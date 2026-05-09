# Phase 04: Technical Debt Quantification

**Phase Duration:** 4-5 days  
**Priority:** Medium-High - Strategic planning value  
**Complexity:** Medium  
**Dependencies:** Phase 00 (Foundation) ✅, Phase 01-03 (metrics integration)  
**Status:** Ready for implementation

All code examples use the `ts-morph` API for AST parsing and traversal. This phase integrates with existing complexity analysis from `tsmorph-analyzer.ts` to provide debt quantification.

## Overview

This phase implements technical debt quantification inspired by CodeScene's behavioral analysis. It provides a Code Health Score, identifies hotspots, calculates a Maintainability Index adapted for React, and helps teams prioritize refactoring efforts with data-driven insights.

### Key Integration Points

This analyzer **integrates with existing complexity analysis** rather than duplicating it:

1. **Input:** Takes `ReactComplexityResult` from `tsmorph-analyzer.ts`
2. **Processing:** Calculates health metrics, maintainability index, and debt interest
3. **Output:** Provides actionable debt quantification and recommendations

The analyzer is designed to work standalone or as part of a larger analysis pipeline, making it suitable for:

- CLI analysis (optional integration)
- Future VSCode extension (Phase 07-08)
- Batch analysis of codebases
- CI/CD quality gates

## Business Value

- Prioritize refactoring with data, not gut feeling
- Quantify technical debt for stakeholder communication
- Track code health trends over time
- Identify high-risk areas before they become critical
- Support technical debt reduction planning

## Agent Assignments

| Agent                    | Role                               | Capacity  |
| ------------------------ | ---------------------------------- | --------- |
| code-complexity-analyzer | Lead implementer, algorithm design | Primary   |
| typescript-engineer      | Type system, scoring models        | Secondary |
| react-engineer           | React-specific adaptations         | Advisory  |

## Execution Strategy

### Milestone 4.1: Code Health Score (Day 1-2)

**Synchronous Tasks:**

1. Design health score model (code-complexity-analyzer)
2. Implement scoring algorithm (code-complexity-analyzer)

**Parallel Tasks:**

- Component health metrics (Sonnet)
- File-level aggregation (Sonnet)

### Milestone 4.2: Hotspot Detection (Day 2-3)

**Synchronous Tasks:**

1. Define hotspot criteria (code-complexity-analyzer)
2. Implement detection logic (Sonnet)

**Parallel Tasks:**

- Priority scoring (Sonnet)
- Cost of delay estimation (Opus)

### Milestone 4.3: Maintainability Index (Day 3-4)

**Synchronous Tasks:**

1. Adapt MI formula for React (code-complexity-analyzer)
2. Implement calculation (Sonnet)

### Milestone 4.4: Integration & Testing (Day 5)

**Parallel Tasks:**

- Integration with existing metrics (Sonnet)
- Schema updates (Haiku)
- Documentation (Haiku)

## Detailed Tasks

### Task 4.0: Barrel Export Updates

**Model:** Haiku (simple exports)

**File 1:** `src/types/index.ts`

Add technical debt type exports:

```typescript
// Export technical debt types
export type {
  CodeHealthMetrics,
  TechnicalDebtHotspot,
  HotspotIssue,
  DebtCategory,
  DebtItem,
  MaintainabilityMetrics,
  MaintainabilityFactors,
  TechnicalDebtInterest,
  PayoffStrategy,
  TechnicalDebtMetrics,
} from './technical-debt';
```

**File 2:** `src/analyzers/index.ts`

Add technical debt analyzer exports:

```typescript
// Technical debt analyzer
export {
  TechnicalDebtAnalyzer,
  analyzeTechnicalDebt,
  type TechnicalDebtAnalyzerOptions,
  type TechnicalDebtAnalyzerResult,
} from './technical-debt-analyzer';
```

### Task 4.1: Technical Debt Types

**Model:** Opus (foundational types)
**File:** `src/types/technical-debt.ts`

