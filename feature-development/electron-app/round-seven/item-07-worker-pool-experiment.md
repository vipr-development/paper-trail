# Item 7 — Wire the Engine's Existing Worker-Thread Pool

> Strong-ROI incremental experiment to unlock real CPU parallelism via the
> AnalysisWorkerPool that already exists (but is disabled) in
> `packages/engine/src/worker-pool.ts`. Staged conduct → analyze →
> validate → proceed-or-stop cadence per the user's standing development
> philosophy.

## Goal

Enable real multi-core CPU parallelism for analysis — first for backfill
(highest-value target), then for foreground bulk runs — by activating the
engine's existing `AnalysisWorkerPool` (currently gated off via
`parallelism.enabled: false` in the desktop's worker config).

If this lands, expected wall-clock impact: **40-60% backfill reduction**
and 30-50% foreground initial-scan reduction on multi-core machines.

If the existing pool turns out to be insufficient (hits DB contention,
crash recovery limits, GC pressure), this item formally validates that
**Option C in item 5** (multi-process pool, much larger architecture work)
is the next step.

## Sequencing note

This item is queued AFTER item 2 (backfill wall-clock improvement) wraps
its current measurement-driven cycle. The item-02 baseline data may
expose a different highest-priority target (e.g., a specific sub-phase
that's surprisingly costly), in which case **that finding takes priority
over starting this item**. Re-evaluate after item 2 closes.

## Why now

Two observations from earlier perf work converge here:

1. **Per-file CPU is conserved across analyses.** We've optimized as
   much as we can without structural change. The only paths to multi-
   minute wall-clock reduction are: file lifecycle reuse (item 5 /
   Option A) OR real parallelism (this item, then item 5 / Option C).

2. **The engine already has worker_thread infrastructure.** `AnalysisWorkerPool`
   in `packages/engine/src/worker-pool.ts` was built for exactly this
   case but the desktop disables it. Investigation cost is low; if it
   works in our bundled environment, we get parallelism nearly for free.

The CLAUDE.md for `@vipr/engine` warns: *"Worker pool resolves its default
entry from the current module directory in both ESM and bundled CommonJS
contexts; bundled clients with relocated worker files still need to pass
workerPath."* So the path resolution may be the gotcha worth investigating
before any wiring work.

## Prerequisites

- **Item 2 (backfill wall-clock) closed.** Either with wins shipped or
  with a documented "no further tactical wins available" decision.
- A clean baseline summary captured AFTER item 2 closes, so this item's
  before/after measurements have a stable reference.
- Familiarity with:
  - [`packages/engine/src/worker-pool.ts`](../../packages/engine/src/worker-pool.ts) — the dormant pool
  - [`packages/engine/src/analysis-worker.ts`](../../packages/engine/src/analysis-worker.ts) — the worker entry point
  - [`packages/engine/src/analysis-engine.ts`](../../packages/engine/src/analysis-engine.ts) (line ~539, `analyzeFilesParallel`) — pool entry condition
  - [`clients/desktop/src/utility/worker.ts`](../../clients/desktop/src/utility/worker.ts) — the desktop's engine config (where `parallelism: { enabled: false }` is set)

## The conduct → analyze → validate → proceed cadence

Each stage follows this pattern. Do not skip stages. Back off to the
previous known-good stage at the first sign of regression or instability.

### Stage 0 — Verify the dormant pool works at all

**Conduct**:
- In a throwaway dev branch, set `parallelism.enabled: true` and provide
  `pluginPackages: [...]` to the engine config in worker.ts
- Set `minFilesForParallelism: 1` so the pool engages even on small batches
- Replace one call site that currently uses `engine.analyzeFile(path)` with
  `engine.analyzeFiles([path])` (single-element array)
- `pnpm --filter @vipr/desktop build && dev` and exercise that path

**Analyze**:
- Did the worker pool spawn worker_threads?
- Did the analysis result match what `analyzeFile` would have returned?
- Any errors related to plugin loading inside workers? (The CLAUDE.md
  gotcha about `pluginPackages` being required is the highest-risk failure
  mode.)
- Any errors related to bundled `workerPath` resolution? (The pool's
  default workerPath uses `import.meta.url` which Vite handles
  differently than tsc.)

**Validate**:
- Run a small batch (e.g., 5 files) through `analyzeFiles([5 files])`
- Compare output deeply against the same files run through `analyzeFile`
  one at a time
- Differential test: every plugin result, every insight, every score
  must match exactly

**Proceed-or-stop**:
- ✅ Output matches → proceed to Stage 1
- ❌ Pool fails to spawn / wrong path / plugin load error → write up
  what's broken in this doc, decide whether to fix the bundled-worker
  path issue or escalate to item 5 / Option C

### Stage 1 — Workers for backfill only

**Conduct**:
- In `clients/desktop/src/utility/worker.ts`, build a separate engine
  configuration for backfill that has `parallelism.enabled: true`
- Modify `analyzeContentBatch` in the worker handler to use
  `engine.analyzeFiles([...])` for the batch instead of looping
  `engine.analyzeContent(file, content)` per item
- Determine pool size: start with `Math.min(cpus().length - 1, 4)` —
  conservative cap to limit memory pressure
- Backfill is already in its own utility process, so worker_thread
  spawning happens within that isolated process — minimal blast radius

**Analyze**:
- Run a 10-commit backfill with the new code
- Inspect: per-batch IPC time, per-file timings, batch size distribution
- Check for: warnings about plugin re-loading, GC pressure (heap usage
  in the utility process), DB write errors
- Verify: snapshot data (snapshot_files, commit_file_changes,
  file_versions) matches what serial backfill would have produced for
  the same commits

**Validate**:
- Run the full 90-day backfill on the baseline repo
- Compare to the prior baseline:
  - **Wall clock**: target 40-60% reduction (e.g. 96 min → 40-58 min)
  - **commitAnalysisTime mean**: should drop proportionally to pool size
  - **commitDbPersistTime**: watch for INCREASE (writer lock contention)
  - **No new errors in `failedCommits` count**
  - **No memory blow-up** (Activity Monitor: utility process RSS should
    stay under 4GB even with 4 workers)
- Differential test: spot-check 10 commits' snapshot_file rows against a
  serial-backfill run of the same commits. They must match.

**Proceed-or-stop**:
- ✅ 40%+ wall clock reduction, no regressions, no DB contention errors → proceed to Stage 2
- ⚠️ <20% wall clock reduction → STOP. Document why (DB contention?
  GC?). Item 7 hit its ceiling; item 5 / Option C is the next step.
- ❌ Any data corruption / regression → revert immediately. Open a bug.

### Stage 2 — Workers for foreground bulk analysis

**Conduct**:
- Foreground typically processes files one at a time (interactive),
  but on workspace open it processes 1000+ files all at once
- Add a queue-depth heuristic to the foreground coordinator:
  - If pending queue size ≥ threshold (e.g. 10 files), route through
    the worker pool via `engine.analyzeFiles([...])`
  - Below threshold, stay with the existing per-file `engine.analyzeFile()`
    for low-latency response on incremental changes
- Pool size: same conservative `Math.min(cpus().length - 1, 4)`
- Reuse the worker pool across batches (don't spawn-and-tear-down)

**Analyze**:
- Run the full foreground baseline (1008 files) with the new code
- Inspect: file dispatch order, batch sizes hit the worker pool
- Verify: incremental edits (1-3 file changes) still go through the
  fast single-file path; no cold-start tax on those
- Check that the dual-path (single-file vs pool) doesn't introduce
  ordering bugs

**Validate**:
- Wall clock target: 30-50% reduction on initial scan (4:42 → 2:30-3:20)
- p95 / p99 file time should improve
- Incremental edit perf should be unchanged
- All foreground test paths pass
- Differential test on the baseline repo: every analysis result, every
  insight, every score matches a serial-foreground run

**Proceed-or-stop**:
- ✅ 30%+ wall clock reduction, no regressions, incremental edits unaffected → done with item 7
- ⚠️ Foreground pool has different ceiling than backfill pool → document
  why; it's still useful even if the lift is modest
- ❌ Regression in incremental editing latency → revert Stage 2 only
  (Stage 1 backfill changes can stand)

### When to back off

Concrete triggers for backing off to the previous stage:

- Test failures introduced (any new red test in a previously-green suite)
- Memory growth that exceeds 1.5× the pre-stage RSS
- DB write errors (`SQLITE_BUSY`, `SQLITE_LOCKED`) that didn't occur
  before
- Differential testing finds even one mismatched analysis result
- Wall-clock improvement < 20% (the experiment's overhead exceeded its
  benefit)

When backing off, document in this file what triggered the back-off and
what next step was chosen (escalate to Option C, fix the underlying
issue and retry, or accept the lower stage as the new equilibrium).

## Acceptance criteria (for closing item 7 successfully)

- Stage 0 validation succeeds (dormant pool works in bundled context)
- Stage 1 ships: backfill uses worker pool, 40%+ wall clock reduction,
  no data integrity regressions, all 4045+ desktop tests pass
- Stage 2 ships: foreground bulk uses worker pool, 30%+ initial-scan
  improvement, no incremental-edit latency regression
- Re-baselined per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md)
  at each stage; baselines committed to `baselines/` for diffing
- This doc updated with actual numbers achieved at each stage

If item 7 stalls at Stage 0 or Stage 1 due to architectural limits
(DB contention, plugin loading, bundle path issues that can't be
trivially fixed), close item 7 with a written assessment and **escalate
to item 5 / Option C** (full multi-process pool — bigger architecture
change, weeks of work).

## Test plan

```bash
# Unit / integration tests at each stage
pnpm --filter @vipr/engine test  # workers tests live here
pnpm --filter @vipr/desktop test -- run src/main/analysis/
pnpm --filter @vipr/desktop test -- run src/utility/

# Full desktop suite (must stay 4045/4045 green)
pnpm --filter @vipr/desktop test

# Full backfill baseline (Stage 1 validation)
# See 04-rebaseline-protocol.md
# ~95 minutes today; should be ~30-50 min after Stage 1

# Differential testing (Stage 1 + Stage 2)
# Run the baseline repo through serial analysis, capture all per-file
# overallScore + insights as a fixture
# Run again through worker-pool analysis, diff
# Any mismatch is a stop signal
```

## Risk + rollback

- **Risk**: medium-high. Real parallelism in a system that's been
  single-threaded for its entire history surfaces latent assumptions.
- **Mitigation 1**: stage-by-stage with explicit back-off triggers.
- **Mitigation 2**: differential testing is non-negotiable. Don't ship
  any stage without it.
- **Mitigation 3**: gate behind a config flag (e.g.,
  `worker.pool.enabled: true` in DESKTOP_CONFIG_DEFAULTS) so users on
  resource-constrained machines can opt out.
- **Rollback**: each stage is its own commit. Revert the offending
  stage; previous stages can stand independently.

## Out of scope

- **Multi-process foreground/backfill** — that's item 5 / Option C, only
  pursued if this item hits its ceiling.
- **File-lifecycle reuse** (Option A) — independent strategic work,
  separately tracked.
- **Settings UI for tuning pool size** — power users can use an env var
  for now; UI work is its own item.
- **Replacing analyzeFile with analyzeFiles wholesale** — keep the
  single-file path for low-latency incremental work. Dual-path is
  intentional.

## Estimated effort

- Stage 0 (validate pool works at all): 1-3 days
- Stage 1 (workers for backfill): 1 week
- Stage 2 (workers for foreground bulk): 1-2 weeks
- Total: 2-4 weeks across multiple sessions, with re-baselining between
  each stage (95 min per backfill baseline run)

Item 7 is the highest-ROI parallelism work available — but only if Stage 0
shows the existing infrastructure is usable. The real cost will be known
after the first 1-3 days.

---

## Stage 0 read-only audit (2026-05-10)

Before writing any spike code, the engine + desktop sources were read
end-to-end to map invariants and risks. **Three architectural blockers
surfaced — all of them on the bundling/distribution side, not the pool
logic itself.** Documenting here so that no spike work is wasted.

### Blocker 1 — Worker entry only supports `analyzeFile(filePath)`, not `analyzeContent(filePath, content)`

`packages/engine/src/analysis-worker.ts` (the worker's message handler)
calls `engine.analyzeFile(filePath)` — which reads the file from disk via
the worker's own ts-morph `Project`. There is no `analyzeContent` branch.

**Why it matters for backfill (the Stage 1 target):** backfill never has
the file on disk at the historical commit's content. It git-fetches each
file's content and feeds it through `engine.analyzeContent(path, content)`
in `analyzeContentBatch` (`clients/desktop/src/utility/analyze-content-batch.ts`,
line 47). The dormant pool **cannot service backfill as written** — the
worker would read whatever content currently exists on disk (or fail with
ENOENT for files that have been deleted in HEAD).

**To unblock**: extend the worker protocol with a second message type
(`{ type: 'analyzeContent', filePath, content }`), serialize content
through `workerData`/`postMessage`, and add a parallel
`AnalysisWorkerPool.analyzeContents(items)` API. Non-trivial because
content payloads can be hundreds of KB per file × hundreds of files in
flight; serialization cost may erode the parallelism win.

**Foreground (Stage 2 target) is unaffected** — foreground uses
`analyzeFile(filePath)` against the live working tree, which the existing
worker protocol supports natively. So the natural Stage 1 target should
flip to **foreground bulk** rather than backfill, contrary to the plan as
originally written.

### Blocker 2 — Engine call site is `analyzeFile`, not `analyzeFiles`

`AnalysisEngine.analyzeFiles([...])` is the only entry that engages the
worker pool (line 546 in `analysis-engine.ts`, gated on
`parallelism.enabled` AND `filePaths.length >= minFilesForParallelism`).
The desktop today calls **`engine.analyzeFile(path)` (singular)** for
every foreground request (worker.ts line 167) — these never hit
`analyzeFiles()` regardless of pool config.

**To unblock**: foreground per-file requests would need to be batched at
the IPC layer (collect N pending `'analyze'` requests within a short
window, dispatch as a single `analyzeFiles([...])` call) — a
non-trivial rework of `coordinator.ts`'s request lifecycle.
Alternatively, add a new IPC message `'analyzeBulk'` that accepts an
array of paths and routes through the pool.

### Blocker 3 — Bundled utility process emits a single `worker.js`; no separate `analysis-worker.js` artifact, no externalized analyzer packages

`vite.utility.config.ts`:

```ts
build: {
  lib: {
    entry: resolve(__dirname, 'src/utility/worker.ts'),
    formats: ['cjs'],
    fileName: () => 'worker.js',
  },
  rollupOptions: {
    external: ['electron', 'better-sqlite3', 'node-llama-cpp', 'node:sqlite', ...builtinModules, ...],
  },
}
```

Two consequences:

1. **No `analysis-worker.js` artifact exists** in the bundled output.
   `getDefaultWorkerPath()` in `worker-pool.ts` resolves
   `<utility-dir>/analysis-worker.js` — that file does not exist.
   `workerPath` MUST be passed explicitly, AND the bundler must be
   reconfigured to emit `analysis-worker.js` as a second `lib.entry`.
2. **Analyzer packages are not external.** `@vipr/engine`, `@vipr/core`,
   `@vipr/react`, `@vipr/nextjs`, `@vipr/typescript`, `@vipr/linting`
   are all rolled INTO `worker.js`. The worker's
   `import('@vipr/core')` (analysis-worker.ts line 82) cannot resolve
   inside `worker_threads` of a packaged app — there's no `node_modules`
   entry for those packages in the Forge-packaged tree.

**To unblock**: either (a) externalize all analyzer packages from the
utility bundle and bundle them as ESM/CJS files alongside `worker.js`
with a custom resolver in `LOCAL_PLUGIN_OVERRIDES`, OR (b) pre-load
plugins in the main worker thread, serialize their plugin metadata, and
have the worker_thread reuse a shared registry — which defeats the
purpose, since each worker thread needs its own analyzer state.

This is by far the heaviest lift of the three blockers. It overlaps
substantially with the work item 5 / Option C would require for
multi-process distribution.

### Verdict — STOP and re-decide

Per the conduct → analyze → validate → proceed-or-stop pattern, the
read-only audit is the **conduct + analyze** of Stage 0. The
**validate** step is "is the dormant pool usable in our bundled
environment without major reworks?" The honest answer is **no**:

- ❌ Pool's worker protocol is incompatible with backfill (Stage 1's
  intended target)
- ❌ Foreground per-file calls don't engage the pool path; batching would
  be required
- ❌ Forge bundler doesn't emit a separate worker entry, and the analyzer
  package distribution is incompatible with `worker_threads` `import()`

These aren't fix-it-in-an-afternoon issues. Each maps to days-to-weeks of
work, and #3 specifically is **architecturally similar** to what
item 5 / Option C (utility-process-based parallelism) would require —
because Electron's `utilityProcess` API uses similar bundling/loading
constraints to `worker_threads`, AND a process-level pool has the
additional benefit of bypassing the single-V8-isolate ceiling entirely.

### Recommended sequencing change

Item 7 in its current form ("wire the existing pool") is no longer the
cheapest experiment. The architectural prerequisites overlap so heavily
with item 5 / Option C that **the cost-to-value comparison has flipped**:

- Original calculus: item 7 = days, item 5 = weeks; do item 7 first
- Revised calculus: item 7 = weeks (because of the three blockers
  above), item 5 = weeks; both require similar infrastructure changes,
  but item 5 yields true multi-core parallelism (no single-V8-isolate
  ceiling) while item 7 yields worker_thread parallelism (still bounded
  by V8 isolate constraints in the parent process)

**Proposal**: close item 7 with the audit above as its written deliverable
and promote **item 5 / Option C** to active. Two reasons:

1. The bundling work (separate worker/process entry, externalized
   analyzer packages, plugin loading from a known path) is required for
   Option C anyway.
2. Option C's process-level parallelism dodges the single-isolate
   ceiling that bounds even successful worker_thread implementations.

The strict win here is that **the audit converted a 2-4 week
"experiment" into a 1-day deliverable**, exactly because we did the
read-only conduct → analyze step before writing code. No stage was
re-run, no backfill baseline was wasted, and the next item (5 / Option
C) inherits the architectural understanding cleanly.

If item 5 / Option C ALSO surfaces architectural blockers when it
reaches its own Stage 0 audit, then the answer is: file-lifecycle reuse
(Option A) becomes the only remaining lever — that's already documented
as item 5's twin track.
