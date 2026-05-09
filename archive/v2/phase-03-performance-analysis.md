# Phase 03: Performance Analysis

**Phase Duration:** 4-5 days  
**Priority:** High - Developer productivity  
**Complexity:** Medium-High  
**Dependencies:** Phase 00 (Foundation) ✅, Phase 02 (Anti-Patterns) ✅  
**Status:** Ready for implementation

All code examples use the `ts-morph` API for AST parsing and traversal.

## Overview

This phase implements a comprehensive performance analysis system that builds upon the existing anti-pattern detection (Phase 02) to provide deeper insights into render optimization, memoization effectiveness, bundle size impact, and performance bottlenecks. It consolidates performance-related data from anti-patterns and adds new metrics-based analysis.

**Key Note:** Phase 02 already implements performance anti-pattern rules (`performance-rules.ts`) including:

- Inline function props detection
- Inline object/array props detection
- Index-as-key detection
- Component-in-render detection

Phase 03 extends this with:

- Quantitative performance scoring
- Memoization effectiveness analysis
- Bundle impact estimation
- Integration with existing anti-pattern system

## Business Value

- Identify performance issues before production
- Reduce unnecessary re-renders through data-driven insights
- Optimize bundle size through code analysis
- Provide actionable optimization suggestions with quantified impact
- Support performance-focused code reviews with concrete metrics

## Scope and Boundaries

### In Scope for Phase 03

✅ Performance metrics and scoring
✅ Render performance analysis (complementing Phase 02 anti-patterns)
✅ Memoization tracking and effectiveness
✅ Bundle size impact estimation
✅ Heavy dependency detection
✅ Code splitting opportunity detection
✅ Integration with existing analyzer architecture
✅ CLI integration and output formatting (using `--performance` flag)

### Out of Scope (Future Phases)

❌ VS Code extension integration (Phase 08)
❌ Auto-fixing performance issues (future refactoring engine enhancement)
❌ Runtime performance profiling (static analysis only)
❌ Network request optimization
❌ Image/asset optimization
❌ Advanced bundle analysis (tree-shaking effectiveness in depth)

## Agent Assignments

| Agent                 | Role                                             | Capacity  |
| --------------------- | ------------------------------------------------ | --------- |
| react-engineer        | Lead implementer, React performance expertise    | Primary   |
| typescript-engineer   | Type system, integration with existing analyzers | Secondary |
| code-quality-analyzer | Test implementation, pattern validation          | Advisory  |

## Execution Strategy

### Milestone 3.1: Type Definitions (Day 1)

**Sequential Tasks:**

1. Create `src/types/performance.ts` with all type definitions
2. Update `src/types/index.ts` to export performance types
3. Create `src/constants/dependencies.ts` with heavy dependency catalog
4. Update `src/constants/index.ts` to export dependencies

**Owner:** typescript-engineer (Opus for types, Haiku for constants)
**Dependencies:** None
**Output:** Type-safe performance types, ready for analyzer implementation

### Milestone 3.2: Core Analyzer Implementation (Day 2-3)

**Sequential Tasks:**

1. Create `src/analyzers/performance-analyzer.ts` extending BaseAnalyzer
2. Implement render performance analysis (inline detection, expensive ops)
3. Implement memoization analysis (count hooks, detect patterns)
4. Implement bundle impact analysis (heavy deps, code splitting)

**Parallel Sub-tasks (Day 3):**

- Render analysis methods (Sonnet)
- Memoization counting (Sonnet)
- Bundle impact estimation (Sonnet)

**Owner:** react-engineer (Opus for architecture, Sonnet for methods)
**Dependencies:** Milestone 3.1
**Output:** Working PerformanceAnalyzer class

### Milestone 3.3: Integration & Testing (Day 4-5)

**Parallel Tasks:**

- Unit test suite (`performance-analyzer.test.ts`) - Sonnet
- Integration with CLI/main analyzer - Sonnet
- Update analyzers barrel export - Haiku
- Documentation and examples - Haiku

**Owner:** code-quality-analyzer + typescript-engineer
**Dependencies:** Milestone 3.2
**Output:** Fully tested and integrated performance analyzer

### Implementation Notes

1. **Start with types first** - This enables type-checking during development
2. **Follow BaseAnalyzer pattern** - Look at `migration-analyzer.ts` as reference
3. **Use ts-morph exclusively** - No Babel dependencies
4. **Leverage existing anti-patterns** - Don't duplicate Phase 02 detection logic
5. **Test incrementally** - Run tests after each major method implementation

## Detailed Tasks

### Task 3.0: Integration Architecture

**Understanding Existing System:**

The performance analyzer must integrate with:

