# Phase 4: Analyzer Refactoring

## Overview

Refactor the React analyzer's monolithic `analyze()` method into separate, focused analysis modules that register with the plugin system. This phase transforms a single 900+ line method handling five different analysis concerns into independent, testable, and parallelizable analysis implementations while maintaining full backward compatibility.

## Objectives

1. Split the monolithic `ReactAnalyzerPlugin.analyze()` method into separate analysis modules
2. Register each analysis with the plugin system using the `IAnalysis` interface
3. Enable parallel execution of analyses within the React plugin
4. Create a core analyzer plugin for framework-agnostic analyses
5. Eliminate code anti-patterns and technical debt during refactoring
6. Maintain 100% backward compatibility with existing metrics and results
7. Improve testability by creating focused, single-responsibility analysis modules
8. Enable selective analysis execution through configuration

## Technical Scope

### Anti-Patterns to Eliminate

**God Class Anti-Pattern**

- Current `ReactAnalyzerPlugin` has 1000+ lines handling 5+ distinct responsibilities
- Single method violates Single Responsibility Principle
- Difficult to test individual analysis dimensions in isolation
- Changes to one analysis risk breaking others

**Primitive Obsession**

- Analysis results passed around as raw objects
- No domain types for individual analysis results
- Missing type safety for analysis metadata

**Feature Envy**

- Main plugin reaching into utility functions spread across multiple files
- Analysis logic coupled to implementation details
- Violation of Tell, Don't Ask principle

**Long Method Anti-Pattern**

- `analyze()` method spans 300+ lines
- Nested helper methods (analyzeStructural, analyzeHooks, etc.) are 100+ lines each
- Complex cognitive load to understand full analysis flow

### Technical Debt to Address

1. **Tight Coupling**: Plugin directly instantiates and orchestrates all analyses
2. **Low Cohesion**: Unrelated analysis dimensions mixed in single class
3. **Poor Testability**: Cannot test analyses in isolation without full plugin
4. **Limited Extensibility**: Adding new analyses requires modifying plugin class
5. **No Separation of Concerns**: Scoring, aggregation, and analysis logic intertwined

## Refactoring Strategy

### Step 1: Extract Analysis Interface

Create the `IAnalysis` interface to standardize all analysis implementations.

```typescript
// common/types/src/analysis/index.ts

export interface IAnalysis<TResult = unknown, TConfig = unknown> {
  /** Unique identifier for this analysis */
  readonly id: string;

  /** Human-readable name */
  readonly name: string;

  /** Analysis category */
  readonly category: AnalysisCategory;

  /** Description of what this analysis does */
  readonly description: string;

  /** Whether this analysis is enabled by default */
  readonly defaultEnabled: boolean;

  /**
   * Execute the analysis on a source file
   */
  analyze(sourceFile: SourceFile, config?: TConfig): AnalysisResult<TResult>;

  /**
   * Validate configuration for this analysis
   */
  validateConfig?(config: TConfig): true | string;

  /**
   * Get default configuration
   */
  getDefaultConfig?(): TConfig;
}

export type AnalysisCategory =
  | 'migrations'
  | 'anti-patterns'
  | 'performance'
  | 'technical-debt'
  | 'security'
  | 'accessibility'
  | 'patterns';

export interface AnalysisResult<TResult = unknown> {
  /** Analysis identifier */
  analysisId: string;

  /** Analysis-specific results */
  data: TResult;

  /** Insights generated */
  insights: ComplexityInsight[];

  /** Optional score contribution (0-100) */
  score?: number;

  /** Execution time in milliseconds */
  executionTimeMs: number;

  /** Metadata */
  metadata?: Record<string, unknown>;
}
```

### Step 2: Extract Analysis Classes from Plugin

Use the **Extract Class** refactoring pattern to create focused analysis modules.

**Before (God Class):**

