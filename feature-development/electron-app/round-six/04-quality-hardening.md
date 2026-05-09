---
id: 04-quality-hardening
title: 'Quality Hardening & Test Coverage (M4)'
phase: 6
dependencies: [01-pipeline-content-retrieval, 02-batch-git-cat-file, 03-batch-analysis-ipc]
status: planned
---

# Quality Hardening & Test Coverage (M4)

## Problem Statement

The S1-S4 implementation audit (`00-s1-s4-implementation-summary.md`) identified six recommendations
(R1-R6) spanning code duplication, silent error handling, unbounded memory, unnecessary git
subprocess calls, test coverage gaps, and a crash-safety window. M1-M3 add further complexity to the
pipeline — a bounded async queue, a `git cat-file --batch` parser, and batch IPC schemas — all of
which modify the same methods that R1-R6 target.

M4 is the consolidation milestone. It lands after M1-M3 and addresses all six recommendations in a
single coordinated pass, ensuring that refactoring targets the final shape of the code rather than an
intermediate state.

Key metrics before M4:

| Metric                                            | Current   | Target                        |
| ------------------------------------------------- | --------- | ----------------------------- |
| `createIncrementalSnapshot` cyclomatic complexity | 14        | <= 8                          |
| `createIncrementalSnapshot` line count            | ~190      | ~100                          |
| `analysisCache` size cap                          | unbounded | configurable (default 10,000) |
| S1 cache hit test coverage                        | none      | dedicated test                |
| S3 `changedFiles` fast-path test coverage         | none      | dedicated test                |
| Corrupted `plugin_results` logging                | silent    | `logger.warn`                 |
| `file_count` UPDATE crash window                  | two-phase | single transaction            |

## New Files

None. All changes are to existing files.

## Modified Files

| File                                                                    | Changes                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `clients/desktop/src/main/analysis/historical-snapshot-service.ts`      | R1: Extract shared analysis loop into `analyzeFileList()`. R2: Add `logger.warn` to `lookupByContentSha` catch block. R3: Add cache size cap with configurable threshold. R6: Move `file_count` UPDATE into the final `txManager.transaction` call. |
| `clients/desktop/src/main/analysis/historical-snapshot-service.test.ts` | R5: Add tests for S1 cache hit path, S3 `changedFiles` fast path, `lookupByContentSha` DB fallback, corrupted JSON handling, `getCacheStats`/`resetCacheCounters`, all-deleted-files commit, and `parentSnapshot.file_count = null`.                |

## Implementation Details

### R1 — Extract Analysis Loop (`analyzeFileList`)

The per-file analysis loop is duplicated between `createSnapshotForCommit` (post-M1: inside
`runConsumer`) and `createIncrementalSnapshot` (post-M3: the cache-miss batch loop). Both loops:

1. Compute content SHA
2. Check S1 cache (`lookupByContentSha`)
3. Call `analyzeContent` or `analyzeContentBatch` on cache miss
4. Handle per-file errors
5. Push results and call `onProgress`

Extract to a private method:

