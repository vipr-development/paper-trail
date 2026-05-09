# Formatter Migration Plan

**Status:** Planned
**Priority:** Medium
**Estimated Effort:** 4-6 hours
**Last Updated:** 2026-01-12

## Overview

This document outlines the plan to complete the formatter abstraction migration for the React analyzer plugin. The infrastructure (`IPluginFormatter` interface and `BasePluginFormatter` base class) has been created, but the React plugin still uses the legacy `aggregateResults` method.

## Motivation

### Current Problems

1. **Mixed Concerns:** The `aggregateResults` method (300+ lines) handles both aggregation logic AND result formatting
2. **Hard to Test:** Formatting logic is tightly coupled with aggregation, making unit testing difficult
3. **Hard to Customize:** No easy way to provide alternative formatting strategies
4. **Code Duplication:** Similar formatting patterns could be shared across plugins (future: Vue, Angular, etc.)

### Benefits of Migration

1. **Separation of Concerns:** Clear boundary between aggregation and formatting
2. **Testability:** Each formatter can be unit tested independently
3. **Reusability:** Base formatter can be shared across plugins
4. **Flexibility:** Easy to provide custom formatters for different output needs
5. **Maintainability:** Smaller, focused classes are easier to understand and modify

## Current State

### Infrastructure (Complete ✅)

**Created files:**

- `common/types/src/plugin/formatter.ts` - `IPluginFormatter` interface
- `analyzers/core/src/utils/base-plugin-formatter.ts` - Base implementation
- Exported from `@vipr/core` and `@vipr/types`

**Interface structure:**

```typescript
interface IPluginFormatter<TMetrics> {
  pluginId: string;
  buildResultMap(results: AnalysisResult[]): Map<string, AnalysisResult>;
  normalizeScores(resultMap: Map<string, AnalysisResult>): Map<string, number>;
  calculateCompositeScore(normalizedScores: Map<string, number>): number;
  formatInsights(insights: ComplexityInsight[]): PluginInsight[];
  buildMetrics(
    resultMap: Map<string, AnalysisResult>,
    additionalData?: Record<string, unknown>
  ): TMetrics;
  buildAnalysisBreakdown(results: AnalysisResult[]): Map<AnalysisId, AnalysisResult>;
  format(results, insights, sourceFile, executionTimeMs, additionalData?): PluginResult<TMetrics>;
}
```

### Legacy Code (To Migrate)

**File:** `analyzers/react/src/plugin.ts`

**Method:** `aggregateResults()` (lines 196-578)

- 383 lines of formatting logic
- Extracts data from 15 analysis results
- Normalizes 15 dimension scores
- Calculates weighted composite score
- Builds complex metrics object with defaults
- Converts insights
- Returns `PluginResult`

## Target State

### New Structure

```
analyzers/react/src/
├── formatter/
│   ├── react-formatter.ts          # ReactPluginFormatter class
│   ├── react-formatter.test.ts     # Unit tests
│   └── metrics-builder.ts          # Helper for building metrics with defaults
├── plugin.ts                        # Updated to use formatter
└── ...
```

### Key Classes

**ReactPluginFormatter:**

```typescript
export class ReactPluginFormatter extends BasePluginFormatter<ReactPluginMetrics> {
  constructor() {
    super({
      pluginId: 'react',
      normalizationReference: NORMALIZATION_REFERENCE,
      weights: COMPLEXITY_WEIGHTS,
      metricsBuilder: buildReactMetrics,
    });
  }

  // Override for React-specific enhancements
  format(results, insights, sourceFile, executionTimeMs, additionalData?) {
    const result = super.format(results, insights, sourceFile, executionTimeMs, additionalData);

    // Add React-specific data
    result.metrics.componentCount = findReactComponents(sourceFile).length;

    return result;
  }
}
```

**Updated Plugin:**

