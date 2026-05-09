---
id: 15-core-shared-traditional-metrics
title: 'Core: Shared Traditional Metrics Cache'
phase: 15
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 15: Core Analyzer — Shared Traditional Metrics Cache

Eliminate 2 of 3 identical full-tree AST traversals per file by sharing the `calculateTraditionalMetrics` result across cyclomatic, Halstead, and maintainability analyses.

## Problem

Three core analyses call `calculateTraditionalMetrics(sourceFile, options)` with identical arguments:

| Analysis                       | File                                                            | Line |
| ------------------------------ | --------------------------------------------------------------- | ---- |
| `CyclomaticComplexityAnalysis` | `analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts` | ~48  |
| `HalsteadMetricsAnalysis`      | `analyzers/core/src/analyses/halstead-metrics-analysis.ts`      | ~45  |
| `MaintainabilityIndexAnalysis` | `analyzers/core/src/analyses/maintainability-index-analysis.ts` | ~100 |

All three use identical options: `{ countJsxAsOperators: true, includeOptionalChaining: true, includeNullishCoalescing: true }`.

Since these analyses run in parallel via `Promise.all` on the same `SourceFile` instance, the same full-tree `forEachDescendant` traversal executes 3 times. This is ~33% of the core plugin's per-file traversal cost.

## Fix

Create a `WeakMap`-based memoization cache in `@vipr/common` (not `@vipr/core`) so that the React plugin's `TechnicalDebtAnalysis` can also consume it in a later phase.

### Step 1: Create `packages/common/src/utils/traditional-metrics-cache.ts`

```typescript
import type { SourceFile } from 'ts-morph';
import type { TraditionalMetrics } from './traditional-metrics';
import { calculateTraditionalMetrics } from './traditional-metrics';

const STANDARD_OPTIONS = {
  countJsxAsOperators: true,
  includeOptionalChaining: true,
  includeNullishCoalescing: true,
} as const;

const cache = new WeakMap<SourceFile, TraditionalMetrics>();

/**
 * Returns traditional metrics (cyclomatic + Halstead) for a source file,
 * computing them at most once per SourceFile instance.
 *
 * Uses a WeakMap so entries are garbage-collected when the SourceFile is released.
 */
export function getTraditionalMetrics(sourceFile: SourceFile): TraditionalMetrics {
  const cached = cache.get(sourceFile);
  if (cached) return cached;
  const result = calculateTraditionalMetrics(sourceFile, STANDARD_OPTIONS);
  cache.set(sourceFile, result);
  return result;
}
```

### Step 2: Export from `@vipr/common`

Add the export to `@vipr/common`'s public API (check the existing export pattern — likely `packages/common/src/index.ts` or the package.json `exports` map).

### Step 3: Replace direct calls in all three analyses

In each of the three analysis files, replace:

```typescript
// BEFORE
import { calculateTraditionalMetrics } from '@vipr/common';
const metrics = calculateTraditionalMetrics(sourceFile, {
  countJsxAsOperators: true,
  includeOptionalChaining: true,
  includeNullishCoalescing: true,
});

// AFTER
import { getTraditionalMetrics } from '@vipr/common';
const metrics = getTraditionalMetrics(sourceFile);
```

## Files Modified

| File                                                            | Change                         |
| --------------------------------------------------------------- | ------------------------------ |
| `packages/common/src/utils/traditional-metrics-cache.ts`        | New file — WeakMap memoization |
| `packages/common/src/index.ts` (or exports)                     | Export `getTraditionalMetrics` |
| `analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts` | Use `getTraditionalMetrics`    |
| `analyzers/core/src/analyses/halstead-metrics-analysis.ts`      | Use `getTraditionalMetrics`    |
| `analyzers/core/src/analyses/maintainability-index-analysis.ts` | Use `getTraditionalMetrics`    |

## Why `@vipr/common` and not `@vipr/core`

The React analyzer's `TechnicalDebtAnalysis` (Phase 18) also needs this cache. `@vipr/react` depends on `@vipr/common` but not on `@vipr/core`. Placing the cache in `@vipr/common` makes it accessible to both.

## Verification

1. `pnpm --filter @vipr/common typecheck` — no type errors
2. `pnpm --filter @vipr/core test` — all tests pass (specifically `cyclomatic-complexity-analysis.test.ts`, `halstead-metrics-analysis.test.ts`, `maintainability-index-analysis.test.ts`)
3. Confirm metrics values are identical — the cache is transparent

## Risk

**Low.** `WeakMap` keyed on `SourceFile` prevents memory leaks. The same `SourceFile` object reference is used across all analyses for a given file (confirmed in engine). The options are hardcoded identically in all three call sites.
