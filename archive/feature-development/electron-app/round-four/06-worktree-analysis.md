---
id: 06-worktree-analysis
title: 'Worktree-Scoped Analysis'
phase: 4
dependencies: [01, 03]
status: planned
---

# Worktree-Scoped Analysis

## Problem Statement

Each git worktree is a separate directory with its own working tree and HEAD. `WorktreeDetectionService` already detects worktrees, but no analysis pipeline handles them. Unlike historical commit analysis (which uses `analyzeContent` with content retrieved via `git show`), worktree files are on disk — `analyzeFile()` should be used for better accuracy (plugin heuristics that depend on file-system context behave correctly when the file exists on disk).

Snapshots must be keyed by both the worktree path and its HEAD SHA, so two worktrees at the same commit SHA coexist as distinct records (this is enforced by the schema added in phase 01). The incremental mode further reduces redundant analysis by carrying over scores for files unchanged relative to the main branch HEAD.

## New File

`clients/desktop/src/main/analysis/worktree-analysis-service.ts`

## Key Types

```typescript
// clients/desktop/src/main/analysis/worktree-analysis-service.ts

export interface WorktreeAnalysisOptions {
  worktreePath: string; // absolute path to the worktree directory
  mainRepoPath: string; // used to validate the worktree belongs to this repo
  incremental?: boolean; // only re-analyze files changed vs. main HEAD
  onProgress?: (processed: number, total: number, currentFile: string) => void;
}

export interface WorktreeAnalysisResult {
  snapshotId: number;
  worktreePath: string;
  filesAnalyzed: number;
  filesCarriedOver: number;
  wasSkipped: boolean;
}

/**
 * Merges WorktreeInfo (from worktree-service.ts) with snapshot status.
 * Field names match WorktreeInfo exactly so spreading `...wt` works without remapping.
 */
export interface WorktreeWithStatus {
  path: string;
  head: string; // WorktreeInfo uses 'head', not 'headSha'
  branch: string | null; // nullable in WorktreeInfo
  detached: boolean;
  locked: boolean; // WorktreeInfo uses 'locked', not 'isLocked'
  lockedReason: string | null;
  bare: boolean; // WorktreeInfo uses 'bare', not 'isBare'
  prunable: boolean;
  snapshotId: number | null;
  hasSnapshot: boolean;
  lastAnalyzedAt: number | null;
}
```

## Class Interface

```typescript
// clients/desktop/src/main/analysis/worktree-analysis-service.ts

import type { WorktreeDetectionService } from '../git/worktree-service';
import type { GitContentService } from '../git/git-content-service';
import type { UtilityProcessManager } from '../analysis/utility-process-manager';
import type { SnapshotRepository } from '../db/interfaces';
import type { DatabaseSync } from 'node:sqlite';

export class WorktreeAnalysisService {
  constructor(
    private worktreeDetection: WorktreeDetectionService,
    private gitContent: GitContentService,
    private utilityManager: UtilityProcessManager,
    private snapshotRepo: SnapshotRepository,
    private db: DatabaseSync
  ) {}

  /** Analyze a worktree, creating a snapshot keyed by worktree_path + HEAD SHA. */
  analyzeWorktree(options: WorktreeAnalysisOptions): Promise<WorktreeAnalysisResult>;

  /** Get all worktrees for a repo with their snapshot status merged in. */
  listWorktreesWithStatus(mainRepoPath: string): Promise<WorktreeWithStatus[]>;

  /** Compare a worktree snapshot to the main branch HEAD snapshot. */
  compareToMain(worktreePath: string, mainRepoPath: string): Promise<SnapshotComparison>;

  /** Guard against path traversal: throws if the path is not a registered worktree. */
  private validateWorktreePath(worktreePath: string, mainRepoPath: string): Promise<void>;
}
```

## Security: Worktree Path Validation

Any IPC call that accepts a `worktreePath` must be validated before any file-system access. Without this guard, a malicious IPC call could supply an arbitrary path (e.g. `/etc`) and trigger analysis of unrelated directories.

```typescript
private async validateWorktreePath(worktreePath: string, mainRepoPath: string): Promise<void> {
  const worktrees = await this.worktreeDetection.listWorktrees(mainRepoPath);
  const valid = worktrees.some(wt => wt.path === worktreePath);
  if (!valid) {
    throw Object.assign(
      new Error(`INVALID_WORKTREE: ${worktreePath} not in repo ${mainRepoPath}`),
      { code: 'INVALID_WORKTREE' }
    );
  }
}
```

Path comparison is exact-match on the absolute path string returned by `WorktreeDetectionService.listWorktrees`. Do not normalize or resolve `worktreePath` from the caller — require the caller to supply the canonical path as returned by the listing API.

## `analyzeWorktree` Implementation