```typescript
export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  private formatter = new ReactPluginFormatter();

  async analyze(sourceFile: SourceFile, config?: AnalyzerConfig): Promise<PluginResult> {
    // ... run analyses ...
    const traditional = calculateTraditionalMetrics(sourceFile, { countJsxAsOperators: true });

    // Use formatter instead of aggregateResults
    return this.formatter.format(
      analysisResults,
      allInsights,
      sourceFile,
      Math.round(performance.now() - startTime),
      { traditional }
    );
  }
}
```

## Implementation Plan

### Step 1: Create Metrics Builder Helper

**File:** `analyzers/react/src/formatter/metrics-builder.ts`

Extract the metrics building logic from `aggregateResults` into a pure function:

```typescript
import type { AnalysisResult } from '@vipr/types';
import type { ReactPluginMetrics } from '../types/complexity-result-types';
// ... import all complexity types

export function buildReactMetrics(
  resultMap: Map<string, AnalysisResult>,
  additionalData?: Record<string, unknown>
): ReactPluginMetrics {
  // Extract data from result map
  const types = resultMap.get('react-types')?.data as TypeAnalyzerComplexity | undefined;
  const accessibility = resultMap.get('react-accessibility')?.data as
    | AccessibilityComplexity
    | undefined;
  // ... extract all 15 analyses

  // Build metrics with defaults
  return {
    structural: structural ?? DEFAULT_STRUCTURAL,
    hooks: hooks ?? DEFAULT_HOOKS,
    temporal: temporal ?? DEFAULT_TEMPORAL,
    coupling: coupling ?? DEFAULT_COUPLING,
    identity: identity ?? DEFAULT_IDENTITY,
    dataflow: dataflow ?? DEFAULT_DATAFLOW,
    antiPatterns: antiPatterns ?? DEFAULT_ANTI_PATTERNS,
    security: security ?? DEFAULT_SECURITY,
    migration: migration ?? DEFAULT_MIGRATION,
    performance: performance ?? DEFAULT_PERFORMANCE,
    reliability: reliability ?? DEFAULT_RELIABILITY,
    technicalDebt: technicalDebt ?? DEFAULT_TECHNICAL_DEBT,
    types: types ?? DEFAULT_TYPES,
    accessibility: accessibility ?? DEFAULT_ACCESSIBILITY,
    traditional: (additionalData?.traditional as any) ?? DEFAULT_TRADITIONAL,
    componentCount: 0, // Will be set by formatter
  };
}

// Define all default values as constants
const DEFAULT_STRUCTURAL: StructuralComplexity = {
  score: 0,
  branches: 0,
  jsxConditionals: 0,
  earlyReturns: 0,
  loops: 0,
  logicalOperators: 0,
};

// ... define all other defaults
```

**Why this step:**

- Separates data extraction from formatting logic
- Makes defaults explicit and testable
- Pure function is easy to unit test
- Can be reused if we add multiple formatter variants

### Step 2: Create ReactPluginFormatter

**File:** `analyzers/react/src/formatter/react-formatter.ts`

```typescript
import { BasePluginFormatter, type BaseFormatterConfig } from '@vipr/core';
import type { SourceFile } from 'ts-morph';
import type { AnalysisResult, ComplexityInsight, PluginResult } from '@vipr/types';
import { COMPLEXITY_WEIGHTS } from '../constants/weights';
import { NORMALIZATION_REFERENCE } from '../constants/thresholds';
import type { ReactPluginMetrics } from '../types/complexity-result-types';
import { buildReactMetrics } from './metrics-builder';
import { findReactComponents } from '../utils/react-helpers';

/**
 * Formatter for React plugin results.
 * Extends BasePluginFormatter with React-specific formatting.
 */
export class ReactPluginFormatter extends BasePluginFormatter<ReactPluginMetrics> {
  constructor() {
    const config: BaseFormatterConfig<ReactPluginMetrics> = {
      pluginId: 'react',
      normalizationReference: NORMALIZATION_REFERENCE,
      weights: COMPLEXITY_WEIGHTS,
      metricsBuilder: buildReactMetrics,
    };
    super(config);
  }

  /**
   * Override format to add React-specific data like component count.
   */
  format(
    results: AnalysisResult[],
    insights: ComplexityInsight[],
    sourceFile: SourceFile,
    executionTimeMs: number,
    additionalData?: Record<string, unknown>
  ): PluginResult<ReactPluginMetrics> {
    // Call base implementation
    const result = super.format(results, insights, sourceFile, executionTimeMs, additionalData);

    // Add React-specific enhancements
    result.metrics.componentCount = findReactComponents(sourceFile).length;

    return result;
  }
}
```

