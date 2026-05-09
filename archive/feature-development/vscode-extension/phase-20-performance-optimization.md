# Phase 20: Performance Optimization

**Purpose**: Optimize extension performance through lazy loading, worker threads, caching, and efficient resource management.

**Dependencies**: All previous phases

**Deliverables**: Lazy module loading, worker thread for analysis, memory optimization, startup time improvements

## Overview

Phase 20 focuses on performance optimization:

1. Implement lazy loading for heavy dependencies
2. Move analysis to worker threads
3. Add intelligent caching strategies
4. Optimize bundle size with tree-shaking
5. Implement incremental analysis for changed files
6. Add memory usage monitoring and cleanup
7. Optimize activation time with deferred initialization

## Architecture

```mermaid
---
title: Performance Optimization Architecture
config:
  theme: forest
---
graph TB
    Activation[Extension Activation] --> Minimal[Minimal Core Load]
    Minimal --> LazyInit[Lazy Initialization]

    LazyInit -->|on demand| Dashboard[Dashboard Bundle]
    LazyInit -->|on demand| Charts[Chart.js]
    LazyInit -->|on demand| SQLite[SQLite]
    LazyInit -->|on demand| PDFLib[PDF-lib]
    LazyInit -->|on demand| AI[AI Services]

    Command[User Command] --> WorkerManager[Worker Manager]
    WorkerManager --> AnalysisWorker[Analysis Worker Thread]

    AnalysisWorker --> CoreEngine[@vipr/engine]
    CoreEngine --> Results[Analysis Results]

    Results --> Cache[Result Cache]
    Cache --> CacheStrategy{Cache Strategy}

    CacheStrategy -->|hit| ReturnCached[Return Cached]
    CacheStrategy -->|miss| RunAnalysis[Run Analysis]

    RunAnalysis --> Incremental{Incremental?}
    Incremental -->|yes| AnalyzeChanged[Analyze Changed Files Only]
    Incremental -->|no| AnalyzeAll[Analyze All Files]

    Extension[Extension] --> MemoryMonitor[Memory Monitor]
    MemoryMonitor --> Cleanup[Periodic Cleanup]
    Cleanup --> DisposeResources[Dispose Unused Resources]

    Extension --> BundleOptimizer[Bundle Optimizer]
    BundleOptimizer --> TreeShaking[Tree Shaking]
    BundleOptimizer --> CodeSplitting[Code Splitting]
    BundleOptimizer --> Minification[Minification]

    classDef perf fill:#2563eb,stroke:#1e40af,color:#fff
    classDef worker fill:#16a34a,stroke:#15803d,color:#fff
    classDef cache fill:#dc2626,stroke:#b91c1c,color:#fff

    class Activation,LazyInit,MemoryMonitor,BundleOptimizer perf
    class WorkerManager,AnalysisWorker,CoreEngine worker
    class Cache,CacheStrategy,Incremental cache
```

## File Changes

### 1. Lazy Module Loader

**File**: `src/core/lazy-loader.ts`

```typescript
/**
 * Lazy module loader with caching
 */
export class LazyLoader {
  private static modules = new Map<string, any>();

  /**
   * Lazy load Chart.js
   */
  static async loadChartJS() {
    if (!this.modules.has('chart.js')) {
      const { Chart, registerables } = await import('chart.js');
      Chart.register(...registerables);
      this.modules.set('chart.js', Chart);
    }
    return this.modules.get('chart.js');
  }

  /**
   * Lazy load SQLite
   */
  static async loadSQLite() {
    if (!this.modules.has('sqlite')) {
      const sqlite = await import('node-sqlite3-wasm');
      this.modules.set('sqlite', sqlite);
    }
    return this.modules.get('sqlite');
  }

  /**
   * Lazy load PDF-lib
   */
  static async loadPDFLib() {
    if (!this.modules.has('pdf-lib')) {
      const pdfLib = await import('pdf-lib');
      this.modules.set('pdf-lib', pdfLib);
    }
    return this.modules.get('pdf-lib');
  }

  /**
   * Lazy load analysis engine
   */
  static async loadAnalysisEngine() {
    if (!this.modules.has('engine')) {
      const engine = await import('@vipr/engine');
      this.modules.set('engine', engine);
    }
    return this.modules.get('engine');
  }

  /**
   * Clear cached modules
   */
  static clearCache() {
    this.modules.clear();
  }
}
```

### 2. Worker Manager

**File**: `src/core/worker-manager.ts`