```mermaid
flowchart TD
    A[analyzeWorktree called] --> B[validateWorktreePath]
    B -->|invalid| Z[throw INVALID_WORKTREE]
    B -->|valid| C[git rev-parse HEAD in worktreePath]
    C --> D{incremental?}

    D -->|false| E[Get all tracked files via gitContent.getTrackedFilesAtCommit(worktreeHeadSha)]
    D -->|true| F[getChangedFilesBetweenCommits mainHeadSha → worktreeHeadSha]
    F --> G[Analyze changed files only]
    G --> H[Carry over scores for unchanged files from main HEAD snapshot]
    H --> I[Merge results]

    E --> J[analyzeFile for each file on disk]
    I --> J
    J --> K[snapshotRepo.createSnapshot ref_type=worktree worktree_path=worktreePath]
    K --> L[Return WorktreeAnalysisResult]
```

**Snapshot storage fields:**

```typescript
await this.snapshotRepo.createSnapshot({
  gitSha: worktreeHeadSha,
  refType: 'worktree',
  worktreePath: options.worktreePath,
  refName: worktreeBranch,
});
```

This satisfies the composite unique index from migration 15: two worktrees at the same `gitSha` are distinct because their `worktree_path` values differ.

## Incremental Analysis

When `incremental: true` is set:

1. Resolve main branch HEAD SHA: `gitContent.resolveRef('HEAD', mainRepoPath)`.
2. Get the list of changed files: `gitContent.getChangedFilesBetweenCommits(mainHeadSha, worktreeHeadSha)`.
3. Analyze only the changed files using `utilityManager.analyzeFile(absolutePath)` where `absolutePath` is resolved relative to `worktreePath`.
4. Carry over unchanged file scores from the main HEAD snapshot using the carry-over SQL established in phase 03:

```sql
INSERT INTO snapshot_files (snapshot_id, file_path, score, plugin_id, report_type, raw_metrics)
SELECT :newSnapshotId, sf.file_path, sf.score, sf.plugin_id, sf.report_type, sf.raw_metrics
FROM snapshot_files sf
WHERE sf.snapshot_id = :mainSnapshotId
  AND sf.file_path NOT IN (SELECT value FROM json_each(:changedPaths));
```

The `filesCarriedOver` count in `WorktreeAnalysisResult` reflects how many rows were inserted via carry-over vs. fresh analysis.

## `listWorktreesWithStatus` Implementation

Merges `WorktreeDetectionService.listWorktrees()` output with `SnapshotRepository` state:

```typescript
async listWorktreesWithStatus(mainRepoPath: string): Promise<WorktreeWithStatus[]> {
  // listWorktrees is async (spawns a child process); snapshotRepo methods are synchronous
  const worktrees = await this.worktreeDetection.listWorktrees(mainRepoPath);
  return worktrees.map(wt => {
    const latest = this.snapshotRepo.getLatestSnapshotForWorktree(wt.path);
    return {
      ...wt,
      snapshotId: latest?.id ?? null,
      hasSnapshot: latest !== null,
      lastAnalyzedAt: latest?.analyzed_at ?? null,
    };
  });
}
```

## Extended IPC Channels

Extend `clients/desktop/src/main/ipc/handlers/worktrees.ts` with the following four handlers.

### `worktrees:analyze`

Trigger full or incremental analysis of a worktree.

**Request:**

```typescript
{ worktreePath: string; mainRepoPath: string; incremental?: boolean }
```

**Response:**

```typescript
WorktreeAnalysisResult;
```

**Error cases:**

| Condition                                                      | Error code               |
| -------------------------------------------------------------- | ------------------------ |
| `worktreePath` not in `WorktreeDetectionService.listWorktrees` | `INVALID_WORKTREE`       |
| Worktree HEAD SHA unresolvable                                 | `WORKTREE_DETACHED_HEAD` |

### `worktrees:getSnapshot`

Return the most recent snapshot for a worktree.

**Request:**

```typescript
{
  worktreePath: string;
}
```

**Response:**

```typescript
SnapshotRecord | null;
```

**Flow:** `snapshotRepo.getLatestSnapshotForWorktree(worktreePath)`.

### `worktrees:compare`

Compare a worktree snapshot to the main branch HEAD snapshot, returning a delta.

**Request:**

```typescript
{
  worktreePath: string;
  mainRepoPath: string;
}
```

**Response:**

```typescript
SnapshotComparison; // see SnapshotRepository interface
```

### `worktrees:listWithStatus`

Return all worktrees for a repo with merged snapshot status.

**Request:**

```typescript
{
  mainRepoPath: string;
}
```

**Response:**

```typescript
WorktreeWithStatus[]
```

## Extended Repository Interface