1. **Anti-Pattern System (Phase 02)**: Already detects performance anti-patterns
2. **Base Analyzer**: Follows established pattern for all analyzers
3. **CLI Output**: Must format results for console and JSON output
4. **Type System**: Leverages existing `CodeLocation`, `Severity`, etc.

**Integration Points:**

```typescript
// src/analyzers/index.ts - Add to barrel exports
export { PerformanceAnalyzer, analyzePerformance } from './performance-analyzer';
export type { PerformanceMetrics } from '../types/performance';
```

### CLI integration (not part of Phase 03, but architecture ready)

```typescript
// src/cli.ts would add:
import { PerformanceAnalyzer } from './analyzers/performance-analyzer';
const perfAnalyzer = new PerformanceAnalyzer(sourceFile);
const perfResult = perfAnalyzer.analyze(filePath);
```

Formatting in the CLI would be similar to prior sections and match the existing theme.

**Anti-Pattern Coordination:**

The performance analyzer should be aware of but not duplicate Phase 02:

- Phase 02 detects WHAT (specific anti-patterns with locations)
- Phase 03 quantifies HOW MUCH (metrics, scores, aggregate impact)
- Phase 03 can reference anti-pattern IDs in optimization opportunities

### Task 3.1: Performance Types

**Model:** Opus (architectural design)
**File:** `src/types/performance.ts`

**Note:** Uses existing `CodeLocation` and `Severity` types from `src/types.ts`

```typescript
import type { CodeLocation, Severity } from '../types';

/**
 * Performance optimization opportunity types
 */
export type OptimizationType =
  | 'add-memo'
  | 'add-useMemo'
  | 'add-useCallback'
  | 'lazy-loading'
  | 'code-splitting'
  | 'virtualization'
  | 'debounce'
  | 'throttle'
  | 'avoid-inline-functions'
  | 'avoid-inline-objects';

/**
 * Individual optimization opportunity
 *
 * This extends the anti-pattern system by adding quantified impact metrics
 */
export interface OptimizationOpportunity {
  /** Optimization type */
  type: OptimizationType;
  /** Code location */
  location: CodeLocation;
  /** Estimated impact level */
  estimatedImpact: 'high' | 'medium' | 'low';
  /** Description of the opportunity */
  reason: string;
  /** How to implement */
  suggestion: string;
  /** Example code (optional) */
  example?: string;
  /** Related anti-pattern ID if from anti-pattern detection */
  antiPatternId?: string;
}

/**
 * Render performance metrics
 */
export interface RenderPerformanceMetrics {
  /** Overall render performance score (0-100, higher is better) */
  score: number;
  /** Risk of unnecessary re-renders (0-100, lower is better) */
  unnecessaryRenderRisk: number;
  /** Count of expensive operations in render */
  expensiveOperationCount: number;
  /** Identified optimization opportunities */
  optimizationOpportunities: OptimizationOpportunity[];
  /** Memoization effectiveness */
  memoizationEffectiveness: number;
  /** Components that may cause parent re-renders */
  reRenderTriggers: ReRenderTrigger[];
  /** Count of anti-patterns from performance rules */
  antiPatternCount: number;
}

/**
 * Re-render trigger analysis
 */
export interface ReRenderTrigger {
  /** Component or hook name */
  source: string;
  /** Location in code */
  location: CodeLocation;
  /** Why this triggers re-renders */
  reason: string;
  /** How often it might trigger */
  frequency: 'every-render' | 'frequently' | 'occasionally';
}

/**
 * Memoization analysis
 */
export interface MemoizationAnalysis {
  /** Total memoized items */
  totalMemoized: number;
  /** useCallback count */
  useCallbackCount: number;
  /** useMemo count */
  useMemoCount: number;
  /** React.memo count */
  reactMemoCount: number;
  /** Potentially unnecessary memoization */
  unnecessaryMemoization: UnnecessaryMemoization[];
  /** Missing memoization opportunities */
  missingMemoization: MissingMemoization[];
}

export interface UnnecessaryMemoization {
  type: 'useCallback' | 'useMemo' | 'memo';
  location: CodeLocation;
  reason: string;
}

export interface MissingMemoization {
  type: 'useCallback' | 'useMemo' | 'memo';
  location: CodeLocation;
  reason: string;
  impact: 'high' | 'medium' | 'low';
}

/**
 * Bundle size impact estimation
 */
export interface BundleSizeMetrics {
  /** Estimated bundle impact in bytes */
  estimatedImpact: number;
  /** Number of imports */
  importCount: number;
  /** Heavy dependencies detected */
  heavyDependencies: HeavyDependency[];
  /** Tree-shaking score (0-100) */
  treeshakingScore: number;
  /** Code splitting opportunities */
  codeSplittingOpportunities: CodeSplittingOpportunity[];
}

export interface HeavyDependency {
  /** Package name */
  name: string;
  /** Estimated size in bytes */
  estimatedSize: number;
  /** Usage pattern */
  usage: 'full' | 'partial';
  /** Lighter alternatives */
  alternatives?: string[];
  /** Is it tree-shakable */
  treeshakable: boolean;
}

export interface CodeSplittingOpportunity {
  /** Component name */
  component: string;
  /** Location */
  location: CodeLocation;
  /** Why this should be split */
  reason: string;
  /** Estimated savings on initial load */
  estimatedSavings: number;
}

/**
 * Complete performance metrics
 * Extends BaseAnalyzerResult for consistency with other analyzers
 */
export interface PerformanceMetrics {
  /** Overall performance score (0-100) */
  score: number;
  /** Render performance analysis */
  renderPerformance: RenderPerformanceMetrics;
  /** Memoization analysis */
  memoization: MemoizationAnalysis;
  /** Bundle size impact */
  bundleImpact: BundleSizeMetrics;
  /** Total optimization opportunities */
  totalOpportunities: number;
}
```