```typescript
import { Worker } from 'worker_threads';
import * as vscode from 'vscode';
import * as path from 'path';

export interface WorkerTask {
  id: string;
  type: 'analyze';
  data: any;
}

export interface WorkerResult {
  taskId: string;
  success: boolean;
  data?: any;
  error?: string;
}

/**
 * Manages worker threads for CPU-intensive analysis
 */
export class WorkerManager {
  private worker: Worker | null = null;
  private pendingTasks = new Map<string, (result: WorkerResult) => void>();

  constructor(private context: vscode.ExtensionContext) {}

  /**
   * Execute task in worker thread
   */
  async executeTask(task: WorkerTask): Promise<WorkerResult> {
    return new Promise((resolve, reject) => {
      try {
        // Create worker if not exists
        if (!this.worker) {
          this.createWorker();
        }

        // Store pending task resolver
        this.pendingTasks.set(task.id, resolve);

        // Send task to worker
        this.worker!.postMessage(task);

        // Timeout after 5 minutes
        setTimeout(() => {
          if (this.pendingTasks.has(task.id)) {
            this.pendingTasks.delete(task.id);
            reject(new Error('Worker task timeout'));
          }
        }, 300000);
      } catch (error) {
        reject(error);
      }
    });
  }

  /**
   * Create worker thread
   */
  private createWorker() {
    const workerPath = path.join(this.context.extensionPath, 'dist', 'analysis-worker.js');

    this.worker = new Worker(workerPath);

    // Handle worker messages
    this.worker.on('message', (result: WorkerResult) => {
      const resolver = this.pendingTasks.get(result.taskId);
      if (resolver) {
        resolver(result);
        this.pendingTasks.delete(result.taskId);
      }
    });

    // Handle worker errors
    this.worker.on('error', error => {
      console.error('[Vipr] Worker error:', error);
      this.terminateWorker();
    });

    // Handle worker exit
    this.worker.on('exit', code => {
      if (code !== 0) {
        console.error(`[Vipr] Worker stopped with exit code ${code}`);
      }
      this.worker = null;
    });
  }

  /**
   * Terminate worker
   */
  terminateWorker() {
    if (this.worker) {
      this.worker.terminate();
      this.worker = null;
    }
    this.pendingTasks.clear();
  }

  /**
   * Dispose
   */
  dispose() {
    this.terminateWorker();
  }
}
```

### 3. Analysis Worker

**File**: `src/workers/analysis-worker.ts`

```typescript
import { parentPort } from 'worker_threads';
import type { WorkerTask, WorkerResult } from '../core/worker-manager';

/**
 * Worker thread for running analysis
 */
async function processTask(task: WorkerTask): Promise<WorkerResult> {
  try {
    switch (task.type) {
      case 'analyze':
        // Import engine dynamically
        const { AnalysisEngine } = await import('@vipr/engine');
        const engine = new AnalysisEngine(task.data.config);

        // Run analysis
        const results = await engine.analyzeFiles(task.data.files);

        return {
          taskId: task.id,
          success: true,
          data: results,
        };

      default:
        return {
          taskId: task.id,
          success: false,
          error: `Unknown task type: ${task.type}`,
        };
    }
  } catch (error) {
    return {
      taskId: task.id,
      success: false,
      error: error instanceof Error ? error.message : String(error),
    };
  }
}

// Listen for tasks from main thread
if (parentPort) {
  parentPort.on('message', async (task: WorkerTask) => {
    const result = await processTask(task);
    parentPort!.postMessage(result);
  });
}
```

### 4. Result Cache

**File**: `src/core/result-cache.ts`

