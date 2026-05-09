# Phase 1: Type System & Interfaces

## Overview

This phase establishes the foundational type system for the plugin architecture refactor. It introduces the `IAnalysis` interface to enable fine-grained analysis registration within plugins, and enhances the `ITechnologyPlugin` interface to support multiple discrete analyses running in parallel. These type definitions provide the structural contract that all subsequent phases depend upon.

## Objectives

1. Design and implement the `IAnalysis<TConfig, TResult>` generic interface for analysis registration
2. Extend `ITechnologyPlugin` with analysis discovery and execution methods
3. Create discriminated unions for standard analysis categories
4. Define type-safe result aggregation structures
5. Ensure backward compatibility with existing plugin implementations
6. Establish type patterns for extensibility

## Technical Scope

### 1. Analysis Interface Design

The core abstraction is `IAnalysis`, which represents a discrete unit of analysis work within a plugin. Each analysis:

- Has a unique identifier and category
- Can be enabled/disabled independently
- Declares its configuration schema via generic type parameter
- Returns strongly-typed results via generic type parameter
- Supports both synchronous and asynchronous execution

### 2. Analysis Categories

Standard analysis categories form a discriminated union, enabling exhaustive pattern matching:

- `migrations` - Migration readiness assessment
- `anti-patterns` - Anti-pattern detection
- `performance` - Performance optimization opportunities
- `technical-debt` - Technical debt hotspots
- `security` - Security vulnerability scanning
- `accessibility` - Accessibility compliance (React-specific)
- `patterns` - Language pattern analysis (TypeScript-specific)

### 3. Plugin Interface Enhancement

The `ITechnologyPlugin` interface gains two new methods:

- `getAnalyses(): IAnalysis[]` - Returns all registered analyses
- `runAnalysis(analysisId, sourceFile, config): Promise<AnalysisResult>` - Executes a specific analysis

These additions are non-breaking. Existing plugins without these methods continue to work via the existing `analyze()` method.

### 4. Result Aggregation Types

Results from parallel analyses must be aggregated with type safety:

- `AnalysisResult<T>` - Generic result wrapper with metadata
- `PluginResult` extended with `analysisBreakdown?: Map<string, AnalysisResult>`
- `AggregatedResult` maintains existing structure, enhanced with analysis-level detail

## Type Definitions

### Core Analysis Interface

```typescript
/**
 * Generic interface for discrete analysis units within a plugin.
 *
 * Type parameters:
 * - TConfig: Configuration schema for this analysis (use `never` if no config)
 * - TResult: Result data structure returned by this analysis
 * - TMetrics: Optional metrics structure for analysis-specific measurements
 */
export interface IAnalysis<TConfig = unknown, TResult = unknown, TMetrics = unknown> {
  /** Unique identifier for this analysis (e.g., "react-accessibility") */
  readonly id: string;

  /** Analysis category (enables grouping and filtering) */
  readonly category: AnalysisCategory;

  /** Human-readable name */
  readonly name: string;

  /** Brief description of what this analysis detects/measures */
  readonly description: string;

  /** Version of this analysis (semantic versioning) */
  readonly version: string;

  /** Whether this analysis is enabled by default */
  readonly enabledByDefault: boolean;

  /** Estimated execution cost (1-5, where 5 is most expensive) */
  readonly executionCost: ExecutionCost;

  /**
   * Validate configuration for this analysis.
   * Called before execution to ensure config schema compliance.
   */
  validateConfig(config: TConfig): true | string;

  /**
   * Execute the analysis on the provided source file.
   *
   * @param sourceFile - ts-morph SourceFile to analyze
   * @param config - Type-safe configuration for this analysis
   * @returns Result containing insights, metrics, and score
   */
  execute(
    sourceFile: SourceFile,
    config?: TConfig
  ): AnalysisResult<TResult, TMetrics> | Promise<AnalysisResult<TResult, TMetrics>>;

  /**
   * Get default configuration for this analysis.
   * Used when no explicit config provided.
   */
  getDefaultConfig(): TConfig;
}
```

### Analysis Categories

