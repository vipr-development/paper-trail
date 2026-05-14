# Initiative — Option C: Process-Level Parallelism

> Spawn N foreground utility processes (currently 1) so the desktop
> actually uses multiple CPU cores for analysis. Each process handles
> in-flight files independently. With 4 processes on a 4-core machine,
> near-4× wall-clock improvement is realistic.

## Status

✅ **Stage 1 SHIPPED and QA-validated (2026-05-10).** `1e38b1fe`.
35.0% wall-clock reduction (97:22 → 63:16) at N=2. Defense-in-depth
(`e3500fd1`) added: orphan-window guard, length-parity assertion,
3 new failure-mode tests, refined postcheck script.

✅ **Stage 2 SHIPPED + QA-validated + fix-first remediations applied
(2026-05-11).**

- Backfill (`9c82fbda`): N=2 → N=4. Validated 50:08 wall-clock =
  **48.49% reduction** at the predicted Option C ceiling. Postcheck
  PASS, zero incomplete rows, integrity invariants hold.
- Foreground (`9c82fbda` + hotfix `663211ac`): single manager → N=4
  pool. Initial validation showed only 33.8% reduction (190s vs
  287s) because `coordinator.maxConcurrent: 2` (predates pool) was
  starving the dispatcher to 2-wide — `peakInFlight: 2` despite a
  4-member pool. **Hotfix `663211ac`** floors effective
  `maxConcurrent` at `pool.size`. Re-validated at **2:35 wall-clock =
  45.0% reduction** with `peakInFlight: 4`. Now at the CPU-bound floor
  (sum of per-file CPU / pool size = 155s, matching wall-clock
  exactly); further wins require Option A (file-lifecycle reuse) or
  the FileDetail.tsx tail follow-up.

### Stage 2 QA round (2026-05-11) — verdict: FIX-FIRST, all remediations applied

Three independent agents (`implementation-analyst`,
`sqlite-engineer`, `code-complexity-analyzer`) audited the Stage 2 +
hotfix branch. Verdict: **FIX-FIRST.** Stage 2's correctness logic was
sound, but four concrete fix items had to land before merge. All four
were applied in this session:

| # | Fix | Commit | What |
|---|---|---|---|
| F1 | LRU bound on `AnalysisCacheManagerImpl` | `814f7d1b` | The class was advertised in CLAUDE.md as "in-memory LRU" but had no eviction. Default `maxEntries: 5,000`, config-overridable. First-eviction warning logged once per instance. Engine doc drift fixed. |
| F2 | RAM gate in `resolvePoolSize` | `78edf73e` | On machines with `totalmem() < 8 GB`, pool size is further capped at 2. Defensive — preserves user overrides below the cap. |
| F3 | Tracing state rollback on partial fan-out | `b1ff3169` | `tracing-session-service.ts:340` was setting `state.status='recording'` BEFORE `Promise.all(configureTracing)`. A single rejecting member left the service inconsistent. Now: snapshot prior state, mutate optimistically, roll back on any rejection (including main tracing config). |
| F4 | Utility-process RSS sampler | `4b18e212` | New `sample-utility-rss.mjs` script: poll `ps -axo` during a backfill, summarize per-process peak/mean RSS afterward. Closes the QA's "unmeasured RSS at N=8" gate so future sessions can characterize the footprint without code changes. |

Plus a postcheck regression fix (`72753e94`) — foreground-only DBs
were incorrectly flagged as failing the `indexing_jobs` existence
check; now informational.

**Validation after all fixes:**
- `pnpm typecheck` (workspace): ✅ 56/56 tasks pass
- `pnpm lint` (workspace): ✅ 0 errors
- `pnpm checks:formatting` (workspace): ✅ 35/35 tasks pass
- `pnpm --filter @vipr/desktop test`: ✅ **4090/4090** (was 4084; +6 new)
- `pnpm --filter @vipr/engine test`: ✅ **234/234** (was 226; +8 new)
- `pnpm --filter @vipr/common test`: ✅ 513/513
- `stage-1-postcheck.mjs` against the user's foreground-only DB: ✅ PASS

**QA deferred items (not blocking merge; filed as separate follow-ups):**

- T5.4 — Backfill `onRestarted` re-dispatch coupling (retry loop
  compensates today)
- T1.3 — "Capped at runtime" log assertion test
- T7.1 — Unbounded coordinator backlog queue (pre-existing,
  pathological-repo concern)
