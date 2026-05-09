---
id: 17-core-syntax-only-return-type
title: 'Core: Syntax-Only Return Type Check in FunctionAnalysis'
phase: 17
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 17: Core Analyzer — Syntax-Only Return Type Check

Replace the expensive `getReturnType()` type-checker invocation with a syntax-only `getReturnTypeNode()` check in `FunctionAnalysis`.

## Problem

In `analyzers/core/src/analyses/function-analysis.ts` (~line 283):

```typescript
const retType = node.getReturnType(); // INVOKES TYPESCRIPT TYPE CHECKER
hasReturnType = retType.getText() !== 'void' && retType.getText() !== 'any';
```

`getReturnType()` triggers TypeScript's language service to infer the return type — significantly more expensive than syntax-only operations. For a file with 20 functions, this runs 20 times, each invoking type resolution.

## Fix

Replace with a syntax-only check for an explicit return type annotation:

```typescript
// BEFORE — type-checker invocation
const retType = node.getReturnType();
hasReturnType = retType.getText() !== 'void' && retType.getText() !== 'any';

// AFTER — syntax-only, no type inference
hasReturnType = node.getReturnTypeNode() !== undefined;
```

`getReturnTypeNode()` returns the explicit type annotation node if present, or `undefined` if the return type is inferred. No type resolution is triggered.

### Semantic Difference

The old check asked: "does this function have a non-void, non-any inferred return type?"
The new check asks: "does this function have an explicit return type annotation?"

The new semantics are actually more useful for code quality metrics — an explicit annotation is a stronger signal of type safety than an inferred type. Functions with `void` return annotations will now show `hasReturnType: true`, which is correct (they have explicit type annotations).

## Files Modified

| File                                               | Change                                                          |
| -------------------------------------------------- | --------------------------------------------------------------- |
| `analyzers/core/src/analyses/function-analysis.ts` | Replace `getReturnType()` with `getReturnTypeNode()` (~2 lines) |

## Verification

1. `pnpm --filter @vipr/core test` — specifically `function-analysis.test.ts`
2. Review test fixtures: if any test asserts `hasReturnType: false` for a function with an explicit `void` annotation, update the expected value to `true`
3. `pnpm --filter @vipr/core typecheck`

## Risk

**Low.** The `hasReturnType` field is used by presenters for display only (insight generation). No downstream logic depends on the void/any distinction. The performance gain is meaningful — eliminating N type-checker invocations per file where N = number of functions.