**Why this structure:**

- Minimal code - most work done by base class
- Clear React-specific customization point (component count)
- Easy to add more React-specific formatting later
- Follows composition over inheritance principle

### Step 3: Add Comprehensive Tests

**File:** `analyzers/react/src/formatter/react-formatter.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { Project } from 'ts-morph';
import { ReactPluginFormatter } from './react-formatter';

describe('ReactPluginFormatter', () => {
  function createSourceFile(code: string) {
    const project = new Project({ useInMemoryFileSystem: true });
    return project.createSourceFile('test.tsx', code);
  }

  describe('buildResultMap', () => {
    it('should create Map from analysis results', () => {
      const formatter = new ReactPluginFormatter();
      const results = [
        {
          analysisId: 'react-hooks',
          data: { score: 50 },
          insights: [],
          score: 50,
          category: 'technical-debt',
          executionTimeMs: 10,
        },
        {
          analysisId: 'react-structural',
          data: { score: 75 },
          insights: [],
          score: 75,
          category: 'technical-debt',
          executionTimeMs: 10,
        },
      ];

      const map = formatter.buildResultMap(results);

      expect(map.size).toBe(2);
      expect(map.get('react-hooks')).toBeDefined();
      expect(map.get('react-structural')).toBeDefined();
    });

    it('should handle empty results', () => {
      const formatter = new ReactPluginFormatter();
      const map = formatter.buildResultMap([]);

      expect(map.size).toBe(0);
    });
  });

  describe('normalizeScores', () => {
    it('should normalize scores using reference values', () => {
      const formatter = new ReactPluginFormatter();
      const resultMap = new Map([
        [
          'react-hooks',
          {
            analysisId: 'react-hooks',
            data: { score: 25 },
            insights: [],
            score: 25,
            category: 'technical-debt',
            executionTimeMs: 10,
          },
        ],
      ]);

      const normalized = formatter.normalizeScores(resultMap);

      expect(normalized.has('hooks')).toBe(true);
      expect(normalized.get('hooks')).toBeGreaterThan(0);
    });

    it('should skip missing analyses', () => {
      const formatter = new ReactPluginFormatter();
      const resultMap = new Map();

      const normalized = formatter.normalizeScores(resultMap);

      expect(normalized.size).toBe(0);
    });
  });

  describe('calculateCompositeScore', () => {
    it('should apply weights correctly', () => {
      const formatter = new ReactPluginFormatter();
      const normalized = new Map([
        ['hooks', 50],
        ['structural', 75],
      ]);

      const score = formatter.calculateCompositeScore(normalized);

      expect(score).toBeGreaterThan(0);
      expect(score).toBeLessThanOrEqual(100);
    });

    it('should return 0 for empty scores', () => {
      const formatter = new ReactPluginFormatter();
      const score = formatter.calculateCompositeScore(new Map());

      expect(score).toBe(0);
    });
  });

  describe('formatInsights', () => {
    it('should convert ComplexityInsight to PluginInsight', () => {
      const formatter = new ReactPluginFormatter();
      const insights = [
        {
          severity: 'warning' as const,
          category: 'performance',
          message: 'Too many re-renders',
          suggestion: 'Use memoization',
        },
      ];

      const formatted = formatter.formatInsights(insights);

      expect(formatted).toHaveLength(1);
      expect(formatted[0].source).toBe('react');
      expect(formatted[0].id).toContain('react-performance');
    });
  });

  describe('buildMetrics', () => {
    it('should apply defaults for missing analyses', () => {
      const formatter = new ReactPluginFormatter();
      const resultMap = new Map();

      const metrics = formatter.buildMetrics(resultMap);

      expect(metrics.hooks.score).toBe(0);
      expect(metrics.structural.score).toBe(0);
      expect(metrics.componentCount).toBe(0);
    });

    it('should extract data from present analyses', () => {
      const formatter = new ReactPluginFormatter();
      const resultMap = new Map([
        [
          'react-hooks',
          {
            analysisId: 'react-hooks',
            data: { score: 50, totalHooks: 5, breakdown: [], customHooks: [] },
            insights: [],
            score: 50,
            category: 'technical-debt',
            executionTimeMs: 10,
          },
        ],
      ]);

      const metrics = formatter.buildMetrics(resultMap);

      expect(metrics.hooks.score).toBe(50);
      expect(metrics.hooks.totalHooks).toBe(5);
    });

    it('should include traditional metrics from additionalData', () => {
      const formatter = new ReactPluginFormatter();
      const resultMap = new Map();
      const additionalData = {
        traditional: {
          cyclomaticComplexity: 10,
          halstead: {
            volume: 100,
            difficulty: 5,
            effort: 500,
            time: 27.78,
            bugs: 0.033,
            uniqOperators: 10,
            uniqOperands: 15,
            totalOperators: 50,
            totalOperands: 75,
            programLength: 125,
            vocabularySize: 25,
          },
        },
      };

      const metrics = formatter.buildMetrics(resultMap, additionalData);

      expect(metrics.traditional.cyclomaticComplexity).toBe(10);
    });
  });

  describe('format (integration)', () => {
    it('should produce complete PluginResult', () => {
      const formatter = new ReactPluginFormatter();
      const sourceFile = createSourceFile(`
        import React, { useState } from 'react';

        export function Counter() {
          const [count, setCount] = useState(0);
          return <div>{count}</div>;
        }
      `);

      const results = [
        {
          analysisId: 'react-hooks',
          data: { score: 50, totalHooks: 1, breakdown: [], customHooks: [] },
          insights: [],
          score: 50,
          category: 'technical-debt' as const,
          executionTimeMs: 10,
        },
      ];

      const result = formatter.format(results, [], sourceFile, 100);

      expect(result.pluginId).toBe('react');
      expect(result.score).toBeGreaterThanOrEqual(0);
      expect(result.executionTimeMs).toBe(100);
      expect(result.metrics).toBeDefined();
      expect(result.analysisBreakdown).toBeDefined();
    });

    it('should include componentCount', () => {
      const formatter = new ReactPluginFormatter();
      const sourceFile = createSourceFile(`
        export function Comp1() { return <div>1</div>; }
        export function Comp2() { return <div>2</div>; }
      `);

      const result = formatter.format([], [], sourceFile, 100);

      expect(result.metrics.componentCount).toBe(2);
    });

    it('should include traditional metrics when provided', () => {
      const formatter = new ReactPluginFormatter();
      const sourceFile = createSourceFile('const x = 1;');
      const traditional = {
        cyclomaticComplexity: 5,
        halstead: {
          volume: 100,
          difficulty: 5,
          effort: 500,
          time: 27.78,
          bugs: 0.033,
          uniqOperators: 10,
          uniqOperands: 15,
          totalOperators: 50,
          totalOperands: 75,
          programLength: 125,
          vocabularySize: 25,
        },
      };

      const result = formatter.format([], [], sourceFile, 100, { traditional });

      expect(result.metrics.traditional.cyclomaticComplexity).toBe(5);
    });
  });
});
```

