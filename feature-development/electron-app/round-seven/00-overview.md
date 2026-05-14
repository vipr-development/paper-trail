# Desktop Analysis Performance Plan

> Rolling sequence of work for ongoing perf and quality improvements to the
> Vipr desktop analysis pipeline. This is a **rolling window** — finished
> items collapse, future items get refined as we learn. Maintained so any
> agent can pick up any item without needing prior conversation context.

## Current rolling sequence (sequential, one commit per item)

| # | Status | Doc | Description |
|---|---|---|---|
| **1** | ✅ Done (`62b55bae`) | n/a | Fix six test regressions surfaced by the Phase 3 merge; verify no regressions; full suite (4043/4043) green |
| **2** | ✅ Done (`82fb5f87`, `bc47853b`) | [`item-02-backfill-wallclock.md`](./item-02-backfill-wallclock.md) | Backfill wall-clock measurement-driven cycle. Shipped wins: hoisted hot-path SQL prepares, sub-phase per-commit instrumentation, log-noise filter, isAnalyzableFile bug fix. Closed: incremental write path is already cheap (<28ms/commit total); dominant cost is `commitAnalysisTime` (66%) which is bounded by per-file CPU floor. Remaining wall-clock reduction requires structural change → item 7. |
| **3** | 📋 Queued | [`item-03-telemetry-tracing.md`](./item-03-telemetry-tracing.md) | Move PerformanceProfile out of console logging and into the `@vipr/tracing` system as a toggle-able feature |
| **4** | 📋 Queued | [`item-04-vipr-config-audit.md`](./item-04-vipr-config-audit.md) | Rigorous audit of existing `vipr.config.json` implementation; maximize default config shape; user-facing docs |
| **5** | ✅ **Stage 1 + 2 SHIPPED** — Option E next | [`item-05-strategic-initiatives.md`](./item-05-strategic-initiatives.md), [`initiative-option-c-process-parallelism.md`](./initiative-option-c-process-parallelism.md), [`initiative-option-e-storage-normalization.md`](./initiative-option-e-storage-normalization.md) | **Option C — Process-level parallelism.** Stage 1 (N=2 backfill): 35% reduction. Stage 2 (N=4 backfill + foreground pool + hotfix): backfill 48.5%, foreground 45% — both at the CPU/parallelism ceiling. **Option E — Storage normalization.** Queued and planned. Audit: 130 MB DB, 97.7% in two tables, ~85% of insight content is static templates. Projected: 130 MB → ~35 MB (73% smaller). Phased dual-write migration; recommended next initiative ahead of Stage 3 hardening. |
| **6** | 📋 Queued | [`item-06-analysis-level-filtering.md`](./item-06-analysis-level-filtering.md) | Architectural improvement: per-analysis preconditions + shared file-capability index (Phase 5 work) |
| **7** | ✅ Done — closed via audit (no code) | [`item-07-worker-pool-experiment.md`](./item-07-worker-pool-experiment.md) | Read-only Stage 0 audit identified 3 architectural blockers (worker-protocol incompatible with backfill; engine call site doesn't engage pool; bundler emits single artifact with rolled-in analyzers). Cost flipped weeks-not-days; sequencing handed off to item 5 / Option C, which inherits the same bundling work but yields true multi-core parallelism. |

Each item's doc is structured the same way (Goal, Why now, Prerequisites,
Implementation, Acceptance criteria, Test plan, Risk/rollback, Out of scope,
Estimated effort). A fresh agent should pick up any single doc and ship it
without needing this conversation.

## Sequencing notes

- **Item 2 closed (2026-05-10).** The post-`bc47853b` baseline showed all
  five new sub-phase metrics on the incremental write path are tiny
  (combined ~28ms/commit, ~1% of `commitAnalysisTime` mean of 9.9s).
  No tactical write-path fix can move backfill wall-clock materially while
  the analysis path is the floor.
- **Item 7 closed by Stage 0 audit (2026-05-10).** Read-only audit of the
  engine + desktop sources surfaced three architectural blockers:
  worker-protocol incompatibility with backfill (no `analyzeContent`
  branch); engine call site doesn't engage the pool path
  (`analyzeFile` singular); bundler emits a single `worker.js` with
  analyzer packages rolled in (no `node_modules` for `import()` from
  worker_threads). The cost calculus flipped weeks-not-days, so
  sequencing was handed off to item 5 / Option C — which requires the
  same bundling work but yields true multi-core parallelism.
