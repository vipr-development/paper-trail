# Presenter Test Implementation Guide

Practical guide for implementing tests for each untested presenter in the React analyzer.

## Table of Contents

1. [Base Presenter Tests](#base-presenter-tests)
2. [Overview Presenter Tests](#overview-presenter-tests)
3. [Performance Presenter Tests](#performance-presenter-tests)
4. [Accessibility Presenter Tests](#accessibility-presenter-tests)
5. [Migration Presenter Tests](#migration-presenter-tests)
6. [Reliability Presenter Tests](#reliability-presenter-tests)
7. [Security Presenter Tests](#security-presenter-tests)
8. [Dataflow Presenter Tests](#dataflow-presenter-tests)
9. [Anti-Pattern Presenter Tests](#anti-pattern-presenter-tests)
10. [Test Utilities](#test-utilities)

## Base Presenter Tests

**File:** `src/presenters/base-presenter.test.ts`

**Purpose:** Test shared utility methods used by all presenters.

```typescript
/**
 * Base React Report Presenter Tests
 *
 * Tests for shared presenter utilities and base functionality.
 */

import { describe, it, expect } from 'vitest';
import { ReactBasePresenter } from './base-presenter';
import type { PluginResult, ReportPresentation } from '@vipr/common';
import { createAnalysisId } from '@vipr/common';

// Create a concrete test implementation of the abstract class
class TestPresenter extends ReactBasePresenter {
  readonly reportType = 'test' as const;
  readonly analysisId = 'react-test';

  present(pluginResult: PluginResult): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Test Report',
      sections: [],
    };
  }
}

describe('ReactBasePresenter', () => {
  let presenter: TestPresenter;

  beforeEach(() => {
    presenter = new TestPresenter();
  });

  describe('canPresent', () => {
    it('should accept React plugin results with matching analysis', () => {
      const pluginResult: PluginResult = {
        pluginId: 'react',
        score: 75,
        insights: [],
        executionTimeMs: 100,
        metrics: {},
        analysisBreakdown: new Map([
          [
            createAnalysisId('react-test'),
            {
              analysisId: 'react-test',
              category: 'test',
              data: {},
              insights: [],
              executionTimeMs: 50,
            },
          ],
        ]),
      };

      expect(presenter.canPresent(pluginResult)).toBe(true);
    });

    it('should reject non-React plugin results', () => {
      const pluginResult: PluginResult = {
        pluginId: 'vue',
        score: 75,
        insights: [],
        executionTimeMs: 100,
        metrics: {},
        analysisBreakdown: new Map(),
      };

      expect(presenter.canPresent(pluginResult)).toBe(false);
    });

    it('should reject results without analysis breakdown', () => {
      const pluginResult: PluginResult = {
        pluginId: 'react',
        score: 75,
        insights: [],
        executionTimeMs: 100,
        metrics: {},
      };

      expect(presenter.canPresent(pluginResult)).toBe(false);
    });

    it('should reject results without matching analysis', () => {
      const pluginResult: PluginResult = {
        pluginId: 'react',
        score: 75,
        insights: [],
        executionTimeMs: 100,
        metrics: {},
        analysisBreakdown: new Map([
          [
            createAnalysisId('react-other'),
            {
              analysisId: 'react-other',
              category: 'other',
              data: {},
              insights: [],
              executionTimeMs: 50,
            },
          ],
        ]),
      };

      expect(presenter.canPresent(pluginResult)).toBe(false);
    });
  });

  describe('createScore', () => {
    it('should create score with default thresholds', () => {
      const score = presenter['createScore'](75, 'Test Score');

      expect(score.value).toBe(75);
      expect(score.maxValue).toBe(100);
      expect(score.label).toBe('Test Score');
      expect(score.thresholds).toEqual({
        excellent: 80,
        good: 60,
        fair: 40,
      });
    });

    it('should create score with custom thresholds', () => {
      const customThresholds = {
        excellent: 90,
        good: 70,
        fair: 50,
      };
      const score = presenter['createScore'](75, 'Test Score', 100, customThresholds);

      expect(score.thresholds).toEqual(customThresholds);
    });

    it('should round score values', () => {
      const score = presenter['createScore'](75.7, 'Test Score');
      expect(score.value).toBe(76);
    });

    it('should support custom max values', () => {
      const score = presenter['createScore'](15, 'Test Score', 20);
      expect(score.maxValue).toBe(20);
    });
  });

  describe('createMetric', () => {
    it('should create metric with minimal options', () => {
      const metric = presenter['createMetric']('test-id', 'Test Metric', 42);

      expect(metric.id).toBe('test-id');
      expect(metric.label).toBe('Test Metric');
      expect(metric.value).toBe(42);
    });

    it('should create metric with all options', () => {
      const metric = presenter['createMetric']('test-id', 'Test Metric', 75, {
        maxValue: 100,
        unit: '%',
        format: 'percentage',
        thresholds: { excellent: 80, good: 60, fair: 40 },
        context: 'Test context',
      });

      expect(metric.maxValue).toBe(100);
      expect(metric.unit).toBe('%');
      expect(metric.format).toBe('percentage');
      expect(metric.thresholds).toBeDefined();
      expect(metric.context).toBe('Test context');
    });

    it('should support string values', () => {
      const metric = presenter['createMetric']('test-id', 'Test Metric', 'high', {
        format: 'text',
      });

      expect(metric.value).toBe('high');
      expect(metric.format).toBe('text');
    });
  });

  describe('createItem', () => {
    it('should create item with minimal options', () => {
      const item = presenter['createItem']('item-1', 'high', 'Test Item');

      expect(item.id).toBe('item-1');
      expect(item.severity).toBe('high');
      expect(item.title).toBe('Test Item');
    });

    it('should create item with all options', () => {
      const item = presenter['createItem']('item-1', 'high', 'Test Item', {
        description: 'Test description',
        location: { file: 'test.tsx', line: 10, column: 5 },
        suggestion: 'Fix this',
        category: 'performance',
        metadata: { complexity: 10 },
        links: { docs: 'https://example.com' },
      });

      expect(item.description).toBe('Test description');
      expect(item.location).toBeDefined();
      expect(item.suggestion).toBe('Fix this');
      expect(item.category).toBe('performance');
      expect(item.metadata).toEqual({ complexity: 10 });
      expect(item.links).toEqual({ docs: 'https://example.com' });
    });
  });

  describe('createSection', () => {
    it('should create section with minimal options', () => {
      const section = presenter['createSection']('test-section', 'Test Section');

      expect(section.id).toBe('test-section');
      expect(section.title).toBe('Test Section');
    });

    it('should create section with all options', () => {
      const score = presenter['createScore'](75, 'Section Score');
      const metrics = [presenter['createMetric']('m1', 'Metric 1', 10)];
      const items = [presenter['createItem']('i1', 'low', 'Item 1')];
      const subsections = [presenter['createSection']('sub1', 'Subsection 1')];

      const section = presenter['createSection']('test-section', 'Test Section', {
        description: 'Test description',
        score,
        metrics,
        items,
        subsections,
        collapsible: true,
        collapsed: false,
        style: 'list',
      });

      expect(section.description).toBe('Test description');
      expect(section.score).toEqual(score);
      expect(section.metrics).toEqual(metrics);
      expect(section.items).toEqual(items);
      expect(section.subsections).toEqual(subsections);
      expect(section.collapsible).toBe(true);
      expect(section.collapsed).toBe(false);
      expect(section.style).toBe('list');
    });
  });

  describe('mapSeverity', () => {
    it('should map critical severity', () => {
      expect(presenter['mapSeverity']('critical')).toBe('critical');
      expect(presenter['mapSeverity']('CRITICAL')).toBe('critical');
    });

    it('should map high severity', () => {
      expect(presenter['mapSeverity']('high')).toBe('high');
      expect(presenter['mapSeverity']('serious')).toBe('high');
      expect(presenter['mapSeverity']('error')).toBe('high');
    });

    it('should map medium severity', () => {
      expect(presenter['mapSeverity']('medium')).toBe('medium');
      expect(presenter['mapSeverity']('moderate')).toBe('medium');
      expect(presenter['mapSeverity']('warning')).toBe('medium');
    });

    it('should map low severity', () => {
      expect(presenter['mapSeverity']('low')).toBe('low');
      expect(presenter['mapSeverity']('minor')).toBe('low');
    });

    it('should default to info for unknown severities', () => {
      expect(presenter['mapSeverity']('unknown')).toBe('info');
      expect(presenter['mapSeverity'](undefined)).toBe('info');
      expect(presenter['mapSeverity']('')).toBe('info');
    });
  });

  describe('mapLocation', () => {
    it('should map complete location', () => {
      const location = presenter['mapLocation']({
        file: 'test.tsx',
        line: 10,
        column: 5,
        endLine: 12,
        endColumn: 15,
      });

      expect(location).toEqual({
        file: 'test.tsx',
        line: 10,
        column: 5,
        endLine: 12,
        endColumn: 15,
      });
    });

    it('should handle alternative line/column properties', () => {
      const location = presenter['mapLocation']({
        file: 'test.tsx',
        startLine: 10,
        startColumn: 5,
        endLine: 12,
        endColumn: 15,
      });

      expect(location?.line).toBe(10);
      expect(location?.column).toBe(5);
    });

    it('should return undefined for undefined location', () => {
      expect(presenter['mapLocation'](undefined)).toBeUndefined();
    });

    it('should handle partial locations', () => {
      const location = presenter['mapLocation']({
        file: 'test.tsx',
        line: 10,
      });

      expect(location?.file).toBe('test.tsx');
      expect(location?.line).toBe(10);
    });
  });

  describe('createCountMetrics', () => {
    it('should create metrics from counts', () => {
      const counts = {
        errors: 5,
        warnings: 3,
        info: 1,
      };

      const metrics = presenter['createCountMetrics'](counts, 'issue');

      expect(metrics).toHaveLength(3);
      expect(metrics[0].id).toBe('issue-errors');
      expect(metrics[0].label).toBe('Errors');
      expect(metrics[0].value).toBe(5);
    });

    it('should filter out zero counts', () => {
      const counts = {
        errors: 5,
        warnings: 0,
        info: 1,
      };

      const metrics = presenter['createCountMetrics'](counts, 'issue');

      expect(metrics).toHaveLength(2);
      expect(metrics.some(m => m.id === 'issue-warnings')).toBe(false);
    });

    it('should handle empty counts', () => {
      const metrics = presenter['createCountMetrics']({}, 'issue');
      expect(metrics).toHaveLength(0);
    });
  });

  describe('formatLabel', () => {
    it('should format camelCase labels', () => {
      expect(presenter['formatLabel']('camelCase')).toBe('Camel Case');
      expect(presenter['formatLabel']('myVariable')).toBe('My Variable');
    });

    it('should format kebab-case labels', () => {
      expect(presenter['formatLabel']('kebab-case')).toBe('Kebab Case');
      expect(presenter['formatLabel']('my-variable')).toBe('My Variable');
    });

    it('should handle single words', () => {
      expect(presenter['formatLabel']('word')).toBe('Word');
    });

    it('should handle acronyms', () => {
      expect(presenter['formatLabel']('jsxElement')).toBe('Jsx Element');
    });
  });
});
```

## Overview Presenter Tests

**File:** `src/presenters/overview-presenter.test.ts`

**Key Tests:**

- Overall complexity score section
- Traditional metrics (cyclomatic, Halstead)
- React-specific dimensions (hooks, effects, coupling)
- Hook breakdown display
- Effect analysis display
- Insight aggregation

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { OverviewPresenter } from './overview-presenter';
import type { PluginResult } from '@vipr/common';

describe('OverviewPresenter', () => {
  let presenter: OverviewPresenter;

  beforeEach(() => {
    presenter = new OverviewPresenter();
  });

  describe('metadata', () => {
    it('should have correct metadata', () => {
      expect(presenter.reportType).toBe('overview');
      expect(presenter.pluginId).toBe('react');
    });
  });

  describe('canPresent', () => {
    it('should accept React results with metrics', () => {
      const result: PluginResult = {
        pluginId: 'react',
        score: 75,
        insights: [],
        executionTimeMs: 100,
        metrics: { traditional: { cyclomaticComplexity: 5 } },
      };

      expect(presenter.canPresent(result)).toBe(true);
    });

    it('should reject results without metrics', () => {
      const result: PluginResult = {
        pluginId: 'react',
        score: 75,
        insights: [],
        executionTimeMs: 100,
      };

      expect(presenter.canPresent(result)).toBe(false);
    });
  });

  describe('present', () => {
    it('should create overview with all sections', () => {
      const result = createMockPluginResultWithMetrics();
      const presentation = presenter.present(result);

      expect(presentation.reportType).toBe('overview');
      expect(presentation.title).toBe('Complexity Overview');
      expect(presentation.sections.length).toBeGreaterThan(0);
    });

    it('should include overall complexity score', () => {
      const result = createMockPluginResultWithMetrics({ score: 85 });
      const presentation = presenter.present(result);

      const overallSection = presentation.sections.find(s => s.id === 'overall');
      expect(overallSection).toBeDefined();
      expect(overallSection?.score?.value).toBe(85);
    });

    it('should include traditional complexity measures', () => {
      const result = createMockPluginResultWithMetrics({
        metrics: {
          traditional: {
            cyclomaticComplexity: 10,
            halstead: { volume: 500, difficulty: 15 },
          },
        },
      });

      const presentation = presenter.present(result);
      const complexitySection = presentation.sections.find(s => s.id === 'complexity-measures');

      expect(complexitySection).toBeDefined();
      expect(complexitySection?.metrics?.length).toBeGreaterThan(0);
    });

    it('should include hook breakdown', () => {
      const result = createMockPluginResultWithMetrics({
        metrics: {
          hooks: {
            score: 15,
            totalHooks: 5,
            breakdown: [
              { name: 'useState', count: 2, complexity: 2 },
              { name: 'useEffect', count: 3, complexity: 5 },
            ],
            customHooks: [],
          },
        },
      });

      const presentation = presenter.present(result);
      const hookSection = presentation.sections.find(s => s.id.includes('hook'));

      expect(hookSection).toBeDefined();
    });

    it('should filter out empty sections', () => {
      const result = createMockPluginResultWithMetrics({
        metrics: { traditional: { cyclomaticComplexity: 1 } },
      });

      const presentation = presenter.present(result);

      // Should only include sections with content
      presentation.sections.forEach(section => {
        const hasContent =
          section.score ||
          (section.metrics && section.metrics.length > 0) ||
          (section.items && section.items.length > 0);
        expect(hasContent).toBe(true);
      });
    });
  });
});

function createMockPluginResultWithMetrics(overrides = {}): PluginResult {
  return {
    pluginId: 'react',
    score: 75,
    insights: [],
    executionTimeMs: 100,
    metrics: {
      traditional: {
        cyclomaticComplexity: 5,
        halstead: { volume: 100, difficulty: 10 },
      },
    },
    ...overrides,
  };
}
```

## Performance Presenter Tests

**File:** `src/presenters/performance-presenter.test.ts`

**Key Tests:**

- Render performance metrics
- Memoization analysis
- Bundle impact assessment
- Optimization opportunities
- Risk level calculation

```typescript
describe('PerformancePresenter', () => {
  // Test render performance section
  it('should display render performance metrics', () => {
    const result = createMockPerformanceResult({
      renderPerformance: {
        expensiveOperations: 5,
        inlineFunctions: 3,
        inlineObjects: 2,
        inlineArrays: 1,
        score: 60,
        risk: 'medium',
      },
    });

    const presentation = presenter.present(result);
    const renderSection = presentation.sections.find(s => s.id.includes('render'));

    expect(renderSection).toBeDefined();
    expect(renderSection?.metrics?.some(m => m.id.includes('expensive'))).toBe(true);
  });

  // Test memoization analysis
  it('should show memoization effectiveness', () => {
    const result = createMockPerformanceResult({
      memoization: {
        useCallbackCount: 3,
        useMemoCount: 2,
        reactMemoCount: 1,
        effectiveness: 75,
        unnecessaryMemoization: [],
        missingMemoization: ['Component1'],
      },
    });

    const presentation = presenter.present(result);
    const memoSection = presentation.sections.find(s => s.id.includes('memo'));

    expect(memoSection).toBeDefined();
  });

  // Test optimization opportunities
  it('should list optimization opportunities', () => {
    const result = createMockPerformanceResult({
      renderPerformance: {
        optimizationOpportunities: [
          {
            type: 'inline-function',
            location: { line: 10 },
            impact: 'medium',
            suggestion: 'Use useCallback',
          },
        ],
      },
    });

    const presentation = presenter.present(result);
    const opportunities = presentation.sections.flatMap(s => s.items || []);

    expect(opportunities.length).toBeGreaterThan(0);
  });
});
```

## Accessibility Presenter Tests

**File:** `src/presenters/accessibility-presenter.test.ts`

**Key Tests:**

- WCAG compliance level
- Violation categorization
- Keyboard navigation score
- Screen reader compatibility
- Severity mapping
- Remediation suggestions

```typescript
describe('AccessibilityPresenter', () => {
  // Test WCAG level display
  it('should show WCAG compliance level', () => {
    const result = createMockAccessibilityResult({
      wcagLevel: 'AA',
      keyboardNavigationScore: 85,
      screenReaderCompatibility: 90,
    });

    const presentation = presenter.present(result);
    expect(presentation.sections[0].score).toBeDefined();
  });

  // Test violation categorization
  it('should categorize violations by severity', () => {
    const result = createMockAccessibilityResult({
      violations: [
        { type: 'missing-alt', severity: 'critical', count: 3 },
        { type: 'color-contrast', severity: 'serious', count: 5 },
      ],
    });

    const presentation = presenter.present(result);
    const violationSection = presentation.sections.find(s => s.id.includes('violation'));

    expect(violationSection?.items?.length).toBeGreaterThan(0);
  });

  // Test remediation suggestions
  it('should provide actionable suggestions', () => {
    const result = createMockAccessibilityResult({
      violations: [
        {
          type: 'missing-label',
          location: { line: 15 },
          suggestion: 'Add aria-label or aria-labelledby',
        },
      ],
    });

    const presentation = presenter.present(result);
    const items = presentation.sections.flatMap(s => s.items || []);

    expect(items.some(i => i.suggestion)).toBe(true);
  });
});
```

## Migration Presenter Tests

**File:** `src/presenters/migration-presenter.test.ts`

**Key Tests:**

- Migration readiness score
- Blocker identification
- Effort estimation
- Codemod suggestions
- Version compatibility

```typescript
describe('MigrationPresenter', () => {
  // Test readiness assessment
  it('should show migration readiness', () => {
    const result = createMockMigrationResult({
      readinessScore: 75,
      targetVersion: '19.0.0',
      blockers: [],
      warnings: [],
    });

    const presentation = presenter.present(result);
    expect(presentation.sections[0].score?.value).toBe(75);
  });

  // Test blocker display
  it('should highlight migration blockers', () => {
    const result = createMockMigrationResult({
      blockers: [
        {
          type: 'legacy-context',
          location: { line: 20 },
          description: 'Uses legacy context API',
        },
      ],
    });

    const presentation = presenter.present(result);
    const blockerItems = presentation.sections
      .flatMap(s => s.items || [])
      .filter(i => i.severity === 'critical');

    expect(blockerItems.length).toBeGreaterThan(0);
  });

  // Test effort estimation
  it('should display effort estimation', () => {
    const result = createMockMigrationResult({
      estimatedEffort: {
        hours: 40,
        complexity: 'moderate',
        breakdown: {
          classComponents: 10,
          deprecatedLifecycles: 5,
          stringRefs: 3,
        },
      },
    });

    const presentation = presenter.present(result);
    const effortSection = presentation.sections.find(s => s.id.includes('effort'));

    expect(effortSection).toBeDefined();
  });
});
```

## Reliability Presenter Tests

**File:** `src/presenters/reliability-presenter.test.ts`

**Key Tests:**

- Error boundary detection
- Error handling patterns
- Resilience metrics
- Failure mode analysis
- Recovery strategies

```typescript
describe('ReliabilityPresenter', () => {
  // Test error boundary coverage
  it('should show error boundary coverage', () => {
    const result = createMockReliabilityResult({
      errorBoundaries: {
        count: 2,
        coverage: 85,
        missingBoundaries: ['Component1'],
      },
    });

    const presentation = presenter.present(result);
    const boundarySection = presentation.sections.find(s => s.id.includes('boundary'));

    expect(boundarySection).toBeDefined();
  });

  // Test error handling patterns
  it('should identify error handling patterns', () => {
    const result = createMockReliabilityResult({
      errorHandling: {
        tryCatchBlocks: 5,
        errorBoundaries: 2,
        fallbackComponents: 1,
        score: 80,
      },
    });

    const presentation = presenter.present(result);
    expect(presentation.sections.length).toBeGreaterThan(0);
  });
});
```

## Security Presenter Tests

**File:** `src/presenters/security-presenter.test.ts`

**Key Tests:**

- Vulnerability detection
- Security risk assessment
- XSS prevention
- Injection attack prevention
- Sensitive data exposure

```typescript
describe('SecurityPresenter', () => {
  // Test vulnerability detection
  it('should identify security vulnerabilities', () => {
    const result = createMockSecurityResult({
      vulnerabilities: [
        {
          type: 'xss',
          severity: 'high',
          location: { line: 15 },
          description: 'Potential XSS vulnerability',
        },
      ],
    });

    const presentation = presenter.present(result);
    const vulnerabilitySection = presentation.sections.find(s => s.id.includes('vulnerability'));

    expect(vulnerabilitySection?.items?.length).toBeGreaterThan(0);
  });

  // Test risk assessment
  it('should display security risk score', () => {
    const result = createMockSecurityResult({
      riskScore: 65,
      riskLevel: 'medium',
    });

    const presentation = presenter.present(result);
    expect(presentation.sections[0].score?.value).toBe(65);
  });
});
```

## Dataflow Presenter Tests

**File:** `src/presenters/dataflow-presenter.test.ts`

**Key Tests:**

- Data flow complexity
- Prop drilling detection
- State management patterns
- Data mutation tracking
- Unidirectional flow validation

```typescript
describe('DataflowPresenter', () => {
  // Test prop drilling detection
  it('should identify prop drilling issues', () => {
    const result = createMockDataflowResult({
      propDrilling: {
        depth: 5,
        affectedComponents: ['Component1', 'Component2'],
        score: 40,
      },
    });

    const presentation = presenter.present(result);
    const drillingSection = presentation.sections.find(s => s.id.includes('drilling'));

    expect(drillingSection).toBeDefined();
  });

  // Test state management patterns
  it('should analyze state management', () => {
    const result = createMockDataflowResult({
      stateManagement: {
        localState: 10,
        contextUsage: 3,
        externalState: 2,
        score: 75,
      },
    });

    const presentation = presenter.present(result);
    expect(presentation.sections.length).toBeGreaterThan(0);
  });
});
```

## Anti-Pattern Presenter Tests

**File:** `src/presenters/anti-pattern-presenter.test.ts`

**Key Tests:**

- Anti-pattern detection
- Code anti-pattern identification
- Refactoring suggestions
- Pattern violation severity
- Best practice compliance

```typescript
describe('AntiPatternPresenter', () => {
  // Test anti-pattern detection
  it('should identify anti-patterns', () => {
    const result = createMockAntiPatternResult({
      antiPatterns: [
        {
          type: 'god-component',
          severity: 'high',
          location: { line: 1 },
          description: 'Component has too many responsibilities',
        },
      ],
    });

    const presentation = presenter.present(result);
    const patternSection = presentation.sections.find(s => s.id.includes('pattern'));

    expect(patternSection?.items?.length).toBeGreaterThan(0);
  });

  // Test refactoring suggestions
  it('should provide refactoring suggestions', () => {
    const result = createMockAntiPatternResult({
      antiPatterns: [
        {
          type: 'prop-drilling',
          suggestion: 'Consider using Context API',
        },
      ],
    });

    const presentation = presenter.present(result);
    const items = presentation.sections.flatMap(s => s.items || []);

    expect(items.some(i => i.suggestion)).toBe(true);
  });
});
```

## Test Utilities

**File:** `src/presenters/__test-utils__/presenter-test-helpers.ts`

```typescript
/**
 * Shared test utilities for presenter tests
 */

import type { PluginResult, AnalysisResult } from '@vipr/common';
import { createAnalysisId } from '@vipr/common';

export function createMockPluginResult(overrides: Partial<PluginResult> = {}): PluginResult {
  return {
    pluginId: 'react',
    score: 75,
    insights: [],
    executionTimeMs: 100,
    metrics: {},
    analysisBreakdown: new Map(),
    ...overrides,
  };
}

export function createMockAnalysisResult(analysisId: string, data: any = {}): AnalysisResult {
  return {
    analysisId,
    category: 'test',
    data,
    insights: [],
    executionTimeMs: 50,
  };
}

export function createPluginResultWithAnalysis(analysisId: string, data: any = {}): PluginResult {
  const analysisResult = createMockAnalysisResult(analysisId, data);

  return createMockPluginResult({
    analysisBreakdown: new Map([[createAnalysisId(analysisId), analysisResult]]),
  });
}

export function createPerformancePluginResult(data: any = {}): PluginResult {
  return createPluginResultWithAnalysis('react-performance', {
    score: 75,
    renderPerformance: {
      expensiveOperations: 0,
      inlineFunctions: 0,
      inlineObjects: 0,
      inlineArrays: 0,
      score: 100,
      risk: 'low',
      ...data.renderPerformance,
    },
    memoization: {
      useCallbackCount: 0,
      useMemoCount: 0,
      reactMemoCount: 0,
      effectiveness: 100,
      unnecessaryMemoization: [],
      missingMemoization: [],
      ...data.memoization,
    },
    bundleImpact: {
      estimatedSize: 0,
      importCount: 0,
      heavyDependencies: [],
      treeshakingScore: 100,
      codeSplittingOpportunities: [],
      ...data.bundleImpact,
    },
    totalOpportunities: 0,
    ...data,
  });
}

export function createAccessibilityPluginResult(data: any = {}): PluginResult {
  return createPluginResultWithAnalysis('react-accessibility', {
    score: 85,
    wcagLevel: 'AA',
    violations: [],
    warnings: [],
    bestPractices: [],
    keyboardNavigationScore: 90,
    screenReaderCompatibility: 90,
    bySeverity: {},
    ...data,
  });
}

// Add more factory functions for other analysis types...
```

## Testing Checklist

For each presenter, ensure the following test coverage:

### Basic Functionality

- [ ] Has correct metadata (reportType, pluginId, analysisId)
- [ ] canPresent() accepts valid results
- [ ] canPresent() rejects invalid results
- [ ] present() returns valid presentation structure

### Data Transformation

- [ ] Transforms analysis data correctly
- [ ] Maps all required fields
- [ ] Applies correct formatting
- [ ] Handles optional fields

### Error Handling

- [ ] Handles missing data gracefully
- [ ] Handles undefined values
- [ ] Handles empty arrays/objects
- [ ] Handles invalid data types

### Score Calculation

- [ ] Displays correct score values
- [ ] Applies correct thresholds
- [ ] Rounds scores appropriately
- [ ] Handles score edge cases (0, 100, >100)

### Metric Formatting

- [ ] Formats counts correctly
- [ ] Formats percentages correctly
- [ ] Formats durations correctly
- [ ] Applies correct units

### Item Creation

- [ ] Maps severity levels correctly
- [ ] Creates items with correct structure
- [ ] Includes locations when available
- [ ] Includes suggestions when available

### Section Organization

- [ ] Creates logical section hierarchy
- [ ] Filters empty sections
- [ ] Orders sections appropriately
- [ ] Sets collapsible state correctly

## Next Steps

1. Start with `base-presenter.test.ts` - establishes patterns
2. Add `overview-presenter.test.ts` - most complex
3. Add remaining presenters in priority order
4. Create shared test utilities
5. Review coverage with `npm test -- --coverage`
6. Refactor common patterns into helpers

## Resources

- Base presenter source: `src/presenters/base-presenter.ts`
- Existing analysis tests: `src/analyses/*.test.ts`
- Testing utilities: `@vipr/testing`
- Type definitions: `@vipr/common`