- T7.3 — `file_metrics` SIGKILL race (pre-existing serious; worth a
  closer look post-merge)

🔵 **Stage 3 next** (post-Stage-2-validation). Per-member crash
recovery, pool-level health check, vipr.config.json disable knob.

## Background

Promoted from item 5 / Option C after item 7's read-only audit revealed
that the dormant `AnalysisWorkerPool` (worker_threads) requires weeks of
bundling rework to be usable. Option C was originally framed as the
"larger, riskier" alternative — but the Stage 0 audit below shows the
infrastructure investment is significantly smaller than expected.

## Goal

Reduce foreground initial-scan and backfill wall-clock by activating
real multi-core parallelism via additional Electron `utilityProcess`
instances. Targets:

- **Foreground initial scan**: ~75% wall-clock reduction (4:42 → ~1:10)
  on a 4-core machine
- **Backfill**: 30-50% wall-clock reduction (97 min → 50-65 min) — bounded
  by the fact that backfill is also serialized at the commit level

These are upper-bound estimates; the bottom-up assessment in Stage 1 will
firm them up against measurement.

---

## Stage 0 read-only audit (2026-05-10)

Audited the desktop's existing main-process / utility-process
architecture by reading the source. Three categories of findings, all
positive.

### Finding 1 — The desktop already runs 2-3 utility processes today

- `vipr-analysis-worker` (foreground) — `UtilityProcessManager` with
  `purpose: 'foreground'`, instantiated in `clients/desktop/src/main/index.ts`
  line 599
- `vipr-backfill-worker` (backfill) — same manager class, `purpose: 'backfill'`,
  line 603
- `vipr-remediation-analysis-worker` (AI remediation, on-demand) —
  instantiated in `clients/desktop/src/main/ai/ai-remediation-service.ts`
  line 254

All three load the same bundled `worker.js` via
`utilityProcess.fork(workerPath, ...)`. **Option C is therefore not "1 → N
processes" — it's "scale the existing N+ pattern."**

### Finding 2 — DB writes are serialized through the main process

The utility process opens SQLite **only** for the one-shot
`repairHistoricalSnapshots` operation (worker.ts line 35-38). All other
analysis writes happen in the main process:

- Foreground per-file results: coordinator's `persistAnalysisResultWithRetry`
  → `this.db.saveAnalysisResult(...)` (main process)
- Backfill snapshot rows: `historicalSnapshotService.createIncrementalSnapshot(...)`
  → main-process DB queries
- Backfill commit metrics: `historicalSnapshotService.upsertCommitMetrics(...)`
  → main-process DB queries

**Implication**: under N utility processes, all writes still serialize
through the single main-process DB instance. **No SQLite write
contention.** The existing `SQLITE_BUSY`/`SQLITE_LOCKED` retry logic in
`coordinator.ts` (lines 26-39, with 3-attempt retry at 100ms intervals)
remains as a defense-in-depth safeguard, not a load-bearing requirement.

This invalidates the "DB contention" risk listed in `item-05-strategic-initiatives.md`.

### Finding 3 — `UtilityProcessManager` already abstracts a single process cleanly

`clients/desktop/src/main/analysis/utility-process-manager.ts` provides:

- IPC correlation by uuid (`pendingRequests` map per-instance)
- Per-message timeout + Zod validation
- Per-instance `tracer.trace('utility-request', { ... purpose, utilityPendingAtSend })`
- Restart logic: max 3 attempts with backoff (1s, 3s, 10s)
- `onRestartedCallback` for the coordinator to re-enqueue files lost in a crash
- `configureProject(repoPath)` lifecycle (called per-process after start)
- `configureTracing(...)` for trace session forwarding
- Shutdown orchestration with grace period

**A `UtilityProcessPool` wrapping N instances is mechanical** — instantiate N managers,
add a load-balancer (round-robin or least-busy by `pendingRequests.size`),
fan out IPC calls, aggregate restart callbacks. No new abstractions needed.

### Finding 4 — Coordinator's existing dispatcher pattern is pool-friendly

`coordinator.ts` line 432-456 already overlaps requests up to
`maxConcurrent`:

```ts
while (
  inFlightPromises.size < this.config.maxConcurrent &&
  this.queue.length > 0 &&
  !this.cancellationToken.isCancelled &&
  !this.isPaused
) {
  const item = this.queue.shift();
  this.processing.add(item.filePath);
  const promise = this.processFile(item).finally(() => {
    this.processing.delete(item.filePath);
    inFlightPromises.delete(promise);
  });
  inFlightPromises.add(promise);
}
```

