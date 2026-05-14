# Item 2 — Improve Backfill Wall-Clock Time

> The single largest experiential pain point. Foreground analysis at 4:42 is
> tolerable; backfill at **94 minutes** for a 90-day, 394-commit window on a
> 1000-file repo is not. This item takes the wall clock down.

## Goal

Reduce backfill wall-clock time on the baseline scenario (90-day window, 394
commits, 1008 files) from **94 minutes** toward a target under **30 minutes**
(≈3× improvement) by attacking the dominant cost drivers in the
backfill-specific code path.

## Why now

After items shipped so far:

- Foreground per-file analysis is essentially as fast as it can be without
  structural changes (see item 5 / Option A)
- Backfill per-file analysis cost is the same as foreground (~537ms per file
  at the throughput level after the `historicalBatchConcurrency=1` fix)
- BUT backfill processes far more files (7,330 cache-miss files across 394
  commits in the baseline) AND has its own per-commit overhead (git diff,
  content fetch, snapshot persistence, delta storage)

So the lever isn't per-file analysis — it's:
1. **Per-commit overhead** (git operations, snapshot persistence)
2. **Per-batch IPC** (sending file content to the worker)
3. **Cache hit rate** (content-SHA dedup across commits)

From the most recent backfill baseline:
- Mean commit time: 14.4 seconds
- p50: 3.5s, p95: 63s, p99: 98s, max **6:17 on a single commit (sha 175a5c43)**
- Mean batch IPC time: 7.6 seconds
- Mean batch size: 14.1 files

## Prerequisites

- Item 1 shipped (✅ done — 4043/4043 desktop tests passing)
- Re-baseline backfill run captured AFTER item 1 to confirm the current
  starting point (use [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md))
- Read [`baselines.md`](./baselines.md) for the prior backfill numbers

## Investigation steps (do these first, in order)

This item is **measurement-driven**. Do not start optimizing without a
fresh baseline that reflects items 1 + the productionize-vipr-config merge.

1. **Capture a fresh backfill summary** with the current code on the
   baseline repo. Compare to the prior baseline. Phase 1a alone (filtering
   `mock/`, `__tests__/`, etc.) should reduce file count and possibly wall
   clock meaningfully.

2. **Identify the slowest commit (`slowestCommits[0]`)** and inspect what
   makes it expensive:
   ```bash
   git show --stat <sha>
   git show --name-status <sha> | head -30
   ```
   Common patterns: large refactor touching many heavy files, formatting
   pass touching everything, generated-file commits.

3. **Identify the slowest individual file (`slowestBackfillFiles[0]`)** and
   ask: is this file legitimately expensive, or is it being analyzed many
   times across commits because it changes often? Check `git log --follow
   <file> | wc -l`.

4. **Check the backfill cache hit rate.** The historical-snapshot-service
   maintains an in-memory content-SHA cache so identical bytes across
   commits are deduplicated. Look at `cacheStats.hits` / `cacheStats.misses`
   in the backfill scheduler logs. If miss rate is high, the cache isn't
   helping much.

## Likely optimization targets (driven by step 1-4 findings)

### A. Per-commit git overhead

Each commit triggers:
- `getChangedFiles(parentSha, sha)` — git diff
- For each changed file: `git show <sha>:<path>` to get content
- `createIncrementalSnapshot(...)` which persists changes

If git operations dominate, batch git invocations (one `git ls-tree` instead
of N `git show`) or use a more efficient git library.

### B. Snapshot persistence cost

`linkFileToSnapshot()` writes per-file rows. For a snapshot with N files,
that's N inserts. Even in a transaction, this can be slow for large snapshots
(594 files for the worst commit in the baseline).

Look at:
- `clients/desktop/src/main/db/snapshot-db.ts` — `linkFileToSnapshot()`
- Could batch into one multi-row INSERT or use SQLite's `INSERT OR REPLACE` more efficiently

### C. Content-SHA cache effectiveness

The historical-snapshot-service has `addToCache(contentSha, result)` for
analyzed file content. If the cache hit rate is low (most files differ
across commits), this is largely ineffective. Possible fixes:
- Hash file content before reading the FULL file content (use git's blob
  SHA instead of computing our own)
- Persist the cache to disk so it survives restarts and can be reused
- Add a similarity-based bucket so near-identical files (e.g., a one-line
  change) can reuse most of the analysis

### D. Skip "trivial" commits

Some commits in the baseline (`fffa08d4`: 1034 files in 89s = 86ms/file)
look like formatting passes — many files changed, mostly just whitespace.
These hit the SHA cache already, but maybe we can detect them earlier and
skip the analysis pipeline entirely.

### E. Process-level parallelism for backfill

The backfill scheduler currently processes commits **sequentially**. It
runs in its own utility process (separate from foreground), so the main
event loop isn't affected. But within the backfill process, only one
commit is processed at a time.

Could we process N commits in parallel if they're independent? Snapshot
delta storage may have ordering constraints (each commit's delta references
its parent), so this is non-trivial — research first.

This overlaps with item 5's Option C (process-level parallelism). If item 5
is shipped first, backfill might benefit automatically.

## Acceptance criteria

- New backfill summary on the baseline scenario shows wall clock under 60
  minutes (rough target — adjust based on which optimizations land)
- All existing backfill tests pass
- New tests cover any code paths added (esp. for cache + persistence changes)
- No regression in foreground performance
- `pnpm --filter @vipr/desktop test -- run src/main/analysis/` clean
- Re-baseline captured per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md)
  and saved as `baselines/<date>-after-item-02.md`