```typescript
// analyzers/react/src/plugin.ts - 1000+ lines
export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  analyze(sourceFile: SourceFile, config?: AnalyzerConfig): PluginResult {
    // 300+ lines orchestrating 5 different analyses
    const structural = this.analyzeStructural(sourceFile, insights);
    const hooks = this.analyzeHooks(sourceFile, insights);
    const temporal = this.analyzeTemporal(sourceFile, insights);
    const coupling = this.analyzeCoupling(sourceFile, insights);
    const identity = this.analyzeIdentity(sourceFile, insights);
    // ... complex aggregation logic
  }

  private analyzeStructural(...) { /* 120 lines */ }
  private analyzeHooks(...) { /* 150 lines */ }
  private analyzeTemporal(...) { /* 200 lines */ }
  private analyzeCoupling(...) { /* 120 lines */ }
  private analyzeIdentity(...) { /* 130 lines */ }
}
```

**After (Focused Classes):**

```typescript
// analyzers/react/src/analyses/structural-analysis.ts
export class StructuralAnalysis implements IAnalysis<StructuralComplexity> {
  readonly id = 'react-structural';
  readonly name = 'React Structural Complexity';
  readonly category = 'technical-debt';
  readonly description = 'Analyzes branching, conditionals, and control flow in React components';
  readonly defaultEnabled = true;

  analyze(sourceFile: SourceFile): AnalysisResult<StructuralComplexity> {
    const startTime = performance.now();
    const insights: ComplexityInsight[] = [];

    const data = this.analyzeStructural(sourceFile, insights);
    const score = this.calculateScore(data);

    return {
      analysisId: this.id,
      data,
      insights,
      score,
      executionTimeMs: performance.now() - startTime,
    };
  }

  private analyzeStructural(
    sourceFile: SourceFile,
    insights: ComplexityInsight[]
  ): StructuralComplexity {
    // Extracted logic from plugin
  }

  private calculateScore(data: StructuralComplexity): number {
    // Isolated scoring logic
  }
}
```

### Step 3: Update Plugin to Use Analysis Registry

Transform plugin from executor to coordinator using the **Strategy Pattern**.

```typescript
// analyzers/react/src/plugin.ts
export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  readonly id = 'react';
  readonly name = 'React Analyzer';
  readonly version = '1.0.0';
  readonly filePatterns = ['**/*.tsx', '**/*.jsx'];

  private analyses: Map<string, IAnalysis> = new Map();

  constructor() {
    this.registerAnalyses();
  }

  private registerAnalyses(): void {
    // Register all React-specific analyses
    this.registerAnalysis(new StructuralAnalysis());
    this.registerAnalysis(new HookAnalysis());
    this.registerAnalysis(new TemporalAnalysis());
    this.registerAnalysis(new CouplingAnalysis());
    this.registerAnalysis(new IdentityAnalysis());
    this.registerAnalysis(new AccessibilityAnalysis());
  }

  private registerAnalysis(analysis: IAnalysis): void {
    this.analyses.set(analysis.id, analysis);
  }

  getAnalyses(): IAnalysis[] {
    return Array.from(this.analyses.values());
  }

  async runAnalysis(
    analysisId: string,
    sourceFile: SourceFile,
    config?: unknown
  ): Promise<AnalysisResult> {
    const analysis = this.analyses.get(analysisId);
    if (!analysis) {
      throw new Error(`Unknown analysis: ${analysisId}`);
    }

    return analysis.analyze(sourceFile, config);
  }

  analyze(sourceFile: SourceFile, config?: AnalyzerConfig): PluginResult {
    const startTime = performance.now();

    // Get enabled analyses
    const enabledAnalyses = this.getEnabledAnalyses(config);

    // Run analyses in parallel
    const analysisResults = await Promise.all(
      enabledAnalyses.map(analysis =>
        this.runAnalysis(analysis.id, sourceFile, config?.analyses?.[analysis.id])
      )
    );

    // Aggregate results
    return this.aggregateResults(analysisResults, startTime);
  }

  private getEnabledAnalyses(config?: AnalyzerConfig): IAnalysis[] {
    return Array.from(this.analyses.values()).filter(analysis => {
      const analysisConfig = config?.analyses?.[analysis.id];
      return analysisConfig?.enabled ?? analysis.defaultEnabled;
    });
  }

  private aggregateResults(results: AnalysisResult[], startTime: number): PluginResult {
    // Aggregate insights
    const insights = results.flatMap(r => r.insights);

    // Aggregate metrics
    const metrics = results.reduce(
      (acc, result) => {
        acc[result.analysisId] = result.data;
        return acc;
      },
      {} as Record<string, unknown>
    );

    // Calculate composite score
    const score = this.calculateCompositeScore(results);

    return {
      pluginId: this.id,
      score,
      insights,
      metrics,
      executionTimeMs: performance.now() - startTime,
    };
  }
}
```