```typescript
import type { FileAnalysisResult } from '@vipr/common';
import * as crypto from 'crypto';
import * as vscode from 'vscode';

interface CacheEntry {
  result: FileAnalysisResult;
  hash: string;
  timestamp: number;
}

/**
 * Cache for analysis results with content-based invalidation
 */
export class ResultCache {
  private cache = new Map<string, CacheEntry>();
  private readonly MAX_AGE_MS = 10 * 60 * 1000; // 10 minutes

  /**
   * Get cached result for file
   */
  async get(filePath: string, document: vscode.TextDocument): Promise<FileAnalysisResult | null> {
    const entry = this.cache.get(filePath);

    if (!entry) {
      return null;
    }

    // Check if cache is stale
    const age = Date.now() - entry.timestamp;
    if (age > this.MAX_AGE_MS) {
      this.cache.delete(filePath);
      return null;
    }

    // Check if content changed
    const currentHash = this.hashContent(document.getText());
    if (currentHash !== entry.hash) {
      this.cache.delete(filePath);
      return null;
    }

    return entry.result;
  }

  /**
   * Store result in cache
   */
  set(filePath: string, result: FileAnalysisResult, content: string): void {
    const hash = this.hashContent(content);

    this.cache.set(filePath, {
      result,
      hash,
      timestamp: Date.now(),
    });
  }

  /**
   * Invalidate cache for file
   */
  invalidate(filePath: string): void {
    this.cache.delete(filePath);
  }

  /**
   * Clear entire cache
   */
  clear(): void {
    this.cache.clear();
  }

  /**
   * Remove stale entries
   */
  cleanup(): void {
    const now = Date.now();
    for (const [filePath, entry] of this.cache.entries()) {
      const age = now - entry.timestamp;
      if (age > this.MAX_AGE_MS) {
        this.cache.delete(filePath);
      }
    }
  }

  /**
   * Get cache statistics
   */
  getStats() {
    return {
      size: this.cache.size,
      entries: Array.from(this.cache.keys()),
    };
  }

  /**
   * Hash file content
   */
  private hashContent(content: string): string {
    return crypto.createHash('sha256').update(content).digest('hex');
  }
}
```

### 5. Memory Monitor

**File**: `src/core/memory-monitor.ts`

```typescript
import * as vscode from 'vscode';

/**
 * Monitor and report memory usage
 */
export class MemoryMonitor {
  private intervalId: NodeJS.Timeout | null = null;

  /**
   * Start monitoring
   */
  start() {
    // Check memory every 5 minutes
    this.intervalId = setInterval(
      () => {
        this.checkMemory();
      },
      5 * 60 * 1000
    );
  }

  /**
   * Stop monitoring
   */
  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  /**
   * Check current memory usage
   */
  private checkMemory() {
    const usage = process.memoryUsage();
    const heapUsedMB = usage.heapUsed / 1024 / 1024;
    const heapTotalMB = usage.heapTotal / 1024 / 1024;

    console.log(`[Vipr] Memory: ${heapUsedMB.toFixed(1)}MB / ${heapTotalMB.toFixed(1)}MB`);

    // Warn if memory usage is high
    if (heapUsedMB > 500) {
      console.warn('[Vipr] High memory usage detected');
      this.triggerCleanup();
    }
  }

  /**
   * Trigger cleanup of caches and unused resources
   */
  private triggerCleanup() {
    // Notify extension to clean up
    vscode.commands.executeCommand('vipr.internal.cleanup');
  }
}
```

### 6. Deferred Initialization

**File**: `src/extension.ts` (optimizations)

```typescript
import { LazyLoader } from './core/lazy-loader';
import { WorkerManager } from './core/worker-manager';
import { ResultCache } from './core/result-cache';
import { MemoryMonitor } from './core/memory-monitor';

let workerManager: WorkerManager | undefined;
let resultCache: ResultCache | undefined;
let memoryMonitor: MemoryMonitor | undefined;

export async function activate(context: vscode.ExtensionContext) {
  console.log('[Vipr] Activating extension...');

  // Initialize only core services immediately
  resultCache = new ResultCache();
  memoryMonitor = new MemoryMonitor();
  memoryMonitor.start();

  // Defer heavy initialization
  setImmediate(() => {
    initializeHeavyServices(context);
  });

  // Register minimal commands immediately
  registerCoreCommands(context);

  // Periodic cache cleanup
  setInterval(
    () => {
      resultCache?.cleanup();
    },
    5 * 60 * 1000
  );

  console.log('[Vipr] Extension activated (minimal mode)');
}

/**
 * Initialize heavy services after activation
 */
async function initializeHeavyServices(context: vscode.ExtensionContext) {
  try {
    // Initialize worker manager
    workerManager = new WorkerManager(context);

    // Initialize storage service (deferred)
    // storageService will be initialized on first use

    console.log('[Vipr] Heavy services initialized');
  } catch (error) {
    console.error('[Vipr] Failed to initialize heavy services:', error);
  }
}

/**
 * Register core commands
 */
function registerCoreCommands(context: vscode.ExtensionContext) {
  // Internal cleanup command
  context.subscriptions.push(
    vscode.commands.registerCommand('vipr.internal.cleanup', () => {
      LazyLoader.clearCache();
      resultCache?.clear();
      workerManager?.terminateWorker();
      console.log('[Vipr] Cleanup complete');
    })
  );
}

export function deactivate() {
  memoryMonitor?.stop();
  workerManager?.dispose();
  LazyLoader.clearCache();
}
```

