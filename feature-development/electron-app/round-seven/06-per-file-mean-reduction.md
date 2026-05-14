# Phase 4 — Per-File Mean Reduction (The Big Lever)

> **The only phase that can move foreground wall clock by minutes rather than
> seconds.** Everything else is housekeeping. This one requires a research
> sub-step before implementation.
>
> **Status (2026-05-09)**:
> - Step 1 (research instrumentation): ✅ shipped in commit `03804da3`
> - Step 2 (implementation): 🔬 GATED on baseline data per the plan's own
>   warning. Re-baseline first, then return to this doc with data in hand.

## Goal

Drive foreground per-file mean from **548ms** to a substantially lower number
(target: <300ms, a ~45% reduction). At current concurrency, this would take
foreground wall clock from 4:38 to under 2:30 on the baseline repo.

## Why now (and the math)

Baseline foreground:
- Wall clock: 277,728ms
- Files: 1013
- Mean per-file: 548ms
- Peak in-flight: 2

`(548ms × 1013 files) ÷ 2 in-flight = 277,562ms ≈ 277,728ms actual`

The math matches to the millisecond. **The system is serial-blocked on
per-file CPU work, two at a time.** Three levers can change this:

1. **Lower mean per-file time** — affects every file, compounds
2. **Raise peak in-flight** — affects total throughput proportionally
3. **Skip more files** — affects file count

Phases 1a, 1b/c, and 3 chip away at lever 3 (some seconds saved). Lever 2
(`coordinator.maxConcurrent` from 2 to 4 or higher) is a single-line change
worth testing as part of this phase. **But lever 1 is the dominant variable
and the only path to "minutes saved."**

Inside the worker, per-file cost is dominated by:
- ts-morph SourceFile creation (parsing + adding to `Project`)
- ~9 TS analyses + ~7 React analyses per file, each performing its own AST
  traversal
- Type checker calls (`getType()`, `getSymbol()`) for files with deep import
  graphs
- SourceFile teardown (removing from `Project`)

The 548ms is split across 16+ analyses. We don't currently know which ones
dominate. **That's the research step.**

## Prerequisites

- Phase 0 + 0.5 instrumentation in place (✅ already shipped)
- Phase 1a shipped (so backfill measurements are honest)
- Re-baselined after Phase 1a + Phase 2

## Step 1 — Research: per-analysis timing

Goal: produce a breakdown of where the 548ms goes inside one file, across all
~16 analyses. Without this, any optimization is a shot in the dark.

### Approach options (pick one)

**Option A: Use existing `@vipr/tracing`** (preferred — leverages existing
infrastructure)

The engine already creates trace spans per analysis at
[`packages/engine/src/analysis-engine.ts:683`](../../packages/engine/src/analysis-engine.ts) (`executeAnalysis`).
Confirm these spans capture duration. If yes, enable tracing for one diagnostic
run, dump the per-analysis timings, and aggregate.

Steps:
1. Read `executeAnalysis` and surrounding code in `analysis-engine.ts`
2. Find where trace data is written (likely a `FileSink` somewhere in the
   utility worker setup)
3. Enable tracing for a foreground run (configure via UI? env var?
   `tracing-session-service.ts`?)
4. Open the test repo, let it analyze
5. Read the trace output, aggregate by analysis ID, compute mean/p95/p99 per
   analysis

**Option B: Add per-analysis aggregation to PerformanceProfile**

Extend the profile module to record per-analysis durations alongside per-file
ones. Same pattern as backfill per-file timings (worker reports, scheduler
records). Use existing trace spans as the source.

Steps:
1. Add `AnalysisTimingSample { analysisId: string, durationMs: number, filePath: string }` to performance-profile.ts
2. Add `recordAnalysisTiming` method
3. Wire from `executeAnalysis` in the engine to the profile (via existing trace, or directly)
4. Surface in the summary as: `analysisTime: { 'core-cyclomatic': PercentileStats, ... }` and `slowestAnalysisInstances: [...]`
5. Add tests
6. Re-baseline to capture per-analysis breakdown

Option A is faster if tracing already captures what we need. Option B is more
durable for ongoing measurement. **Read the tracing code first; choose based
on what's there.**

### Output of Step 1

A document at `.claude/plans/baselines/<date>-per-analysis-breakdown.md` with:

- Top 5–10 most expensive analyses by mean duration
- Top 5–10 most expensive analyses by total time spent (mean × invocations)
- Per-analysis p95/p99 to identify outlier-prone analyses
- Recommendation: which analyses to investigate first for optimization

This becomes the input to Step 2.

## Step 2 — Implementation: targeted optimizations

Driven by Step 1 data. Likely candidates (do not commit to these without data):

### Hypothesis A — Shared AST traversal across analyses

If many analyses each do their own full-tree walk, build an AST visitor index
once per file and reuse across analyses. The engine's `buildASTIndex` at
[`analysis-engine.ts:1128`](../../packages/engine/src/analysis-engine.ts) may
already do part of this; check what it covers.

Estimated impact: 30–50% per-file reduction if analyses are traversal-bound.