### Step 4: Create Core Analyzer Plugin

Extract framework-agnostic analyses into a reusable core plugin.

```typescript
// analyzers/core/src/plugin.ts
export class CoreAnalyzerPlugin implements ITechnologyPlugin {
  readonly id = 'core';
  readonly name = 'Core Analyzer';
  readonly version = '1.0.0';
  readonly filePatterns = ['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx'];

  private analyses: Map<string, IAnalysis> = new Map();

  constructor() {
    this.registerAnalyses();
  }

  private registerAnalyses(): void {
    this.registerAnalysis(new CyclomaticComplexityAnalysis());
    this.registerAnalysis(new HalsteadMetricsAnalysis());
    this.registerAnalysis(new CognitiveComplexityAnalysis());
  }

  canHandle(sourceFile: SourceFile): boolean {
    // Handles all JavaScript/TypeScript files
    return /\.(ts|tsx|js|jsx)$/.test(sourceFile.getFilePath());
  }

  // ... similar structure to ReactAnalyzerPlugin
}
```

## Code Patterns

### Pattern 1: Extract Class Refactoring

**Objective**: Break down the god class into focused, cohesive classes.

**Before**:

```typescript
export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  analyze(sourceFile: SourceFile): PluginResult {
    const insights: ComplexityInsight[] = [];
    const structural = this.analyzeStructural(sourceFile, insights);
    const hooks = this.analyzeHooks(sourceFile, insights);
    // ... 5 more analyses
    // Complex aggregation logic
    return { ... };
  }

  private analyzeStructural(sourceFile: SourceFile, insights: ComplexityInsight[]) {
    // 120 lines of structural analysis
  }

  private analyzeHooks(sourceFile: SourceFile, insights: ComplexityInsight[]) {
    // 150 lines of hook analysis
  }
}
```

**After**:

```typescript
// analyzers/react/src/analyses/structural-analysis.ts
export class StructuralAnalysis implements IAnalysis<StructuralComplexity> {
  readonly id = 'react-structural';
  readonly name = 'React Structural Complexity';
  readonly category = 'technical-debt';

  analyze(sourceFile: SourceFile): AnalysisResult<StructuralComplexity> {
    const insights: ComplexityInsight[] = [];
    const data = this.computeStructuralMetrics(sourceFile, insights);
    return {
      analysisId: this.id,
      data,
      insights,
      score: this.calculateScore(data),
      executionTimeMs: 0,
    };
  }

  private computeStructuralMetrics(
    sourceFile: SourceFile,
    insights: ComplexityInsight[]
  ): StructuralComplexity {
    // Focused structural analysis logic
  }
}
```

**Benefits**:

- Each class has single responsibility
- Testable in isolation
- Reusable across plugins
- Clear boundaries

### Pattern 2: Strategy Pattern for Analysis Execution

**Objective**: Enable runtime selection and parallel execution of analyses.

**Implementation**:

```typescript
export class ReactAnalyzerPlugin implements ITechnologyPlugin {
  private analyses: Map<string, IAnalysis> = new Map();

  async analyze(sourceFile: SourceFile, config?: AnalyzerConfig): Promise<PluginResult> {
    // Select enabled analyses based on config
    const enabledAnalyses = this.getEnabledAnalyses(config);

    // Execute strategies in parallel
    const results = await Promise.all(
      enabledAnalyses.map(analysis => analysis.analyze(sourceFile, config?.analyses?.[analysis.id]))
    );

    // Aggregate using template method
    return this.aggregateResults(results);
  }
}
```

**Benefits**:

- Analyses are interchangeable
- Easy to add/remove analyses
- Parallel execution out of the box
- Configuration-driven

### Pattern 3: Introduce Domain Types

**Objective**: Replace primitive obsession with meaningful domain types.

**Before**:

```typescript
analyze(sourceFile: SourceFile): PluginResult {
  const structural = this.analyzeStructural(sourceFile);
  // structural is just a plain object
  const score = structural.score;
  const branches = structural.branches;
  // ...
}
```

**After**:

```typescript
class StructuralComplexityResult {
  constructor(
    public readonly score: number,
    public readonly branches: number,
    public readonly jsxConditionals: number,
    public readonly earlyReturns: number,
    public readonly loops: number,
    public readonly logicalOperators: number
  ) {}

  isHighComplexity(): boolean {
    return this.score > THRESHOLDS.structural.warning;
  }

  getInsights(): ComplexityInsight[] {
    const insights: ComplexityInsight[] = [];
    if (this.isHighComplexity()) {
      insights.push({
        severity: 'warning',
        category: 'structural',
        message: `High structural complexity (${this.score})`,
        suggestion: 'Break down complex conditional rendering',
      });
    }
    return insights;
  }
}
```

**Benefits**:

- Type safety
- Encapsulated business logic
- Self-documenting
- Testable behavior

## File Changes

### New Files to Create

#### 1. Analysis Interface

**File**: `common/types/src/analysis/index.ts`

```typescript
export interface IAnalysis<TResult = unknown, TConfig = unknown> { ... }
export interface AnalysisResult<TResult = unknown> { ... }
export type AnalysisCategory = ...;
```

#### 2. Structural Analysis

**File**: `analyzers/react/src/analyses/structural-analysis.ts`

- Extract structural complexity analysis
- 120 lines focused on branches, conditionals, JSX patterns
- Includes scoring and insight generation

#### 3. Hook Analysis

**File**: `analyzers/react/src/analyses/hook-analysis.ts`

- Extract hook complexity analysis
- 150 lines focused on useState, useEffect, custom hooks
- Dependency tracking and weighting

#### 4. Temporal Analysis

**File**: `analyzers/react/src/analyses/temporal-analysis.ts`

- Extract temporal complexity analysis
- 200 lines focused on effects, lifecycles, subscriptions
- Risk detection for cleanup patterns

#### 5. Coupling Analysis

**File**: `analyzers/react/src/analyses/coupling-analysis.ts`

- Extract coupling complexity analysis
- 120 lines focused on props, context, children patterns
- Dependency injection analysis

#### 6. Identity Analysis

**File**: `analyzers/react/src/analyses/identity-analysis.ts`

- Extract identity/reference stability analysis
- 130 lines focused on useCallback, useMemo, inline references
- Memoization pattern detection

#### 7. Accessibility Analysis

**File**: `analyzers/react/src/analyses/accessibility-analysis.ts`

- New analysis for a11y patterns
- ARIA attributes, semantic HTML, keyboard navigation
- WCAG compliance checks

#### 8. Analysis Index

**File**: `analyzers/react/src/analyses/index.ts`

```typescript
export { StructuralAnalysis } from './structural-analysis';
export { HookAnalysis } from './hook-analysis';
export { TemporalAnalysis } from './temporal-analysis';
export { CouplingAnalysis } from './coupling-analysis';
export { IdentityAnalysis } from './identity-analysis';
export { AccessibilityAnalysis } from './accessibility-analysis';
```