### Step 4: Update Plugin to Use Formatter

**File:** `analyzers/react/src/plugin.ts`

**Changes:**

1. Import the formatter
2. Create formatter instance in constructor
3. Replace `aggregateResults` call with `formatter.format`
4. Remove `aggregateResults` method (lines 196-578)

```typescript
// Add import
import { ReactPluginFormatter } from './formatter/react-formatter';

export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  // ... existing fields ...
  private formatter = new ReactPluginFormatter();

  async analyze(sourceFile: SourceFile, config?: AnalyzerConfig): Promise<PluginResult> {
    const startTime = performance.now();
    const enabledAnalyses = this.getEnabledAnalyses(config);
    const analysisResults: AnalysisResult[] = [];
    const allInsights: ComplexityInsight[] = [];

    for (const analysis of enabledAnalyses) {
      const result = await Promise.resolve(analysis.execute(sourceFile));
      analysisResults.push(result);
      allInsights.push(...result.insights);
    }

    const traditional = calculateTraditionalMetrics(sourceFile, {
      countJsxAsOperators: true,
      includeOptionalChaining: false,
      includeNullishCoalescing: false,
    });

    // Use formatter instead of aggregateResults
    return this.formatter.format(
      analysisResults,
      allInsights,
      sourceFile,
      Math.round(performance.now() - startTime),
      { traditional }
    );
  }

  // DELETE aggregateResults method (lines 196-578)
}
```