### Task 3.2: Performance Analyzer Implementation

**Model:** Opus (complex analysis logic)
**File:** `src/analyzers/performance-analyzer.ts`

**Integration Strategy:**

- Extends `BaseAnalyzer<TResult>` from `base-analyzer-tsmorph.ts`
- Imports existing anti-pattern detection from Phase 02
- Uses ts-morph for all AST traversal and analysis
- Follows the pattern established by `migration-analyzer.ts` and `reliability-analyzer-tsmorph.ts`

````typescript
import { SourceFile, Node, SyntaxKind } from 'ts-morph';
import { BaseAnalyzer, BaseAnalyzerResult } from './base-analyzer-tsmorph';
import type {
  PerformanceMetrics,
  RenderPerformanceMetrics,
  MemoizationAnalysis,
  BundleSizeMetrics,
  OptimizationOpportunity,
  ReRenderTrigger,
  HeavyDependency,
  CodeSplittingOpportunity,
} from '../types/performance';
import { HEAVY_DEPENDENCIES } from '../constants/dependencies';
import type { CodeLocation } from '../types';

/**
 * Performance Analyzer using ts-morph
 *
 * Analyzes React components for performance issues including:
 * - Render optimization opportunities
 * - Memoization effectiveness
 * - Bundle size impact
 *
 * Integrates with Phase 02 anti-pattern detection to provide comprehensive
 * performance insights.
 *
 * @example
 * ```typescript
 * const project = new Project();
 * const sourceFile = project.createSourceFile('temp.tsx', code);
 * const analyzer = new PerformanceAnalyzer(sourceFile);
 * const result = analyzer.analyze();
 * console.log(result.score); // 75
 * ```
 */
export class PerformanceAnalyzer extends BaseAnalyzer<PerformanceMetrics & BaseAnalyzerResult> {
  protected analyzeAST(): Omit<PerformanceMetrics & BaseAnalyzerResult, 'metadata'> {
    const renderPerformance = this.analyzeRenderPerformance();
    const memoization = this.analyzeMemoization();
    const bundleImpact = this.analyzeBundleImpact();

    const totalOpportunities =
      renderPerformance.optimizationOpportunities.length +
      memoization.missingMemoization.length +
      bundleImpact.codeSplittingOpportunities.length;

    // Calculate overall score
    const score = this.calculateOverallScore(renderPerformance, memoization, bundleImpact);

    return {
      score,
      renderPerformance,
      memoization,
      bundleImpact,
      totalOpportunities,
    };
  }

  private analyzeRenderPerformance(): RenderPerformanceMetrics {
    const opportunities: OptimizationOpportunity[] = [];
    const reRenderTriggers: ReRenderTrigger[] = [];
    let expensiveOperationCount = 0;
    let antiPatternCount = 0;

    // Detect inline objects/functions in JSX (complement to anti-pattern rules)
    this.ast.forEachDescendant(node => {
      // Detect expensive operations in render
      if (Node.isCallExpression(node)) {
        if (this.isExpensiveRenderOperation(node)) {
          expensiveOperationCount++;
          opportunities.push({
            type: 'add-useMemo',
            location: this.getLocation(node),
            estimatedImpact: 'medium',
            reason: 'Expensive computation in render without memoization',
            suggestion: 'Wrap in useMemo to prevent recalculation on every render',
          });
        }
      }

      // Detect large lists without virtualization
      if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
        this.checkListVirtualization(node, opportunities);
      }

      // Detect inline functions and objects (cross-reference with anti-patterns)
      if (Node.isJsxAttribute(node)) {
        this.checkJSXAttributePerformance(node, opportunities, reRenderTriggers);
      }
    });

