---
id: 18-react-technical-debt-dedup
title: 'React: Eliminate TechnicalDebtAnalysis Halstead Reimplementation'
phase: 18
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: [15]
---

# Phase 18: React Analyzer — Eliminate TechnicalDebtAnalysis Halstead Reimplementation

Replace React's private Halstead metric reimplementation with the shared `getTraditionalMetrics` cache from Phase 15, and remove the uncached `calculateCyclomaticComplexity` call.

## Problem

In `analyzers/react/src/analyses/technical-debt-analysis.ts`:

1. **Line ~134**: `calculateCyclomaticComplexity(sourceFile)` — calls the `@vipr/common` function directly without using the ASTIndex and without caching. If the core plugin also analyzes this file, this is a redundant traversal.

2. **Lines ~1163-1266**: `private calculateHalsteadMetrics(sourceFile, astIndex)` — a complete reimplementation of Halstead metric computation. This duplicates what `calculateTraditionalMetrics` in `@vipr/common` already provides.

When core and React plugins both analyze the same `.tsx` file, Halstead is computed 3 times: once by core's cyclomatic analysis, once by core's Halstead analysis, and once by React's technical debt analysis.

## Fix

Replace both with the shared `getTraditionalMetrics` from `@vipr/common` (created in Phase 15).

### Step 1: Replace the two calls

```typescript
// BEFORE (~lines 133-134):
const halstead = this.calculateHalsteadMetrics(sourceFile, astIndex);
const cyclomaticComplexity = calculateCyclomaticComplexity(sourceFile);

// AFTER:
import { getTraditionalMetrics } from '@vipr/common';
const traditionalMetrics = getTraditionalMetrics(sourceFile);
const halstead = traditionalMetrics.halstead;
const cyclomaticComplexity = traditionalMetrics.cyclomaticComplexity;
```

### Step 2: Delete the private method

Remove the entire `private calculateHalsteadMetrics()` method (~lines 1163-1266, approximately 100-150 lines). Remove any associated private helper methods that are only used by this method.

### Step 3: Verify property shape compatibility

The private implementation returns a Halstead object. Compare its shape with what `calculateTraditionalMetrics` returns in the `halstead` field. They must have the same properties used by `TechnicalDebtAnalysis`:

- `volume`
- `difficulty`
- `effort`
- `bugs`
- `vocabulary`
- `length`

Read both implementations to confirm the return shapes match. If the private implementation returns additional properties not in the shared version, check whether `TechnicalDebtAnalysis` actually uses them.

## Pre-implementation Validation

**Before deleting anything**, run both implementations against the same test fixture and compare outputs. The two Halstead implementations were coded independently and may produce slightly different numbers due to:

- Different operator/operand classification
- Different handling of JSX operators
- Different optional chaining / nullish coalescing treatment

If values diverge, document the differences and adjust the shared implementation's options or accept the shared implementation's values (since it's the canonical implementation).

## Files Modified

| File                                                      | Change                                                                                      |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `analyzers/react/src/analyses/technical-debt-analysis.ts` | Replace private Halstead + cyclomatic calls with `getTraditionalMetrics`, delete ~150 lines |

## Dependencies

- **Phase 15 must land first** — `getTraditionalMetrics` must be exported from `@vipr/common`

## Verification

1. `pnpm --filter @vipr/react test` — specifically `technical-debt-analysis.test.ts`
2. `pnpm --filter @vipr/react typecheck`
3. If test values change slightly due to implementation differences, update expected values with a comment explaining why

## Risk

**Medium.** The two Halstead implementations may produce slightly different numbers. Run characterization tests before and after to quantify any difference.
