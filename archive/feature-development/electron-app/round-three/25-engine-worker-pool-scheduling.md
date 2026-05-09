---
id: 25-engine-worker-pool-scheduling
title: 'Engine: Work-Stealing Worker Pool Scheduling'
phase: 25
agents: [typescript-engineer, vitest-engineer]
status: pending
depends_on: [13, 14, 15]
---

# Phase 25: Engine — Work-Stealing Worker Pool Scheduling

Replace the round-robin batch dispatch in the worker pool with a work-stealing queue for near-linear multi-core scaling on large projects.

## Problem

In `packages/engine/src/worker-pool.ts`, the `AnalysisWorkerPool` uses round-robin batch dispatch:

```typescript
// ~line 206
const workerIndex = this.nextWorkerId % this.workers.length;
this.nextWorkerId++;
```

Files are pre-partitioned into fixed batches of `batchSize` (default 5) and assigned to workers round-robin. This has two problems:

### Problem A: Round-Robin Ignores File Size

A batch of five 2,000-line files and a batch of five 50-line files are dispatched identically. The large-file batch takes 10x longer, leaving other workers idle while the main thread waits on `Promise.all`.

### Problem B: Batch Boundaries Create Stragglers

`Promise.all` waits for all batches to complete. If one worker gets unlucky (larger files, cold disk cache), the entire `analyzeFiles()` call blocks on it.

## Fix

Replace fixed-batch round-robin with a pull-based work-stealing queue:

### Architecture

```
Main Thread                    Worker Threads
┌─────────────┐               ┌──────────┐
│ File Queue   │◄─── ready ───│ Worker 0 │
│ [f1,f2,...fN]│──── file ───►│          │
│             │◄─── result ──│          │
│             │               └──────────┘
│             │               ┌──────────┐
│             │◄─── ready ───│ Worker 1 │
│             │──── file ───►│          │
│             │◄─── result ──│          │
│             │               └──────────┘
└─────────────┘               ...
```

1. Main thread holds a queue of file paths
2. Workers send a `'ready'` message when idle (at startup and after completing each file)
3. Main thread responds with the next file from the queue
4. When the queue is empty, main thread sends a `'done'` signal
5. Results are collected as they arrive — no `Promise.all` over fixed batches

### Implementation Sketch

```typescript
class AnalysisWorkerPool {
  private fileQueue: string[] = [];
  private resultCollector: Map<string, AggregatedAnalysisResult> = new Map();
  private resolveAll: (() => void) | null = null;
  private pendingCount = 0;

  async analyzeFiles(filePaths: string[]): Promise<Map<string, AggregatedAnalysisResult>> {
    this.fileQueue = [...filePaths];
    this.resultCollector.clear();
    this.pendingCount = 0;

    return new Promise(resolve => {
      this.resolveAll = () => resolve(new Map(this.resultCollector));

      // Each worker requests work by posting 'ready'
      for (const worker of this.workers) {
        this.dispatchNextFile(worker);
      }
    });
  }

  private dispatchNextFile(worker: Worker) {
    const filePath = this.fileQueue.shift();
    if (filePath) {
      this.pendingCount++;
      worker.postMessage({ type: 'analyze', filePath });
    } else if (this.pendingCount === 0) {
      // All files dispatched AND all results received
      this.resolveAll?.();
    }
  }

  private handleWorkerMessage(worker: Worker, message: WorkerMessage) {
    if (message.type === 'result') {
      this.resultCollector.set(message.filePath, message.result);
      this.pendingCount--;
      this.dispatchNextFile(worker); // Pull next file
    }
  }
}
```

### Worker Side

The worker's message handler needs a minor update — instead of receiving a batch of files, it receives individual files:

```typescript
// analysis-worker.ts
parentPort.on('message', async message => {
  if (message.type === 'analyze') {
    const result = await engine.analyzeFile(message.filePath);
    parentPort.postMessage({ type: 'result', filePath: message.filePath, result });
  }
});
```

### Backward Compatibility

Keep the existing batch-mode API as a fallback or remove it entirely. The work-stealing model subsumes batch mode — a batch is just N sequential dispatches to the same worker.

## Investigation Required

Before implementing, read:

1. `packages/engine/src/worker-pool.ts` — understand the current message protocol, serialization, and error handling
2. `packages/engine/src/analysis-worker.ts` — understand the worker's current message handler
3. How errors are propagated from workers to the main thread
4. How worker initialization (plugin loading, engine setup) works — this must remain unchanged

## Files Modified

| File                                     | Change                                                |
| ---------------------------------------- | ----------------------------------------------------- |
| `packages/engine/src/worker-pool.ts`     | Replace round-robin dispatch with work-stealing queue |
| `packages/engine/src/analysis-worker.ts` | Update message handler for per-file dispatch          |

## Dependencies

Phases 13-15 should land first so that each worker benefits from hash deduplication, async cache writes, and shared metrics caching.

## Verification

1. `pnpm --filter @vipr/engine test` — all worker pool tests pass
2. `pnpm --filter @vipr/engine typecheck`
3. Benchmark: analyze a project with mixed file sizes (some large, some small) and verify:
   - All CPU cores are utilized
   - No workers are idle while others are busy
   - Total wall-clock time is lower than with round-robin

## Risk

**Medium-high.** This is the most substantial change in the optimization series. The worker pool's message protocol, error handling, and serialization all need careful attention. Test thoroughly with:

- Empty file lists
- Single file
- More files than workers
- Files that cause analysis errors
- Worker crashes mid-analysis