    // Calculate risk score
    const unnecessaryRenderRisk = this.calculateRenderRisk(opportunities, reRenderTriggers);
    const memoizationEffectiveness = this.calculateMemoizationEffectiveness();
    const score = Math.max(0, 100 - unnecessaryRenderRisk);

    return {
      score,
      unnecessaryRenderRisk,
      expensiveOperationCount,
      optimizationOpportunities: opportunities,
      memoizationEffectiveness,
      reRenderTriggers,
      antiPatternCount,
    };
  }

  private checkJSXAttributePerformance(
    node: Node,
    opportunities: OptimizationOpportunity[],
    triggers: ReRenderTrigger[]
  ): void {
    if (!Node.isJsxAttribute(node)) return;

    const nameNode = node.getNameNode();
    if (!Node.isIdentifier(nameNode)) return;
    const attrName = nameNode.getText();

    const initializer = node.getInitializer();
    if (!initializer || !Node.isJsxExpression(initializer)) return;

    const expr = initializer.getExpression();
    if (!expr) return;

    // Inline functions
    if (Node.isArrowFunction(expr) || Node.isFunctionExpression(expr)) {
      if (attrName.startsWith('on')) {
        triggers.push({
          source: `inline ${attrName}`,
          location: this.getLocation(node),
          reason: 'Inline function creates new reference each render',
          frequency: 'every-render',
        });
      }
    }

    // Inline objects (except style which is common)
    if (Node.isObjectLiteralExpression(expr) && attrName !== 'style') {
      triggers.push({
        source: `inline object in ${attrName}`,
        location: this.getLocation(node),
        reason: 'Inline object creates new reference each render',
        frequency: 'every-render',
      });

      opportunities.push({
        type: 'add-useMemo',
        location: this.getLocation(node),
        estimatedImpact: 'medium',
        reason: `Inline object in "${attrName}" prop`,
        suggestion: 'Extract to useMemo or constant outside component',
        antiPatternId: 'inline-object-props',
      });
    }

    // Inline arrays
    if (Node.isArrayLiteralExpression(expr)) {
      triggers.push({
        source: `inline array in ${attrName}`,
        location: this.getLocation(node),
        reason: 'Inline array creates new reference each render',
        frequency: 'every-render',
      });
    }
  }

  private isExpensiveRenderOperation(node: Node): boolean {
    if (!Node.isCallExpression(node)) return false;

    const expr = node.getExpression();

    // Check if inside useMemo/useCallback already
    let parent = node.getParent();
    while (parent) {
      if (Node.isCallExpression(parent)) {
        const parentExpr = parent.getExpression();
        if (Node.isIdentifier(parentExpr)) {
          const name = parentExpr.getText();
          if (['useMemo', 'useCallback'].includes(name)) {
            return false; // Already memoized
          }
        }
      }
      parent = parent.getParent();
    }

    // Check for expensive operations
    if (Node.isPropertyAccessExpression(expr)) {
      const method = expr.getName();
      // Array methods that iterate
      if (['map', 'filter', 'reduce', 'sort', 'find', 'every', 'some'].includes(method)) {
        return true;
      }
    }

    // JSON.parse/stringify
    if (Node.isPropertyAccessExpression(expr)) {
      const obj = expr.getExpression();
      if (Node.isIdentifier(obj) && obj.getText() === 'JSON') {
        return true;
      }
    }

    return false;
  }

  private checkListVirtualization(node: Node, opportunities: OptimizationOpportunity[]): void {
    // Check for .map() calls that might benefit from virtualization
    if (!Node.isJsxElement(node)) return;

    const children = node.getJsxChildren();
    children.forEach(child => {
      if (Node.isJsxExpression(child)) {
        const expr = child.getExpression();
        if (!expr || !Node.isCallExpression(expr)) return;

        const callExpr = expr.getExpression();
        if (Node.isPropertyAccessExpression(callExpr)) {
          if (callExpr.getName() === 'map') {
            // Heuristic: if mapping over a prop or state, might be large
            opportunities.push({
              type: 'virtualization',
              location: this.getLocation(node),
              estimatedImpact: 'low',
              reason: 'List rendering detected - consider virtualization for large lists',
              suggestion: 'For lists > 100 items, use react-window or react-virtualized',
            });
          }
        }
      }
    });
  }

  private analyzeMemoization(): MemoizationAnalysis {
    let useCallbackCount = 0;
    let useMemoCount = 0;
    let reactMemoCount = 0;

    this.ast.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();

      if (Node.isIdentifier(expr)) {
        const name = expr.getText();
        if (name === 'useCallback') useCallbackCount++;
        if (name === 'useMemo') useMemoCount++;
        if (name === 'memo') reactMemoCount++;
      }

      if (Node.isPropertyAccessExpression(expr)) {
        const obj = expr.getExpression();
        if (Node.isIdentifier(obj) && obj.getText() === 'React') {
          if (expr.getName() === 'memo') reactMemoCount++;
        }
      }
    });

    return {
      totalMemoized: useCallbackCount + useMemoCount + reactMemoCount,
      useCallbackCount,
      useMemoCount,
      reactMemoCount,
      unnecessaryMemoization: [], // Would require more sophisticated analysis
      missingMemoization: [], // Populated by render analysis
    };
  }

  private analyzeBundleImpact(): BundleSizeMetrics {
    const imports: string[] = [];
    const heavyDependencies: HeavyDependency[] = [];
    const codeSplittingOpportunities: CodeSplittingOpportunity[] = [];

    // Analyze imports
    this.ast.getImportDeclarations().forEach(importDecl => {
      const moduleSpecifier = importDecl.getModuleSpecifierValue();
      imports.push(moduleSpecifier);

      // Check if it's a known heavy dependency
      const heavyDep = HEAVY_DEPENDENCIES.find(dep => dep.name === moduleSpecifier);
      if (heavyDep) {
        // Check if using partial imports
        const namedImports = importDecl.getNamedImports();
        const hasNamespaceImport = importDecl.getNamespaceImport();
        const isPartial = namedImports.length > 0 && !hasNamespaceImport;

        heavyDependencies.push({
          ...heavyDep,
          usage: isPartial ? 'partial' : 'full',
        });
      }
    });

    // Check for lazy loading opportunities
    this.ast.getVariableDeclarations().forEach(varDecl => {
      const init = varDecl.getInitializer();
      if (!init) return;

      // Check for default export of large components
      if (Node.isArrowFunction(init) || Node.isFunctionExpression(init)) {
        const lineCount = init.getEndLineNumber() - init.getStartLineNumber();

        if (lineCount > 100) {
          const name = varDecl.getName();
          codeSplittingOpportunities.push({
            component: name,
            location: this.getLocation(varDecl),
            reason: 'Large component (>100 lines) - consider lazy loading',
            estimatedSavings: lineCount * 50, // Rough estimate: 50 bytes per line
          });
        }
      }
    });

    // Also check function declarations
    this.ast.getFunctions().forEach(fn => {
      const name = fn.getName();
      if (!name) return;

      const lineCount = fn.getEndLineNumber() - fn.getStartLineNumber();
      if (lineCount > 100) {
        codeSplittingOpportunities.push({
          component: name,
          location: this.getLocation(fn),
          reason: 'Large component (>100 lines) - consider lazy loading',
          estimatedSavings: lineCount * 50,
        });
      }
    });

    const estimatedImpact = heavyDependencies.reduce((sum, dep) => sum + dep.estimatedSize, 0);
    const treeshakingScore = this.calculateTreeshakingScore(imports, heavyDependencies);

    return {
      estimatedImpact,
      importCount: imports.length,
      heavyDependencies,
      treeshakingScore,
      codeSplittingOpportunities,
    };
  }

  private calculateRenderRisk(
    opportunities: OptimizationOpportunity[],
    triggers: ReRenderTrigger[]
  ): number {
    let risk = 0;

    triggers.forEach(t => {
      if (t.frequency === 'every-render') risk += 10;
      else if (t.frequency === 'frequently') risk += 5;
      else risk += 2;
    });

    opportunities.forEach(o => {
      if (o.estimatedImpact === 'high') risk += 15;
      else if (o.estimatedImpact === 'medium') risk += 8;
      else risk += 3;
    });

    return Math.min(100, risk);
  }

  private calculateMemoizationEffectiveness(): number {
    // Placeholder - would require more sophisticated analysis
    return 75;
  }

  private calculateTreeshakingScore(imports: string[], heavyDeps: HeavyDependency[]): number {
    if (imports.length === 0) return 100;

    const partialImports = heavyDeps.filter(d => d.usage === 'partial').length;
    const totalHeavy = heavyDeps.length;

    if (totalHeavy === 0) return 100;

    return Math.round((partialImports / totalHeavy) * 100);
  }

  private calculateOverallScore(
    render: RenderPerformanceMetrics,
    memo: MemoizationAnalysis,
    bundle: BundleSizeMetrics
  ): number {
    // Weighted average
    const renderWeight = 0.5;
    const memoWeight = 0.3;
    const bundleWeight = 0.2;

    return Math.round(
      render.score * renderWeight +
        memo.totalMemoized * 5 * memoWeight + // Rough heuristic
        bundle.treeshakingScore * bundleWeight
    );
  }
}