### Step 5: Update Package Exports

**File:** `analyzers/react/src/index.ts`

Add formatter to public API:

```typescript
// Formatter
export { ReactPluginFormatter } from './formatter/react-formatter';
export { buildReactMetrics } from './formatter/metrics-builder';
```

This allows external consumers to:

- Use the formatter independently
- Create custom formatters by extending it
- Access the metrics builder for testing

### Step 6: Update Plugin Tests

**File:** `analyzers/react/src/plugin.test.ts`

Update existing tests to verify formatter integration:

```typescript
describe('ReactAnalyzerPlugin', () => {
  // ... existing tests ...

  it('should use formatter to produce results', async () => {
    const plugin = new ReactAnalyzerPlugin();
    const sourceFile = createSourceFile(`
      export function Counter() {
        const [count, setCount] = useState(0);
        return <div>{count}</div>;
      }
    `);

    const result = await plugin.analyze(sourceFile);

    expect(result.pluginId).toBe('react');
    expect(result.score).toBeGreaterThanOrEqual(0);
    expect(result.metrics).toBeDefined();
    expect(result.metrics.componentCount).toBeGreaterThan(0);
  });
});
```

## Testing Strategy

### Unit Tests

1. **Metrics Builder Tests** (`metrics-builder.test.ts`)
   - Test default value application
   - Test data extraction from result map
   - Test additionalData handling
   - Test edge cases (empty, partial data)

2. **Formatter Tests** (`react-formatter.test.ts`)
   - Test each method independently
   - Test integration via `format()` method
   - Test React-specific enhancements (component count)
   - Test with various analysis combinations

### Integration Tests

1. **Plugin Tests** (`plugin.test.ts`)
   - Verify formatter is used correctly
   - Test with real analysis results
   - Test with various source files
   - Verify output structure matches expectations

### Regression Tests

1. **Output Comparison**
   - Run plugin with sample files before migration
   - Capture output
   - Run after migration
   - Compare outputs (should be identical)

2. **Performance Tests**
   - Measure analysis time before/after
   - Should be similar or faster
   - No performance regression

## Rollback Plan

### If Issues Discovered

1. **Keep Legacy Code Temporarily**
   - Don't delete `aggregateResults` immediately
   - Rename to `aggregateResultsLegacy`
   - Add feature flag to switch between implementations