```typescript
/**
 * Analyze a list of files, checking the content-SHA cache before sending to
 * the utility process. Returns results in the same order as the input paths.
 *
 * This method is the single implementation of the analyze-with-cache pattern
 * used by both createSnapshotForCommit and createIncrementalSnapshot.
 */
private async analyzeFileList(
  contentMap: Map<string, string>,
  paths: string[],
  options: {
    onProgress?: (processed: number, total: number, currentFile: string) => void;
    progressOffset?: number; // For incremental: filesCarriedOver
  }
): Promise<AnalyzeFileListResult> {
  const results: FileResult[] = [];
  let analyzed = 0;
  let skipped = 0;
  let failed = 0;
  const offset = options.progressOffset ?? 0;
  const total = offset + paths.length;

  // First pass: resolve cache hits, collect misses
  const cacheMissBatch: Array<{ path: string; content: string; contentSha: string }> = [];

  for (const path of paths) {
    const content = contentMap.get(path) ?? null;
    if (content === null) {
      skipped++;
      results.push({ path, result: null, overallScore: null, contentSha: null });
      continue;
    }

    const contentSha = createHash('sha256').update(content).digest('hex');
    const cached = this.lookupByContentSha(contentSha);
    if (cached) {
      this.analysisCacheHits++;
      analyzed++;
      options.onProgress?.(offset + analyzed, total, path);
      results.push({
        path,
        result: { pluginResults: cached.pluginResults } as RuntimeAggregatedResult,
        overallScore: cached.overallScore,
        contentSha,
      });
    } else {
      this.analysisCacheMisses++;
      cacheMissBatch.push({ path, content, contentSha });
    }
  }

  // Second pass: analyze cache misses in batches (M3)
  const BATCH_SIZE = 10;
  for (let i = 0; i < cacheMissBatch.length; i += BATCH_SIZE) {
    const batch = cacheMissBatch.slice(i, i + BATCH_SIZE);
    const batchResults = await this.utilityManager.analyzeContentBatch(
      batch.map(b => ({ filePath: b.path, content: b.content }))
    );

    for (let j = 0; j < batch.length; j++) {
      const item = batch[j]!;
      const batchResult = batchResults[j]!;

      if (batchResult.result === null) {
        logger.warn('Error analyzing file', { path: item.path, error: batchResult.error });
        failed++;
        results.push({ path: item.path, result: null, overallScore: null, contentSha: null });
      } else {
        const overallScore = aggregateFileScore(batchResult.result);
        analyzed++;
        options.onProgress?.(offset + analyzed, total, item.path);
        results.push({
          path: item.path,
          result: batchResult.result,
          overallScore,
          contentSha: item.contentSha,
        });
        this.analysisCache.set(item.contentSha, {
          overallScore,
          pluginResults: batchResult.result.pluginResults,
        });
      }
    }
  }

  return { results, analyzed, skipped, failed };
}
```

Both `createSnapshotForCommit` and `createIncrementalSnapshot` call `analyzeFileList` instead of
inlining the loop. The method signature is stable — M1's `runConsumer` can delegate to it, and M3's
batch logic is embedded within it.

### R2 — Log Corrupted `plugin_results`

Replace the silent `catch` in `lookupByContentSha`:

```typescript
// Before (silent)
} catch {
  return null;
}

// After
} catch (err) {
  logger.warn('Failed to parse cached plugin_results, re-analyzing', {
    contentSha,
    error: err instanceof Error ? err.message : String(err),
  });
  return null;
}
```

Behavior is unchanged — `null` triggers re-analysis — but the corruption is now diagnosable in logs.

### R3 — Analysis Cache Size Cap

Add a configurable cap to prevent unbounded memory growth during large backfills:

```typescript
private static readonly DEFAULT_CACHE_CAP = 10_000;
private readonly cacheCap: number;

constructor(options?: { cacheCap?: number }) {
  this.cacheCap = options?.cacheCap ?? HistoricalSnapshotService.DEFAULT_CACHE_CAP;
}

private addToCache(contentSha: string, entry: CachedAnalysis): void {
  if (this.analysisCache.size >= this.cacheCap) {
    // Evict oldest entry (first key in insertion-order Map)
    const oldest = this.analysisCache.keys().next().value;
    if (oldest !== undefined) {
      this.analysisCache.delete(oldest);
    }
  }
  this.analysisCache.set(contentSha, entry);
}
```

This is a simple FIFO eviction using `Map`'s insertion-order guarantee. It avoids adding an LRU
dependency while bounding memory at ~10 MB (10,000 entries x ~1 KB each). The cap is configurable
for testing and for repositories with unusual file counts.

All `this.analysisCache.set(...)` calls are replaced with `this.addToCache(...)`.

### R4 — Deleted-File Awareness in S3 Path

When `options.changedFiles` is provided (S3 fast path), `deletedJsPaths` is hardcoded to `[]`. This
means deleted files appear in `pathsToAnalyze`, triggering a `git show` that returns null. The file
is correctly excluded from the snapshot but wastes a subprocess call per deleted file.

The fix changes `changedFiles` from `string[]` to the richer shape already available from
`getCommitRange`:

```typescript
// In HistoricalSnapshotOptions (or the relevant options type)
changedFiles?: Array<{ status: string; path: string }>;

// In createIncrementalSnapshot, S3 branch:
if (options.changedFiles) {
  const analyzable = options.changedFiles.filter(f => isAnalyzableFile(f.path));
  changedJsPaths = analyzable.filter(f => f.status !== 'D').map(f => f.path);
  deletedJsPaths = analyzable.filter(f => f.status === 'D').map(f => f.path);
} else {
  // fallback: getChangedFilesWithStatus(...)
}
```

This eliminates the extra `git show` calls for deleted files on the S3 path. The
`BackfillScheduler` already has `commit.changedFiles` with status information from `getCommitRange`
— it passes the richer type instead of mapping to `string[]`.

### R5 — Test Coverage Additions

Add to `historical-snapshot-service.test.ts`:

```typescript
describe('S1 content-SHA cache', () => {
  it('skips analyzeContent when content SHA matches a prior analysis', async () => {
    // First call: cache miss, analyzeContent called
    await service.createSnapshotForCommit({
      commitSha: 'abc1234',
      // ... files: { 'src/a.ts': 'const x = 1;' }
    });

    // Second call with same file content at a different commit
    await service.createSnapshotForCommit({
      commitSha: 'def5678',
      // ... files: { 'src/a.ts': 'const x = 1;' } — identical content
    });

    // analyzeContent should be called once for 'src/a.ts', not twice
    expect(analyzeContentSpy).toHaveBeenCalledTimes(1);
  });

  it('falls back to DB lookup when in-memory cache is empty', async () => {
    // Pre-seed the DB with a known content SHA and result
    // Create a new service instance (empty in-memory cache)
    // Call createSnapshotForCommit with the same content
    // Assert analyzeContent was not called (DB hit)
    // Assert the DB query was executed
  });

  it('logs and re-analyzes when plugin_results JSON is corrupted', async () => {
    // Pre-seed the DB with a content SHA whose plugin_results is invalid JSON
    const logSpy = vi.spyOn(logger, 'warn');
    await service.createSnapshotForCommit({
      commitSha: 'abc1234',
      // ... file with content matching the corrupted SHA
    });

    expect(logSpy).toHaveBeenCalledWith(
      'Failed to parse cached plugin_results, re-analyzing',
      expect.objectContaining({ contentSha: expect.any(String) })
    );
    expect(analyzeContentSpy).toHaveBeenCalled(); // Fell through to analysis
  });
});

describe('S3 changedFiles fast path', () => {
  it('does not call getChangedFilesWithStatus when changedFiles is provided', async () => {
    const spy = vi.spyOn(gitContent, 'getChangedFilesWithStatus');

    await service.createIncrementalSnapshot({
      commitSha: 'abc1234',
      parentSha: 'parent123',
      changedFiles: [{ status: 'M', path: 'src/modified.ts' }],
    });

    expect(spy).not.toHaveBeenCalled();
  });

  it('excludes deleted files from analysis when changedFiles includes status', async () => {
    await service.createIncrementalSnapshot({
      commitSha: 'abc1234',
      parentSha: 'parent123',
      changedFiles: [
        { status: 'M', path: 'src/modified.ts' },
        { status: 'D', path: 'src/deleted.ts' },
      ],
    });

    // Only 'src/modified.ts' should be analyzed
    expect(analyzeContentSpy).toHaveBeenCalledWith(
      expect.stringContaining('modified.ts'),
      expect.any(String)
    );
    expect(analyzeContentSpy).not.toHaveBeenCalledWith(
      expect.stringContaining('deleted.ts'),
      expect.any(String)
    );
  });
});

describe('cache telemetry', () => {
  it('getCacheStats returns accumulated hits and misses', async () => {
    service.resetCacheCounters();

    // Run two snapshots where second has cache hits
    await service.createSnapshotForCommit({
      /* first commit */
    });
    await service.createSnapshotForCommit({
      /* second commit, same content */
    });

    const stats = service.getCacheStats();
    expect(stats.hits).toBeGreaterThan(0);
    expect(stats.hits + stats.misses).toBeGreaterThan(0);
  });

  it('resetCacheCounters zeroes both counters', () => {
    service.resetCacheCounters();
    expect(service.getCacheStats()).toEqual({ hits: 0, misses: 0 });
  });
});

describe('edge cases', () => {
  it('handles all-deleted-files commit', async () => {
    await service.createIncrementalSnapshot({
      commitSha: 'abc1234',
      parentSha: 'parent123',
      changedFiles: [
        { status: 'D', path: 'src/a.ts' },
        { status: 'D', path: 'src/b.ts' },
      ],
    });

    // No files analyzed, carry-over minus deletions
    expect(analyzeContentSpy).not.toHaveBeenCalled();
  });

  it('handles parentSnapshot.file_count = null', async () => {
    // Mock parent snapshot with null file_count
    mockGetSnapshotByContext.mockResolvedValueOnce({
      id: 1,
      file_count: null,
    });

    // Should not throw — file_count coerced to 0 via ?? operator
    await expect(
      service.createIncrementalSnapshot({
        commitSha: 'abc1234',
        parentSha: 'parent123',
        changedFiles: [{ status: 'M', path: 'src/a.ts' }],
      })
    ).resolves.toBeDefined();
  });
});
```