/**
 * Convenience function to analyze performance from source code
 *
 * @param source - Source code to analyze
 * @returns Performance analysis result
 *
 * @example
 * ```typescript
 * const result = analyzePerformance(`
 *   function MyComponent({ items }) {
 *     const filtered = items.filter(i => i.active);
 *     return <List data={filtered} onClick={() => alert('hi')} />;
 *   }
 * `);
 * console.log(result.renderPerformance.unnecessaryRenderRisk);
 * ```
 */
export function analyzePerformance(source: string): PerformanceMetrics & BaseAnalyzerResult {
  const { Project } = require('ts-morph');
  const project = new Project({ useInMemoryFileSystem: true });
  const sourceFile = project.createSourceFile('temp.tsx', source);
  const analyzer = new PerformanceAnalyzer(sourceFile);
  return analyzer.analyze();
}
````

### Task 3.3: Heavy Dependencies Catalog

**Model:** Haiku (data entry)
**File:** `src/constants/dependencies.ts`

**Note:** This creates a new file that exports alongside existing constants. Will be re-exported through `src/constants/index.ts`.

```typescript
import type { HeavyDependency } from '../types/performance';

/**
 * Catalog of known heavy npm dependencies
 *
 * Sizes are approximate minified+gzipped sizes based on bundlephobia.com data.
 * Used by performance analyzer to detect bundle bloat opportunities.
 *
 * @module constants/dependencies
 */
export const HEAVY_DEPENDENCIES: Omit<HeavyDependency, 'usage'>[] = [
  {
    name: 'moment',
    estimatedSize: 67_000, // 67KB
    alternatives: ['date-fns', 'dayjs', 'luxon'],
    treeshakable: false,
  },
  {
    name: 'lodash',
    estimatedSize: 72_000, // 72KB
    alternatives: ['lodash-es', 'radash', 'remeda'],
    treeshakable: false,
  },
  {
    name: 'lodash-es',
    estimatedSize: 72_000,
    alternatives: ['radash', 'remeda'],
    treeshakable: true,
  },
  {
    name: 'axios',
    estimatedSize: 14_000, // 14KB
    alternatives: ['ky', 'fetch (native)'],
    treeshakable: false,
  },
  {
    name: 'rxjs',
    estimatedSize: 47_000, // 47KB
    alternatives: [],
    treeshakable: true,
  },
  {
    name: 'chart.js',
    estimatedSize: 65_000, // 65KB
    alternatives: ['lightweight-charts', 'uplot'],
    treeshakable: true,
  },
  {
    name: 'antd',
    estimatedSize: 350_000, // 350KB (full import)
    alternatives: ['@radix-ui', '@headlessui/react'],
    treeshakable: true,
  },
  {
    name: '@mui/material',
    estimatedSize: 300_000, // 300KB (full import)
    alternatives: ['@radix-ui', '@headlessui/react'],
    treeshakable: true,
  },
  {
    name: 'framer-motion',
    estimatedSize: 50_000, // 50KB
    alternatives: ['motion', '@formkit/auto-animate'],
    treeshakable: true,
  },
  {
    name: 'react-spring',
    estimatedSize: 40_000, // 40KB
    alternatives: ['motion', '@formkit/auto-animate'],
    treeshakable: true,
  },
];
```