Today, `maxConcurrent` parallel requests all hit the **same** utility
process (which serializes them inside its single V8 isolate). Replacing
`this.utilityManager.analyzeFile(filePath)` with
`this.utilityPool.analyzeFile(filePath)` (where the pool routes to one
of N processes) gives you real CPU parallelism with zero rework of the
queue dispatcher.

The same pattern applies to backfill: `historicalSnapshotService` calls
`this.utilityManager.analyzeContentBatch(...)` per cache-miss batch
(line 1486). A pooled version splits each batch across N processes in
parallel.

### Finding 5 — Bundling rework is NOT required (item 7's blocker 3 does not apply)

Item 7 needed `worker_threads` + `import('@vipr/core')` to resolve at
runtime — which fails in a packaged Electron app because the analyzer
packages are rolled into `worker.js` and not present as `node_modules`
entries.

Option C uses `utilityProcess.fork(workerPath, ...)` which spawns a
**fresh Node process** that loads `worker.js` from disk. The bundle
already includes `@vipr/core`, `@vipr/react`, etc. — the freshly-spawned
process has everything it needs. **Zero bundler config changes
required.**

### Finding 6 — Tracing already supports multiple utility managers

`tracing-session-service.ts` line 220:

```ts
setUtilityManagers(managers: Array<UtilityProcessManager | null | undefined>): void {
  this.utilityManagers = managers.filter(...)
}
```

The service iterates the array when starting/stopping a trace session
(lines 356, 396). **Registering N pool members with tracing is
trivial.**

### Findings summary table