```typescript
/**
 * Technical Debt Analysis Types
 *
 * Types for quantifying technical debt, code health, and maintainability
 * Integrates with existing complexity analysis from tsmorph-analyzer
 *
 * @module types/technical-debt
 */

import type { CodeLocation } from '../types';

/**
 * Overall code health metrics
 */
export interface CodeHealthMetrics {
  /** Overall health score (0-10 scale, like CodeScene) */
  overallHealth: number;
  /** Health grade */
  grade: 'A' | 'B' | 'C' | 'D' | 'F';
  /** Health trend (requires historical data, defaults to 'stable') */
  trend: 'improving' | 'stable' | 'degrading';
  /** Technical debt score (higher = more debt) */
  technicalDebtScore: number;
  /** Estimated maintenance burden (hours/month) */
  maintenanceBurden: number;
  /** Identified hotspots */
  hotspots: TechnicalDebtHotspot[];
  /** Debt breakdown by category */
  debtByCategory: DebtCategory[];
}

/**
 * Technical debt hotspot
 */
export interface TechnicalDebtHotspot {
  /** File path */
  file: string;
  /** Component name if applicable */
  component?: string;
  /** Complexity score */
  complexity: number;
  /** Estimated bug probability (0-1) */
  bugProbability: number;
  /** Priority score for addressing */
  priorityScore: number;
  /** Estimated cost of not fixing (hours) */
  costOfDelay: number;
  /** Specific issues identified */
  issues: HotspotIssue[];
  /** Recommended actions */
  recommendations: string[];
}

export interface HotspotIssue {
  type: 'complexity' | 'coupling' | 'duplication' | 'testing' | 'documentation';
  severity: 'high' | 'medium' | 'low';
  description: string;
  location?: CodeLocation;
}

/**
 * Debt category breakdown
 */
export interface DebtCategory {
  category: string;
  score: number;
  items: DebtItem[];
}

export interface DebtItem {
  description: string;
  location: CodeLocation;
  effort: number; // hours to fix
  impact: 'high' | 'medium' | 'low';
}

/**
 * Maintainability Index (adapted for React)
 */
export interface MaintainabilityMetrics {
  /** Classic Maintainability Index (0-100) */
  maintainabilityIndex: number;
  /** React-adapted MI */
  reactMaintainabilityIndex: number;
  /** Component factors */
  factors: MaintainabilityFactors;
}

export interface MaintainabilityFactors {
  /** Halstead Volume contribution */
  halsteadVolume: number;
  /** Cyclomatic Complexity contribution */
  cyclomaticComplexity: number;
  /** Lines of Code contribution */
  linesOfCode: number;
  /** Hook complexity contribution (React-specific) */
  hookComplexity: number;
  /** Coupling complexity contribution (React-specific) */
  couplingComplexity: number;
  /** Type coverage contribution */
  typeCoverage: number;
}

/**
 * Technical debt interest model
 */
export interface TechnicalDebtInterest {
  /** Initial debt (hours to fix all issues) */
  principalDebt: number;
  /** Additional cost per month if not fixed */
  interestRate: number;
  /** Factors contributing to interest */
  compoundingFactors: string[];
  /** Strategies to pay down debt */
  payoffStrategies: PayoffStrategy[];
}

export interface PayoffStrategy {
  name: string;
  effort: number; // hours to implement
  benefit: number; // hours saved per month
  breakEvenPoint: number; // months until ROI positive
  priority: number; // 1-10
}

/**
 * Complete technical debt metrics
 */
export interface TechnicalDebtMetrics {
  /** Overall score (0-100, higher is better/less debt) */
  score: number;
  /** Code health metrics */
  codeHealth: CodeHealthMetrics;
  /** Maintainability metrics */
  maintainability: MaintainabilityMetrics;
  /** Debt interest model */
  debtInterest: TechnicalDebtInterest;
}
```

### Task 4.2: Technical Debt Analyzer

**Model:** Opus (core algorithm)
**File:** `src/analyzers/technical-debt-analyzer.ts`

````typescript
/**
 * Technical Debt Analyzer
 *
 * Quantifies technical debt using code health metrics, maintainability index,
 * and debt interest calculations. Integrates with existing complexity analysis.
 *
 * @module analyzers/technical-debt-analyzer
 */

import type { SourceFile } from 'ts-morph';
import { BaseAnalyzer, type BaseAnalyzerResult } from './base-analyzer-tsmorph';
import type {
  TechnicalDebtMetrics,
  CodeHealthMetrics,
  MaintainabilityMetrics,
  TechnicalDebtHotspot,
  TechnicalDebtInterest,
  MaintainabilityFactors,
  PayoffStrategy,
  HotspotIssue,
  DebtCategory,
} from '../types/technical-debt';
import type { ReactComplexityResult } from '../types';

/**
 * Configuration options for technical debt analyzer
 */
export interface TechnicalDebtAnalyzerOptions {
  /** Existing complexity result to incorporate */
  complexityResult?: ReactComplexityResult;
  /** Lines of code in the file */
  linesOfCode?: number;
  /** File path being analyzed */
  filePath?: string;
  /** Component name if known */
  componentName?: string;
}

/**
 * Technical Debt Analyzer Result
 */
