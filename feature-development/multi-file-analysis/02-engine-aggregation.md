---
id: 02-engine-aggregation
---

# Phase 2: Engine Aggregation

## Overview

This phase enhances the `AnalysisEngine` to support batch analysis with progress reporting, aggregation, and caching. The engine orchestrates file discovery, parallel analysis, and result aggregation into `BatchAnalysisResult`.

## Current Implementation

The existing `analyzeDirectory()` method (line 293-299 in `packages/engine/src/analysis-engine.ts`):

```typescript
async analyzeDirectory(
  dirPath: string,
  patterns: string[] = ['**/*.ts', '**/*.tsx']
): Promise<AggregatedResult[]> {
  const sourceFiles = this.project.addSourceFilesAtPaths(patterns.map(p => `${dirPath}/${p}`));
  return this.analyzeFiles(sourceFiles.map(sf => sf.getFilePath()));
}
```

**Limitations**:

- Returns array of individual results (no aggregation)
- No progress reporting
- No filtering options
- No statistics computation

## Design Goals

1. **Backward Compatibility**: Keep existing signature, add new overload
2. **Progress Reporting**: Emit events during analysis
3. **Efficient Aggregation**: Compute statistics incrementally
4. **Smart Caching**: Cache batch results by directory + options
5. **Resource Management**: Respect concurrency limits

## New Types

Add to `packages/engine/src/analysis-engine.ts`:

```typescript
/**
 * Callback invoked during batch analysis to report progress.
 */
export interface ProgressCallback {
  /**
   * Called when file discovery completes.
   * @param totalFiles - Number of files discovered
   */
  onDiscovery?: (totalFiles: number) => void;

  /**
   * Called before analyzing each file.
   * @param filePath - Path to the file being analyzed
   * @param current - Current file index (1-based)
   * @param total - Total number of files
   */
  onFileStart?: (filePath: string, current: number, total: number) => void;

  /**
   * Called after analyzing each file.
   * @param filePath - Path to the analyzed file
   * @param result - Analysis result
   * @param current - Current file index (1-based)
   * @param total - Total number of files
   */
  onFileComplete?: (
    filePath: string,
    result: AggregatedResult,
    current: number,
    total: number
  ) => void;

  /**
   * Called when a file fails to analyze.
   * @param filePath - Path to the failed file
   * @param error - Error that occurred
   */
  onFileError?: (filePath: string, error: Error) => void;

  /**
   * Called when batch analysis completes.
   * @param result - Final batch result
   */
  onComplete?: (result: BatchAnalysisResult) => void;
}

/**
 * Options for batch analysis.
 */
export interface BatchAnalysisOptions {
  /** File glob patterns (default: ['**\/*.ts', '**\/*.tsx']) */
  patterns?: string[];

  /** Progress callback */
  onProgress?: ProgressCallback;

  /** Minimum score threshold (exclude files above this) */
  minScore?: number;

  /** Maximum score threshold (exclude files below this) */
  maxScore?: number;

  /** Minimum severity to include (critical, warning, info) */
  minSeverity?: Severity;

  /** Parallel file analysis limit (default: 10) */
  concurrency?: number;

  /** Enable batch result caching (default: true) */
  enableBatchCache?: boolean;

  /** Include files with no issues (default: true) */
  includeCleanFiles?: boolean;
}
```

## Enhanced Method Signatures

### Method Overloads

