---
id: 21-react-helpers-type-checker-removal
title: 'React: Remove Type-Checker Calls in react-helpers.ts'
phase: 21
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 21: React Analyzer — Remove Type-Checker Calls in Traversal Callbacks

Replace `getType()`, `getReturnType()`, and `getTypeAtLocation()` calls inside traversal callbacks in `react-helpers.ts` with syntax-only alternatives.

## Problem

In `analyzers/react/src/utils/react-helpers.ts`, five type-checker invocations occur inside hot paths:

| Line | Call                           | Context                                                      |
| ---- | ------------------------------ | ------------------------------------------------------------ |
| ~510 | `fn.getReturnType().getText()` | `getInferredReturnType()` — standalone utility, lower impact |
| ~561 | `propsParam.getType()`         | `extractPropsInfo()` — invoked for every component           |
| ~568 | `prop.getTypeAtLocation(fn)`   | Inside props iteration — invoked per prop per component      |
| ~612 | `desc.getType()`               | Inside a `forEachDescendant` callback — invoked per node     |
| ~640 | `firstArg.getType()`           | Inside a traversal callback — invoked per matching call      |

Each `getType()` / `getReturnType()` / `getTypeAtLocation()` call invokes the TypeScript language service, which is orders of magnitude more expensive than syntax-only operations. When called inside traversal callbacks, these are the most expensive per-node operations in the React analyzer.

## Fix

Replace each with syntax-only alternatives where feasible.

### `extractPropsInfo` (~line 561)

```typescript
// BEFORE — type-checker
const propsType = propsParam.getType();
const properties = propsType.getProperties();
analysis.totalPropsCount = properties.length;

// AFTER — syntax-only via type annotation node
const typeNode = propsParam.getTypeNode();
if (typeNode && Node.isTypeLiteral(typeNode)) {
  analysis.totalPropsCount = typeNode.getProperties().length;
} else if (typeNode && Node.isTypeReference(typeNode)) {
  // For interface-typed props (e.g., `props: ButtonProps`), resolve the referenced type
  // This still avoids getType() — use the source file's type alias/interface lookup
  const typeName = typeNode.getTypeName().getText();
  const sourceFile = propsParam.getSourceFile();
  const iface = sourceFile.getInterface(typeName);
  const typeAlias = sourceFile.getTypeAlias(typeName);
  if (iface) {
    analysis.totalPropsCount = iface.getProperties().length;
  } else if (typeAlias) {
    const aliasType = typeAlias.getTypeNode();
    if (aliasType && Node.isTypeLiteral(aliasType)) {
      analysis.totalPropsCount = aliasType.getProperties().length;
    }
  }
}
```

### `desc.getType()` (~line 612) and `firstArg.getType()` (~line 640)

Read the surrounding code to understand what these checks accomplish. Common patterns:

- If checking for `any` type: replace with `Node.isAnyKeyword` on the type annotation
- If checking for specific type names: use `getText()` on the type annotation node
- If the type check is essential for correctness and cannot be done syntactically: consider caching the result or deferring the check to a post-traversal phase

### Investigation Required

For each call site, the implementing agent must:

1. Read the surrounding 20 lines of code
2. Determine what the type information is used for
3. Design the syntax-only replacement
4. If a syntax-only replacement is not possible (e.g., the check requires resolved type information from imported interfaces), document why and leave the call with a `// @todo` comment noting the type-checker cost

## Files Modified

| File                                         | Change                                                                  |
| -------------------------------------------- | ----------------------------------------------------------------------- |
| `analyzers/react/src/utils/react-helpers.ts` | Replace type-checker calls with syntax-only alternatives where feasible |

## Verification

1. `pnpm --filter @vipr/react test` — run all React tests (helpers are used by 12 analyses)
2. `pnpm --filter @vipr/react typecheck`
3. For `extractPropsInfo`: if `totalPropsCount` changes for interface-typed components, update test expectations with a comment explaining the semantic difference

## Risk

**Medium.** The `propsCount` metric may produce different values for components with interface-typed props that are imported from other files (the syntax-only approach cannot resolve cross-file type references). This is an acceptable trade-off — the metric is approximate by nature. Type-checker calls that genuinely cannot be replaced should be left in place with documentation.