export interface TechnicalDebtAnalyzerResult extends TechnicalDebtMetrics, BaseAnalyzerResult {}

/**
 * Technical Debt Analyzer
 *
 * Analyzes code health, calculates maintainability index, and estimates
 * technical debt interest. Uses existing complexity metrics as input.
 *
 * @example
 * ```typescript
 * const project = new Project();
 * const sourceFile = project.createSourceFile('temp.tsx', code);
 * const complexityResult = analyzeComplexity(code);
 * const analyzer = new TechnicalDebtAnalyzer(sourceFile, {
 *   complexityResult,
 *   linesOfCode: code.split('\n').length
 * });
 * const result = analyzer.analyze();
 * console.log(result.codeHealth.grade); // 'B'
 * ```
 */
export class TechnicalDebtAnalyzer extends BaseAnalyzer<TechnicalDebtAnalyzerResult> {
  private options: TechnicalDebtAnalyzerOptions;

  constructor(ast: SourceFile, options: TechnicalDebtAnalyzerOptions = {}) {
    super(ast);
    this.options = options;
  }

  /**
   * Analyze technical debt
   */
  protected analyzeAST(): Omit<TechnicalDebtAnalyzerResult, 'metadata'> {
    const codeHealth = this.analyzeCodeHealth();
    const maintainability = this.analyzeMaintainability();
    const debtInterest = this.calculateDebtInterest(codeHealth, maintainability);

    // Overall score (inverse of debt - higher is better)
    const score = this.calculateOverallScore(codeHealth, maintainability);

    return {
      score,
      codeHealth,
      maintainability,
      debtInterest,
    };
  }

  // ==========================================================================
  // Code Health Analysis
  // ==========================================================================

  /**
   * Analyze overall code health
   */
  private analyzeCodeHealth(): CodeHealthMetrics {
    const hotspots = this.identifyHotspots();
    const technicalDebtScore = this.calculateDebtScore(hotspots);
    const maintenanceBurden = this.estimateMaintenanceBurden(hotspots);

    // Calculate overall health (0-10 scale)
    const overallHealth = Math.max(0, 10 - technicalDebtScore / 10);

    return {
      overallHealth,
      grade: this.healthToGrade(overallHealth),
      trend: 'stable', // Would need historical data
      technicalDebtScore,
      maintenanceBurden,
      hotspots,
      debtByCategory: this.categorizeDebt(hotspots),
    };
  }

  /**
   * Identify technical debt hotspots based on complexity metrics
   */
  private identifyHotspots(): TechnicalDebtHotspot[] {
    const hotspots: TechnicalDebtHotspot[] = [];
    const complexityResult = this.options.complexityResult;

    if (!complexityResult) {
      return hotspots;
    }

    // Check if this file is a hotspot based on complexity
    // Threshold: total complexity > 50 indicates a hotspot
    if (complexityResult.total > 50) {
      const issues = this.collectIssues(complexityResult);
      const bugProbability = this.estimateBugProbability(complexityResult);
      const priorityScore = this.calculatePriority(complexityResult, bugProbability);

      hotspots.push({
        file: this.options.filePath || 'current',
        component: this.options.componentName,
        complexity: complexityResult.total,
        bugProbability,
        priorityScore,
        costOfDelay: this.estimateCostOfDelay(complexityResult, priorityScore),
        issues,
        recommendations: this.generateRecommendations(complexityResult, issues),
      });
    }

    return hotspots;
  }

  /**
   * Collect specific issues from complexity analysis
   */
  private collectIssues(result: ReactComplexityResult): HotspotIssue[] {
    const issues: HotspotIssue[] = [];

    // High structural complexity
    if (result.structural.score > 20) {
      issues.push({
        type: 'complexity',
        severity: result.structural.score > 40 ? 'high' : 'medium',
        description: `High structural complexity (${result.structural.score})`,
      });
    }

    // Too many hooks
    if (result.hooks.totalHooks > 10) {
      issues.push({
        type: 'complexity',
        severity: result.hooks.totalHooks > 15 ? 'high' : 'medium',
        description: `Too many hooks (${result.hooks.totalHooks})`,
      });
    }

    // High temporal complexity (effects)
    if (result.temporal.score > 15) {
      issues.push({
        type: 'complexity',
        severity: result.temporal.score > 25 ? 'high' : 'medium',
        description: `High temporal complexity (${result.temporal.totalEffects} effects)`,
      });
    }

    // High coupling
    if (result.coupling.score > 15) {
      issues.push({
        type: 'coupling',
        severity: result.coupling.score > 25 ? 'high' : 'medium',
        description: `High coupling (${result.coupling.propsCount} props, ${result.coupling.contextConsumers} contexts)`,
      });
    }

    return issues;
  }