### R6 — Single-Transaction `file_count` Correction

Move the `UPDATE snapshots SET file_count` into the existing final transaction:

```typescript
// Before: two-phase write
this.createSnapshotSafe(db, { ... file_count: parentFileCount }); // Phase 1: placeholder

// ... analysis ...

if (actualFileCount !== parentFileCount) {
  db.prepare('UPDATE snapshots SET file_count = ? WHERE id = ?')   // Phase 2: correction
    .run(actualFileCount, snapshotId);
}

// After: single transaction
this.createSnapshotSafe(db, { ... file_count: parentFileCount }); // Placeholder (outside tx)

// ... analysis ...

await this.txManager.transaction(db => {
  // Existing: INSERT snapshot_files rows
  for (const result of fileResults) {
    db.prepare('INSERT INTO snapshot_files ...').run(...);
  }

  // Existing: upsertCommitMetrics
  this.upsertCommitMetrics(db, snapshotId, ...);

  // NEW: correct file_count atomically with the rest
  if (actualFileCount !== parentFileCount) {
    db.prepare('UPDATE snapshots SET file_count = ? WHERE id = ?')
      .run(actualFileCount, snapshotId);
  }
});
```

If the process crashes before the transaction commits, the placeholder `file_count` persists but no
`snapshot_files` rows exist either — the snapshot is incomplete and will be re-created on the next
backfill run (existing skip-if-exists logic checks for `snapshot_files` presence). This eliminates
the window where `file_count` is wrong but `snapshot_files` are fully written.

## Constraints

### R1 Extraction Must Preserve M1 Pipeline Compatibility

If M1 (producer-consumer pipeline) is implemented, the `runConsumer` function delegates to
`analyzeFileList` rather than inlining the analysis loop. The `PipelineQueue` remains in
`runConsumer`; `analyzeFileList` receives a pre-built `contentMap` from the consumer's accumulated
dequeue results. This keeps the pipeline coordination separate from the analysis-with-cache logic.

### R3 Cache Cap Must Not Regress S1 Hit Rate

The default cap of 10,000 entries is chosen to exceed the file count of typical repositories (most
JS/TS projects have fewer than 5,000 unique files). For mono-repos exceeding this threshold, the cap
is configurable. The FIFO eviction order means the oldest entries (earliest commits in a
oldest-first backfill) are evicted first — these are the least likely to be cache hits for later
commits where files have diverged.

### R4 Type Change Requires BackfillScheduler Update

Changing `changedFiles` from `string[]` to `Array<{ status: string; path: string }>` is a breaking
change to `HistoricalSnapshotOptions`. The `BackfillScheduler` is the only caller — it already has
the richer type from `getCommitRange` and currently maps to `string[]`. The fix is to stop mapping
and pass the full objects.

### R5 Tests Must Not Depend on Implementation Internals

Cache-hit tests should verify behavior (analyzeContent call count) rather than inspecting the
`analysisCache` Map directly. This keeps tests stable if the caching mechanism changes.

