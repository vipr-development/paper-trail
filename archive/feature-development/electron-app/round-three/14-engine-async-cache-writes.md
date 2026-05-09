---
id: 14-engine-async-cache-writes
title: 'Engine: Fire-and-Forget Persistent Cache Writes'
phase: 14
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: []
---

# Phase 14: Engine — Fire-and-Forget Persistent Cache Writes

Remove the `await` on persistent cache writes so analysis results are returned immediately without blocking on disk I/O.

## Problem

In `packages/engine/src/analysis-engine.ts` (~line 340-343):

```typescript
if (this.persistentCache) {
  const pluginVersions = this.getPluginVersions();
  await this.persistentCache.set(contentHash, pluginVersions, aggregated); // BLOCKS HERE
}
```

`analyzeFile` waits for the disk write to complete before returning the result. The caller never needs the write to complete synchronously — the result is already computed. For 1,000 files on initial analysis, this adds 5-15 seconds of unnecessary blocking.

## Fix

Remove the `await` and add a `.catch` to prevent unhandled rejection:

```typescript
if (this.persistentCache) {
  const pluginVersions = this.getPluginVersions();
  this.persistentCache
    .set(contentHash, pluginVersions, aggregated)
    .catch(err =>
      logger.warn(
        `Persistent cache write failed for ${filePath}: ${err instanceof Error ? err.message : String(err)}`
      )
    );
}
```

## Files Modified

| File                                     | Change                                                        |
| ---------------------------------------- | ------------------------------------------------------------- |
| `packages/engine/src/analysis-engine.ts` | Remove `await` on `persistentCache.set`, add `.catch` handler |

## Test Considerations

Check `packages/engine/src/analysis-engine.test.ts` for tests that:

1. Call `analyzeFile` then immediately assert on `persistentCache.get` — these may need a `vi.waitFor()` or `await vi.advanceTimersToNextTimerAsync()` since the write is now async
2. Mock `persistentCache.set` to verify it was called — these still work because the call still happens, just not awaited

The persistent cache already handles `ENOSPC`/`EACCES` gracefully, so fire-and-forget is safe.

## Verification

1. `pnpm --filter @vipr/engine test` — all tests pass (fix any timing-dependent tests)
2. `pnpm --filter @vipr/engine typecheck` — no type errors

## Risk

**Low.** No semantic change to analysis results. Only side effect: if the process exits immediately after analysis, a cache entry may not persist. This is acceptable — a cache miss on next run simply re-analyzes.