```typescript
/**
 * Standard analysis categories.
 * Using string literal union for exhaustive type checking.
 */
export type AnalysisCategory =
  | 'migrations'
  | 'anti-patterns'
  | 'performance'
  | 'technical-debt'
  | 'security'
  | 'accessibility'
  | 'patterns';

/**
 * Execution cost hint for scheduling and parallelization.
 */
export type ExecutionCost = 1 | 2 | 3 | 4 | 5;

/**
 * Type guard for analysis category validation.
 */
export function isAnalysisCategory(value: string): value is AnalysisCategory {
  return [
    'migrations',
    'anti-patterns',
    'performance',
    'technical-debt',
    'security',
    'accessibility',
    'patterns',
  ].includes(value);
}
```

### Analysis Result Types

```typescript
/**
 * Result returned by an analysis execution.
 * Generic over result data and metrics.
 */
export interface AnalysisResult<TResult = unknown, TMetrics = unknown> {
  /** Analysis identifier that produced this result */
  analysisId: string;

  /** Analysis category */
  category: AnalysisCategory;

  /** Optional score contribution (0-100, higher is better) */
  score?: number;

  /** Insights generated by this analysis */
  insights: ComplexityInsight[];

  /** Analysis-specific result data */
  data: TResult;

  /** Analysis-specific metrics */
  metrics?: TMetrics;

  /** Execution time in milliseconds */
  executionTimeMs: number;

  /** Warnings encountered (non-fatal) */
  warnings?: string[];
}

/**
 * Branded type for analysis IDs to prevent string confusion.
 */
export type AnalysisId = string & { readonly __brand: 'AnalysisId' };

/**
 * Helper to create branded AnalysisId.
 */
export function createAnalysisId(id: string): AnalysisId {
  return id as AnalysisId;
}
```

### Enhanced Plugin Interface

```typescript
/**
 * Enhanced ITechnologyPlugin interface with analysis registration support.
 * Maintains backward compatibility - new methods are optional.
 */
export interface ITechnologyPlugin {
  // Existing members (unchanged)
  readonly id: string;
  readonly name: string;
  readonly version: string;
  readonly filePatterns: string[];
  canHandle(sourceFile: SourceFile): boolean;
  analyze(sourceFile: SourceFile, config?: AnalyzerConfig): PluginResult | Promise<PluginResult>;
  getRules(): Rule[];
  onRegister?(): void | Promise<void>;
  onUnregister?(): void | Promise<void>;
  validateConfig?(config: Record<string, unknown>): true | string;
  getDefaultConfig?(): Record<string, unknown>;

  // New methods for analysis registration
  /**
   * Get all analyses registered by this plugin.
   * Enables engine to discover and selectively execute analyses.
   *
   * @returns Array of registered analyses (empty array if none)
   */
  getAnalyses?(): IAnalysis[];

  /**
   * Execute a specific analysis by ID.
   * Provides fine-grained control over analysis execution.
   *
   * @param analysisId - ID of the analysis to run
   * @param sourceFile - Source file to analyze
   * @param config - Optional configuration for the analysis
   * @returns Promise resolving to analysis result
   * @throws Error if analysisId is not registered
   */
  runAnalysis?(
    analysisId: AnalysisId,
    sourceFile: SourceFile,
    config?: AnalyzerConfig
  ): Promise<AnalysisResult>;
}
```

### Enhanced Result Types

```typescript
/**
 * Extended PluginResult with analysis breakdown.
 * Maintains backward compatibility - analysisBreakdown is optional.
 */
export interface PluginResult<T = unknown> {
  pluginId: string;
  metrics: T;
  insights: PluginInsight[];
  score?: number;
  weight?: number;
  executionTimeMs?: number;
  warnings?: string[];

  /**
   * Optional breakdown of results by analysis.
   * Populated when plugin uses analysis registration system.
   */
  analysisBreakdown?: Map<AnalysisId, AnalysisResult>;
}

/**
 * Analysis execution error with context.
 */
export interface AnalysisError {
  analysisId: AnalysisId;
  category: AnalysisCategory;
  error: Error;
  sourceFile?: string;
}

/**
 * Enhanced AggregatedResult (existing interface, shown for context).
 * No changes needed - already supports Map-based plugin results.
 */
export interface AggregatedResult {
  filePath: string;
  analyzedAt: string;
  overallScore: number;
  overallGrade: string;
  pluginResults: Map<string, PluginResult>;
  insights: ComplexityInsight[];
  errors: PluginError[];
}
```

