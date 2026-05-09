---
id: 13-engine-hash-deduplication
title: 'Engine: Deduplicate Content Hash Computation'
phase: 13
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 13: Engine — Deduplicate Content Hash Computation

Eliminate redundant SHA-256 hash computations in the analysis pipeline. Currently `computeContentHash` is called 8-11 times per cold file on identical content.

## Problem

In `packages/engine/src/analysis-engine.ts`, the same file content is hashed multiple times during a single `analyzeFile` call:

1. **`analyzeFile` (~line 294)** — `computeContentHash(content)` for persistent cache lookup
2. **`checkAnalysisCache` (~line 780)** — `computeContentHash(sourceFile.getText())` called once per analysis (5 analyses = 5 calls)
3. **`cacheAnalysisResult` (~line 810)** — `computeContentHash(sourceFile.getText())` called once per analysis (5 more calls)

For a cold-cache file with 5 core analyses, this produces 11 SHA-256 computations on the same string.

## Fix

Compute the hash exactly once in `analyzeFile` and thread it through `AnalysisExecutionContext` to all downstream consumers.

### Step 1: Extend `AnalysisExecutionContext`

In `analysis-engine.ts`, add `contentHash` to the context interface:

```typescript
interface AnalysisExecutionContext {
  sourceFile: SourceFile;
  pluginId: string;
  config?: AnalyzerConfig;
  timeout: number;
  cache: AnalysisCacheManager;
  logger: PluginLogger;
  astIndex: ASTIndex;
  contentHash: string; // ADD — computed once per file, reused by all analyses
}
```

### Step 2: Thread the hash from `analyzeFile` into `runPluginsInParallel`

The hash is already computed at ~line 294. Pass it as a parameter to `runPluginsInParallel`, then include it in the context construction (~line 904):

```typescript
const context: AnalysisExecutionContext = {
  sourceFile,
  pluginId: plugin.id,
  config,
  timeout: this.timeout,
  cache: this.cacheManager,
  logger: pluginLogger,
  astIndex,
  contentHash, // ADD
};
```

### Step 3: Replace redundant calls in `checkAnalysisCache` and `cacheAnalysisResult`

Both methods currently call `computeContentHash(sourceFile.getText())`. Replace with `context.contentHash`.

## Files Modified

| File                                     | Change                                                                                       |
| ---------------------------------------- | -------------------------------------------------------------------------------------------- |
| `packages/engine/src/analysis-engine.ts` | Add `contentHash` to context interface, compute once, thread through, remove redundant calls |

## Verification

1. `pnpm --filter @vipr/engine test` — all tests pass
2. `pnpm --filter @vipr/engine typecheck` — no type errors
3. Confirm `computeContentHash` is called exactly once per `analyzeFile` invocation (search for call sites after the change)

## Risk

**Low.** `AnalysisExecutionContext` is defined and consumed entirely within `analysis-engine.ts`. No external consumers. The hash value itself is unchanged — only the number of times it is computed changes.