```typescript
export class AnalysisEngine {
  /**
   * Analyze a directory and return individual file results (existing behavior).
   */
  async analyzeDirectory(dirPath: string, patterns?: string[]): Promise<AggregatedResult[]>;

  /**
   * Analyze a directory with options and return batch result.
   */
  async analyzeDirectory(
    dirPath: string,
    options: BatchAnalysisOptions
  ): Promise<BatchAnalysisResult>;

  /**
   * Implementation (combining both signatures).
   */
  async analyzeDirectory(
    dirPath: string,
    patternsOrOptions?: string[] | BatchAnalysisOptions
  ): Promise<AggregatedResult[] | BatchAnalysisResult> {
    // Determine which signature was called
    const isBatchMode =
      patternsOrOptions &&
      !Array.isArray(patternsOrOptions) &&
      typeof patternsOrOptions === 'object';

    if (isBatchMode) {
      return this.analyzeDirectoryBatch(dirPath, patternsOrOptions);
    }

    // Legacy behavior
    const patterns = (patternsOrOptions as string[]) ?? ['**/*.ts', '**/*.tsx'];
    const sourceFiles = this.project.addSourceFilesAtPaths(patterns.map(p => `${dirPath}/${p}`));
    return this.analyzeFiles(sourceFiles.map(sf => sf.getFilePath()));
  }

  /**
   * Internal method for batch analysis.
   */
  private async analyzeDirectoryBatch(
    dirPath: string,
    options: BatchAnalysisOptions
  ): Promise<BatchAnalysisResult> {
    // Implementation below
  }
}
```

## Implementation Steps

### Step 1: File Discovery

```typescript
private async discoverFiles(
  dirPath: string,
  patterns: string[]
): Promise<string[]> {
  const sourceFiles = this.project.addSourceFilesAtPaths(
    patterns.map(p => `${dirPath}/${p}`)
  );

  return sourceFiles.map(sf => sf.getFilePath());
}
```

### Step 2: Progress-Aware Batch Analysis

```typescript
private async analyzeDirectoryBatch(
  dirPath: string,
  options: BatchAnalysisOptions
): Promise<BatchAnalysisResult> {
  const startTime = performance.now();

  // 1. Discover files
  const patterns = options.patterns ?? ['**/*.ts', '**/*.tsx'];
  const filePaths = await this.discoverFiles(dirPath, patterns);

  options.onProgress?.onDiscovery?.(filePaths.length);

  // 2. Analyze files with progress
  const results: AggregatedResult[] = [];
  const errors: Array<{ filePath: string; error: Error }> = [];
  const concurrency = options.concurrency ?? 10;

  await this.executeWithConcurrencyLimit(
    filePaths,
    concurrency,
    async (filePath, index) => {
      options.onProgress?.onFileStart?.(filePath, index + 1, filePaths.length);

      try {
        const result = await this.analyzeFile(filePath);

        // Apply filters
        if (this.shouldIncludeFile(result, options)) {
          results.push(result);
          options.onProgress?.onFileComplete?.(
            filePath,
            result,
            index + 1,
            filePaths.length
          );
        }
      } catch (error) {
        const err = error instanceof Error ? error : new Error(String(error));
        errors.push({ filePath, error: err });
        options.onProgress?.onFileError?.(filePath, err);
      }
    }
  );

  // 3. Aggregate results
  const batchResult = await this.aggregateBatchResults(
    dirPath,
    results,
    errors,
    performance.now() - startTime
  );

  options.onProgress?.onComplete?.(batchResult);

  return batchResult;
}
```

### Step 3: File Filtering

```typescript
private shouldIncludeFile(
  result: AggregatedResult,
  options: BatchAnalysisOptions
): boolean {
  // Filter by score range
  if (options.minScore !== undefined && result.overallScore < options.minScore) {
    return false;
  }
  if (options.maxScore !== undefined && result.overallScore > options.maxScore) {
    return false;
  }

  // Filter by severity
  if (options.minSeverity) {
    const severityLevels = { info: 0, warning: 1, critical: 2 };
    const minLevel = severityLevels[options.minSeverity];
    const hasRelevantInsights = result.insights.some(
      insight => severityLevels[insight.severity] >= minLevel
    );
    if (!hasRelevantInsights) {
      return false;
    }
  }

  // Filter clean files
  if (!options.includeCleanFiles && result.insights.length === 0) {
    return false;
  }

  return true;
}
```

### Step 4: Result Aggregation