- **Item 5 / Option C — Stage 1 SHIPPED and QA-validated (2026-05-10).**
  Single 90-day backfill on the baseline repo delivered **35.0%
  wall-clock reduction** (97:22 → 63:16) at N=2, comfortably past the
  25% Stage 1 target. Independent QA (`.claude/qa-stage-1/`) confirmed:
  fan-out logic is structurally correct; the 9 zero-state snapshots
  flagged by the initial postcheck are **structural artifacts** —
  commits that modified only non-analyzable files (docs, configs, lock
  files), reproduced cleanly by `historical-snapshot-service.ts:1989`
  early-exit fall-through. `is_complete=1` on all 9 + valid
  `parent_snapshot_id` = state reconstructible via parent chain.
  Stage 1 ships.
- **Defense-in-depth before Stage 2** (✅ shipped `e3500fd1`):
  orphan-window guard in `createSnapshot{ForCommit,Incremental}Internal`
  (try/catch + row delete on rejection), length-parity assertion in
  `analyzeContentBatchParallel` (prevents silent IPC truncation), 3 new
  failure-mode tests, postcheck script refinement (C3 invariant fixed,
  C7 false positive dropped, signature now includes deltas).
- **Stage 2 SHIPPED + QA-validated + fix-first remediations applied
  (2026-05-11):**
  - **Backfill** (`9c82fbda`): ✅ 48.49% reduction (97:22 → 50:08), at
    the predicted Option C ceiling.
  - **Foreground** (`9c82fbda` + hotfix `663211ac`): ✅ 45.0% reduction
    (4:42 → 2:35), `peakInFlight: 4`. Now CPU-bound (`sum of per-file
    CPU / pool.size`); further wins require Option A or the
    FileDetail.tsx tail follow-up.
  - **QA round** (three independent agents): verdict FIX-FIRST. Four
    remediations applied:
    - **F1** (`814f7d1b`) — LRU bound on `AnalysisCacheManagerImpl`
      (was advertised as LRU but had no eviction). Default
      `maxEntries: 5,000`, config-overridable. Engine CLAUDE.md doc
      drift fixed.
    - **F2** (`78edf73e`) — RAM gate in `resolvePoolSize`: on
      `totalmem() < 8 GB`, pool further capped at 2.
    - **F3** (`b1ff3169`) — Tracing service rolls back state on
      `Promise.all` rejection during fan-out (was committing
      `status='recording'` before utility-manager ACKs).
    - **F4** (`4b18e212`) — New `sample-utility-rss.mjs` script for
      measuring per-process RSS during a backfill.
  - Plus postcheck regression fix (`72753e94`) — foreground-only DBs no
    longer flagged as failing the `indexing_jobs` existence check.
  - **Tests after fixes:** desktop 4090/4090, engine 234/234, common
    513/513. Workspace typecheck/lint/format all clean.
  - **QA-deferred items** filed for later (not blocking):
    [`followup-option-c-stage2-deferred.md`](./followup-option-c-stage2-deferred.md)
    — `file_metrics` SIGKILL race (pre-existing serious), backfill
    `onRestarted` coupling, unbounded coordinator queue,
    `resolvePoolSize` log assertion.
- **Next strategic initiative — Option E (Storage normalization).** Audit
  against the user's actual workspace DB after Stage 2 validation:
  **130 MB total**, of which **127 MB** lives in `analyses` (84.8 MB) +
  `snapshot_files` (42.5 MB). Sample inspection of a single React
  insight showed ~85% of bytes are static template content
  (prompt.template, explanation, detailsMarkdown, suggestion).
  Projected reduction to **~35 MB (73% smaller)** by hoisting templates
  to client-bundled static manifests. **Phase 0 consumer audit complete
  (2026-05-11):** [`followup-option-e-phase0-consumer-audit.md`](./followup-option-e-phase0-consumer-audit.md)
  catalogues ~103 files to update; manifest schema designed for
  Option F extension. See
  [`initiative-option-e-storage-normalization.md`](./initiative-option-e-storage-normalization.md)
  for the full 5-phase plan; Phase 1 (manifest generator) is next.
- **Following Option E — Option F (Documentation as a client).**
  Pre-launch foundation work. The founder confirmed
  ([memory note](../../../../.claude/projects/-Users-jamesleebaker-Codespace-vipr/memory/feedback_pre_launch_foundation.md))
  that "having a proper documentation site that is consumable across
  marketing and desktop would be a great component of a successful
  launch." Reshape: move `@vipr/documentation` to `clients/documentation`
  as a buildable artifact; have analyzers OWN per-analysis docs;
  marketing/desktop/VSCode/CLI consume the docs client's built output.
  Drift surface confirmed physical: 30+ hand-maintained
  `content/analysis/*.md` files already mirror analyzer code. ~4 weeks
  across 6 phases. See
  [`initiative-option-f-documentation-as-client.md`](./initiative-option-f-documentation-as-client.md).

