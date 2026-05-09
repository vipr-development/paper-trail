---
id: 22-nextjs-security-import-graph-cache
title: 'Next.js: Cache ImportGraphBuilder at Plugin Level'
phase: 22
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 22: Next.js Analyzer — Cache ImportGraphBuilder at Plugin Level

Eliminate per-file O(V+E) import graph rebuilds in Next.js `SecurityAnalysis` by caching the graph across files within an analysis run.

## Problem

In `analyzers/nextjs/src/analyses/security-analysis.ts` (~lines 331-333):

```typescript
if (hasUseClientDirective(sourceFile)) {
  const project = sourceFile.getProject();
  const importGraph = new ImportGraphBuilder(project); // O(V+E) — walks ALL source files
  importGraph.build();
  const violations = detectBoundaryViolations(sourceFile, importGraph);
}
```

`ImportGraphBuilder.build()` traverses every source file in the project to build the import graph. This is called for every client-component file analyzed. In a project with 200 files and 50 client components, the full import graph is rebuilt 50 times.

The plugin class (`NextJsAnalyzerPlugin`) has an `importGraphCache` and `getOrBuildImportGraph()` method, but the `SecurityAnalysis` class cannot access them because it only receives `sourceFile` in `execute()`.

## Fix

Use a module-level `WeakMap` cache keyed on the `Project` instance. The `Project` object is stable for the lifetime of an analysis run:

```typescript
// At module level in security-analysis.ts
const importGraphCache = new WeakMap<Project, ImportGraph>();

// Inside the detection method:
if (hasUseClientDirective(sourceFile)) {
  const project = sourceFile.getProject();
  let importGraph = importGraphCache.get(project);
  if (!importGraph) {
    importGraph = new ImportGraphBuilder(project);
    importGraph.build();
    importGraphCache.set(project, importGraph);
  }
  const violations = detectBoundaryViolations(sourceFile, importGraph);
}
```

### Why WeakMap on Project

- `WeakMap` ensures the cached graph is garbage-collected when the `Project` is released
- The `Project` instance is shared across all files in an analysis session
- If a new `Project` is created (e.g., watch mode re-initialization), a fresh graph is built

### Alternative: Thread Through Config

If the implementing agent finds that `NextJsAnalysisContext` (passed via `ExtendedAnalyzerConfig`) is a better home for the cached graph, use that instead. Read how `buildNextJsContext()` in `analyzers/nextjs/src/shared/nextjs-context.ts` works and whether the import graph can be attached there.

## Files Modified

| File                                                 | Change                                                            |
| ---------------------------------------------------- | ----------------------------------------------------------------- |
| `analyzers/nextjs/src/analyses/security-analysis.ts` | Add module-level WeakMap cache for ImportGraph (~6 lines changed) |

## Verification

1. `pnpm --filter @vipr/nextjs test` — specifically `server-client-import-chain.test.ts`, `security-helpers.test.ts`, and any security analysis tests
2. `pnpm --filter @vipr/nextjs typecheck`
3. Verify the graph is built exactly once per project (add a debug log or breakpoint to confirm)

## Risk

**Low-medium.** The cached graph captures project state at first build. If source files change mid-analysis (unlikely in current engine flow), the cache may be stale. For the sequential-within-session model, this is safe. Add a comment documenting the cache lifetime assumption.
