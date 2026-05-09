---
id: 23-nextjs-jsx-descent-consolidation
title: 'Next.js: Consolidate JSX Element Descent in Helpers'
phase: 23
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 23: Next.js Analyzer — Consolidate JSX Element Descent in Helpers

Eliminate 6 redundant `getDescendantsOfKind` calls across 3 helper functions by collecting JSX elements once and sharing.

## Problem

In `analyzers/nextjs/src/utils/nextjs-helpers.ts`, three functions each independently collect JSX elements:

- `getImageUsages()` — calls `getDescendantsOfKind(JsxSelfClosingElement)` + `getDescendantsOfKind(JsxOpeningElement)`
- `getLinkUsages()` — same two calls
- `getScriptUsages()` — same two calls

When an analysis calls all three helpers (e.g., `rendering-analysis.ts`), the same two `getDescendantsOfKind` traversals run 3 times each = 6 total traversals.

## Fix

### Option A: Pre-Collection Function (Preferred)

Add a collection function and optional parameter to the three helpers:

```typescript
export interface JsxElementCollection {
  selfClosing: JsxSelfClosingElement[];
  opening: JsxOpeningElement[];
}

/**
 * Collect all JSX elements once. Pass to getImageUsages/getLinkUsages/getScriptUsages
 * to avoid repeated traversals.
 */
export function collectJsxElements(sourceFile: SourceFile): JsxElementCollection {
  return {
    selfClosing: sourceFile.getDescendantsOfKind(SyntaxKind.JsxSelfClosingElement),
    opening: sourceFile.getDescendantsOfKind(SyntaxKind.JsxOpeningElement),
  };
}

// Update each helper to accept optional pre-collected elements:
export function getImageUsages(
  sourceFile: SourceFile,
  jsxElements?: JsxElementCollection
): Array<JsxOpeningElement | JsxSelfClosingElement> {
  const nextImports = getNextJsImports(sourceFile);
  const imageLocalName = getLocalNameForImport(nextImports, 'next/image');
  if (!imageLocalName) return [];

  const elements = jsxElements ?? collectJsxElements(sourceFile);
  return [
    ...elements.selfClosing.filter(e => e.getTagNameNode().getText() === imageLocalName),
    ...elements.opening.filter(e => e.getTagNameNode().getText() === imageLocalName),
  ];
}
// Same pattern for getLinkUsages and getScriptUsages
```

### Option B: ASTIndex Integration

If the ASTIndex is available in the calling analyses, use it directly:

```typescript
const selfClosing = getNodesByKind(astIndex, SyntaxKind.JsxSelfClosingElement);
const opening = getNodesByKind(astIndex, SyntaxKind.JsxOpeningElement);
```

This requires the calling analysis to pass the ASTIndex to the helper, which is a slightly larger change.

### Updating Call Sites

Find all analyses that call multiple helpers on the same file. Update them to collect once:

```typescript
// BEFORE
const imageUsages = getImageUsages(sourceFile);
const linkUsages = getLinkUsages(sourceFile);
const scriptUsages = getScriptUsages(sourceFile);

// AFTER
const jsxElements = collectJsxElements(sourceFile);
const imageUsages = getImageUsages(sourceFile, jsxElements);
const linkUsages = getLinkUsages(sourceFile, jsxElements);
const scriptUsages = getScriptUsages(sourceFile, jsxElements);
```

The optional parameter means existing single-caller usages continue to work unchanged.

## Files Modified

| File                                             | Change                                                    |
| ------------------------------------------------ | --------------------------------------------------------- |
| `analyzers/nextjs/src/utils/nextjs-helpers.ts`   | Add `collectJsxElements`, add optional param to 3 helpers |
| Calling analyses (e.g., `rendering-analysis.ts`) | Update to collect once and pass through                   |

## Verification

1. `pnpm --filter @vipr/nextjs test` — specifically `nextjs-helpers.test.ts` and analysis tests
2. `pnpm --filter @vipr/nextjs typecheck`

## Risk

**Low.** The optional parameter is backward-compatible. Single callers work unchanged. Multi-callers get a performance improvement.