### Helper Types for Implementation

```typescript
/**
 * Base abstract class for implementing analyses.
 * Reduces boilerplate and enforces patterns.
 */
export abstract class BaseAnalysis<
  TConfig = unknown,
  TResult = unknown,
  TMetrics = unknown,
> implements IAnalysis<TConfig, TResult, TMetrics> {
  abstract readonly id: string;
  abstract readonly category: AnalysisCategory;
  abstract readonly name: string;
  abstract readonly description: string;
  abstract readonly version: string;
  abstract readonly enabledByDefault: boolean;
  abstract readonly executionCost: ExecutionCost;

  abstract execute(
    sourceFile: SourceFile,
    config?: TConfig
  ): AnalysisResult<TResult, TMetrics> | Promise<AnalysisResult<TResult, TMetrics>>;

  /**
   * Default implementation returns true (no validation).
   * Override to implement schema validation.
   */
  validateConfig(config: TConfig): true | string {
    return true;
  }

  /**
   * Default implementation returns empty object cast to TConfig.
   * Override to provide meaningful defaults.
   */
  getDefaultConfig(): TConfig {
    return {} as TConfig;
  }

  /**
   * Utility to create result object with standard fields.
   */
  protected createResult(
    data: TResult,
    options: {
      score?: number;
      insights?: ComplexityInsight[];
      metrics?: TMetrics;
      executionTimeMs: number;
      warnings?: string[];
    }
  ): AnalysisResult<TResult, TMetrics> {
    return {
      analysisId: this.id,
      category: this.category,
      data,
      score: options.score,
      insights: options.insights ?? [],
      metrics: options.metrics,
      executionTimeMs: options.executionTimeMs,
      warnings: options.warnings,
    };
  }
}

/**
 * Type utility for extracting config type from IAnalysis.
 */
export type AnalysisConfig<T> = T extends IAnalysis<infer C, unknown, unknown> ? C : never;

/**
 * Type utility for extracting result type from IAnalysis.
 */
export type AnalysisResultData<T> = T extends IAnalysis<unknown, infer R, unknown> ? R : never;

/**
 * Type utility for extracting metrics type from IAnalysis.
 */
export type AnalysisMetrics<T> = T extends IAnalysis<unknown, unknown, infer M> ? M : never;
```

## File Changes

### Files to Create

#### `common/types/src/analysis/IAnalysis.ts`

New file containing:

- `IAnalysis<TConfig, TResult, TMetrics>` interface
- `AnalysisCategory` type and related types
- `ExecutionCost` type
- `AnalysisResult<TResult, TMetrics>` interface
- `AnalysisId` branded type and factory
- `AnalysisError` interface
- `BaseAnalysis` abstract class
- Type utilities (`AnalysisConfig`, `AnalysisResultData`, `AnalysisMetrics`)
- Type guard `isAnalysisCategory`

#### `common/types/src/analysis/categories.ts`

New file for analysis category constants and metadata:

```typescript
/**
 * Metadata for standard analysis categories.
 */
export interface AnalysisCategoryMetadata {
  id: AnalysisCategory;
  name: string;
  description: string;
  applicableTo: ('react' | 'typescript' | 'javascript' | 'all')[];
}

/**
 * Metadata registry for standard categories.
 */
export const ANALYSIS_CATEGORIES: Record<AnalysisCategory, AnalysisCategoryMetadata> = {
  migrations: {
    id: 'migrations',
    name: 'Migration Readiness',
    description: 'Assess readiness for framework/library version upgrades',
    applicableTo: ['all'],
  },
  'anti-patterns': {
    id: 'anti-patterns',
    name: 'Anti-Pattern Detection',
    description: 'Identify code patterns that violate best practices',
    applicableTo: ['all'],
  },
  performance: {
    id: 'performance',
    name: 'Performance Analysis',
    description: 'Detect performance bottlenecks and optimization opportunities',
    applicableTo: ['all'],
  },
  'technical-debt': {
    id: 'technical-debt',
    name: 'Technical Debt',
    description: 'Identify code quality issues and maintainability concerns',
    applicableTo: ['all'],
  },
  security: {
    id: 'security',
    name: 'Security Analysis',
    description: 'Scan for security vulnerabilities and unsafe patterns',
    applicableTo: ['all'],
  },
  accessibility: {
    id: 'accessibility',
    name: 'Accessibility',
    description: 'Validate WCAG compliance and accessibility best practices',
    applicableTo: ['react'],
  },
  patterns: {
    id: 'patterns',
    name: 'Language Patterns',
    description: 'Analyze language-specific patterns and idioms',
    applicableTo: ['typescript'],
  },
} as const;
```