#### 9. Core Analyzer Plugin

**File**: `analyzers/core/src/plugin.ts`

- New plugin for framework-agnostic analyses
- Cyclomatic complexity, Halstead metrics
- Base anti-patterns (generic code anti-patterns)

#### 10. Core Analyses

**Files**:

- `analyzers/core/src/analyses/cyclomatic-analysis.ts`
- `analyzers/core/src/analyses/halstead-analysis.ts`
- `analyzers/core/src/analyses/cognitive-analysis.ts`
- `analyzers/core/src/analyses/index.ts`

### Files to Modify

#### 1. React Plugin

**File**: `analyzers/react/src/plugin.ts`

**Changes**:

- Remove private analysis methods (720 lines)
- Add analysis registry
- Add `getAnalyses()` method
- Add `runAnalysis()` method
- Update `analyze()` to coordinate registered analyses
- Add parallel execution logic
- Maintain backward compatibility in result format

**Size**: Reduce from 1000+ lines to ~200 lines

#### 2. Plugin Interface

**File**: `common/types/src/plugin/index.ts`

**Changes**:

```typescript
export interface ITechnologyPlugin {
  // ... existing properties

  /**
   * Get all analyses provided by this plugin
   */
  getAnalyses(): IAnalysis[];

  /**
   * Run a specific analysis by ID
   */
  runAnalysis(
    analysisId: string,
    sourceFile: SourceFile,
    config?: unknown
  ): Promise<AnalysisResult>;
}
```

#### 3. Analyzer Config

**File**: `common/types/src/core/index.ts`

**Changes**:

```typescript
export interface AnalyzerConfig {
  // ... existing config

  /**
   * Analysis-specific configuration
   */
  analyses?: Record<
    string,
    {
      enabled?: boolean;
      config?: unknown;
    }
  >;
}
```

#### 4. Analysis Engine

**File**: `analyzers/core/src/engine/analysis-engine.ts`

**Changes**:

- Update to support analysis-level parallelization
- Add analysis result aggregation
- Maintain plugin result format for backward compatibility

## Dependencies

### Prerequisite Phases

**Phase 1: Type System & Interfaces** (Complete)

- `IAnalysis` interface must exist
- `AnalysisResult` type must be defined
- Analysis category enum must be defined

**Phase 2: Plugin Discovery & Loading** (Not required)

- This phase is independent of plugin discovery
- Can proceed without plugin loader

**Phase 3: Engine Enhancements** (Not required)

- Can refactor without engine changes
- Engine updates can come after

### Internal Dependencies

1. **Analysis Interface** must be created before analysis classes
2. **Analysis Classes** must be extracted before updating plugin
3. **Plugin Updates** must maintain backward compatibility
4. **Core Plugin** can be developed in parallel with React refactoring

### External Dependencies

- `ts-morph` for AST analysis
- `@vipr/types` for shared types
- `@vipr/core` for utilities (scoring, normalization)

## Acceptance Criteria

### Functional Requirements

1. All existing analyses produce identical results to current implementation
2. All analysis dimensions (structural, hooks, temporal, coupling, identity) work independently
3. Analyses execute in parallel within the plugin
4. Plugin aggregates analysis results into the same format as before
5. Configuration allows enabling/disabling individual analyses
6. Core analyzer plugin provides framework-agnostic analyses
7. Accessibility analysis detects basic a11y violations

### Code Quality Requirements

1. Each analysis class is under 200 lines
2. Each analysis has a single, well-defined responsibility
3. Analysis classes are independent and have no cross-dependencies
4. All code anti-patterns identified in Technical Scope are eliminated
5. Cyclomatic complexity of plugin class reduced by 80%
6. Test coverage for each analysis is at least 90%

### Performance Requirements

