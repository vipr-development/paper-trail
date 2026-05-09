---
id: 19-react-security-astindex
title: 'React: Wire SecurityAnalysis to ASTIndex'
phase: 19
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 19: React Analyzer — Wire SecurityAnalysis to ASTIndex

Replace the full-tree `forEachDescendant` traversal in React's `SecurityAnalysis` with ASTIndex lookups. The analysis currently ignores the available ASTIndex with an explicit comment acknowledging the debt.

## Problem

In `analyzers/react/src/analyses/security-analysis.ts` (~lines 191-194):

```typescript
if (astIndex) {
  // For security analysis, use traversal even with astIndex for simplicity
  sourceFile.forEachDescendant(node => {  // ignores astIndex entirely
```

With `executionCost: 3`, this analysis runs on most React files and always performs a full-tree traversal regardless of whether the ASTIndex is available.

## Fix

Refactor the `detectVulnerabilities` method (or equivalent) to use ASTIndex lookups for each vulnerability category.

### Approach

The vulnerability detection checks specific node kinds. Map each vulnerability type to its `SyntaxKind`:

| Vulnerability                   | Node Kind(s) to Source from ASTIndex  |
| ------------------------------- | ------------------------------------- |
| XSS (`dangerouslySetInnerHTML`) | `SyntaxKind.JsxAttribute`             |
| `eval()` usage                  | `SyntaxKind.CallExpression`           |
| Sensitive variable names        | `SyntaxKind.VariableDeclaration`      |
| `innerHTML` assignment          | `SyntaxKind.PropertyAccessExpression` |
| Unsafe URL patterns             | `SyntaxKind.StringLiteral`            |

### Implementation Pattern

```typescript
if (astIndex) {
  // XSS: check JsxAttribute nodes for dangerouslySetInnerHTML
  const jsxAttributes = getNodesByKind(astIndex, SyntaxKind.JsxAttribute);
  for (const attr of jsxAttributes) {
    // existing check logic — unchanged
  }

  // eval(): check CallExpression nodes
  const callExpressions = getNodesByKind(astIndex, SyntaxKind.CallExpression);
  for (const call of callExpressions) {
    // existing eval check logic — unchanged
  }
  // ... other vulnerability categories
} else {
  // Keep existing forEachDescendant fallback
}
```

The vulnerability detection logic (string comparisons, pattern matching) inside each node check does not change — only how the candidate nodes are sourced.

### Investigation Required

Before implementing, read the current `forEachDescendant` callback thoroughly and:

1. List every `SyntaxKind` the callback checks for
2. Map each to an ASTIndex lookup
3. Verify no vulnerability checks depend on traversal order or parent context that would be lost with index-based lookup
4. If any checks need parent context, use `node.getParent()` (O(1) in ts-morph) after sourcing from the index

## Files Modified

| File                                                | Change                                                     |
| --------------------------------------------------- | ---------------------------------------------------------- |
| `analyzers/react/src/analyses/security-analysis.ts` | Refactor `detectVulnerabilities` to use ASTIndex fast path |

## Verification

1. `pnpm --filter @vipr/react test` — specifically `security-analysis.test.ts`
2. `pnpm --filter @vipr/react typecheck`
3. Verify all existing vulnerability test cases still detect the same vulnerabilities

## Risk

**Low-medium.** The node-check logic is unchanged. The risk is missing a `SyntaxKind` in the ASTIndex path that the `forEachDescendant` path would have caught. The fallback path ensures correctness if `astIndex` is absent.
