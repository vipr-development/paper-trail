---
id: 24-typescript-astindex-integration
title: 'TypeScript: Wire ASTIndex Across All 9 Analyses'
phase: 24
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 24: TypeScript Analyzer — Wire ASTIndex Across All 9 Analyses

Add ASTIndex integration to all 9 TypeScript analyses, which currently have zero index usage. This is the biggest structural gap across all plugins — 34+ independent traversals run unconditionally with no shared pre-computation.

## Problem

The TypeScript plugin:

- Has no `buildTypeScriptContext()` equivalent (unlike React's `buildReactContext()` or Next.js's `buildNextJsContext()`)
- Does not accept `ExtendedAnalyzerConfig<ASTIndex>` in any analysis
- All 9 analyses use `getDescendantsOfKind` or `forEachDescendant` independently

### Traversal Count by Analysis

| Analysis                          | File                              | Traversals |
| --------------------------------- | --------------------------------- | ---------: |
| `structural-quality-analysis.ts`  | 9 `getDescendantsOfKind` calls    |          9 |
| `type-guards-analysis.ts`         | 7 `getDescendantsOfKind` calls    |          7 |
| `module-augmentation-analysis.ts` | 6 `getDescendantsOfKind` calls    |          6 |
| `declaration-shape-analysis.ts`   | 4                                 |          4 |
| `type-complexity-analysis.ts`     | 3 + nested per-alias traversals   |         3+ |
| `generics-analysis.ts`            | 2 + nested per-generic traversals |         2+ |
| `type-safety-analysis.ts`         | 1 `forEachDescendant`             |          1 |
| `utility-types-analysis.ts`       | 1                                 |          1 |
| `import-discipline-analysis.ts`   | 1                                 |          1 |

**Total: 34+ traversals on the same file** that could be reduced to hash-map lookups.

## Fix

### Step 1: Verify the Engine Passes ASTIndex to TypeScript Plugin

Read the TypeScript plugin's `plugin.ts` to confirm:

- Does it implement `getAnalyses()` (enhanced path) or only `analyze()` (legacy path)?
- If it uses `getAnalyses()`, the engine already passes `ExtendedAnalyzerConfig<ASTIndex>` to each analysis's `execute()` method
- If it uses the legacy `analyze()` path, the plugin needs to be updated to use `getAnalyses()` — or the `analyze()` method needs to extract the `astIndex` from the config and thread it to analyses

### Step 2: Update Each Analysis to Accept ASTIndex

For each of the 9 analyses, update the `execute` signature:

```typescript
// BEFORE
execute(sourceFile: SourceFile, _config?: unknown): AnalysisResult<T> {

// AFTER
execute(
  sourceFile: SourceFile,
  config?: ExtendedAnalyzerConfig<ASTIndex>
): AnalysisResult<T> {
  const astIndex = config?.astIndex;
```

### Step 3: Replace `getDescendantsOfKind` with ASTIndex Lookups

For each `getDescendantsOfKind` call:

```typescript
// BEFORE
const throwNodes = sourceFile.getDescendantsOfKind(SyntaxKind.ThrowStatement);

// AFTER
const throwNodes = astIndex
  ? getNodesByKind(astIndex, SyntaxKind.ThrowStatement)
  : sourceFile.getDescendantsOfKind(SyntaxKind.ThrowStatement);
```

Keep the fallback for standalone usage outside the engine.

### Priority: Start with the Two Highest-Impact Files

**`structural-quality-analysis.ts`** — 9 `getDescendantsOfKind` calls at lines ~128, 150, 161, 172, 189, 203, 208, 214, 217. Each is a straightforward replacement.

**`type-complexity-analysis.ts`** — Has a `sourceFile.forEachDescendant` at ~line 156 counting `ConditionalType`, `UnionType`, `IntersectionType`, `MappedType`, etc. Replace with:

```typescript
if (astIndex) {
  conditionalBranches = getNodesByKind(astIndex, SyntaxKind.ConditionalType).length;
  // For union/intersection, may need to sum member counts — read existing logic
} else {
  sourceFile.forEachDescendant(/* existing */);
}
```

Also has nested `node.forEachDescendant` per type alias (~lines 284, 310) for recursive type detection. These are scoped to individual type alias sub-trees and are acceptable — do not change.

### Remaining 7 Analyses

Apply the same pattern. Each analysis is straightforward:

1. Accept `config?: ExtendedAnalyzerConfig<ASTIndex>`
2. Extract `astIndex`
3. Replace each `getDescendantsOfKind` with an ASTIndex lookup + fallback
4. Leave `forEachDescendant` calls that are on sub-nodes (not `sourceFile`)

## Files Modified

| File                                                                | Change                                                 |
| ------------------------------------------------------------------- | ------------------------------------------------------ |
| `analyzers/typescript/src/plugin.ts`                                | Verify/enable ASTIndex threading (may need no changes) |
| `analyzers/typescript/src/analyses/structural-quality-analysis.ts`  | 9 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/type-complexity-analysis.ts`     | 3 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/type-guards-analysis.ts`         | 7 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/module-augmentation-analysis.ts` | 6 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/declaration-shape-analysis.ts`   | 4 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/generics-analysis.ts`            | 2 ASTIndex replacements                                |
| `analyzers/typescript/src/analyses/type-safety-analysis.ts`         | 1 ASTIndex replacement                                 |
| `analyzers/typescript/src/analyses/utility-types-analysis.ts`       | 1 ASTIndex replacement                                 |
| `analyzers/typescript/src/analyses/import-discipline-analysis.ts`   | 1 ASTIndex replacement                                 |

## Verification

1. `pnpm --filter @vipr/typescript test` — all 9 analysis test files must pass
2. `pnpm --filter @vipr/typescript typecheck`
3. For `structural-quality-analysis.ts` and `type-complexity-analysis.ts`: add a test that runs both the legacy and ASTIndex paths on the same fixture and asserts identical counts

## Risk

**Medium.** This is a broad change touching 9 files. Each replacement is mechanical, but the volume means careful attention to:

- Matching the exact `SyntaxKind` for each existing `getDescendantsOfKind` call
- Preserving return type (some `getDescendantsOfKind` calls return typed arrays that may need casting from the ASTIndex's generic `Node[]`)
- Not breaking the fallback path for standalone usage

## Import Requirements

The analyses need to import from `@vipr/engine`:

```typescript
import type { ExtendedAnalyzerConfig, ASTIndex } from '@vipr/engine';
import { getNodesByKind } from '@vipr/engine';
```

Verify `@vipr/typescript` has `@vipr/engine` as a dependency. If not, add it to `package.json`.