```typescript
private async aggregateBatchResults(
  rootPath: string,
  files: AggregatedResult[],
  errors: Array<{ filePath: string; error: Error }>,
  executionTimeMs: number
): Promise<BatchAnalysisResult> {
  // Import helpers from Phase 1
  const {
    calculateSummary,
    createScoreDistribution,
    detectTechnology,
  } = await import('@vipr/common/types/output/batch');

  // Calculate summary statistics
  const scores = files.map(f => f.overallScore);
  const summary = {
    totalFiles: files.length + errors.length,
    analyzedFiles: files.length,
    skippedFiles: errors.length,
    averageScore: scores.length > 0 ? scores.reduce((a, b) => a + b, 0) / scores.length : 0,
    minScore: scores.length > 0 ? Math.min(...scores) : 0,
    maxScore: scores.length > 0 ? Math.max(...scores) : 0,
    executionTimeMs,
  };

  // Group by technology
  const technologies = this.computeTechnologyBreakdown(files);

  // Score distribution
  const scoreDistribution = createScoreDistribution(files);

  // Issue statistics
  const issues = this.computeIssueStatistics(files);

  // Complexity distribution
  const complexity = this.computeComplexityDistribution(files);

  // Recommendations
  const recommendations = this.generateRecommendations(files);

  return {
    type: 'batch',
    analyzedAt: new Date().toISOString(),
    rootPath,
    files,
    summary,
    technologies,
    scoreDistribution,
    issues,
    complexity,
    recommendations,
  };
}
```

### Step 5: Technology Breakdown

```typescript
private computeTechnologyBreakdown(
  files: AggregatedResult[]
): TechnologyBreakdown[] {
  const { detectTechnology } = require('@vipr/common/types/output/batch');

  const groups = new Map<Technology, AggregatedResult[]>();

  for (const file of files) {
    const tech = detectTechnology(file);
    if (!groups.has(tech)) {
      groups.set(tech, []);
    }
    groups.get(tech)!.push(file);
  }

  return Array.from(groups.entries()).map(([technology, techFiles]) => {
    const scores = techFiles.map(f => f.overallScore);
    return {
      technology,
      fileCount: techFiles.length,
      averageScore: scores.reduce((a, b) => a + b, 0) / scores.length,
      minScore: Math.min(...scores),
      maxScore: Math.max(...scores),
      files: techFiles.map(f => f.filePath),
    };
  });
}
```

### Step 6: Issue Statistics

```typescript
private computeIssueStatistics(files: AggregatedResult[]): IssueStatistics {
  const categoryMap = new Map<string, {
    count: number;
    fileCount: Set<string>;
    severityCounts: { critical: number; warning: number; info: number };
    files: Map<string, number>;
  }>();

  let totalIssues = 0;
  const overallSeverityCounts = { critical: 0, warning: 0, info: 0 };
  const fileIssueCounts = new Map<string, { total: number; critical: number }>();

  for (const file of files) {
    let fileTotal = 0;
    let fileCritical = 0;

    for (const insight of file.insights) {
      totalIssues++;
      overallSeverityCounts[insight.severity]++;
      fileTotal++;

      if (insight.severity === 'critical') {
        fileCritical++;
      }

      if (!categoryMap.has(insight.category)) {
        categoryMap.set(insight.category, {
          count: 0,
          fileCount: new Set(),
          severityCounts: { critical: 0, warning: 0, info: 0 },
          files: new Map(),
        });
      }

      const category = categoryMap.get(insight.category)!;
      category.count++;
      category.fileCount.add(file.filePath);
      category.severityCounts[insight.severity]++;

      const fileCount = category.files.get(file.filePath) ?? 0;
      category.files.set(file.filePath, fileCount + 1);
    }

    if (fileTotal > 0) {
      fileIssueCounts.set(file.filePath, { total: fileTotal, critical: fileCritical });
    }
  }

  // Convert to IssueTypeStatistics
  const byCategory: IssueTypeStatistics[] = Array.from(categoryMap.entries()).map(
    ([category, data]) => ({
      category,
      count: data.count,
      fileCount: data.fileCount.size,
      severityCounts: data.severityCounts,
      affectedFiles: Array.from(data.files.entries())
        .map(([filePath, count]) => ({ filePath, count }))
        .sort((a, b) => b.count - a.count),
    })
  );

  // Most affected files
  const mostAffectedFiles = Array.from(fileIssueCounts.entries())
    .map(([filePath, { total, critical }]) => ({
      filePath,
      issueCount: total,
      criticalCount: critical,
    }))
    .sort((a, b) => b.issueCount - a.issueCount)
    .slice(0, 20); // Top 20

  return {
    totalIssues,
    byCategory,
    severityCounts: overallSeverityCounts,
    mostAffectedFiles,
  };
}
```