## Polish / follow-ups (not blocking the rolling sequence)

- [`followup-performance-profile-ingestion.md`](./followup-performance-profile-ingestion.md) —
  encapsulate the nested per-(file, analysis) ingestion loops in
  `backfill-scheduler.ts` and `coordinator.ts` behind two new
  `PerformanceProfile.ingest*` methods. ~40 lines removed; eliminates
  duplication between foreground and backfill paths; pure refactor.
  Land **after** Option C Stage 1's differential test passes so the
  refactor has a clean behavioral baseline. ~1.5 hours.
- [`followup-progress-estimation-ux.md`](./followup-progress-estimation-ux.md) —
  user-reported UX rough edge: ETA briefly shows "very large / 23 min"
  in the first 5-10 seconds of foreground analysis before stabilizing
  to the actual ~3 minutes. Cheap fix: suppress the ETA until a
  stability threshold (5% of files OR 30s elapsed). ~2-3 hours.
- [`followup-backfill-progress-ux.md`](./followup-backfill-progress-ux.md) —
  extends the progress-estimation work to backfill across two new
  surfaces: title-bar history tooltip + bottom progress bar. Includes
  pre-flight accuracy fix (`historicalMsPerCommit` floor removal, plus
  pool-size invalidation), live ETA stability gate (shared helper with
  the foreground UX item), and a bottom-bar layout redesign that gives
  commit info more horizontal space alongside an inline ETA. ~2.5 days
  total; soft dependency on the algorithm helper above.
- [`followup-filedetail-tail-latency.md`](./followup-filedetail-tail-latency.md) —
  `FileDetail.tsx` is the dominant single-file analysis cost across
  both backfill and foreground (12s of CPU per pass; 4 separate
  analyses each at ~4s). Suspected redundant AST traversal across
  analyses. Concrete kickoff target for the future Option A
  (file-lifecycle reuse) strategic initiative. Investigation ~half a day.
- [`followup-option-c-stage2-deferred.md`](./followup-option-c-stage2-deferred.md) —
  four QA-deferred items from the Stage 2 audit:
  - D1 `file_metrics` SIGKILL race (pre-existing serious — orphan-window
    guard covers snapshot rows but not mid-write file_metrics)
  - D2 backfill pool `onRestarted` re-dispatch coupling (retry loop
    compensates today)
  - D3 unbounded coordinator backlog queue (pathological-repo
    concern)
  - D4 `resolvePoolSize` "capped at runtime" log assertion test
    (~30 min observability gap)
- **The conduct → analyze → validate → proceed-or-stop pattern** applies
  especially to item 7 (and any future strategic-initiative work). Each
  stage validates positive outcomes without regression before proceeding;
  back off to last known-good when current fidelity becomes unstable.

## Branch + commit context

- **Branch**: `claude/unruffled-hawking-f89a04`
- **Base**: `origin/main` (rebased through the productionize-vipr-config merge)
- **Active commits on branch** (newest first):
  - `62b55bae` — Item 1: fix six test regressions from the merge (4043/4043 green)
  - `360d182b` — Phase 4 perf: AST-only paths in core-functions + ts-declaration-shape
  - `329454ba` — Desktop maxConcurrentAnalyses=1 (honest per-analysis timing)
  - `2298a5d7` — Bundler-safe static JSON imports for analyzer default config
  - `03804da3` — Per-analysis timing on AggregatedResult + profile aggregation
  - `034c13b7` — Merge productionize-vipr-config into branch
  - `b160553f` — historicalBatchConcurrency=1 default
  - `c141beff` — Fix isAnalyzableFile drift in two services
  - (older Phase 0 / 0.5 instrumentation commits)

## Why this exists

In a baseline measurement on the Vipr monorepo's `clients/desktop` (1013 source
files), foreground analysis took **4:38 wall clock** and a 90-day backfill
(394 commits, 7330 cache-miss files) took **94:41**. CPU was pegged
throughout. The user wants to support larger repos without pegging machines
and to keep navigation responsive after analysis completes.

After items shipped so far:
- Foreground: 4:39 → 4:42 (essentially unchanged; per-analysis CPU was conserved)
- Per-analysis instrumentation now honest (was an artifact of `maxConcurrentAnalyses=Infinity`)
- Critical bug fix: backfill no longer wastes time on `mock/`, `__tests__/`, etc.
- Per-plugin / per-analysis enablement wired (user can opt out via `vipr.config.json`)
- Phase 1b/c default exclusion expansion folded in via the merge