```typescript
private USE_FORMATTER = process.env.USE_FORMATTER !== 'false';

async analyze(sourceFile: SourceFile, config?: AnalyzerConfig): Promise<PluginResult> {
  // ... run analyses ...

  if (this.USE_FORMATTER) {
    return this.formatter.format(analysisResults, allInsights, sourceFile, executionTimeMs, { traditional });
  } else {
    return this.aggregateResultsLegacy(analysisResults, traditional, allInsights, sourceFile, startTime);
  }
}
```

2. **Gradual Migration**
   - Ship formatter in one release
   - Default to legacy implementation
   - Enable via feature flag for testing
   - Switch default in next release
   - Remove legacy in following release

### Verification Steps

Before considering migration complete:

1. ✅ All existing tests pass
2. ✅ New formatter tests pass (>90% coverage)
3. ✅ Manual testing with sample projects
4. ✅ Output comparison shows identical results
5. ✅ No performance regression
6. ✅ Code review approval
7. ✅ Documentation updated

## Success Criteria

Migration is successful when:

1. **Functionality:** All tests pass, outputs match legacy implementation
2. **Code Quality:** Formatter has >90% test coverage
3. **Performance:** No regression in analysis time
4. **Maintainability:** Code is clearer, methods are `<50` lines each
5. **Flexibility:** Easy to add new formatting strategies

## Estimated Timeline

| Phase     | Task                          | Estimated Time |
| --------- | ----------------------------- | -------------- |
| 1         | Create metrics builder        | 1 hour         |
| 2         | Create ReactPluginFormatter   | 1 hour         |
| 3         | Write formatter tests         | 2 hours        |
| 4         | Update plugin                 | 30 minutes     |
| 5         | Update exports & plugin tests | 30 minutes     |
| 6         | Regression testing            | 1 hour         |
| **Total** |                               | **6 hours**    |

## Future Enhancements

Once migration is complete, these become easier:

1. **Alternative Formatters:**
   - `CompactReactFormatter` - minimal output for CI
   - `VerboseReactFormatter` - detailed output for debugging
   - `JSONReactFormatter` - structured JSON for tooling

2. **Shared Formatters:**
   - `BaseFrameworkFormatter` - shared by React, Vue, Angular plugins
   - Reduces duplication across plugins

3. **Format Extensions:**
   - Custom metric aggregation strategies
   - Pluggable insight formatters
   - Output templates

## Related Documents

- [Phase 04: Analyzer Refactoring](../v3/phase-04-analyzer-refactoring.md) - Overview of analyzer refactoring work

**Note:** The BasePluginFormatter and IPluginFormatter interfaces are defined in the source code:

- Base class: `analyzers/core/src/utils/base-plugin-formatter.ts`
- Interface: `packages/common/src/types/plugin/formatter.ts`

## Questions & Decisions

### Open Questions

1. **Should we support multiple formatters per plugin?**
   - Pro: More flexibility for different use cases
   - Con: Added complexity
   - **Decision:** Start with single formatter, add multi-formatter support if needed

2. **Should formatter be injectable via config?**
   - Pro: Users can provide custom formatters
   - Con: More API surface
   - **Decision:** Make it a constructor parameter, document how to extend

3. **Should we extract default values to separate file?**
   - Pro: Easier to maintain, can be imported for testing
   - Con: More files
   - **Decision:** Yes, create `defaults.ts` if defaults file grows >100 lines

### Decisions Made

1. ✅ Use composition (extend BasePluginFormatter) over creating from scratch
2. ✅ Keep formatter in separate directory for clear organization
3. ✅ Extract metrics builder as separate testable function
4. ✅ Provide rollback strategy via feature flag if needed
5. ✅ Add comprehensive tests before migration

---

**Next Steps:**

1. Review this plan with team
2. Schedule implementation (estimate 6 hours)
3. Create tracking issue
4. Execute migration following this plan
5. Update this document with any learnings