### Files to Modify

#### `common/types/src/analysis/index.ts`

Add exports:

```typescript
// Existing exports (unchanged)
export * from './security';
export * from './accessibility';
export * from './technical-debt';
export * from './performance';

// New exports
export * from './IAnalysis';
export * from './categories';
```

#### `common/types/src/plugin/index.ts`

Update imports and interface:

```typescript
// Add to imports
import type { IAnalysis, AnalysisResult, AnalysisId } from '../analysis';

// Update ITechnologyPlugin interface - add new optional methods
export interface ITechnologyPlugin {
  // ... existing members ...

  getAnalyses?(): IAnalysis[];

  runAnalysis?(
    analysisId: AnalysisId,
    sourceFile: SourceFile,
    config?: AnalyzerConfig
  ): Promise<AnalysisResult>;
}

// Update PluginResult interface - add optional field
export interface PluginResult<T = unknown> {
  // ... existing members ...

  analysisBreakdown?: Map<AnalysisId, AnalysisResult>;
}
```

## Dependencies

- **None** - This is the foundational phase that other phases depend on
- Requires only existing dependencies: `ts-morph` for `SourceFile` type

## Acceptance Criteria

### Type Safety

- [ ] `IAnalysis` generic interface correctly constrains config and result types
- [ ] `AnalysisCategory` discriminated union enables exhaustive pattern matching
- [ ] `AnalysisId` branded type prevents accidental string assignment
- [ ] Type utilities correctly extract generic type parameters
- [ ] No use of `any` or `unknown` escape hatches in public API

### Backward Compatibility

- [ ] Existing `ITechnologyPlugin` implementations compile without modification
- [ ] New methods (`getAnalyses`, `runAnalysis`) are optional
- [ ] `PluginResult.analysisBreakdown` is optional
- [ ] No breaking changes to `AggregatedResult`, `PluginInsight`, or `ComplexityInsight`

### Extensibility

- [ ] `AnalysisCategory` can be extended via string literal union
- [ ] Custom analysis categories supported (though not in standard set)
- [ ] `IAnalysis` generics support arbitrary config/result shapes
- [ ] `BaseAnalysis` abstract class reduces boilerplate

### Code Quality

- [ ] All public interfaces have TSDoc comments with `@example` blocks
- [ ] Generic type parameters documented with clear constraints
- [ ] Variance of generic parameters considered (all covariant in this design)
- [ ] No circular type dependencies

### Compilation

- [ ] `common/types` package builds successfully with strict mode
- [ ] No TypeScript errors with `strictNullChecks`, `noUncheckedIndexedAccess`
- [ ] Type exports correctly available to consumers

### Testing

- [ ] Unit tests for type guards (`isAnalysisCategory`)
- [ ] Unit tests for `createAnalysisId` factory
- [ ] Unit tests for `BaseAnalysis` default implementations
- [ ] Type tests ensuring generic constraint enforcement

### Documentation

- [ ] TSDoc comments on all exported types
- [ ] Code examples in comments demonstrate usage patterns
- [ ] Type parameter constraints explained
- [ ] Migration guide from old plugin pattern to new (conceptual, no code changes needed)

## Recommended Claude Model

**Claude Sonnet 4.5**

Reasoning:

- Complex type system design requires deep TypeScript expertise
- Generic type constraints and variance require careful consideration
- Backward compatibility constraints demand architectural thinking
- Type utility design requires understanding of mapped types and conditional types
- This phase sets the foundation - errors here cascade to all subsequent phases