### Hypothesis B — Type checker call reduction

If certain analyses make many `getType()` / `getSymbol()` calls, those force
the TS compiler to resolve symbols across the import graph — expensive for
files like `FileDetail.tsx` that import 50+ things. Look for patterns like:
- Repeated `getType()` calls on the same node within one analysis
- Type-checker calls that could be batched or cached
- Analyses that need only AST shape, not type info — confirm they don't
  accidentally trigger type resolution

Estimated impact: dramatic for type-heavy files (could turn 12s outliers into
3s); modest overall.

### Hypothesis C — Bump `coordinator.maxConcurrent` from 2 to 4

Single-line change in `client-constants.ts`. Independent of the other work.
**Caveat**: each in-flight request goes to the same utility process and
shares the same V8 isolate. So 4 in-flight may behave like Phase 2's 4-way
concurrency problem (no real parallelism, just interleaving with worse JIT).

Test it with measurement, don't ship it on hypothesis. Compare wall clock at
maxConcurrent=2 vs 4 on the same warm cache.

If it doesn't help: stay at 2 and look at process-level parallelism (run
the analysis worker as 2 separate utility processes, each handling 2
in-flight). That's a much larger change.

### Hypothesis D — Project / SourceFile lifecycle

Each `analyzeFile` adds a SourceFile to the shared `Project`, runs analyses,
and removes it ([`analysis-engine.ts:516`](../../packages/engine/src/analysis-engine.ts)).
The teardown may force re-resolution of imports for the next file. Investigate
whether SourceFile reuse (or batched lifecycle) helps.

Estimated impact: unknown — needs measurement.

## Step 3 — Validation

Re-baseline per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md).

Critical metrics:

- **Foreground per-file `totalMs` mean**: must drop. Target <300ms.
- **Foreground wall clock**: must drop proportionally.
- **Backfill per-file `analyzeMs` mean** (with concurrency=1 from Phase 2): same files run through the same analyses, so should drop similarly.
- **`slowestFiles` list**: outliers should also drop. If outliers stay high
  but mean drops, you've optimized the easy cases but missed the hard ones.

Repeat measurement to confirm variance is acceptable (run twice, take median).

## Acceptance criteria

- Per-analysis breakdown document exists in `.claude/plans/baselines/`
- Foreground per-file `totalMs` mean is measurably lower than baseline (548ms)
  - Stretch: <300ms
  - Minimum acceptable: <450ms (~18% reduction, ≈45 seconds saved on baseline repo)
- No regression in correctness (existing analyzer tests pass)
- No regression in backfill (per-file timing also drops or stays same)
- All TS/React analyzer tests pass

## Test plan

```bash
# Per-analyzer
pnpm --filter @vipr/core test
pnpm --filter @vipr/react test
pnpm --filter @vipr/typescript test
pnpm --filter @vipr/nextjs test

# Engine
pnpm --filter @vipr/engine test

# Desktop integration
pnpm --filter @vipr/desktop typecheck
pnpm --filter @vipr/desktop test -- run src/main/analysis/

# Re-baseline (full procedure)
# See 04-rebaseline-protocol.md
```

## Risk + rollback

- **Highest-risk phase.** Touches the analysis pipeline. Bugs here can
  silently produce wrong analysis results — much worse than slow analysis.
- **Mitigation 1**: small commits. One hypothesis at a time. Each ships
  independently.
- **Mitigation 2**: differential testing. After each commit, compare
  `slowestFiles` lists and any analyses' output against baseline. If a file's
  reported metrics changed, that's a regression.
- **Mitigation 3**: the analyzer test suites should catch most semantic
  regressions. Trust them, but run differential analysis as a backup.
- **Rollback**: revert per-commit. Each hypothesis is independent.

## Out of scope

- **Do NOT** rewrite the engine. Targeted optimizations only.
- **Do NOT** introduce parallelism inside the worker via worker_threads. The
  engine has a `parallelism` config but it's correctly disabled for desktop;
  don't re-enable it without a separate planning effort.
- **Do NOT** change presenter or registry behavior. Those are downstream.
- **Do NOT** add per-plugin / per-analysis enablement here — that's Phase 3c.
- **Do NOT** optimize for one repo's specifics. Verify any change improves
  things on a second repo too (e.g., a Next.js project) before shipping.

## Estimated effort

- Step 1 (research): 1–2 days
- Step 2 (implementation, all hypotheses tried): 1–2 weeks of focused work
- Step 3 (validation): hours per iteration, but iterations may be many

This is the longest phase by far. Plan accordingly.

## Sequencing

This phase can run in parallel with Phase 3 (config productization) since
they touch different layers. **Phase 4 should not be blocked by Phase 3**,
even if Phase 3 is still landing.

## Dependencies on other phases

- **Phase 0 + 0.5**: required (PerformanceProfile + per-file attribution)
- **Phase 1a**: required (so baseline measurements are honest)
- **Phase 2**: recommended (so backfill per-file timings are honest)
- **Phases 1b/c, 3**: independent — Phase 4 work doesn't depend on them
