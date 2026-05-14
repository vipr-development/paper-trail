# Option C — Stage 1 Differential Test Plan

> Manual validation procedure for the backfill `UtilityProcessPool` change
> (Stage 1 of `initiative-option-c-process-parallelism.md`). Snapshot-chain
> integrity is non-negotiable; any mismatch reverts the change immediately.

## Why differential rather than just unit testing

Unit tests verify the pool's routing/fan-out logic in isolation. The
**differential test** verifies the end-to-end invariant that matters for
the user: **a pooled backfill produces byte-identical persisted state to
a single-process backfill**, for the same input commits.

Anything less than byte-identical persistence is a regression — downstream
consumers (history queries, churn metrics, A/B deltas, file-version
lookups) all depend on that chain.

## Pre-conditions

1. Stage 1 code is built and runnable in `dev` mode
2. The baseline test repo (or a small fixture repo) has a known git history
   of at least 5 consecutive commits with non-trivial file changes
3. Two clean `.vipr/` database states are available (the test will trash
   one of them per run)
4. The `WORKER_HISTORICAL_BATCH_CONCURRENCY=1` env var is set (avoids
   confounding concurrency with parallelism)

## Procedure

### Step 1 — Capture single-process baseline

1. Set `STAGE_1_BACKFILL_POOL_SIZE = 1` in `clients/desktop/src/main/index.ts`
   (degenerate pool; degrades to single-process behavior identical to pre-Stage 1)
2. Rebuild: `pnpm --filter @vipr/desktop build`
3. Wipe the desktop's workspace DB for the test repo
4. Open the repo, let backfill run on the first 5 commits only (cancel the rest)
5. Capture the persisted state — see "What to capture" below
6. Save as `baseline-1proc.sql.dump`

### Step 2 — Run pooled (N=2) over the same 5 commits

1. Restore `STAGE_1_BACKFILL_POOL_SIZE = 2`
2. Rebuild: `pnpm --filter @vipr/desktop build`
3. Wipe the workspace DB again
4. Open the repo, let backfill run on the same first 5 commits, cancel the rest
5. Capture the persisted state
6. Save as `pooled-2proc.sql.dump`

### Step 3 — Diff

```bash
diff <(sort baseline-1proc.sql.dump) <(sort pooled-2proc.sql.dump)
```

**Expected**: empty diff. **Any non-empty diff is a STOP signal — revert
the Stage 1 commit and investigate.**

## What to capture

For the 5 test commits, dump these tables sorted deterministically:

```sql
-- Snapshot rows (one per commit)
SELECT git_sha, branch, ref_type, ref_name, is_draft, overall_score,
       file_count, analyzed_at
FROM snapshots
WHERE git_sha IN ('<sha1>', '<sha2>', '<sha3>', '<sha4>', '<sha5>')
ORDER BY git_sha;

-- File-level snapshot rows for those snapshots
SELECT s.git_sha, sf.file_path, sf.content_sha, sf.score,
       sf.cyclomatic, sf.cognitive, sf.maintainability,
       sf.lines, sf.functions
FROM snapshot_files sf
JOIN snapshots s ON sf.snapshot_id = s.id
WHERE s.git_sha IN ('<sha1>', '<sha2>', '<sha3>', '<sha4>', '<sha5>')
ORDER BY s.git_sha, sf.file_path;

-- Commit file-change events
SELECT commit_sha, file_path, change_type, prior_content_sha, content_sha
FROM commit_file_changes
WHERE commit_sha IN ('<sha1>', '<sha2>', '<sha3>', '<sha4>', '<sha5>')
ORDER BY commit_sha, file_path;

-- File-version content rows
SELECT content_sha, length(content) as content_bytes
FROM file_versions
WHERE content_sha IN (
  SELECT DISTINCT content_sha FROM snapshot_files sf
  JOIN snapshots s ON sf.snapshot_id = s.id
  WHERE s.git_sha IN ('<sha1>', '<sha2>', '<sha3>', '<sha4>', '<sha5>')
)
ORDER BY content_sha;

-- Commit metrics (aggregated per commit)
SELECT commit_sha, total_files, average_score, total_complexity,
       total_lines, average_maintainability
FROM commit_metrics
WHERE commit_sha IN ('<sha1>', '<sha2>', '<sha3>', '<sha4>', '<sha5>')
ORDER BY commit_sha;
```

## Acceptance criteria

- All 5 SELECTs return identical rows between baseline and pooled
- No new errors / warnings in the dev log between baseline and pooled runs
- Wall-clock time on the pooled run is at least 15% lower than baseline
  (validates the parallelism is actually engaging, not just the pool wiring)
- Memory: combined RSS of `vipr-backfill-worker-0` + `vipr-backfill-worker-1`
  during the pooled run does not exceed ~1.5× the single-process baseline RSS

## What to do if the diff is non-empty

1. **Revert immediately** — `git revert <stage-1-commit>` and re-deploy
2. Capture the diff verbatim and which rows differ
3. Categorize:
   - **Score / metric difference**: likely a non-determinism in the
     analysis pipeline that the single-process path masked. Investigate
     ts-morph state isolation across worker processes.
   - **content_sha or file_versions difference**: serialization issue —
     likely in `serializeAggregatedResult` / `deserializeAggregatedResult`.
     Should not happen since serialization wasn't touched, but verify.
   - **Missing rows**: the pool dropped or duplicated work (look at
     `partitionContiguous` — likely an off-by-one).
   - **Extra rows**: pool member returned duplicate IDs (uuid collision?
     extremely unlikely but worth checking).
4. Open a follow-up item documenting what was found

## Calibration note

This procedure is intentionally manual for Stage 1 because automating
the dual-build + DB-wipe cycle inside vitest is high-effort for a
one-time gate. If Stage 1 ships clean, Stage 2 should consider
investing in an automated fixture-based variant (smaller repo, faster
iteration) so Stage 3 hardening work is gated by a green automated check
rather than a manual one.