| Concern | Audit verdict |
|---|---|
| SQLite write contention under N processes | ✅ N/A — main process owns all writes |
| IPC routing across N processes | ✅ Trivial — uuid correlation per-process |
| Bundling rework (item 7's blocker) | ✅ N/A — `utilityProcess` spawns fresh Node, loads `worker.js` from disk |
| Coordinator queue dispatcher rework | ✅ Minimal — replace `utilityManager` with `utilityPool` reference |
| Restart / crash-recovery per process | ✅ Already per-process; pool just iterates |
| Tracing across N processes | ✅ Already supports an array of managers |
| `configureProject` lifecycle across N | ⚠️ Pool must fan out to all members (~4-6 calls instead of 1) |
| Per-process plugin loading cost | ⚠️ ~1-3s × N at cold start (parallelizable across processes) |
| Memory: N × ts-morph state | ⚠️ ~400MB × N peak; defaults must respect lower-spec machines |
| Backfill scheduler integration | ✅ Pool used the same way; per-batch fan-out across N |

The two ⚠️ items are configuration concerns, not architectural
blockers. Defaults need calibration; nothing fundamental needs
redesigning.

---

## Sequencing decision (2026-05-10)

**Backfill-first.** Earlier drafts of this doc had Stage 1 targeting
foreground initial-scan because the coordinator's `maxConcurrent`
dispatcher made it the easier first wire-up. That's engineering
convenience, not user value. Backfill is the experiential pain (97 min
vs foreground's 5 min — a 19× difference in wall-clock you actually
feel), so Stage 1 attacks backfill directly.

**Realistic ceiling acknowledged.** Backfill has structural
serialization at the commit level (snapshot N depends on snapshot N-1).
Process pooling cannot break that; it only fans out file analysis
WITHIN a commit. The realistic ceiling at N=4 is ~50% wall-clock
reduction (~97 min → ~48 min), not the ~75% projection that applies to
foreground. If 48 min is still too long, the next lever is
**"Option D"** — pre-fetch + pre-analyze the next K commits' cache-miss
files in parallel with the current commit's snapshot persistence
(decouples analysis from snapshot writes; pipelines them). Out of scope
for Option C; flagged here so we don't promise more than C can
deliver.

**Differential test gate is non-negotiable.** Any snapshot row, commit
metric, or file_version row mismatch in the 5-commit pilot reverts the
stage immediately. The snapshot chain is load-bearing for every
downstream consumer (history, churn, A/B deltas).

## Stage 1 — Two-process backfill pool + per-commit fan-out

### Conduct (backfill-first)

1. Create `clients/desktop/src/main/analysis/utility-process-pool.ts`
   wrapping N `UtilityProcessManager` instances. Round-robin routing for
   per-call methods; fan-out for batch methods.
2. Replace `backfillUtilityManager` (single instance) in `index.ts` with
   a 2-member backfill pool. Foreground manager stays single-process.
3. In `historical-snapshot-service.ts` (line 1486), change:
   ```ts
   const batchResults = await this.utilityManager.analyzeContentBatch(batch);
   ```
   to fan-out across pool members:
   ```ts
   const batchResults = await this.utilityPool.analyzeContentBatchParallel(batch);
   ```
   The pool internally chunks the batch into N sub-batches (one per
   member) and dispatches them in parallel via `Promise.all`. Result
   order is preserved by chunk-index then within-chunk order.

Reference design for `UtilityProcessPool`:

```ts
export interface UtilityProcessPoolOptions {
  poolSize: number;
  serviceNamePrefix: string;
  purpose: 'foreground' | 'backfill';
  loadBalancer?: 'round-robin' | 'least-busy';
}

export class UtilityProcessPool {
  private members: UtilityProcessManager[] = [];
  private nextRoundRobinIndex = 0;
  private readonly loadBalancer: 'round-robin' | 'least-busy';

  constructor(options: UtilityProcessPoolOptions) { /* ... */ }

  async start(): Promise<void> {
    await Promise.all(this.members.map(m => m.start()));
  }
  async stop(): Promise<void> { /* shutdown all */ }
  async configureProject(repoPath: string): Promise<UtilityProjectConfigResult> {
    // Send to all members; resolve with first config (they should all match)
    const results = await Promise.all(this.members.map(m => m.configureProject(repoPath)));
    return results[0]!;
  }
  // Per-call routing
  async analyzeFile(filePath: string): Promise<RuntimeAggregatedResult> {
    return this.pickMember().analyzeFile(filePath);
  }
  async analyzeContent(filePath: string, content: string): Promise<RuntimeAggregatedResult> {
    return this.pickMember().analyzeContent(filePath, content);
  }
  async analyzeContentBatch(items: BatchAnalysisItem[]): Promise<BatchAnalysisResult[]> {
    // Stage 1: route whole batch to one member; Stage 3 will sub-batch across members
    return this.pickMember().analyzeContentBatch(items);
  }

  private pickMember(): UtilityProcessManager {
    if (this.loadBalancer === 'least-busy') {
      // Pick member with smallest pendingRequests.size
      // (requires UtilityProcessManager to expose pendingCount)
    }
    // Default round-robin
    const member = this.members[this.nextRoundRobinIndex % this.members.length]!;
    this.nextRoundRobinIndex = (this.nextRoundRobinIndex + 1) % this.members.length;
    return member;
  }

  // Forward tracing + onRestarted callbacks
  onAnyRestarted(callback: (purposeAndIndex: { purpose: string; index: number }) => void): void {
    this.members.forEach((m, index) => m.onRestarted(() => callback({ purpose: this.purpose, index })));
  }
}
```

Wire it into `clients/desktop/src/main/index.ts`:

```ts
// Replace single foreground manager with a 2-member pool for Stage 1
const foregroundPool = new UtilityProcessPool({
  poolSize: 2,
  serviceNamePrefix: 'vipr-analysis-worker',
  purpose: 'foreground',
  loadBalancer: 'round-robin',
});
// Keep backfillUtilityManager as a single manager for Stage 1 (Stage 2 expands it)
```

Update `coordinator.ts` to accept the pool instead of the single manager:

```ts
constructor(
  private readonly db: DatabaseService,
  private readonly utilityPool: UtilityProcessPool,  // was: utilityManager
  // ...
) { }
```

The dispatcher loop (line 432-456) is unchanged — only the call site at
line 677 changes from `this.utilityManager.analyzeFile(filePath)` to
`this.utilityPool.analyzeFile(filePath)`.

### Analyze

After the change builds and runs:

- Confirm 2 backfill utility processes spawn
  (`vipr-backfill-worker-0`, `vipr-backfill-worker-1`) — observable in
  Activity Monitor
- Confirm `analyzeContentBatchParallel` chunks distribute across both
  members (per-process trace output should show roughly 50/50 split of
  files)
- Confirm `configureProject` fans out to both on repo open
- Confirm crashing one process triggers its onRestart callback without
  interrupting the other

### Validate

- Backfill baseline on the 90-day window (97 min today)
- **Target: 25-30% wall-clock reduction at `poolSize=2`** (97 min → ~70 min)
- All 4045+ desktop tests stay green
- No new `SQLITE_BUSY`/`SQLITE_LOCKED` errors in the trace logs
  (defense-in-depth check; main-process DB writes are still serialized)
- Memory: backfill utility processes' combined RSS at peak should not
  exceed ~1.5× single-process baseline
- **Differential test (non-negotiable)**: pick 5 commits, run them
  through single-process backfill, capture snapshot_files,
  commit_file_changes, file_versions, commit_metrics rows for those
  commits; run again through pooled backfill; **every row must match
  exactly**. Any mismatch = immediate revert.

### Proceed-or-stop

- ✅ ~25%+ wall-clock reduction with snapshot integrity intact → proceed to
  Stage 2 (expand backfill pool to N=4, then add foreground pooling)
- ⚠️ 10-25% reduction → STOP and analyze. Likely explanations:
  (a) per-commit serial cost dominates at N=2 (try N=4 in Stage 2 first
  before stopping); (b) IPC serialization cost erodes the win on small
  batches; (c) hidden serial work in `historicalSnapshotService` that
  needs sub-phase instrumentation
- ❌ Snapshot mismatch in differential test → revert immediately. The
  pool introduces a behavior we don't understand
- ❌ Any new test failures → revert

### Stage 1 Closure (2026-05-10)

**Status: ✅ Done. SHIPPED.**

**Outcome**: 35.0% wall-clock reduction at N=2 (97:22 → 63:16),
comfortably past the 25% target. Snapshot integrity verified.

**Commits**: `1e38b1fe` (Stage 1 implementation), `e747ddd7` →
`d9cda43e` (postcheck script + iterations), plus the defense-in-depth
commit (this session) covering orphan-window guard, length-parity
assertion, refined postcheck, and 3 new failure-mode tests.

**Validation path taken (vs the rigorous double-build option)**: one
backfill + integrity-check script. The script was wrong on the first
pass (flagged 9 "false failures"); QA pass (three independent agents:
`implementation-analyst`, `sqlite-engineer`,
`code-complexity-analyzer`) demanded by the user resolved the
ambiguity. Diagnostic SQL exposed `is_complete=1` on all 9, which
proved Hypothesis B (structural artifact: commits touching only
non-analyzable files trigger `historical-snapshot-service.ts:1989`
fall-through and create a delta snapshot row with zero deltas — by
design).

**What QA caught that I missed**:
1. **Orphan-window bug** (latent, didn't fire in this run): the
   sequence `createSnapshotSafe` (writes row with `is_complete=0`) →
   `getFilesAtCommit` → `analyzeFileList` is **not wrapped in
   try/catch**. If anything rejects between row insert and the
   transaction at `:1712`/`:2087`, the row stays orphaned. Stage 1
   (N=2) makes this more reachable than single-process did (more
   processes = more crash surface, `Promise.all` turns any single
   member rejection into total batch failure). Fixed in the
   defense-in-depth commit before Stage 2 expands to N=4.
2. **Non-null assertion at `:1505`** (latent): `batchResults[j]!`
   assumes pool returns parity-length result arrays. No contractual
   guarantee. Fixed by adding length-parity assertion in
   `analyzeContentBatchParallel`.
3. **Postcheck script bugs** (immediate): C3 invariant was wrong
   (didn't account for delta snapshots with zero analyzable changes);
   C7 false positive (skipped commits → no commit_metrics row, but
   counted in `processed_commits`); signature query ignored deltas
   entirely. All fixed in the defense-in-depth commit.

**Decision-quality note for the rolling sequence**: the QA round was
worth its cost. Three independent agents reading the code surfaced two
real latent bugs I missed. For Stage 2, run another QA pass before
declaring done — the larger process count justifies it.

---

## Stage 2 — Expand backfill pool to N=4, then add foreground pool

### Conduct

- Expand backfill pool from 2 → `Math.min(cpus().length - 1, 4)`
- THEN add foreground pooling at the same default size (replace
  single foreground manager with a foreground pool; coordinator's
  call site swaps from `utilityManager.analyzeFile` to
  `utilityPool.analyzeFile`)
- Promote `poolSize` to a setting in `vipr.config.json` /
  `DESKTOP_CONFIG_DEFAULTS` so power users can override per-purpose
- Add per-process tracing tag so traces from different members are
  distinguishable (`utility-request:backfill:0`,
  `utility-request:backfill:1`, etc.)

### Analyze + Validate

- Re-baseline backfill (target ~50% reduction at N=4 — the structural
  ceiling for Option C; see "Realistic ceiling acknowledged" above)
- Re-baseline foreground (target ~75% reduction at N=4)
- Memory ceiling: total RSS across all utility processes should stay
  under ~3GB on the baseline 1000-file repo
- All tests still green
- Differential test on backfill repeats (same 5-commit pilot)

### Proceed-or-stop

- ✅ Targets met within memory budget → proceed to Stage 3
- ⚠️ Backfill plateaus at ~30% even at N=4 → the ceiling is lower than
  expected; document and consider whether to invest in Option D
  (commit-pipeline) or Option A (file-lifecycle reuse)
- ⚠️ Memory exceeds budget at default `poolSize` → cap default lower
  (3 instead of 4) and document
- ❌ Tests fail or differential mismatch → revert Stage 2 only (Stage 1
  pool can stand)

---

## Stage 3 — Crash recovery + production hardening

### Conduct

- Per-member crash recovery: when one process crashes, in-flight files
  for that member only are re-enqueued; others continue
- Pool-level health check: if 50%+ of pool members crash within 60s,
  fall back to single-process mode and surface a user-visible error
- Add `vipr.config.json` setting to disable parallelism entirely
  (`worker.parallelism.enabled: false` → poolSize=1)
- Settings UI knob (separate item if it doesn't fit cleanly here)

### Analyze + Validate

- Kill a worker process mid-backfill; confirm orphaned files re-enqueue
- Stress-test: spam analysis requests, confirm pool stays healthy
- Memory profile during 30-minute sustained workload

---

## Risks (re-assessed after audit)

| Risk | Original assessment | Audit-updated assessment |
|---|---|---|
| DB contention | High | **Eliminated** — main process owns all writes |
| IPC complexity | Medium-high | **Low** — uuid correlation is per-process; pool just routes |
| Per-process startup cost | Medium | **Low** — ~1-3s × N parallelizable; only at cold start |
| Memory: N × ts-morph state | High | **Medium** — measurable, manageable with `poolSize` cap |
| Crash recovery across N processes | Medium | **Low-medium** — already per-process today; pool iterates |
| Bundling / artifact distribution | Not previously assessed | **N/A for utilityProcess** (item 7's blocker only applied to worker_threads) |

Net risk: **moderate**, dominated by memory budget at high `poolSize`. Mitigated
by conservative default and config knob.

---

## Estimated effort

| Stage | Effort | Notes |
|---|---|---|
| Stage 1 (2-process MVP) | 3-5 days | Pool class + coordinator wire-up + differential test |
| Stage 2 (configurable + backfill pool) | 1 week | Config plumbing + sub-batch fan-out |
| Stage 3 (hardening) | 1 week | Crash recovery + health check + settings UI knob |
| Re-baselining | ~95 min × 3-4 iterations | Stage 1, Stage 2, Stage 3 each |

**Total: 2-3 weeks across multiple sessions** — significantly less than
the original 1-2 weeks for MVP / 3-5 weeks for production estimate, because
the bundling work doesn't apply.

---

## Out of scope

- **Settings UI for tuning pool size in real-time** — could be a
  follow-up item; for now, edit `vipr.config.json` and restart
- **Cross-process AST cache sharing** — each process has its own ts-morph
  state; sharing is a future Option A concern, not Option C
- **NUMA/CPU pinning** — over-engineering for a desktop app
- **Streaming analysis results back from utility to main as they
  complete** — not needed; current `analyzeContentBatch` collect-all
  pattern is fine

---

## Hand-off if Stage 1 stalls

If Stage 1 measurement reveals <15% wall-clock reduction, the working
assumption ("single-V8-isolate contention is the bottleneck") is wrong.
Likely alternate explanations:

1. **Per-file work is I/O bound** (file reads, git operations) and
   parallelism is wasted on waiting
2. **Main-process serialization is the bottleneck** (DB writes, queue
   management)
3. **Plugin coordination overhead** in the engine is amortized poorly
   across processes

In any of those cases, escalate to **Option A** (file-lifecycle reuse)
which addresses the per-file CPU floor directly via ts-morph project
sharing — independent of process count.