## Edge Cases

| Scenario                                                                       | Handling                                                                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cache cap reached during a single commit with 15,000 unique files              | FIFO eviction keeps the most recent 10,000 entries. Files evicted mid-commit will be re-analyzed if encountered again (unlikely in a single commit). Cross-commit hits for evicted entries fall through to the DB fallback in `lookupByContentSha`.                   |
| `analyzeFileList` receives empty `paths` array                                 | Returns `{ results: [], analyzed: 0, skipped: 0, failed: 0 }`. No IPC messages sent.                                                                                                                                                                                  |
| `changedFiles` contains a file with unknown status (e.g., `U` for unmerged)    | `isAnalyzableFile` filters by extension, not status. The file will be fetched and analyzed normally. If the content is unavailable at the commit, `getFileAtCommit` returns null and the file is skipped.                                                             |
| DB fallback in `lookupByContentSha` returns a row with `null` `plugin_results` | `JSON.parse(null)` returns `null`. The `if (!parsed)` guard triggers and returns `null`, falling through to analysis. No crash.                                                                                                                                       |
| `txManager.transaction` throws during the combined write                       | All writes (snapshot_files, metrics, file_count correction) are rolled back atomically. The snapshot row (created before the transaction) persists with placeholder values but no files — the next backfill run will detect the incomplete snapshot and re-create it. |

## Testing Strategy

### Unit Tests: `analyzeFileList` extraction

Verify that `analyzeFileList` produces identical results to the previous inline loops by running the
existing `historical-snapshot-service.test.ts` suite without modification. If all tests pass after
the extraction, the refactoring is behavior-preserving.

### Unit Tests: Cache cap behavior

```typescript
describe('analysis cache cap', () => {
  it('evicts oldest entry when cap is reached', () => {
    const service = new HistoricalSnapshotService({ cacheCap: 3 });

    // Add 4 entries
    service['addToCache']('sha1', mockEntry);
    service['addToCache']('sha2', mockEntry);
    service['addToCache']('sha3', mockEntry);
    service['addToCache']('sha4', mockEntry);

    // sha1 should be evicted
    expect(service['analysisCache'].has('sha1')).toBe(false);
    expect(service['analysisCache'].has('sha4')).toBe(true);
    expect(service['analysisCache'].size).toBe(3);
  });
});
```

### Integration Tests: R5 test cases

See the full test listings in the Implementation Details > R5 section above. These tests target
the highest-severity coverage gaps identified in the S1-S4 audit.

### Regression: Existing test suite

All existing tests in `historical-snapshot-service.test.ts` and `git-content-service.test.ts` must
pass without modification after M4 lands. The extraction (R1) and transaction consolidation (R6) are
strictly behavior-preserving refactors.

## Acceptance Criteria

- [ ] `createIncrementalSnapshot` cyclomatic complexity is <= 8 (measured by the core analyzer or
      manually counted)
- [ ] `createSnapshotForCommit` and `createIncrementalSnapshot` both delegate to `analyzeFileList`
      with no duplicated analysis logic
- [ ] `lookupByContentSha` logs a warning with `contentSha` and error message when `JSON.parse`
      fails
- [ ] `analysisCache` does not grow beyond the configured cap (default 10,000); FIFO eviction is
      verified by unit test
- [ ] `changedFiles` option accepts `Array<{ status: string; path: string }>` and correctly
      separates deleted files from modified/added files
- [ ] `BackfillScheduler` passes the richer `changedFiles` type without mapping to `string[]`
- [ ] S1 cache hit test: `analyzeContent` is not called for content with a previously computed SHA
- [ ] S1 DB fallback test: in-memory cache miss falls through to DB query and populates the cache
- [ ] S3 fast-path test: `getChangedFilesWithStatus` is not called when `changedFiles` is provided
- [ ] Corrupted JSON test: `logger.warn` is called and the file is re-analyzed
- [ ] All-deleted-files test: no files analyzed, correct `file_count` in snapshot
- [ ] `file_count` UPDATE is inside the final `txManager.transaction` call
- [ ] All existing `historical-snapshot-service` tests pass without modification
- [ ] No new `any` types introduced