Add to `clients/desktop/src/main/db/interfaces.ts` → `SnapshotRepository`:

```typescript
/**
 * Get all snapshots taken within a specific worktree, ordered by analyzed_at DESC.
 * Synchronous — the underlying DatabaseSync layer is always synchronous.
 */
getSnapshotsByWorktree(worktreePath: string): SnapshotRecord[];

/**
 * Get the most recent snapshot for a worktree. Returns null if none exists.
 * Synchronous — the underlying DatabaseSync layer is always synchronous.
 */
getLatestSnapshotForWorktree(worktreePath: string): SnapshotRecord | null;
```

SQL implementation of `getLatestSnapshotForWorktree`:

```sql
SELECT * FROM snapshots
WHERE worktree_path = :worktreePath
  AND ref_type = 'worktree'
  AND is_draft = 0
ORDER BY analyzed_at DESC
LIMIT 1;
```

## Testing

File: `clients/desktop/src/main/analysis/worktree-analysis-service.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
// Mock WorktreeDetectionService, GitContentService, UtilityProcessManager, SnapshotRepository.

describe('WorktreeAnalysisService', () => {
  describe('analyzeWorktree', () => {
    it('rejects paths not returned by WorktreeDetectionService.listWorktrees', async () => {
      // Arrange: worktreeDetection.listWorktrees returns [{ path: '/repo/wt1' }].
      // Act: analyzeWorktree({ worktreePath: '/tmp/evil', mainRepoPath: '/repo' }).
      // Assert: throws with code 'INVALID_WORKTREE'.
    });

    it('stores snapshot with ref_type=worktree and worktree_path set', async () => {
      // Arrange: valid worktree path; spy snapshotRepo.createSnapshot.
      // Assert: createSnapshot called with { refType: 'worktree', worktreePath: '/repo/wt1' }.
    });

    it('incremental mode only calls analyzeFile for changed files', async () => {
      // Arrange: getChangedFilesBetweenCommits returns ['src/a.ts']; two files total.
      // Assert: utilityManager.analyzeFile called once (for 'src/a.ts'), not twice.
    });

    it('incremental mode carries over scores for unchanged files', async () => {
      // Arrange: one changed file, one unchanged; mainSnapshot has snapshot_files for both.
      // Assert: result.filesCarriedOver === 1.
    });

    it('two worktrees at same commit SHA produce distinct snapshots', async () => {
      // Arrange: two calls with same headSha but different worktreePaths.
      // Assert: snapshotRepo.createSnapshot called twice with distinct worktree_path values.
    });

    it('calls onProgress for each file analyzed', async () => {
      // Arrange: three files in worktree; track onProgress calls.
      // Assert: onProgress called three times with increasing processed count.
    });
  });

  describe('listWorktreesWithStatus', () => {
    it('merges worktree info with snapshot status from DB', async () => {
      // Arrange: two worktrees; one has a snapshot, one does not.
      // Assert: result[0].hasSnapshot===true, result[1].hasSnapshot===false.
    });

    it('returns hasSnapshot=false for unanalyzed worktrees', async () => {
      // Arrange: snapshotRepo.getLatestSnapshotForWorktree returns null.
      // Assert: hasSnapshot===false and snapshotId===null.
    });

    it('populates lastAnalyzedAt from snapshot.analyzed_at', async () => {
      // Arrange: latest snapshot has analyzed_at=1700000000.
      // Assert: result.lastAnalyzedAt===1700000000.
    });
  });

  describe('compareToMain', () => {
    it('rejects worktree path not in WorktreeDetectionService listing', async () => {
      // Assert: throws with code 'INVALID_WORKTREE'.
    });

    it('returns SnapshotComparison from snapshotRepo', async () => {
      // Arrange: valid paths; mock snapshotRepo.compareSnapshots.
      // Assert: result matches mock SnapshotComparison.
    });
  });
});
```

## Phase Verification Checklist

| Criterion                                                                      | How to Verify                                                                     |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| Path traversal prevented via `validateWorktreePath`                            | Unit test: `rejects paths not returned by WorktreeDetectionService.listWorktrees` |
| Snapshot stored with `ref_type='worktree'` and `worktree_path` set             | Unit test: `stores snapshot with ref_type=worktree`                               |
| Two worktrees at same commit SHA produce distinct snapshot records             | Unit test: `two worktrees at same commit SHA produce distinct snapshots`          |
| Incremental mode skips unchanged files                                         | Unit test: `incremental mode only calls analyzeFile for changed files`            |
| Carry-over count reported correctly                                            | Unit test: `incremental mode carries over scores for unchanged files`             |
| `listWorktreesWithStatus` returns `hasSnapshot=false` for unanalyzed worktrees | Unit test: `returns hasSnapshot=false for unanalyzed worktrees`                   |