  /**
   * Estimate bug probability using Halstead bugs metric
   */
  private estimateBugProbability(result: ReactComplexityResult): number {
    // Based on Halstead bugs estimate and complexity
    const halsteadBugs = result.traditional.halstead.bugs;
    const complexityFactor = result.total / 100;

    // Combine estimates
    const probability = Math.min(1, halsteadBugs + complexityFactor * 0.3);

    return Math.round(probability * 100) / 100;
  }

  /**
   * Calculate priority score for addressing a hotspot
   */
  private calculatePriority(result: ReactComplexityResult, bugProb: number): number {
    // Priority based on:
    // - Bug probability (40%)
    // - Overall complexity (30%)
    // - Risky effects (20%)
    // - Coupling (10%)

    const bugWeight = bugProb * 40;
    const complexityWeight = (result.total / 100) * 30;
    const effectWeight = ((result.temporal.riskyEffects ?? 0) / 5) * 20;
    const couplingWeight = (result.coupling.score / 30) * 10;

    return Math.min(10, Math.round(bugWeight + complexityWeight + effectWeight + couplingWeight));
  }

  /**
   * Estimate cost of delay (hours per month if not fixed)
   */
  private estimateCostOfDelay(result: ReactComplexityResult, priority: number): number {
    // Estimate hours per month cost of not fixing
    // Based on complexity and priority

    const baseHours = result.total * 0.1; // Base: 0.1 hours per complexity point
    const priorityMultiplier = 1 + priority / 10;

    return Math.round(baseHours * priorityMultiplier);
  }

  /**
   * Generate actionable recommendations
   */
  private generateRecommendations(
    result: ReactComplexityResult,
    _issues: HotspotIssue[]
  ): string[] {
    const recommendations: string[] = [];

    if (result.hooks.totalHooks > 8) {
      recommendations.push('Extract related hooks into custom hooks');
    }

    if (result.temporal.totalEffects > 3) {
      recommendations.push('Consolidate effects or extract into custom hooks');
    }

    if (result.structural.branches > 10) {
      recommendations.push('Split into smaller, focused components');
    }

    if (result.coupling.propsCount > 10) {
      recommendations.push('Group related props or use composition');
    }

    if (result.temporal.riskyEffects && result.temporal.riskyEffects > 0) {
      recommendations.push('Review effects for cleanup and dependency issues');
    }

    return recommendations;
  }

  // ==========================================================================
  // Maintainability Analysis
  // ==========================================================================

  /**
   * Analyze maintainability using classic and React-adapted MI
   */
  private analyzeMaintainability(): MaintainabilityMetrics {
    const complexityResult = this.options.complexityResult;
    const linesOfCode = this.options.linesOfCode ?? 100;

    if (!complexityResult) {
      return {
        maintainabilityIndex: 100,
        reactMaintainabilityIndex: 100,
        factors: this.emptyFactors(),
      };
    }

    const factors = this.calculateFactors(complexityResult, linesOfCode);

    // Classic MI formula: 171 - 5.2*ln(V) - 0.23*CC - 16.2*ln(LOC)
    // Normalized to 0-100
    const halsteadVolume = complexityResult.traditional.halstead.volume || 1;
    const cyclomatic = complexityResult.traditional.cyclomaticComplexity;

    const classicMI =
      Math.max(
        0,
        Math.min(
          100,
          171 - 5.2 * Math.log(halsteadVolume) - 0.23 * cyclomatic - 16.2 * Math.log(linesOfCode)
        )
      ) / 1.71; // Normalize to 0-100

    // React-adapted MI includes hook and coupling complexity
    const reactMI = this.calculateReactMI(classicMI, complexityResult);

    return {
      maintainabilityIndex: Math.round(classicMI),
      reactMaintainabilityIndex: Math.round(reactMI),
      factors,
    };
  }

  /**
   * Calculate maintainability factors
   */
  private calculateFactors(result: ReactComplexityResult, loc: number): MaintainabilityFactors {
    return {
      halsteadVolume: result.traditional.halstead.volume,
      cyclomaticComplexity: result.traditional.cyclomaticComplexity,
      linesOfCode: loc,
      hookComplexity: result.hooks.score,
      couplingComplexity: result.coupling.score,
      typeCoverage: 100, // Would need type analyzer integration
    };
  }