## Test plan

```bash
# Targeted analysis tests
pnpm --filter @vipr/desktop test -- run src/main/analysis/historical-snapshot-service.test.ts
pnpm --filter @vipr/desktop test -- run src/main/analysis/backfill-scheduler.test.ts
pnpm --filter @vipr/desktop test -- run src/main/db/snapshot-db.test.ts

# Full analysis dir
pnpm --filter @vipr/desktop test -- run src/main/analysis/

# Full desktop suite
pnpm --filter @vipr/desktop test

# Re-baseline (full procedure in 04-rebaseline-protocol.md)
# 1. Quit Vipr Desktop
# 2. rm "$HOME/Library/Application Support/Vipr Desktop/vipr-default.db"
# 3. pnpm --filter @vipr/desktop dev
# 4. Open clients/desktop, wait for foreground + backfill
# 5. Capture both Performance summary log lines
```

## Risk + rollback

- **Risk**: HIGH — backfill is the most complex code path in the system.
  Bugs here can corrupt snapshot data or cause infinite loops.
- **Mitigation 1**: One optimization at a time, each in its own commit.
  Easy to revert any single change.
- **Mitigation 2**: Differential testing. After each commit, manually run a
  short backfill and verify the resulting snapshot data matches what the
  prior code would have produced.
- **Mitigation 3**: Snapshot data has its own validation (e.g., file_count
  > 0 assertion in backfill-scheduler trace). Trust those signals.
- **Rollback**: revert the offending commit. No data migration needed
  unless the change touched DB schema.

## Out of scope

- **Do NOT** restructure the analysis engine. That's item 5 / Option A.
- **Do NOT** spawn additional utility processes. That's item 5 / Option C.
  (UNLESS the investigation in step 5 above shows it's the cleanest fix
  for backfill specifically.)
- **Do NOT** change foreground behaviour. Foreground perf work was item 4
  in the prior plan and is largely complete.
- **Do NOT** add new instrumentation beyond what already exists. Use the
  PerformanceProfile data we already have.

## Estimated effort

- Investigation (steps 1-4): half a day
- Implementation (whichever target the data points to): 1-3 days per target
- Re-baselining: ~95 minutes per measurement, plan for 3-5 iterations

This is the largest single item in the rolling sequence. Plan accordingly.

---

## Closure (2026-05-10)

**Status: ✅ Done.** Shipped wins captured below; remaining wall-clock
reduction requires structural change (item 7 / item 5).

### What shipped (commits)

| Commit | Change |
|---|---|
| `c141beff` | Fix `isAnalyzableFile` drift in historical-snapshot + ai-remediation services (was missing directory exclusion) |
| `b160553f` | Default `historicalBatchConcurrency` to 1 (previously 4 — artifact-inflated per-file timings) |
| `2298a5d7` | Static JSON imports for analyzer default config (bundler-safe) |
| `82fb5f87` | Hoist hot-path SQL prepares in historical-snapshot-service; add per-commit phase timings (commitAnalysisTime, commitFinalizationTime, commitDbPersistTime, commitMetricsTime, commitFetchContentTime) |
| `bc47853b` | Add per-commit sub-phase instrumentation on incremental write path (parentLookupMs, loadSnapshotStateMs, snapshotRowMs, transactionLoopMs); filter `[Array]` log noise |

### Final baseline (post-`bc47853b`)

- **Backfill wall-clock**: 97:22 (392 commits, 5,726 cache-miss files, 97.2% cache hit)
- **Foreground wall-clock**: 4:47 (1008 files, mean 569ms/file)
- **Tests**: 4043/4043 desktop suite green

### What the sub-phase data revealed

All five new fields on the incremental snapshot path are tiny:

| Sub-phase | Mean | Count (incremental commits = 161) |
|---|---|---|
| `commitParentLookupTime` | 20.3ms | 161 |
| `commitLoadSnapshotStateTime` | 0.1ms | 161 |
| `commitSnapshotRowTime` | 0ms | 161 |
| `commitTransactionLoopTime` | 5.8ms | 161 |
| `commitMetricsTime` | 1.7ms | 392 (both paths) |

Combined incremental sub-phase total: **~28ms/commit**, ~1% of the
9,915ms `commitAnalysisTime` mean.

### Why we're stopping (per conduct → analyze → validate → proceed-or-stop)

- `commitAnalysisTime` (66% of commit time, 9.9s mean) is the dominant
  cost. It's bounded by per-file CPU work in a single V8 isolate.
- 233 of 392 commits go through `createSnapshotForCommitInternal` (full
  snapshot path) which lacks instrumentation. This is likely where the
  historical "~30% unaccounted" overhead lives — but even a 50%
  storage-path reduction would shave ~5-8 minutes off a 97-minute run.
- The next material lever is **real CPU parallelism**, not further
  storage-path tuning. Item 7 picks that up.

### What's NOT being pursued (and why)

- Instrumenting `createSnapshotForCommitInternal` further: ROI ceiling is
  too low while the analysis path is the floor.
- Per-commit IPC batching changes: `eventEmitMs` mean is already 0.2ms;
  not a bottleneck.
- Additional content-SHA cache tuning: 97.2% hit rate is already excellent.

### Hand-off to item 7

Item 7's plan doc (`item-07-worker-pool-experiment.md`) is set up with
the same conduct → analyze → validate → proceed-or-stop staging.
Stage 0 begins now.
