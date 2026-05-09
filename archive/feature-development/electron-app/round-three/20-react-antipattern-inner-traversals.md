---
id: 20-react-antipattern-inner-traversals
title: 'React: Remove Inner forEachDescendant in Anti-Pattern Analysis'
phase: 20
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 20: React Analyzer — Remove Inner `forEachDescendant` in Anti-Pattern Analysis

Eliminate full-file traversals that fire inside the "optimized" ASTIndex code path of `detectAntiPatternsOptimized`.

## Problem

In `analyzers/react/src/analyses/anti-pattern-analysis.ts`, the optimized path (`detectAntiPatternsOptimized`) correctly sources top-level nodes from the ASTIndex but then calls `sourceFile.forEachDescendant` in at least 3 inner locations (~lines 1076, 1175, 1696). These are sub-analyses that fire inside the already-optimized path, defeating the purpose of the ASTIndex.

## Investigation Required

The implementing agent must:

1. Run `grep -n "forEachDescendant" analyzers/react/src/analyses/anti-pattern-analysis.ts` to find all call sites
2. For each call site, determine:
   - **Is it on `sourceFile`?** (full-file traversal) — these should be replaced with ASTIndex lookups
   - **Is it on a specific sub-node?** (e.g., a function body, a JSX element) — these are scoped and acceptable; leave them
3. Also check calls to `react-helpers.ts` functions from within the optimized path — some helper functions internally call `forEachDescendant` on the source file

## Fix Pattern

For each full-file `sourceFile.forEachDescendant` found inside the optimized path:

```typescript
// BEFORE — full-file traversal inside optimized path
sourceFile.forEachDescendant(node => {
  if (Node.isCallExpression(node)) {
    // check for setState in render
  }
});

// AFTER — use ASTIndex
const callExpressions = getNodesByKind(astIndex, SyntaxKind.CallExpression);
for (const node of callExpressions) {
  // same check for setState in render
}
```

### Sub-Node Traversals

If a `forEachDescendant` is called on a function body or component node (not `sourceFile`), it is already scoped to a small sub-tree. These are acceptable and should not be changed unless profiling shows they're a bottleneck.

## Files Modified

| File                                                    | Change                                                                              |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `analyzers/react/src/analyses/anti-pattern-analysis.ts` | Replace full-file `forEachDescendant` calls in optimized path with ASTIndex lookups |

## Verification

1. `pnpm --filter @vipr/react test` — specifically `anti-pattern-analysis.test.ts` and any integration tests
2. `pnpm --filter @vipr/react typecheck`
3. The legacy `detectAntiPatternsLegacy` path remains untouched — if any replacement introduces a regression, tests will catch it

## Risk

**Low.** The legacy fallback remains. Each replacement follows the same pattern used throughout the optimized path. The main risk is a missing `SyntaxKind` — verify by reading each callback's conditions.