## Assigned Subagents

### Primary: `@agent-typescript-engineer`

Responsibilities:

- Design `IAnalysis` interface with appropriate generic constraints
- Ensure variance correctness in generic parameters
- Create type utilities for config/result extraction
- Design branded types for type safety

Skills Required:

- Advanced TypeScript generics
- Mapped types and conditional types
- Type inference and variance
- Structural typing patterns

## Implementation Notes

### Generic Constraint Considerations

The `IAnalysis` interface uses three generic parameters with default `unknown`:

```typescript
IAnalysis<TConfig = unknown, TResult = unknown, TMetrics = unknown>
```

**Why `unknown` instead of `never` or `void`?**

- `unknown` allows type narrowing and is safer than `any`
- `never` would prevent assignment of any concrete type
- `void` is semantically incorrect (these are data, not absence of return)

Consumers can override:

```typescript
// No config needed
interface NoConfigAnalysis extends IAnalysis<never, MyResult, MyMetrics> {}

// Simple config
interface SimpleAnalysis extends IAnalysis<{ threshold: number }, MyResult> {}

// Complex config with branded types
type AccessibilityConfig = {
  wcagLevel: 'A' | 'AA' | 'AAA';
  checkAriaRoles: boolean;
};
interface AccessibilityAnalysis extends IAnalysis<AccessibilityConfig, A11yResult> {}
```

### Branded Type Pattern

`AnalysisId` uses the branded type pattern to prevent string confusion:

```typescript
type AnalysisId = string & { readonly __brand: 'AnalysisId' };
```

This prevents accidental usage:

```typescript
function runAnalysis(id: AnalysisId) {}

runAnalysis('some-string'); // Error: not an AnalysisId
runAnalysis(createAnalysisId('some-string')); // OK
```

### Optional Method Pattern

Adding methods to `ITechnologyPlugin` without breaking changes:

```typescript
interface ITechnologyPlugin {
  // Existing required methods...

  // New optional methods
  getAnalyses?(): IAnalysis[];
  runAnalysis?(id: AnalysisId, ...): Promise<AnalysisResult>;
}
```

The engine will check for method presence:

```typescript
if (plugin.getAnalyses) {
  const analyses = plugin.getAnalyses();
  // New path: run analyses in parallel
} else {
  // Legacy path: call plugin.analyze() directly
}
```

### Discriminated Union Pattern

`AnalysisCategory` enables exhaustive pattern matching:

```typescript
function processResult(result: AnalysisResult): void {
  switch (result.category) {
    case 'migrations':
      // TypeScript knows category is 'migrations'
      break;
    case 'anti-patterns':
      break;
    case 'performance':
      break;
    case 'technical-debt':
      break;
    case 'security':
      break;
    case 'accessibility':
      break;
    case 'patterns':
      break;
    // No default needed - TypeScript ensures exhaustiveness
  }
}
```

## Risk Assessment

### Low Risk

- Type definitions are compile-time only (no runtime impact)
- Existing code unaffected (backward compatible not needed since we're starting with new code)
- Isolated to `common/types` package

### Medium Risk

- Generic type constraints too restrictive → Mitigate: use `unknown` defaults, allow wide constraints
- Branded types cause friction → Mitigate: provide factory functions, document pattern

### Mitigation Strategies

1. **Incremental Validation**: Compile `common/types` against existing consumers before proceeding
2. **Type Tests**: Write type-level tests ensuring constraints work as expected
3. **Documentation**: Provide clear examples of generic type usage
4. **Review**: Architecture review before Phase 2 begins

## Next Phase Dependencies

**Phase 2** (Plugin Discovery & Loading) requires:

- `IAnalysis` interface for discovering registered analyses
- `AnalysisCategory` for filtering analyses
- `AnalysisId` for referencing analyses

**Phase 3** (Engine Enhancements) requires:

- `IAnalysis.execute()` signature for parallel execution
- `AnalysisResult` for aggregating parallel results
- `PluginResult.analysisBreakdown` for result decomposition

**Phase 4** (Analyzer Refactoring) requires:

- `BaseAnalysis` abstract class for implementation
- `IAnalysis` interface for registering analyses
- Type utilities for config/result extraction
