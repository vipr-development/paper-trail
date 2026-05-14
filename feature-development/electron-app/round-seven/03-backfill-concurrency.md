# Phase 2 — Drop `historicalBatchConcurrency` to 1

> Cleanup, not perf. Validates a finding from baseline analysis. Throughput
> should be unchanged or slightly better.

## Goal

Set `historicalBatchConcurrency` from `4` to `1` in the desktop config
defaults. This stops the worker from running 4 backfill files concurrently
inside a single V8 isolate, which produced no real throughput gain but
inflated per-file timing 4× and likely degraded JIT optimization.

## Why now

Baseline backfill data showed:

- Per-file `analyzeMs` (worker-reported, single-file timing): mean **2,038ms**
- Per-file throughput (`batchTime ÷ batchSize`): mean **537ms**
- Foreground per-file `totalMs`: mean **548ms**

The 4× delta between worker-reported per-file time (2,038ms) and throughput
per-file time (537ms) is exactly the `historicalBatchConcurrency` value (4).
This confirms the concurrency setting causes 4 files to share a single CPU
event loop — they each appear 4× slower while running, but throughput is
unchanged because they're not actually parallel.

Why this is bad even though throughput is fine:

1. **V8 JIT optimization works per-function.** Running 4 concurrent calls
   spreads the optimizer's budget thinner. Single-flight likely warms JIT
   better and may give a small (5–10%) throughput improvement.
2. **Per-file metrics become honest.** With concurrency=1, the per-file
   `analyzeMs` reflects real cost, making backfill data directly comparable
   to foreground.
3. **Memory pressure reduces.** 4 concurrent ts-morph SourceFile parses
   means 4× the AST memory at peak. Sequential keeps it bounded.
4. **`historicalBatchSize: 32` still amortizes IPC overhead** (one IPC call
   per 32 files), so we're not losing the batching win — just the (illusory)
   concurrency.

## Prerequisites

- Phase 1a shipped (so backfill is no longer wasting time on excluded files)
- Optional: Phase 1b/c shipped (further reduces noise)
- Re-baseline captured between Phase 1a and this phase per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md)

## Implementation

### Step 1 — Update both config defaults

File: [`packages/common/src/client-constants.ts`](../../packages/common/src/client-constants.ts) (around line 361 and 401)

Two locations (dev + prod):

```ts
// Around line 361 (dev block)
worker: {
  // ...
  historicalBatchConcurrency: 4,  // ← change to 1
  // ...
},

// Around line 401 (prod block)
worker: {
  // ...
  historicalBatchConcurrency: 4,  // ← change to 1
  // ...
},
```

Change both to `1`.

### Step 2 — Verify the env override path still works

File: [`clients/desktop/src/shared/config/config.ts`](../../clients/desktop/src/shared/config/config.ts) (around lines 255 and 361)

The override pattern:

```ts
historicalBatchConcurrency: getEnvNumber(
  'WORKER_HISTORICAL_BATCH_CONCURRENCY',
  DESKTOP_CONFIG_DEFAULTS.dev.worker.historicalBatchConcurrency
),
```

Confirm the env var name. Document in commit message that users can
re-enable concurrency via `WORKER_HISTORICAL_BATCH_CONCURRENCY=4` if they
have a specific reason to.

### Step 3 — Update relevant tests

If `client-constants.test.ts` or any desktop config test asserts the default
value of `historicalBatchConcurrency`, update those assertions to `1`.

```bash
grep -rn "historicalBatchConcurrency" clients/desktop/src/ packages/common/src/
```

### Step 4 — Comment the rationale near the constant

Add a brief comment so future readers don't innocently bump it back:

```ts
/**
 * Concurrency limit for analyzeContentBatch in the worker.
 *
 * Set to 1: ts-morph analyses are CPU-bound JS in a single V8 isolate.
 * Higher values produce no real parallelism (just cooperative interleaving)
 * and historically inflated per-file timing 4× without improving throughput.
 * Override with WORKER_HISTORICAL_BATCH_CONCURRENCY env var if needed.
 *
 * Batching IPC is preserved via historicalBatchSize (32 files per round-trip).
 */
historicalBatchConcurrency: 1,
```

## Acceptance criteria

- Both `dev` and `prod` defaults set to `1`
- Existing tests pass (or are updated to expect `1`)
- New backfill run shows: per-file `analyzeMs` mean drops to ~500ms (matching foreground)
- Wall clock for a comparable backfill is **same or slightly faster**
- `pnpm --filter @vipr/common typecheck && pnpm --filter @vipr/desktop typecheck` clean
- `pnpm --filter @vipr/desktop test -- run src/main/analysis/` clean

## Test plan

```bash
pnpm --filter @vipr/common typecheck
pnpm --filter @vipr/common test
pnpm --filter @vipr/desktop typecheck
pnpm --filter @vipr/desktop test -- run src/main/analysis/

# Re-baseline (see 04-rebaseline-protocol.md):
# 1. Quit the desktop app
# 2. rm "$HOME/Library/Application Support/Vipr Desktop/vipr-default.db"
# 3. Open clients/desktop in the app, wait for foreground + backfill to complete
# 4. Capture both Performance summary log lines
# 5. Compare per-file analyzeMs and total wall clock vs the previous backfill baseline
```

## Risk + rollback

- **Risk**: very low. Worst case throughput regresses by a measurable but
  small amount (e.g., -5% from lost cooperative interleaving). Easy to revert.
- **Rollback**: change the constant back to 4.
- **Watch out for**: tests that assume concurrent execution (e.g., timing
  assertions). Search for `historicalBatchConcurrency` test references.

## Out of scope

- **Do NOT** change `historicalBatchSize` (still 32) — that's the right batch
  size for IPC amortization
- **Do NOT** change `coordinator.maxConcurrent` — that's a separate question
  (Phase 4 considers bumping it)
- **Do NOT** add new instrumentation — Phase 0.5 already captures what we need
- **Do NOT** restructure the worker — `analyzeContentBatch` stays as-is, just
  with concurrency=1

## Estimated effort

15–30 minutes for the change + tests. Plus ~95 minutes for the re-baseline
backfill run. Single commit.

## Commit message template

```
chore(common): set historicalBatchConcurrency default to 1

Baseline data showed 4× concurrent execution inside a single V8 isolate
produced no real throughput gain (mean batchTime ÷ batchSize ≈ foreground
per-file mean) but inflated per-file analyzeMs from ~537ms (throughput) to
~2038ms (worker-reported). It also likely diluted JIT optimization budget
across 4 concurrent calls.

batchSize stays at 32 to keep IPC amortization.

Override with WORKER_HISTORICAL_BATCH_CONCURRENCY=N for users who need to
test other values.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```