**After creation, update `src/constants/index.ts`:**

```typescript
// Add to existing exports
export { HEAVY_DEPENDENCIES } from './dependencies';
export type { HeavyDependency } from '../types/performance';
```

## Acceptance Criteria

### Functional Requirements

- [ ] Performance types defined with proper TypeScript types
- [ ] PerformanceAnalyzer extends BaseAnalyzer following established patterns
- [ ] Detect inline functions/objects in JSX props (complementing anti-pattern rules)
- [ ] Identify expensive computations in render (not already memoized)
- [ ] Track useCallback/useMemo/memo usage counts
- [ ] Detect heavy dependency imports with size estimates
- [ ] Suggest code splitting opportunities for large components
- [ ] Calculate performance score (0-100)
- [ ] Integration with existing anti-pattern detection from Phase 02
- [ ] Export through analyzers barrel file

### Quality Requirements

- [ ] `<5`% false positive rate for performance issues
- [ ] Correct detection of memoization patterns
- [ ] Accurate heavy dependency detection
- [ ] All tests passing (unit + integration)
- [ ] Type-safe (no `any` types)
- [ ] Follows codebase patterns (matches other analyzers)

### Performance Requirements

- [ ] Analysis < 300ms for typical component (~200 LOC)
- [ ] No memory leaks in analysis
- [ ] Efficient AST traversal (single pass where possible)

