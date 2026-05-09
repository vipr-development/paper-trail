---
id: 26-enable-worker-parallelism-defaults
title: 'Engine: Enable Worker Parallelism by Default'
phase: 26
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: [25]
---

# Phase 26: Engine — Enable Worker Parallelism by Default

Turn on the worker thread pool for large file sets where it provides near-linear speedup. Currently the infrastructure is complete but disabled by default.

## Problem

In `packages/engine/src/analysis-engine.ts`, the worker pool requires explicit opt-in:

```typescript
parallelism: {
  enabled: true,         // OFF by default
  poolSize: cpus,
  minFilesForParallelism: 4,
  pluginPackages: [...],
}
```

For projects with 50+ files, single-threaded analysis leaves most CPU cores idle.

## Fix

### Step 1: Update Default Configuration

Change the default `parallelism.enabled` to `true` when the file count exceeds `minFilesForParallelism`. Read the existing config structure to find where defaults are set.

### Step 2: Auto-Detect Plugin Packages

The `pluginPackages` config is required for workers to load plugins. This should be auto-detected from the registered plugins rather than requiring manual configuration. Read how plugins are registered in the engine and whether their package names are available at runtime.

### Step 3: Sensible Defaults

_NOTE: We should confirm this is a best practice and strike a balance between sensible and benefically opportunistic, in terms of how large to make the pool size_.

```typescript
parallelism: {
  enabled: true,
  poolSize: Math.max(1, os.cpus().length - 1),  // Leave 1 core for main thread
  minFilesForParallelism: 10,  // Don't spawn workers for tiny projects
  batchSize: 1,  // After Phase 25, dispatch individual files
}
```

### Desktop vs CLI Considerations

- **CLI**: Workers are appropriate. No UI thread to protect.
- **Desktop (Electron)**: Workers run in utility processes, not the main process. The existing `AnalysisCoordinator` already manages concurrency. Verify that enabling worker parallelism in the engine doesn't conflict with the coordinator's own concurrency limits (check `clients/desktop/src/shared/config/config.ts` `coordinator.maxConcurrent`).

## Investigation Required

1. Read how `parallelism` config is currently constructed in both CLI and desktop paths
2. Read how `pluginPackages` is populated — can it be auto-detected?
3. Check if the desktop's `AnalysisCoordinator` wraps the engine or runs alongside it
4. Verify worker pool graceful shutdown on process exit

## Files Modified

| File                                           | Change                                 |
| ---------------------------------------------- | -------------------------------------- |
| `packages/engine/src/analysis-engine.ts`       | Update default parallelism config      |
| Possibly `clients/cli/` and `clients/desktop/` | Verify compatibility with new defaults |

## Dependencies

Phase 25 (work-stealing) should land first for optimal scheduling.

## Verification

1. `pnpm test` — full workspace tests pass
2. `pnpm --filter @vipr/engine test` — specifically worker pool tests
3. Benchmark: compare analysis time for a 100+ file project with parallelism on vs off
4. Desktop: verify the app doesn't freeze during analysis (workers should not block the renderer)

## Risk

**Medium.** Enabling parallelism changes the execution model. Edge cases to watch:

- Worker startup failures (e.g., plugin loading errors)
- Memory pressure on machines with many cores spawning many workers
- Race conditions in result aggregation
- Test isolation — some tests may assume single-threaded execution