### Step 7: Complexity Distribution

```typescript
private computeComplexityDistribution(
  files: AggregatedResult[]
): ComplexityDistribution | undefined {
  const { calculateDistribution } = require('@vipr/common/types/output/batch');

  // Extract metrics from core plugin results
  const cyclomaticValues: number[] = [];
  const halsteadVolumes: number[] = [];
  const maintainabilityValues: number[] = [];

  const fileComplexities: Array<{
    filePath: string;
    score: number;
    metric: string;
    value: number;
  }> = [];

  for (const file of files) {
    const coreResult = file.pluginResults.get('core');
    if (!coreResult?.metrics) continue;

    const metrics = coreResult.metrics as any; // Type varies by plugin

    if (metrics.cyclomatic) {
      cyclomaticValues.push(metrics.cyclomatic);
      fileComplexities.push({
        filePath: file.filePath,
        score: file.overallScore,
        metric: 'Cyclomatic Complexity',
        value: metrics.cyclomatic,
      });
    }

    if (metrics.halstead?.volume) {
      halsteadVolumes.push(metrics.halstead.volume);
    }

    if (metrics.maintainability?.index) {
      maintainabilityValues.push(metrics.maintainability.index);
    }
  }

  if (cyclomaticValues.length === 0) {
    return undefined; // No complexity data available
  }

  // Most complex files
  const mostComplexFiles = fileComplexities
    .sort((a, b) => b.value - a.value)
    .slice(0, 20)
    .map(f => ({
      filePath: f.filePath,
      score: f.score,
      primaryComplexity: {
        metric: f.metric,
        value: f.value,
      },
    }));

  return {
    cyclomatic: cyclomaticValues.length > 0
      ? calculateDistribution(cyclomaticValues, 'Cyclomatic Complexity')
      : undefined,
    halsteadVolume: halsteadVolumes.length > 0
      ? calculateDistribution(halsteadVolumes, 'Halstead Volume')
      : undefined,
    maintainability: maintainabilityValues.length > 0
      ? calculateDistribution(maintainabilityValues, 'Maintainability Index')
      : undefined,
    mostComplexFiles,
  };
}
```

### Step 8: Generate Recommendations