The remaining lever for raw wall-clock reduction is in items 2 (backfill) and
5 (Option A file lifecycle, Option C process parallelism).

## Reference docs

- [`baselines.md`](./baselines.md) — raw `Performance summary` JSON for diffing
- [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md) — measurement procedure
- [`05-config-worktree-review.md`](./05-config-worktree-review.md) — verdict on the merge that landed (kept for historical context)

The original per-phase docs (`01-bug-fix-historical-filter.md`,
`02-exclusion-list-expansion.md`, `03-backfill-concurrency.md`,
`05-config-productization.md`, `06-per-file-mean-reduction.md`) describe
shipped work and remain on disk for historical context until item 5
replaces them.

## Decision log (do not re-litigate)

- **The engine's worker pool path is dead code in the desktop.** Worker calls
  `analyzeFile`/`analyzeContent` (single-file), never `analyzeFiles`. The
  `cpus().length - 1` thread count from `analysis-engine.ts` does not apply.
- **DB and IPC are NOT bottlenecks.** Per-file `dbPersistMs` mean is 1.2ms;
  `eventEmitMs` mean is 0.2ms.
- **`snapshot_file_deltas(snapshot_id, file_path)` UNIQUE constraint already
  creates an autoindex** — no additional index needed.
- **`useFileList.ts` is dead code.** No consumers.
- **`Files.tsx` already paginates / virtualizes appropriately.**
- **The `facets/index.ts` barrel is intentional** — serves the dependency
  graph UI and is a valid opt-in anti-pattern target.
- **Per-file cost is approximately conserved across analyses.** Reducing one
  analysis's type-checker work tends to shift cost into another. Real gains
  require structural changes (file lifecycle, real parallelism) — see items
  2 and 5.
- **`maxConcurrentAnalyses=Infinity` was a footgun in a single V8 isolate** —
  same pattern as `historicalBatchConcurrency=4`. Both now bounded.
- **The TypeScript type checker has a per-file warmup cost** that any single
  analysis will eventually pay. Optimizing one analysis pushes work to the
  next. Item 5 (Option A) addresses this at the project level.
- **The desktop already runs 2-3 utility processes today** (vipr-analysis-worker
  for foreground, vipr-backfill-worker for backfill, vipr-remediation-analysis-worker
  on demand for AI). All load the same bundled `worker.js` via
  `utilityProcess.fork(workerPath)`. Option C is "scale the existing pattern,"
  not a green-field architecture change.
- **All SQLite writes are owned by the main process.** Utility processes
  open the DB only for the one-shot `repairHistoricalSnapshots` operation.
  Therefore N utility processes do NOT contend on the DB — the main
  process serializes all `saveAnalysisResult`, `upsertCommitMetrics`, and
  snapshot inserts. This invalidates the "DB contention" risk in Option C's
  original framing.
- **Item 7's bundling blocker does NOT apply to Option C.** `utilityProcess`
  spawns a fresh Node process that loads `worker.js` from disk; the bundle
  already includes all analyzer packages. No bundler config changes
  required. (Worker_threads needed `import('@vipr/core')` to resolve via
  node_modules — that path is what was broken in the packaged app.)
- **Item 2 verdict (2026-05-10): the incremental write path is already cheap.**
  Sub-phase instrumentation (`bc47853b`) showed combined per-commit write
  cost on the incremental path is ~28ms (~1% of 9.9s `commitAnalysisTime`
  mean). The dominant cost is structural — single-thread CPU floor on
  per-file analysis. Tactical write-path tuning is closed; item 7 is the
  next lever.
- **`createSnapshotForCommitInternal` (full snapshot path) is uninstrumented**
  and likely holds the historical "~30% unaccounted" overhead. Even a 50%
  reduction there would only shave ~5-8 min off a 97-min run, so it's not
  worth pursuing ahead of item 7. Revisit only if item 7 + item 5 both stall.

## How to execute an item

Each item doc is structured the same way:

- **Goal** — one sentence
- **Why now** — context + supporting baseline numbers
- **Prerequisites** — what must be true before starting
- **Implementation** — file paths, line numbers, exact changes
- **Acceptance criteria** — how you know it's done correctly
- **Test plan** — what to run, what should pass
- **Risk + rollback** — what could go wrong, how to recover
- **Out of scope** — what NOT to do in this item
- **Estimated effort** — a calibrated guess

When an item ships:
1. Mark it ✅ Done in the table above with the commit SHA
2. Move new follow-up work into a new item doc (don't overload finished docs)
3. Update the "Status as of <date>" lines below the table if needed

When in doubt about scope, check the "Out of scope" section before adding
work mid-stream.