  /**
   * Calculate React-adapted Maintainability Index
   */
  private calculateReactMI(classicMI: number, result: ReactComplexityResult): number {
    // Adjust classic MI with React-specific factors
    const hookPenalty = result.hooks.score > 20 ? (result.hooks.score - 20) * 0.5 : 0;
    const couplingPenalty = result.coupling.score > 15 ? (result.coupling.score - 15) * 0.3 : 0;
    const temporalPenalty = result.temporal.riskyEffects ? result.temporal.riskyEffects * 2 : 0;

    return Math.max(0, classicMI - hookPenalty - couplingPenalty - temporalPenalty);
  }

  // ==========================================================================
  // Debt Interest Calculation
  // ==========================================================================

  /**
   * Calculate technical debt interest model
   */
  private calculateDebtInterest(
    health: CodeHealthMetrics,
    maint: MaintainabilityMetrics
  ): TechnicalDebtInterest {
    const principalDebt = health.hotspots.reduce((sum, h) => sum + h.costOfDelay * 10, 0);
    const interestRate = 100 - maint.reactMaintainabilityIndex; // Higher complexity = higher interest

    const compoundingFactors: string[] = [];
    if (health.overallHealth < 5) compoundingFactors.push('Low code health');
    if (maint.reactMaintainabilityIndex < 50) compoundingFactors.push('Poor maintainability');
    if (health.hotspots.length > 3) compoundingFactors.push('Multiple hotspots');

    const payoffStrategies = this.generatePayoffStrategies(health, maint);

    return {
      principalDebt,
      interestRate,
      compoundingFactors,
      payoffStrategies,
    };
  }

  /**
   * Generate payoff strategies based on hotspots
   */
  private generatePayoffStrategies(
    health: CodeHealthMetrics,
    _maint: MaintainabilityMetrics
  ): PayoffStrategy[] {
    const strategies: PayoffStrategy[] = [];

    // Sort hotspots by priority
    const priorityHotspots = [...health.hotspots].sort((a, b) => b.priorityScore - a.priorityScore);

    priorityHotspots.slice(0, 3).forEach((hotspot, index) => {
      const effort = hotspot.complexity * 0.5; // Hours to refactor
      const benefit = hotspot.costOfDelay;
      const breakEvenPoint = benefit > 0 ? Math.ceil(effort / benefit) : Infinity;

      strategies.push({
        name: `Refactor ${hotspot.component || hotspot.file}`,
        effort,
        benefit,
        breakEvenPoint,
        priority: 10 - index,
      });
    });

    return strategies;
  }

  // ==========================================================================
  // Utility Methods
  // ==========================================================================

  /**
   * Calculate overall debt score from hotspots
   */
  private calculateDebtScore(hotspots: TechnicalDebtHotspot[]): number {
    return hotspots.reduce((sum, h) => sum + h.priorityScore * 5, 0);
  }

  /**
   * Estimate maintenance burden (hours per month)
   */
  private estimateMaintenanceBurden(hotspots: TechnicalDebtHotspot[]): number {
    return hotspots.reduce((sum, h) => sum + h.costOfDelay, 0);
  }

  /**
   * Convert health score to letter grade
   */
  private healthToGrade(health: number): 'A' | 'B' | 'C' | 'D' | 'F' {
    if (health >= 8) return 'A';
    if (health >= 6) return 'B';
    if (health >= 4) return 'C';
    if (health >= 2) return 'D';
    return 'F';
  }

  /**
   * Categorize debt by issue type
   */
  private categorizeDebt(hotspots: TechnicalDebtHotspot[]): DebtCategory[] {
    // Group issues by category
    const categories = new Map<
      string,
      { description: string; effort: number; impact: 'high' | 'medium' | 'low' }[]
    >();

    hotspots.forEach(h => {
      h.issues.forEach(issue => {
        if (!categories.has(issue.type)) {
          categories.set(issue.type, []);
        }
        categories.get(issue.type)!.push({
          description: issue.description,
          effort: 2, // Default estimate
          impact: issue.severity,
        });
      });
    });

    return Array.from(categories.entries()).map(([category, items]) => ({
      category,
      score: items.length * 10,
      items: items.map(item => ({
        ...item,
        location: { line: 0, column: 0 }, // Would need actual locations
      })),
    }));
  }

  /**
   * Calculate overall score (weighted combination)
   */
  private calculateOverallScore(health: CodeHealthMetrics, maint: MaintainabilityMetrics): number {
    // Weighted combination
    const healthWeight = 0.6;
    const maintWeight = 0.4;

    return Math.round(
      health.overallHealth * 10 * healthWeight + maint.reactMaintainabilityIndex * maintWeight
    );
  }

