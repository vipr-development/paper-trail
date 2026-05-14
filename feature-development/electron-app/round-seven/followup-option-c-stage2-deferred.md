# Follow-up — Option C Stage 2 QA Deferred Items

> Four items the Stage 2 QA round (2026-05-11) flagged but explicitly
> deferred — not blocking the Stage 2 merge, but worth addressing in
> future sessions. Listed in rough priority order. None require
> coordinated change to ship in isolation.

## Status

📋 Queued. Independent of Option E, Stage 3, or any active perf work.

## Items

### D1 — `file_metrics` SIGKILL race (CLOSED — not a real bug)

**Status:** CLOSED 2026-05-11 during PR #71 review remediation. The
original QA finding asserted that partial `file_metrics` writes could
attach to commits that appear complete. Verification on remediation
shows:

1. There is no `file_metrics` table in any migration. The only
   `file_metrics` references in the codebase are CTE aliases (e.g.
   `WITH file_metrics AS (SELECT ... FROM commit_file_changes ...)`
   in `clients/desktop/src/main/db/history-queries.ts:819` and
   `clients/desktop/src/main/services/dependency-service.ts:443`).
2. All per-commit writes (`snapshot_files` / `snapshot_file_deltas`,
   `commit_file_changes`, `commit_metrics`, plus the final
   `UPDATE snapshots SET is_complete=1`) happen inside a single
   `TransactionManager.transaction()` call wrapped in
   `BEGIN TRANSACTION` / `COMMIT` (see
   `clients/desktop/src/main/db/database-service.ts:96–123`).
3. SQLite is in WAL mode; on SIGKILL the uncommitted pages roll back
   atomically. The only artifact that can survive is the snapshot row
   itself (inserted by `createSnapshotSafe` BEFORE the transaction
   with `is_complete=0`).
4. The repair sweep at
   `clients/desktop/src/main/analysis/historical-snapshot-repair.ts`
   deletes any such orphan row (including its associated
   `commit_file_changes` / `commit_metrics`). PR #71 wires the sweep
   at workspace open as well as post-analysis so a crash-then-restart
   sequence is cleaned up before the user has to re-trigger analysis.

No code change needed for D1. The pre-existing risk is bounded to the
snapshot-row orphan, which the existing guard already handles.

---

### D2 — Backfill pool `onRestarted` re-dispatch coupling

**Severity per QA:** minor. The current retry loop (`MAX_COMMIT_RETRIES=2`)
in `backfill-scheduler.ts` compensates today.

**Finding:** when a backfill pool member crashes mid-batch, the
in-flight per-commit batch fails. The scheduler catches the failure
and retries the commit (up to MAX_COMMIT_RETRIES). There's no explicit
"wait for the crashed member to restart" coupling — the retry might
re-dispatch the batch before the new process is ready.

**Why this works today:** `UtilityProcessManager.onRestarted` triggers
a restart on exit code != 0, and the pool's round-robin dispatch
implicitly skips members in startup state because their next
`postMessage` will wait for the IPC channel.

**Why it's worth tightening:** at higher pool sizes (Stage 3 might
push beyond N=4), more concurrent restarts become possible. A coherent
"crashed member restarts → in-flight files for that member re-dispatch
to a healthy member" semantic is cleaner than retry-after-fail.

**Effort:** ~1-2 days. Touches `utility-process-pool.ts` (onRestarted
state-machine), `utility-process-manager.ts` (restart lifecycle), and
`backfill-scheduler.ts` (retry path).

---

### D3 — Unbounded coordinator backlog queue

**Severity per QA:** minor. **Pre-existing.** Pathological-repo concern.

**Finding:** `coordinator.ts`'s internal queue (`this.queue[]`) holds
pending `QueuedFile` objects until the dispatcher picks them up. On a
50k-file monorepo, that's 50,000 objects (~few MB of structured data).
Not catastrophic but bounded only by total file count.

**Why it matters:** as Vipr targets larger repos, the in-memory
queue's working set grows linearly. Stage 3 hardening or Option D
(commit pipelining) might compound this.

**Investigation:**
1. Profile actual queue depth on a 10k-file repo. Is the queue
   draining fast enough that depth stays bounded in practice?
2. If depth is consistently > N×batchSize, consider streaming files
   into the queue (lazy enumeration) instead of bulk-enqueueing.

**Effort:** ~1 day investigation, ~2-3 days for a streaming refactor
if warranted.

---

### D4 — `resolvePoolSize` "capped at runtime" log assertion test

**Severity per QA:** minor. Observability gap only.

**Finding:** `utility-process-pool.ts:51-59` emits an `info`-level log
when the resolved pool size differs from configured. No test asserts
the log fires under the cap-triggered path. The cap logic itself is
tested; the log emission is not.

**Why it's worth doing:** future refactors of `resolvePoolSize` could
break the observability without any test catching it. A founder
hitting a low-RAM machine would see `poolSize=2` with no clear "why"
indicator if the log silently regressed.

**Fix shape:** extend the existing `resolvePoolSize` test block in
`utility-process-pool.test.ts` to assert `logger.info` was called with
the correct payload structure (purpose, configured, cpuBound, ramBound,
resolved).

**Effort:** ~30 minutes.

---

## When to do these

D1 first (it's the only "serious" — and it's pre-existing, so any
future user-visible crash mid-write would hit this). D4 is the
cheapest and worth picking up in any session that touches
`resolvePoolSize`. D2 and D3 can wait until either a pathological-repo
issue surfaces or Stage 3 work makes them more relevant.

None of these block Option E (storage normalization) or Stage 3
(crash recovery + production hardening). They're queued so they don't
get lost.