```typescript
private generateRecommendations(
  files: AggregatedResult[]
): FileRecommendation[] {
  const recommendations: FileRecommendation[] = [];

  for (const file of files) {
    const criticalIssues = file.insights.filter(i => i.severity === 'critical').length;
    const totalIssues = file.insights.length;
    const score = file.overallScore;

    // Only recommend files that need attention
    if (score >= 70 && criticalIssues === 0) {
      continue;
    }

    const reasons: string[] = [];
    let primaryReason = '';
    let effort: 'low' | 'medium' | 'high' = 'low';

    // Determine primary reason and effort
    if (score < 30) {
      primaryReason = `Very low score (${score}/100)`;
      effort = 'high';
    } else if (criticalIssues > 5) {
      primaryReason = `${criticalIssues} critical issues`;
      effort = 'high';
    } else if (criticalIssues > 0) {
      primaryReason = `${criticalIssues} critical issue${criticalIssues > 1 ? 's' : ''}`;
      effort = 'medium';
    } else if (score < 50) {
      primaryReason = `Low score (${score}/100)`;
      effort = 'medium';
    } else {
      primaryReason = `Below threshold (${score}/100)`;
      effort = 'low';
    }

    // Secondary reasons
    if (totalIssues > 10) {
      reasons.push(`${totalIssues} total issues`);
    }

    const coreResult = file.pluginResults.get('core');
    if (coreResult?.metrics) {
      const metrics = coreResult.metrics as any;
      if (metrics.cyclomatic > 20) {
        reasons.push(`High cyclomatic complexity (${metrics.cyclomatic})`);
      }
    }

    recommendations.push({
      filePath: file.filePath,
      score,
      criticalIssues,
      totalIssues,
      primaryReason,
      secondaryReasons: reasons,
      estimatedEffort: effort,
    });
  }

  // Sort by priority (critical issues, then score)
  return recommendations
    .sort((a, b) => {
      if (a.criticalIssues !== b.criticalIssues) {
        return b.criticalIssues - a.criticalIssues;
      }
      return a.score - b.score;
    })
    .slice(0, 50); // Top 50 recommendations
}
```

## Caching Strategy

### Batch Cache Entry

```typescript
interface BatchCacheEntry {
  result: BatchAnalysisResult;
  timestamp: number;
  optionsHash: string;
}

private batchCache: Map<string, BatchCacheEntry> = new Map();

/**
 * Generate cache key from directory and options.
 */
private getBatchCacheKey(dirPath: string, options: BatchAnalysisOptions): string {
  const optionsHash = this.hashBatchOptions(options);
  return `${dirPath}::${optionsHash}`;
}

/**
 * Hash batch options for cache key.
 */
private hashBatchOptions(options: BatchAnalysisOptions): string {
  const relevant = {
    patterns: options.patterns,
    minScore: options.minScore,
    maxScore: options.maxScore,
    minSeverity: options.minSeverity,
    includeCleanFiles: options.includeCleanFiles,
  };
  return JSON.stringify(relevant);
}

/**
 * Check batch cache before analysis.
 */
private getCachedBatchResult(
  dirPath: string,
  options: BatchAnalysisOptions
): BatchAnalysisResult | null {
  if (!options.enableBatchCache) return null;

  const key = this.getBatchCacheKey(dirPath, options);
  const entry = this.batchCache.get(key);

  if (!entry) return null;

  const now = Date.now();
  const ttl = this.config.cacheTTL ?? 60000;

  if (now - entry.timestamp > ttl) {
    this.batchCache.delete(key);
    return null;
  }

  return entry.result;
}

/**
 * Cache batch result after analysis.
 */
private setCachedBatchResult(
  dirPath: string,
  options: BatchAnalysisOptions,
  result: BatchAnalysisResult
): void {
  if (!options.enableBatchCache) return;

  const key = this.getBatchCacheKey(dirPath, options);
  this.batchCache.set(key, {
    result,
    timestamp: Date.now(),
    optionsHash: this.hashBatchOptions(options),
  });
}
```

## Updated Execute with Concurrency

Enhance the existing helper to support indexed execution:

```typescript
private async executeWithConcurrencyLimit<T, R>(
  items: T[],
  limit: number,
  executor: (item: T, index: number) => Promise<R>
): Promise<R[]> {
  const results: R[] = [];
  const executing: Promise<void>[] = [];
  let index = 0;

  for (const item of items) {
    const currentIndex = index++;
    const promise = executor(item, currentIndex).then(result => {
      results.push(result);
      executing.splice(executing.indexOf(promise), 1);
    });

    executing.push(promise);

    if (executing.length >= limit) {
      await Promise.race(executing);
    }
  }

  await Promise.all(executing);
  return results;
}
```

## Testing Strategy

### Unit Tests