### Documentation Requirements

- [ ] JSDoc comments for all public APIs
- [ ] Examples in documentation
- [ ] Integration notes for consuming analyzers

## Testing Instructions

### Unit Tests

**File:** `src/analyzers/performance-analyzer.test.ts`

Follow the pattern established in existing analyzer tests:

- `migration-analyzer.test.ts`
- `reliability-analyzer-tsmorph.test.ts`
- `dataflow-analyzer-tsmorph.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { Project } from 'ts-morph';
import { PerformanceAnalyzer } from './performance-analyzer';

describe('PerformanceAnalyzer', () => {
  function analyze(code: string) {
    const project = new Project({ useInMemoryFileSystem: true });
    const sourceFile = project.createSourceFile('test.tsx', code);
    const analyzer = new PerformanceAnalyzer(sourceFile);
    return analyzer.analyze();
  }

  describe('Render Performance', () => {
    it('detects inline function props', () => {
      const code = `
        function Component() {
          return <button onClick={() => console.log('hi')} />;
        }
      `;
      const result = analyze(code);
      expect(result.renderPerformance.reRenderTriggers.length).toBeGreaterThan(0);
    });

    it('detects inline object props', () => {
      const code = `
        function Component() {
          return <List config={{ size: 10, color: 'red' }} />;
        }
      `;
      const result = analyze(code);
      expect(result.renderPerformance.reRenderTriggers.length).toBeGreaterThan(0);
    });

    it('detects expensive operations', () => {
      const code = `
        function Component({ items }) {
          const sorted = items.sort((a, b) => a.value - b.value);
          return <div>{sorted.length}</div>;
        }
      `;
      const result = analyze(code);
      expect(result.renderPerformance.expensiveOperationCount).toBeGreaterThan(0);
    });

    it('does not flag memoized operations', () => {
      const code = `
        function Component({ items }) {
          const sorted = useMemo(() => items.sort((a, b) => a.value - b.value), [items]);
          return <div>{sorted.length}</div>;
        }
      `;
      const result = analyze(code);
      expect(result.renderPerformance.expensiveOperationCount).toBe(0);
    });
  });

  describe('Memoization Analysis', () => {
    it('counts useCallback usage', () => {
      const code = `
        function Component() {
          const handler = useCallback(() => {}, []);
          return <button onClick={handler} />;
        }
      `;
      const result = analyze(code);
      expect(result.memoization.useCallbackCount).toBe(1);
    });

    it('counts useMemo usage', () => {
      const code = `
        function Component({ items }) {
          const filtered = useMemo(() => items.filter(x => x.active), [items]);
          return <List data={filtered} />;
        }
      `;
      const result = analyze(code);
      expect(result.memoization.useMemoCount).toBe(1);
    });

    it('counts React.memo usage', () => {
      const code = `
        const Component = memo(function Component() {
          return <div>Hello</div>;
        });
      `;
      const result = analyze(code);
      expect(result.memoization.reactMemoCount).toBe(1);
    });
  });

  describe('Bundle Impact', () => {
    it('detects heavy dependencies', () => {
      const code = `
        import moment from 'moment';
        import _ from 'lodash';
        
        function Component() {
          return <div>{moment().format()}</div>;
        }
      `;
      const result = analyze(code);
      expect(result.bundleImpact.heavyDependencies.length).toBe(2);
      expect(result.bundleImpact.heavyDependencies[0].name).toBe('moment');
    });

    it('distinguishes full vs partial imports', () => {
      const code = `
        import { debounce } from 'lodash-es';
        
        function Component() {
          return <div>Test</div>;
        }
      `;
      const result = analyze(code);
      const lodashDep = result.bundleImpact.heavyDependencies.find(d => d.name === 'lodash-es');
      expect(lodashDep?.usage).toBe('partial');
    });

    it('suggests code splitting for large components', () => {
      const code = `
        function LargeComponent() {
          ${Array(120).fill('const x = 1;').join('\n')}
          return <div>Large</div>;
        }
      `;
      const result = analyze(code);
      expect(result.bundleImpact.codeSplittingOpportunities.length).toBeGreaterThan(0);
    });
  });

  describe('Overall Scoring', () => {
    it('produces score between 0 and 100', () => {
      const code = `
        function Component() {
          return <div>Hello</div>;
        }
      `;
      const result = analyze(code);
      expect(result.score).toBeGreaterThanOrEqual(0);
      expect(result.score).toBeLessThanOrEqual(100);
    });

    it('lowers score for performance issues', () => {
      const goodCode = `
        function GoodComponent({ onClick }) {
          return <button onClick={onClick}>Click</button>;
        }
      `;
      const badCode = `
        function BadComponent() {
          const items = [1,2,3,4,5].map(x => x * 2).filter(x => x > 5);
          return <button onClick={() => console.log('hi')}>Click</button>;
        }
      `;
      const goodResult = analyze(goodCode);
      const badResult = analyze(badCode);
      expect(badResult.score).toBeLessThan(goodResult.score);
    });
  });
});
```

