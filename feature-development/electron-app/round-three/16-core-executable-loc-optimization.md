---
id: 16-core-executable-loc-optimization
title: 'Core: Fix calculateExecutableLOC O(n*depth) Ancestor Walks'
phase: 16
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 16: Core Analyzer — Fix `calculateExecutableLOC` O(n\*depth) Ancestor Walks

Replace per-node `getAncestors()` calls with ASTIndex-based lookups to eliminate the O(n\*depth) complexity in executable LOC counting.

## Problem

In `analyzers/core/src/analyses/maintainability-index-analysis.ts` (~lines 215-267), `calculateExecutableLOC` uses `forEachDescendant` and calls `node.getAncestors()` for every statement node matched:

```typescript
sourceFile.forEachDescendant((node) => {
  if (isExecutableStatement(node)) {
    const ancestors = node.getAncestors();         // O(depth) walk UP the tree
    const inTypeOnly = ancestors.some(a =>         // scan ancestors
      Node.isInterfaceDeclaration(a) || Node.isTypeAliasDeclaration(a)
    );
    const functionAncestors = ancestors.filter(a =>  // filter ancestors
      Node.isArrowFunction(a) || Node.isFunctionDeclaration(a) || ...
    );
    if (!inTypeOnly && functionAncestors.length <= 1) {
      executableStatements++;
    }
  }
});
```

For a file with 100 statements nested 4 levels deep, this produces 400 ancestor-chain traversals — O(n \* depth).

## Fix

Accept `astIndex` via `ExtendedAnalyzerConfig` (already passed by the engine) and use it to pre-compute exclusion sets, reducing ancestor walks to only the statement nodes.

### Step 1: Accept `config` in `execute()`

The engine already passes `ExtendedAnalyzerConfig<ASTIndex>` as the second argument to `execute()`. Update the signature:

```typescript
import type { ExtendedAnalyzerConfig } from '@vipr/engine';
import { getNodesByKind } from '@vipr/engine';
import type { ASTIndex } from '@vipr/engine';

execute(
  sourceFile: SourceFile,
  config?: ExtendedAnalyzerConfig<ASTIndex>
): AnalysisResult<MaintainabilityIndexData> {
  const astIndex = config?.astIndex;
  // ...
  const linesOfCode = this.calculateExecutableLOC(sourceFile, astIndex);
}
```

### Step 2: Rewrite `calculateExecutableLOC` with ASTIndex fast path

When `astIndex` is available:

1. Collect all statement nodes from the index by their `SyntaxKind` — O(1) per kind lookup
2. Build a `Set` of type-only container nodes (interfaces + type aliases) from the index — typically very few
3. Build a `Set` of function container nodes from the index
4. For each statement node, walk ancestors only to check containment — but now the ancestor walk is on the smaller set of statement nodes, and the type-only/function checks use O(1) set membership

```typescript
private calculateExecutableLOC(sourceFile: SourceFile, astIndex?: ASTIndex): number {
  if (!astIndex) {
    return this.calculateExecutableLOCLegacy(sourceFile); // preserve existing logic
  }

  const EXECUTABLE_KINDS = [
    SyntaxKind.ExpressionStatement, SyntaxKind.ReturnStatement,
    SyntaxKind.VariableStatement, SyntaxKind.IfStatement,
    SyntaxKind.ForStatement, SyntaxKind.ForInStatement,
    SyntaxKind.ForOfStatement, SyntaxKind.WhileStatement,
    SyntaxKind.DoStatement, SyntaxKind.SwitchStatement,
    SyntaxKind.BreakStatement, SyntaxKind.ContinueStatement,
    SyntaxKind.ThrowStatement, SyntaxKind.TryStatement,
  ];

  // Collect all executable statement nodes from index
  const statementNodes = EXECUTABLE_KINDS.flatMap(
    kind => [...getNodesByKind(astIndex, kind)]
  );

  // Pre-build exclusion sets from index (O(1) lookups)
  const typeOnlyContainers = new Set([
    ...getNodesByKind(astIndex, SyntaxKind.InterfaceDeclaration),
    ...getNodesByKind(astIndex, SyntaxKind.TypeAliasDeclaration),
  ]);

  const FUNCTION_KINDS = [
    SyntaxKind.ArrowFunction, SyntaxKind.FunctionDeclaration,
    SyntaxKind.FunctionExpression, SyntaxKind.MethodDeclaration,
    SyntaxKind.Constructor, SyntaxKind.GetAccessor, SyntaxKind.SetAccessor,
  ];
  const functionContainers = new Set(
    FUNCTION_KINDS.flatMap(kind => [...getNodesByKind(astIndex, kind)])
  );

  let count = 0;
  for (const node of statementNodes) {
    const ancestors = node.getAncestors();
    const inTypeOnly = ancestors.some(a => typeOnlyContainers.has(a));
    if (inTypeOnly) continue;
    const fnAncestors = ancestors.filter(a => functionContainers.has(a));
    if (fnAncestors.length <= 1) count++;
  }

  return Math.max(1, count);
}
```

This reduces from "O(all*nodes * depth) visiting every node" to "O(statement*nodes * depth) visiting only statements", where statements are typically 10-20% of total nodes.

### Step 3: Rename old method as `calculateExecutableLOCLegacy`

Keep the existing implementation as a fallback for when `astIndex` is not available (e.g., standalone usage outside the engine).

## Files Modified

| File                                                            | Change                                                                      |
| --------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `analyzers/core/src/analyses/maintainability-index-analysis.ts` | Accept `config` param, add ASTIndex fast path for LOC, keep legacy fallback |

## Verification

1. `pnpm --filter @vipr/core test` — specifically `maintainability-index-analysis.test.ts`
2. `pnpm --filter @vipr/core typecheck`
3. **Critical**: Add a test that runs both the legacy and ASTIndex paths on the same fixture and asserts identical LOC counts

## Risk

**Low-medium.** The set of statement `SyntaxKind`s must exactly match the existing `isExecutableStatement` check. Verify by reading the current implementation's conditions and mapping each to a `SyntaxKind`. The ancestor walk is still present but runs on fewer nodes.