Create `packages/engine/src/analysis-engine-batch.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { AnalysisEngine } from './analysis-engine';
import { isBatchResult } from '@vipr/common';

describe('AnalysisEngine - Batch Analysis', () => {
  it('should return batch result when called with options', async () => {
    const engine = new AnalysisEngine();
    // Register plugins...

    const result = await engine.analyzeDirectory('src/', {
      patterns: ['**/*.ts'],
    });

    expect(isBatchResult(result)).toBe(true);
  });

  it('should return array when called with patterns', async () => {
    const engine = new AnalysisEngine();
    const result = await engine.analyzeDirectory('src/', ['**/*.ts']);

    expect(Array.isArray(result)).toBe(true);
  });

  it('should emit progress events', async () => {
    const engine = new AnalysisEngine();
    const events: string[] = [];

    await engine.analyzeDirectory('src/', {
      onProgress: {
        onDiscovery: () => events.push('discovery'),
        onFileStart: () => events.push('file-start'),
        onFileComplete: () => events.push('file-complete'),
        onComplete: () => events.push('complete'),
      },
    });

    expect(events).toContain('discovery');
    expect(events).toContain('complete');
  });

  it('should filter files by score', async () => {
    const engine = new AnalysisEngine();
    const result = await engine.analyzeDirectory('src/', {
      minScore: 50,
    });

    expect(result.files.every(f => f.overallScore >= 50)).toBe(true);
  });

  it('should cache batch results', async () => {
    const engine = new AnalysisEngine({ enableCache: true });

    const result1 = await engine.analyzeDirectory('src/', {});
    const result2 = await engine.analyzeDirectory('src/', {});

    expect(result1).toBe(result2); // Same reference = cached
  });
});
```

### Integration Tests

```typescript
describe('AnalysisEngine - Batch Integration', () => {
  it('should analyze real directory and compute statistics', async () => {
    const engine = new AnalysisEngine();
    // Register plugins...

    const result = await engine.analyzeDirectory('fixtures/test-project', {
      patterns: ['**/*.ts'],
    });

    expect(result.summary.totalFiles).toBeGreaterThan(0);
    expect(result.technologies.length).toBeGreaterThan(0);
    expect(result.scoreDistribution.length).toBe(5);
  });
});
```

## Migration Path

### Backward Compatibility

Existing code continues to work unchanged:

```typescript
// Old code (still works)
const results = await engine.analyzeDirectory('src/');
// Returns: AggregatedResult[]

// New code (batch mode)
const batchResult = await engine.analyzeDirectory('src/', {
  onProgress: {
    /* ... */
  },
});
// Returns: BatchAnalysisResult
```

### Gradual Migration

Consumers can migrate incrementally:

1. Keep existing calls as-is
2. Add progress callbacks where needed
3. Update to batch mode for new features

## Performance Considerations

### Memory Usage

For large directories (1000+ files):

- Use streaming aggregation (don't hold all results in memory)
- Implement pagination at the engine level
- Consider file-based caching for very large projects

### Concurrency

Default concurrency of 10 balances:

- CPU utilization (ts-morph parsing)
- Memory pressure (AST storage)
- I/O throughput (file reads)

Users can adjust based on their system:

```typescript
await engine.analyzeDirectory('src/', {
  concurrency: 20, // More aggressive
});
```

## Next Steps

1. **Implement enhanced `analyzeDirectory()`** with batch support
2. **Add progress callback support**
3. **Implement aggregation helpers**
4. **Write comprehensive tests**
5. **Proceed to Phase 3**: Batch presenters

## Success Criteria

- [ ] Backward compatible with existing `analyzeDirectory()` calls
- [ ] Progress callbacks fire correctly
- [ ] Batch result contains all required statistics
- [ ] Caching works for batch results
- [ ] Performance acceptable for 1000+ files
- [ ] Full test coverage

## Files Modified

- `packages/engine/src/analysis-engine.ts` - Enhanced `analyzeDirectory()`, batch aggregation
- `packages/engine/src/analysis-engine-batch.test.ts` - New unit tests