### Manual Testing

Run tests for modified files only (as per user rule):

```bash
# Run the performance analyzer tests
npm run test -- performance-analyzer.test.ts

# If you also modify types
npm run test -- performance-analyzer.test.ts types/performance.ts
```

### Integration Testing

1. **Test with sample components:**

```bash
cat > /tmp/PerformanceTest.tsx << 'EOF'
function Component({ items, config }) {
  const filtered = items.filter(i => i.active).map(i => i.name);

  return (
    <List
      data={filtered}
      config={{ ...config, extra: true }}
      onClick={() => console.log('clicked')}
    />
  );
}
EOF

npm run analyze -- /tmp/PerformanceTest.tsx --verbose
# Expected: Detects inline object, inline function, expensive filter/map
```

2. **Test bundle impact detection:**

```bash
cat > /tmp/HeavyImports.tsx << 'EOF'
import moment from 'moment';
import _ from 'lodash';

export function DateFormatter({ date }) {
  return <span>{moment(date).format('YYYY-MM-DD')}</span>;
}
EOF

npm run analyze -- /tmp/HeavyImports.tsx --json | jq '.performance.bundleImpact'
# Expected: Lists moment and lodash as heavy dependencies with alternatives
```

3. **Test with existing sample components:**

```bash
npm run analyze -- src/sample-components/ProblematicComponent.tsx --json | jq '.performance'
# Should detect performance issues in the sample
```

## Estimated Effort

| Task                          | Model  | Estimated Time |
| ----------------------------- | ------ | -------------- |
| 3.1 Performance Types         | Opus   | 1.5 hours      |
| 3.2 Dependencies Catalog      | Haiku  | 0.5 hours      |
| 3.3 PerformanceAnalyzer Class | Opus   | 3 hours        |
| 3.4 Render Analysis Methods   | Sonnet | 2 hours        |
| 3.5 Memoization Analysis      | Sonnet | 1.5 hours      |
| 3.6 Bundle Analysis           | Sonnet | 2 hours        |
| 3.7 Unit Tests                | Sonnet | 3 hours        |
| 3.8 Integration & Exports     | Haiku  | 1 hour         |
| 3.9 Documentation             | Haiku  | 0.5 hours      |
| **Total**                     |        | **15 hours**   |

## Implementation Checklist

### Pre-Implementation

- [ ] Review existing anti-pattern rules in `src/rules/anti-patterns/performance-rules.ts`
- [ ] Review BaseAnalyzer pattern in `src/analyzers/base-analyzer-tsmorph.ts`
- [ ] Review migration-analyzer as reference implementation
- [ ] Understand existing types in `src/types.ts`

### Implementation Order

1. [ ] Create `src/types/performance.ts`
2. [ ] Create `src/constants/dependencies.ts`
3. [ ] Update barrel exports in `types/index.ts` and `constants/index.ts`
4. [ ] Create `src/analyzers/performance-analyzer.ts` skeleton
5. [ ] Implement render performance analysis
6. [ ] Implement memoization analysis
7. [ ] Implement bundle impact analysis
8. [ ] Create `src/analyzers/performance-analyzer.test.ts`
9. [ ] Write unit tests for each analysis method
10. [ ] Update `src/analyzers/index.ts` to export PerformanceAnalyzer
11. [ ] Manual integration testing with sample components
12. [ ] Update documentation

### Post-Implementation

- [ ] Run all tests: `npm run test -- performance-analyzer.test.ts`
- [ ] Type check: `npm run checks:types`
- [ ] Lint: `npm run checks:linting`
- [ ] Test with real components: `npm run analyze -- src/sample-components/ProblematicComponent.tsx`
- [ ] Verify JSON output format
- [ ] Update schema.json if needed for output format

---

**Document Version:** 2.0  
**Created:** January 10, 2026  
**Last Updated:** January 10, 2026  
**Status:** Ready for Implementation - Aligned with ts-morph architecture