1. Parallel execution completes in 70% of sequential time
2. Individual analysis execution time is tracked
3. No performance regression compared to current implementation
4. Memory usage does not increase by more than 10%

### Backward Compatibility Requirements

1. Plugin result format unchanged
2. All existing metrics available in same structure
3. Insight format unchanged
4. Score calculation produces identical results
5. Existing tests pass without modification

### Testing Requirements

1. Unit tests for each analysis class
2. Integration tests for plugin coordination
3. Regression tests comparing old vs new results
4. Performance benchmarks for parallel execution
5. Configuration tests for enabling/disabling analyses

### Documentation Requirements

1. Each analysis class has comprehensive JSDoc
2. Analysis interface documented with examples
3. Migration guide for adding new analyses
4. Architecture diagram showing analysis flow
5. Performance characteristics documented

## Recommended Claude Model

**Primary Model**: Claude Opus 4.5

- Complex refactoring requiring deep understanding of code structure
- Multiple interconnected changes across many files
- Need for careful preservation of behavior during extraction
- High risk of introducing subtle bugs

**Secondary Model**: Claude Sonnet 4.5

- Suitable for creating individual analysis classes after extraction strategy is defined
- Good for implementing tests
- Can handle documentation tasks

## Assigned Subagents

### Lead Refactoring Agent

**Model**: Opus 4.5
**Responsibilities**:

- Plan extraction strategy
- Identify code anti-patterns and technical debt
- Define refactoring sequence to avoid breaking changes
- Review all code changes for safety

### Analysis Extraction Agent

**Model**: Opus 4.5
**Responsibilities**:

- Extract analysis classes from plugin
- Ensure extracted code is functionally identical
- Create domain types for analysis results
- Implement scoring logic in analysis classes

### Testing Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Create unit tests for each analysis
- Build regression test suite
- Implement performance benchmarks
- Verify backward compatibility

### Documentation Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Document analysis interfaces
- Create architecture diagrams
- Write migration guides
- Update JSDoc comments

## Risk Mitigation

### High-Risk Areas

1. **Behavioral Changes**: Extracting analysis logic may introduce subtle differences
   - **Mitigation**: Extensive regression testing, side-by-side comparison

2. **Parallel Execution Bugs**: Race conditions or shared state issues
   - **Mitigation**: Ensure analyses are pure functions, no shared mutable state

3. **Scoring Differences**: Changes in aggregation may alter final scores
   - **Mitigation**: Pin test cases with known scores, validate identical results

4. **Performance Regression**: Overhead from abstraction layers
   - **Mitigation**: Performance benchmarks, profiling before/after

### Breaking Change Prevention

1. Maintain exact result structure in plugin
2. Keep all existing public APIs unchanged
3. Use feature flags to gradually migrate to new system
4. Provide legacy adapter if needed

### Rollback Strategy

1. Keep original plugin code in separate file as reference
2. Use version control tags before major changes
3. Maintain backward compatibility adapter
4. Implement progressive rollout with feature flags

## Metrics for Success

### Code Quality Metrics

- Lines per class: Target < 200 (currently 900+)
- Cyclomatic complexity: Reduce by 80%
- Method length: Target < 50 lines per method
- Class cohesion: Increase to > 0.8 (LCOM4)
- Coupling: Reduce afferent coupling by 60%

### Performance Metrics

- Parallel execution speedup: Target 30% faster
- Memory overhead: Keep under 10% increase
- Analysis isolation overhead: < 5ms per analysis
- Total execution time: No regression

### Testing Metrics

- Unit test coverage: > 90% per analysis
- Integration test coverage: > 85%
- Regression test suite: 100% passing
- Performance test baseline: Established

### Maintainability Metrics

- Time to add new analysis: Reduce from 4 hours to 30 minutes
- Bug fix cycle time: Reduce by 50%
- Code review time: Reduce by 60%
- Developer onboarding: Reduce from 2 days to 4 hours