  /**
   * Empty factors for when no complexity data available
   */
  private emptyFactors(): MaintainabilityFactors {
    return {
      halsteadVolume: 0,
      cyclomaticComplexity: 0,
      linesOfCode: 0,
      hookComplexity: 0,
      couplingComplexity: 0,
      typeCoverage: 100,
    };
  }
}

// ============================================================================
// Convenience Function
// ============================================================================

/**
 * Analyze technical debt from source code
 *
 * @param source - Source code to analyze
 * @param options - Analysis options
 * @returns Technical debt analysis result
 *
 * @example
 * ```typescript
 * import { Project } from 'ts-morph';
 * import { analyzeTechnicalDebt } from './technical-debt-analyzer';
 * import { analyzeReactComplexity } from '../tsmorph-analyzer';
 *
 * const code = `...`;
 * const complexity = analyzeReactComplexity(code);
 * const result = analyzeTechnicalDebt(code, {
 *   complexityResult: complexity,
 *   linesOfCode: code.split('\n').length,
 *   filePath: 'MyComponent.tsx'
 * });
 * console.log(result.codeHealth.grade);
 * ```
 */
export function analyzeTechnicalDebt(
  source: string,
  options: TechnicalDebtAnalyzerOptions = {}
): TechnicalDebtAnalyzerResult {
  const { Project } = require('ts-morph');
  const project = new Project({ useInMemoryFileSystem: true });
  const sourceFile = project.createSourceFile('temp.tsx', source);
  const analyzer = new TechnicalDebtAnalyzer(sourceFile, options);
  return analyzer.analyze();
}
````

## Acceptance Criteria

### Functional Requirements

- [ ] Calculate Code Health Score (0-10 scale)
- [ ] Identify technical debt hotspots
- [ ] Calculate Maintainability Index (classic and React-adapted)
- [ ] Estimate maintenance burden
- [ ] Generate refactoring recommendations
- [ ] Calculate debt interest model
- [ ] Integrate with existing complexity analysis
- [ ] Export types and analyzer from barrel exports

### Scoring Accuracy

- [ ] Health score correlates with perceived code quality
- [ ] MI aligns with industry benchmarks (classic MI formula)
- [ ] Hotspot detection identifies actual problem areas
- [ ] Priority scoring is sensible and actionable

### Integration Requirements

- [ ] Export types from `src/types/index.ts`
- [ ] Export analyzer from `src/analyzers/index.ts`
- [ ] Works with existing `ReactComplexityResult`
- [ ] Compatible with CLI and future VSCode extension

### Code Quality

- [ ] All tests pass
- [ ] No TypeScript errors
- [ ] Consistent with existing analyzer patterns
- [ ] Well-documented with JSDoc comments

## Testing Instructions

### Unit Tests

**File:** `src/analyzers/technical-debt-analyzer.test.ts`

Create comprehensive unit tests following the patterns established in other analyzer tests:

```typescript
/**
 * Technical Debt Analyzer Tests
 *
 * @module analyzers/technical-debt-analyzer.test
 */

import { describe, it, expect } from 'vitest';
import { Project } from 'ts-morph';
import {
  TechnicalDebtAnalyzer,
  analyzeTechnicalDebt,
  type TechnicalDebtAnalyzerOptions,
} from './technical-debt-analyzer';
import { analyzeReactComplexity } from '../tsmorph-analyzer';

describe('TechnicalDebtAnalyzer', () => {
  const createAnalyzer = (code: string, options: Partial<TechnicalDebtAnalyzerOptions> = {}) => {
    const project = new Project({ useInMemoryFileSystem: true });
    const sourceFile = project.createSourceFile('test.tsx', code);

    // Get complexity result
    const complexityResult = analyzeReactComplexity(code);
    const linesOfCode = code.split('\n').length;

    return new TechnicalDebtAnalyzer(sourceFile, {
      complexityResult,
      linesOfCode,
      ...options,
    });
  };

  describe('Code Health Analysis', () => {
    it('should identify simple component with good health', () => {
      const code = `
        function SimpleButton({ label }: { label: string }) {
          return <button>{label}</button>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.codeHealth.grade).toBe('A');
      expect(result.codeHealth.hotspots.length).toBe(0);
      expect(result.codeHealth.overallHealth).toBeGreaterThan(8);
    });

    it('should identify complex component as hotspot', () => {
      const code = `
        function ComplexComponent({ data }: { data: any[] }) {
          const [state1, setState1] = useState(0);
          const [state2, setState2] = useState("");
          const [state3, setState3] = useState(false);
          
          useEffect(() => {
            // Effect 1
          }, [state1]);
          
          useEffect(() => {
            // Effect 2
          }, [state2]);
          
          useEffect(() => {
            // Effect 3
          }, [state3]);
          
          const value1 = useMemo(() => data.filter(x => x.active), [data]);
          const value2 = useMemo(() => data.map(x => x.id), [data]);
          
          const handler1 = useCallback(() => setState1(1), []);
          const handler2 = useCallback(() => setState2("x"), []);
          
          if (state1 > 0) {
            if (state2) {
              if (state3) {
                return <div>Nested</div>;
              }
            }
          }
          
          return (
            <div>
              {data.map(item => (
                <div key={item.id}>
                  {item.active ? <span>{item.name}</span> : null}
                </div>
              ))}
            </div>
          );
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.codeHealth.hotspots.length).toBeGreaterThan(0);
      expect(result.codeHealth.grade).toMatch(/[CDF]/);
      expect(result.codeHealth.overallHealth).toBeLessThan(6);
    });

    it('should estimate bug probability', () => {
      const code = `
        function BuggyComponent() {
          const [count, setCount] = useState(0);
          
          useEffect(() => {
            setCount(count + 1); // Bug: stale closure
          }, []); // Bug: missing dependency
          
          return <div>{count}</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      if (result.codeHealth.hotspots.length > 0) {
        const hotspot = result.codeHealth.hotspots[0];
        expect(hotspot.bugProbability).toBeGreaterThan(0);
      }
    });
  });

  describe('Maintainability Analysis', () => {
    it('should calculate high MI for simple component', () => {
      const code = `
        function SimpleComponent() {
          return <div>Hello</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.maintainability.maintainabilityIndex).toBeGreaterThan(80);
      expect(result.maintainability.reactMaintainabilityIndex).toBeGreaterThan(80);
    });

    it('should calculate lower MI for complex component', () => {
      const code = `
        function ComplexComponent({ items }: { items: any[] }) {
          const [state, setState] = useState(0);
          
          useEffect(() => { }, [state]);
          useEffect(() => { }, [state]);
          useEffect(() => { }, [state]);
          
          const filtered = useMemo(() => 
            items.filter(x => x.active).map(x => x.id), 
            [items]
          );
          
          if (state > 10) {
            return <div>A</div>;
          } else if (state > 5) {
            return <div>B</div>;
          } else {
            return <div>C</div>;
          }
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.maintainability.maintainabilityIndex).toBeLessThan(80);
    });

    it('should include React-specific factors in MI', () => {
      const code = `
        function ReactHeavyComponent() {
          const [a, setA] = useState(0);
          const [b, setB] = useState(0);
          const [c, setC] = useState(0);
          
          useEffect(() => { }, [a, b, c]);
          
          const memoized = useMemo(() => a + b + c, [a, b, c]);
          
          return <div>{memoized}</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.maintainability.factors.hookComplexity).toBeGreaterThan(0);
      expect(result.maintainability.reactMaintainabilityIndex).toBeLessThanOrEqual(
        result.maintainability.maintainabilityIndex
      );
    });
  });

  describe('Debt Interest Calculation', () => {
    it('should calculate debt interest for component with issues', () => {
      const code = `
        function ProblematicComponent() {
          const [state, setState] = useState(0);
          
          useEffect(() => { }, []);
          useEffect(() => { }, []);
          useEffect(() => { }, []);
          
          if (state > 0) {
            if (state > 5) {
              if (state > 10) {
                return <div>Deeply nested</div>;
              }
            }
          }
          
          return <div>{state}</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      expect(result.debtInterest.principalDebt).toBeGreaterThan(0);
      expect(result.debtInterest.interestRate).toBeGreaterThan(0);
    });

    it('should generate payoff strategies', () => {
      const code = `
        function ComplexComponent() {
          // Complex component code
          const [a, setA] = useState(0);
          const [b, setB] = useState(0);
          
          useEffect(() => { }, [a]);
          useEffect(() => { }, [b]);
          
          if (a > 0) {
            if (b > 0) {
              return <div>Nested</div>;
            }
          }
          
          return <div>Default</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      if (result.codeHealth.hotspots.length > 0) {
        expect(result.debtInterest.payoffStrategies.length).toBeGreaterThan(0);
        result.debtInterest.payoffStrategies.forEach(strategy => {
          expect(strategy.name).toBeTruthy();
          expect(strategy.effort).toBeGreaterThan(0);
          expect(strategy.priority).toBeGreaterThan(0);
        });
      }
    });
  });

  describe('Convenience Function', () => {
    it('should analyze technical debt from source string', () => {
      const code = `
        function TestComponent() {
          const [count, setCount] = useState(0);
          return <button onClick={() => setCount(count + 1)}>{count}</button>;
        }
      `;

      const complexity = analyzeReactComplexity(code);
      const result = analyzeTechnicalDebt(code, {
        complexityResult: complexity,
        linesOfCode: code.split('\n').length,
      });

      expect(result.score).toBeGreaterThanOrEqual(0);
      expect(result.score).toBeLessThanOrEqual(100);
      expect(result.codeHealth).toBeDefined();
      expect(result.maintainability).toBeDefined();
      expect(result.debtInterest).toBeDefined();
      expect(result.metadata).toBeDefined();
    });
  });

  describe('Recommendations', () => {
    it('should generate actionable recommendations', () => {
      const code = `
        function ComponentWithIssues() {
          const [s1, setS1] = useState(0);
          const [s2, setS2] = useState(0);
          const [s3, setS3] = useState(0);
          const [s4, setS4] = useState(0);
          const [s5, setS5] = useState(0);
          const [s6, setS6] = useState(0);
          
          useEffect(() => { }, [s1]);
          useEffect(() => { }, [s2]);
          useEffect(() => { }, [s3]);
          useEffect(() => { }, [s4]);
          
          return <div>Many hooks</div>;
        }
      `;

      const analyzer = createAnalyzer(code);
      const result = analyzer.analyze();

      if (result.codeHealth.hotspots.length > 0) {
        const hotspot = result.codeHealth.hotspots[0];
        expect(hotspot.recommendations.length).toBeGreaterThan(0);
        expect(hotspot.recommendations.some(r => r.includes('hook'))).toBe(true);
      }
    });
  });
});
```

### Manual Testing

1. **Test Code Health Score**

   ```bash
   # Run only technical debt tests
   npm run test -- technical-debt-analyzer.test.ts
   ```

2. **Test with Sample Components**

   ```bash
   # Analyze simple component (should have high health)
   npm run analyze -- src/sample-components/SimpleButton.tsx

   # Look for technical debt metrics in output
   # Expected: Grade A, low debt score, high maintainability
   ```

3. **Test with Complex Component**

   ```bash
   # Analyze complex component (should identify hotspots)
   npm run analyze -- src/sample-components/ComplexDashboard.tsx

   # Expected: Lower grade, hotspots identified, recommendations provided
   ```

## Estimated Effort

| Task                           | Model  | Estimated Time |
| ------------------------------ | ------ | -------------- |
| 4.0 Barrel Export Updates      | Haiku  | 0.5 hours      |
| 4.1 Technical Debt Types       | Opus   | 1.5 hours      |
| 4.2 Technical Debt Analyzer    | Opus   | 4 hours        |
| 4.3 Unit Tests                 | Sonnet | 3 hours        |
| 4.4 CLI Integration (optional) | Sonnet | 2 hours        |
| 4.5 Documentation              | Haiku  | 1 hour         |
| **Total**                      |        | **12 hours**   |

### Optional Task 4.4: CLI Integration

If you want to expose technical debt metrics via CLI, add to `src/cli.ts`:

```typescript
// In the main analyze function, after complexity analysis:
if (options.includeDebt) {
  const { TechnicalDebtAnalyzer } = await import('./analyzers/technical-debt-analyzer');
  const debtAnalyzer = new TechnicalDebtAnalyzer(sourceFile, {
    complexityResult: result,
    linesOfCode: sourceCode.split('\n').length,
    filePath: file,
  });

  const debtResult = debtAnalyzer.analyze();
  // Add to output
}
```

This is optional as Phase 04 focuses on the core analyzer implementation. CLI integration can be done later as needed.

---

## Implementation Notes

### Dependencies

This phase requires:

- ✅ Phase 00 (Foundation) - Base analyzer and type system
- ✅ `tsmorph-analyzer.ts` - For `ReactComplexityResult` input
- ✅ Existing type system and utilities

No external dependencies are required beyond what's already in the project.

### Architecture Decisions

1. **Analyzer Pattern:** Follows `BaseAnalyzer` pattern established in Phase 00
2. **Separation of Concerns:** Debt analysis is separate from complexity analysis
3. **Composability:** Can be used standalone or integrated into larger pipelines
4. **Type Safety:** Full TypeScript types with proper exports

### Migration Notes

- No backwards compatibility concerns (pre-launch)
- No Babel migration needed (ts-morph only)
- Clean integration with existing codebase

---

**Document Version:** 2.0  
**Created:** January 10, 2026  
**Last Updated:** January 10, 2026  
**Status:** Ready for Implementation  
**Reviewed:** ✅ Against current codebase (ts-morph, Phase 00 foundation)