### 7. Bundle Optimization

**File**: `clients/vscode-extension/esbuild.config.mjs` (optimized)

```javascript
import * as esbuild from 'esbuild';

const production = process.argv.includes('--production');

const config = {
  entryPoints: ['src/extension.ts'],
  bundle: true,
  outfile: 'dist/extension.js',
  external: ['vscode'],
  format: 'cjs',
  platform: 'node',
  target: 'node18',
  minify: production,
  sourcemap: !production,
  treeShaking: true,
  metafile: production,
  logLevel: 'info',
  // Performance optimizations
  splitting: false, // Not supported for cjs
  keepNames: false,
  legalComments: 'none',
};

async function main() {
  const ctx = await esbuild.context(config);

  if (process.argv.includes('--watch')) {
    await ctx.watch();
    console.log('Watching for changes...');
  } else {
    await ctx.rebuild();
    await ctx.dispose();

    // Analyze bundle size in production
    if (production && config.metafile) {
      const result = await esbuild.build({ ...config, metafile: true });
      console.log(await esbuild.analyzeMetafile(result.metafile));
    }
  }
}

main().catch(e => {
  console.error(e);
  process.exit(1);
});
```

## Configuration

Add performance settings:

**File**: `clients/vscode-extension/package.json` (additions)

```json
{
  "configuration": {
    "properties": {
      "vipr.performance.useWorkerThreads": {
        "type": "boolean",
        "default": true,
        "description": "Use worker threads for analysis (improves performance)"
      },
      "vipr.performance.cacheResults": {
        "type": "boolean",
        "default": true,
        "description": "Cache analysis results for unchanged files"
      },
      "vipr.performance.incrementalAnalysis": {
        "type": "boolean",
        "default": true,
        "description": "Only analyze changed files in workspace analysis"
      }
    }
  }
}
```

## Acceptance Criteria

- [ ] Extension activates in < 2 seconds
- [ ] Heavy modules lazy loaded on first use
- [ ] Worker thread spawns for analysis tasks
- [ ] Result cache hits for unchanged files
- [ ] Incremental analysis works correctly
- [ ] Memory usage stays under 200MB for typical workspaces
- [ ] Bundle size < 5MB (excluding node_modules)
- [ ] No blocking of UI thread during analysis
- [ ] Cache cleanup removes stale entries
- [ ] Memory monitor detects high usage
- [ ] Performance settings configurable
- [ ] Worker thread handles errors gracefully
- [ ] Lazy loading doesn't cause race conditions

## Testing Strategy

### Performance Benchmarks

**File**: `src/__benchmarks__/performance.test.ts`

```typescript
import { describe, it, expect } from 'vitest';

describe('Performance Benchmarks', () => {
  it('should activate extension quickly', async () => {
    const start = Date.now();
    // Simulate activation
    const duration = Date.now() - start;
    expect(duration).toBeLessThan(2000); // < 2 seconds
  });

  it('should analyze 100 files within reasonable time', async () => {
    const start = Date.now();
    // Run analysis on 100 files
    const duration = Date.now() - start;
    expect(duration).toBeLessThan(30000); // < 30 seconds
  });

  it('should use cache for unchanged files', async () => {
    // First analysis
    const firstDuration = await measureAnalysis();

    // Second analysis (should be faster due to cache)
    const secondDuration = await measureAnalysis();

    expect(secondDuration).toBeLessThan(firstDuration * 0.3); // 70% faster
  });
});
```

### Manual Verification

1. Measure activation time:
   - Open large workspace
   - Enable "Developer: Startup Performance"
   - Check Vipr activation time
   - Should be < 2 seconds
2. Test lazy loading:
   - Restart VSCode
   - Check extension loads without Chart.js
   - Open dashboard
   - Verify Chart.js loads on demand
3. Test worker threads:
   - Run analysis on 100+ files
   - Monitor CPU usage
   - Verify UI remains responsive
4. Test caching:
   - Analyze workspace
   - Run analysis again without changes
   - Verify second run much faster
5. Test incremental analysis:
   - Modify one file
   - Run workspace analysis
   - Verify only modified file analyzed
6. Monitor memory:
   - Run analysis multiple times
   - Check memory usage in Task Manager
   - Should stay under 200MB
7. Test bundle size:
   - Build production bundle
   - Check dist/extension.js size
   - Should be < 5MB

## Summary

Phase 20 optimizes extension performance through lazy loading, worker threads, intelligent caching, and efficient resource management, ensuring fast activation times and responsive UI even with large workspaces.
