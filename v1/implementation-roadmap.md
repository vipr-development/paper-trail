# Implementation Roadmap: React Analyzer Enhancements

**Document Purpose:** Technical implementation guide for enhancing the React analyzer with migration, performance, technical debt, and anti-pattern detection capabilities.

**Target Audience:** Development team implementing the enhancements.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Phase 1: Core Enhancements](#phase-1-core-enhancements)
3. [Phase 2: VS Code Extension Foundation](#phase-2-vs-code-extension-foundation)
4. [Technical Specifications](#technical-specifications)
5. [Testing Strategy](#testing-strategy)
6. [Performance Considerations](#performance-considerations)

---

## Architecture Overview

### Current Architecture

```
analyzers/react/
├── src/
│   ├── analyzer.ts              ← Main orchestrator
│   ├── type-analyzer.ts         ← Specialized analyzer
│   ├── dataflow-analyzer.ts     ← Specialized analyzer
│   ├── reliability-analyzer.ts  ← Specialized analyzer
│   ├── refactoring-engine.ts    ← Suggestion generator
│   ├── constants.ts             ← Weights and thresholds
│   ├── types.ts                 ← Type definitions
│   └── utils/
│       ├── ast-helpers.ts
│       └── scoring.ts
└── schema.json
```

### Proposed Enhanced Architecture

```
analyzers/react/
├── src/
│   ├── analyzer.ts              ← Main orchestrator (enhanced)
│   ├── analyzers/               ← NEW: Modular analyzer directory
│   │   ├── base-analyzer.ts     ← Abstract base class
│   │   ├── migration-analyzer.ts      ← NEW
│   │   ├── performance-analyzer.ts    ← NEW
│   │   ├── antipattern-analyzer.ts   ← NEW
│   │   ├── security-analyzer.ts       ← NEW
│   │   ├── accessibility-analyzer.ts  ← NEW
│   │   ├── type-analyzer.ts           ← MOVE HERE
│   │   ├── dataflow-analyzer.ts       ← MOVE HERE
│   │   └── reliability-analyzer.ts    ← MOVE HERE
│   ├── rules/                   ← NEW: Rule definitions
│   │   ├── anti-patterns/
│   │   │   ├── hooks-rules.ts
│   │   │   ├── performance-rules.ts
│   │   │   └── state-rules.ts
│   │   ├── security/
│   │   │   ├── xss-rules.ts
│   │   │   └── injection-rules.ts
│   │   └── a11y/
│   │       └── wcag-rules.ts
│   ├── plugins/                 ← NEW: Plugin system
│   │   ├── plugin-interface.ts
│   │   └── plugin-loader.ts
│   ├── utils/
│   │   ├── ast-helpers.ts
│   │   ├── scoring.ts
│   │   ├── react-version.ts     ← NEW: Version detection
│   │   └── pattern-matchers.ts  ← NEW: Pattern recognition
│   ├── constants/               ← NEW: Organized constants
│   │   ├── weights.ts
│   │   ├── thresholds.ts
│   │   ├── react-api-catalog.ts ← NEW: React APIs by version
│   │   └── anti-patterns.ts     ← NEW: Pattern definitions
│   ├── refactoring-engine.ts
│   ├── types/                   ← NEW: Organized types
│   │   ├── core.ts
│   │   ├── migration.ts         ← NEW
│   │   ├── performance.ts       ← NEW
│   │   ├── anti-patterns.ts     ← NEW
│   │   └── index.ts
│   └── cli.ts
├── schema.json                  ← UPDATE: Extended schema
└── config/
    └── default-config.json      ← NEW: Default configuration
```

---

## Phase 1: Core Enhancements

### Milestone 1.1: Migration Analysis (2-3 weeks)

#### 1.1.1 React Version Detection

**File:** `src/utils/react-version.ts`

```typescript
import * as t from '@babel/types';
import { NodePath } from '@babel/traverse';

export interface ReactVersionInfo {
  detectedVersion: string | null;
  versionRange: string | null;
  confidenceLevel: 'high' | 'medium' | 'low';
  detectionMethod: 'packageJson' | 'imports' | 'apiUsage';
  evidence: VersionEvidence[];
}

interface VersionEvidence {
  indicator: string;
  version: string;
  confidence: number;
  location?: CodeLocation;
}

export class ReactVersionDetector {
  private apiByVersion: Map<string, string[]>;

  constructor() {
    // Load React API catalog
    this.apiByVersion = this.loadApiCatalog();
  }

  /**
   * Detect React version from AST analysis
   */
  detectFromAST(ast: t.File): ReactVersionInfo {
    const evidence: VersionEvidence[] = [];

    // 1. Check imports for version-specific APIs
    traverse(ast, {
      ImportDeclaration: path => {
        if (path.node.source.value === 'react') {
          path.node.specifiers.forEach(spec => {
            if (t.isImportSpecifier(spec)) {
              const importedName = spec.imported.name;
              const version = this.getVersionForAPI(importedName);
              if (version) {
                evidence.push({
                  indicator: `Import: ${importedName}`,
                  version,
                  confidence: 0.9,
                  location: this.getLocation(path),
                });
              }
            }
          });
        }
      },

      // 2. Check for version-specific hook usage
      CallExpression: path => {
        const callee = path.node.callee;
        if (t.isIdentifier(callee)) {
          const hookName = callee.name;
          if (this.isReact19Hook(hookName)) {
            evidence.push({
              indicator: `Hook: ${hookName}`,
              version: '19.0.0',
              confidence: 0.95,
            });
          }
        }
      },
    });

    return this.analyzeEvidence(evidence);
  }

  /**
   * Detect from package.json
   */
  detectFromPackageJson(packageJson: any): ReactVersionInfo {
    const reactVersion = packageJson.dependencies?.react || packageJson.devDependencies?.react;

    if (reactVersion) {
      return {
        detectedVersion: this.parseVersion(reactVersion),
        versionRange: reactVersion,
        confidenceLevel: 'high',
        detectionMethod: 'packageJson',
        evidence: [
          {
            indicator: 'package.json',
            version: reactVersion,
            confidence: 1.0,
          },
        ],
      };
    }

    return this.createUnknownVersion();
  }

  private loadApiCatalog(): Map<string, string[]> {
    // Load from constants/react-api-catalog.ts
    return new Map([
      ['19.0.0', ['use', 'useOptimistic', 'useActionState', 'useFormStatus']],
      ['18.0.0', ['useId', 'useDeferredValue', 'useTransition', 'useSyncExternalStore']],
      ['17.0.0', ['startTransition']],
      [
        '16.8.0',
        ['useState', 'useEffect', 'useContext', 'useReducer', 'useCallback', 'useMemo', 'useRef'],
      ],
    ]);
  }

  private isReact19Hook(hookName: string): boolean {
    return ['use', 'useOptimistic', 'useActionState', 'useFormStatus', 'useFormState'].includes(
      hookName
    );
  }

  private analyzeEvidence(evidence: VersionEvidence[]): ReactVersionInfo {
    if (evidence.length === 0) {
      return this.createUnknownVersion();
    }

    // Weight evidence by confidence and frequency
    const versionScores = new Map<string, number>();

    evidence.forEach(e => {
      const current = versionScores.get(e.version) || 0;
      versionScores.set(e.version, current + e.confidence);
    });

    // Get highest scoring version
    let maxScore = 0;
    let detectedVersion = null;

    versionScores.forEach((score, version) => {
      if (score > maxScore) {
        maxScore = score;
        detectedVersion = version;
      }
    });

    const confidenceLevel = maxScore >= 2 ? 'high' : maxScore >= 1 ? 'medium' : 'low';

    return {
      detectedVersion,
      versionRange: `>=${detectedVersion}`,
      confidenceLevel,
      detectionMethod: 'apiUsage',
      evidence,
    };
  }

  private parseVersion(versionString: string): string {
    // Handle ^18.0.0, ~18.0.0, >=18.0.0, etc.
    return versionString.replace(/^[\^~>=<]+/, '');
  }

  private createUnknownVersion(): ReactVersionInfo {
    return {
      detectedVersion: null,
      versionRange: null,
      confidenceLevel: 'low',
      detectionMethod: 'apiUsage',
      evidence: [],
    };
  }
}
```

#### 1.1.2 Deprecated API Catalog

**File:** `src/constants/react-api-catalog.ts`

```typescript
export interface DeprecatedAPI {
  name: string;
  deprecatedIn: string;
  removedIn: string | null;
  replacement: string;
  automatable: boolean; // Can codemod fix this?
  codemods: string[];
  reason: string;
  migrationGuide: string;
}

export const DEPRECATED_APIS: DeprecatedAPI[] = [
  {
    name: 'componentWillMount',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'constructor or componentDidMount',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
  },
  {
    name: 'componentWillReceiveProps',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'getDerivedStateFromProps or componentDidUpdate',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
  },
  {
    name: 'componentWillUpdate',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'getSnapshotBeforeUpdate',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
  },
  {
    name: 'findDOMNode',
    deprecatedIn: '16.3.0',
    removedIn: null, // Still exists but discouraged
    replacement: 'refs with useRef or createRef',
    automatable: false,
    codemods: [],
    reason: 'Breaks encapsulation, prevents optimizations',
    migrationGuide: 'https://react.dev/reference/react-dom/findDOMNode',
  },
  {
    name: 'defaultProps',
    deprecatedIn: '18.3.0',
    removedIn: null,
    replacement: 'Default parameters or destructuring',
    automatable: true,
    codemods: ['default-props-to-default-parameters'],
    reason: 'Moving to ES6 defaults',
    migrationGuide:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#removed-deprecated-apis',
  },
  {
    name: 'propTypes',
    deprecatedIn: '15.5.0',
    removedIn: null, // Still in separate package
    replacement: 'TypeScript or Flow',
    automatable: false,
    codemods: [],
    reason: 'Type systems are better',
    migrationGuide: 'https://legacy.reactjs.org/docs/typechecking-with-proptypes.html',
  },
  {
    name: 'Legacy Context',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'Context API (React.createContext)',
    automatable: false,
    codemods: [],
    reason: 'New Context API is more efficient',
    migrationGuide: 'https://react.dev/reference/react/createContext',
  },
  {
    name: 'String Refs',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'Callback refs or createRef',
    automatable: true,
    codemods: ['string-ref-to-callback-ref'],
    reason: 'Performance and predictability issues',
    migrationGuide: 'https://react.dev/reference/react/Component#legacy-refs',
  },
];

export const REACT_19_BREAKING_CHANGES = [
  {
    change: 'Errors in render are not re-thrown',
    impact: 'Error boundaries behave differently',
    action: 'Review error boundary implementations',
    automatable: false,
  },
  {
    change: 'Removed: propTypes and defaultProps',
    impact: 'TypeScript/prop-types needed',
    action: 'Migrate to TypeScript or separate prop-types package',
    automatable: true,
    codemods: ['default-props-to-default-parameters'],
  },
  {
    change: 'Ref cleanup functions',
    impact: 'Ref callbacks can return cleanup',
    action: 'Review ref callback patterns',
    automatable: false,
  },
  {
    change: 'useContext now eagerly reads context',
    impact: 'May expose bugs in conditional context usage',
    action: 'Review conditional useContext calls',
    automatable: false,
  },
];
```

#### 1.1.3 Migration Analyzer Implementation

**File:** `src/analyzers/migration-analyzer.ts`

```typescript
import * as t from '@babel/types';
import traverse from '@babel/traverse';
import { ReactVersionDetector } from '../utils/react-version';
import { DEPRECATED_APIS, REACT_19_BREAKING_CHANGES } from '../constants/react-api-catalog';

export interface MigrationMetrics {
  versionInfo: ReactVersionInfo;
  readinessScore: number; // 0-100
  targetVersion: string;
  blockers: MigrationBlocker[];
  warnings: MigrationWarning[];
  opportunities: MigrationOpportunity[];
  estimatedEffort: MigrationEffort;
  codemods: string[];
}

export class MigrationAnalyzer {
  private versionDetector: ReactVersionDetector;
  private targetVersion: string;

  constructor(targetVersion: string = '19.0.0') {
    this.versionDetector = new ReactVersionDetector();
    this.targetVersion = targetVersion;
  }

  analyze(ast: t.File, packageJson?: any): MigrationMetrics {
    // 1. Detect current version
    const versionInfo = packageJson
      ? this.versionDetector.detectFromPackageJson(packageJson)
      : this.versionDetector.detectFromAST(ast);

    // 2. Find deprecated API usage
    const deprecations = this.findDeprecatedAPIs(
      ast,
      versionInfo.detectedVersion,
      this.targetVersion
    );

    // 3. Find class components that should be migrated
    const classComponents = this.findClassComponents(ast);

    // 4. Check for breaking changes
    const breakingChanges = this.checkBreakingChanges(ast, this.targetVersion);

    // 5. Find modernization opportunities
    const opportunities = this.findModernizationOpportunities(ast);

    // 6. Calculate readiness score
    const readinessScore = this.calculateReadinessScore({
      deprecations,
      classComponents,
      breakingChanges,
      opportunities,
    });

    // 7. Estimate effort
    const estimatedEffort = this.estimateEffort({
      deprecations,
      classComponents,
      breakingChanges,
    });

    // 8. Collect applicable codemods
    const codemods = this.collectCodemods(deprecations);

    return {
      versionInfo,
      readinessScore,
      targetVersion: this.targetVersion,
      blockers: [...deprecations, ...breakingChanges],
      warnings: [],
      opportunities,
      estimatedEffort,
      codemods,
    };
  }

  private findDeprecatedAPIs(
    ast: t.File,
    currentVersion: string,
    targetVersion: string
  ): MigrationBlocker[] {
    const blockers: MigrationBlocker[] = [];

    traverse(ast, {
      // Check lifecycle methods
      ClassMethod: path => {
        const methodName = path.node.key.name;
        const deprecatedAPI = DEPRECATED_APIS.find(api => api.name === methodName);

        if (deprecatedAPI && this.isDeprecatedInTarget(deprecatedAPI, targetVersion)) {
          blockers.push({
            type: deprecatedAPI.removedIn ? 'breaking-change' : 'deprecated-api',
            severity: deprecatedAPI.removedIn ? 'critical' : 'high',
            location: this.getLocation(path),
            description: `${methodName} is deprecated: ${deprecatedAPI.reason}`,
            replacementSuggestion: deprecatedAPI.replacement,
            codemods: deprecatedAPI.codemods,
            migrationGuide: deprecatedAPI.migrationGuide,
          });
        }
      },

      // Check for findDOMNode
      CallExpression: path => {
        if (this.isFindDOMNodeCall(path.node)) {
          const deprecatedAPI = DEPRECATED_APIS.find(api => api.name === 'findDOMNode');
          blockers.push({
            type: 'deprecated-api',
            severity: 'high',
            location: this.getLocation(path),
            description: 'findDOMNode is deprecated',
            replacementSuggestion: deprecatedAPI.replacement,
            codemods: [],
            migrationGuide: deprecatedAPI.migrationGuide,
          });
        }
      },

      // Check for string refs
      JSXAttribute: path => {
        if (path.node.name.name === 'ref' && t.isStringLiteral(path.node.value)) {
          blockers.push({
            type: 'deprecated-api',
            severity: 'medium',
            location: this.getLocation(path),
            description: 'String refs are deprecated',
            replacementSuggestion: 'Use callback refs or createRef/useRef',
            codemods: ['string-ref-to-callback-ref'],
            migrationGuide: 'https://react.dev/reference/react/Component#legacy-refs',
          });
        }
      },
    });

    return blockers;
  }

  private findClassComponents(ast: t.File): ClassComponentInfo[] {
    const classComponents: ClassComponentInfo[] = [];

    traverse(ast, {
      ClassDeclaration: path => {
        if (this.isReactComponent(path.node)) {
          classComponents.push({
            name: path.node.id?.name || 'Anonymous',
            location: this.getLocation(path),
            lifecycleMethods: this.getLifecycleMethods(path.node),
            complexity: this.estimateClassComplexity(path.node),
            canAutoConvert: this.canAutoConvertToFunction(path.node),
          });
        }
      },
    });

    return classComponents;
  }

  private findModernizationOpportunities(ast: t.File): MigrationOpportunity[] {
    const opportunities: MigrationOpportunity[] = [];

    traverse(ast, {
      // HOC → Custom Hook opportunities
      CallExpression: path => {
        if (this.isHigherOrderComponent(path.node)) {
          opportunities.push({
            type: 'hoc-to-hook',
            location: this.getLocation(path),
            description: 'HOC could be replaced with custom hook',
            benefit: 'Simpler composition, better TypeScript support',
            effort: 'medium',
            automatable: false,
          });
        }
      },

      // Render props → Hook opportunities
      JSXElement: path => {
        if (this.isRenderPropsPattern(path.node)) {
          opportunities.push({
            type: 'render-props-to-hook',
            location: this.getLocation(path),
            description: 'Render props pattern could use custom hook',
            benefit: 'Cleaner syntax, reusable logic',
            effort: 'medium',
            automatable: false,
          });
        }
      },
    });

    return opportunities;
  }

  private calculateReadinessScore(data: any): number {
    const { deprecations, classComponents, breakingChanges } = data;

    let score = 100;

    // Deduct for blockers
    score -= breakingChanges.filter(b => b.severity === 'critical').length * 20;
    score -= deprecations.filter(d => d.severity === 'critical').length * 15;
    score -= deprecations.filter(d => d.severity === 'high').length * 10;
    score -= deprecations.filter(d => d.severity === 'medium').length * 5;

    // Deduct for class components (5 points each)
    score -= Math.min(classComponents.length * 5, 30);

    return Math.max(0, Math.min(100, score));
  }

  private estimateEffort(data: any): MigrationEffort {
    const { deprecations, classComponents, breakingChanges } = data;

    // Estimate hours
    let hours = 0;

    // Automated fixes are cheap
    const automated = deprecations.filter(d => d.codemods.length > 0);
    hours += automated.length * 0.5; // 30 min per auto fix (review time)

    // Manual fixes are expensive
    const manual = deprecations.filter(d => d.codemods.length === 0);
    hours += manual.length * 2; // 2 hours per manual fix

    // Class component conversions
    hours += classComponents.reduce((sum, cc) => {
      return sum + (cc.canAutoConvert ? 1 : 4);
    }, 0);

    // Breaking changes
    hours += breakingChanges.length * 3;

    // Automation potential
    const totalIssues = deprecations.length + classComponents.length;
    const automatable = automated.length + classComponents.filter(cc => cc.canAutoConvert).length;
    const automationPotential = totalIssues > 0 ? (automatable / totalIssues) * 100 : 0;

    return {
      hours: Math.ceil(hours),
      complexity: this.categorizeComplexity(hours),
      automationPotential,
    };
  }

  private categorizeComplexity(hours: number): string {
    if (hours < 2) return 'trivial';
    if (hours < 8) return 'simple';
    if (hours < 20) return 'moderate';
    if (hours < 40) return 'complex';
    return 'very-complex';
  }

  private collectCodemods(deprecations: MigrationBlocker[]): string[] {
    const codemods = new Set<string>();
    deprecations.forEach(d => {
      d.codemods.forEach(cm => codemods.add(cm));
    });
    return Array.from(codemods);
  }
}
```

### Milestone 1.2: Anti-Pattern Detection (2-3 weeks)

**File:** `src/analyzers/antipattern-analyzer.ts`

```typescript
import * as t from '@babel/types';
import traverse from '@babel/traverse';
import { ANTI_PATTERN_RULES } from '../rules/anti-patterns';

export interface AntiPatternMetrics {
  totalAntiPatterns: number;
  byCategory: Record<string, AntiPattern[]>;
  criticalCount: number;
  autoFixableCount: number;
  impactScore: number; // 0-100
}

export class AntiPatternAnalyzer {
  private rules: AntiPatternRule[];

  constructor() {
    this.rules = ANTI_PATTERN_RULES;
  }

  analyze(ast: t.File): AntiPatternMetrics {
    const antiPatterns: AntiPattern[] = [];

    // Run all rule checks
    this.rules.forEach(rule => {
      const violations = rule.check(ast);
      antiPatterns.push(...violations);
    });

    // Categorize results
    const byCategory = this.categorize(antiPatterns);
    const criticalCount = antiPatterns.filter(ap => ap.severity === 'critical').length;
    const autoFixableCount = antiPatterns.filter(ap => ap.autoFixable).length;
    const impactScore = this.calculateImpactScore(antiPatterns);

    return {
      totalAntiPatterns: antiPatterns.length,
      byCategory,
      criticalCount,
      autoFixableCount,
      impactScore,
    };
  }

  private categorize(antiPatterns: AntiPattern[]): Record<string, AntiPattern[]> {
    const categories: Record<string, AntiPattern[]> = {};

    antiPatterns.forEach(ap => {
      if (!categories[ap.category]) {
        categories[ap.category] = [];
      }
      categories[ap.category].push(ap);
    });

    return categories;
  }

  private calculateImpactScore(antiPatterns: AntiPattern[]): number {
    let score = 0;

    antiPatterns.forEach(ap => {
      switch (ap.severity) {
        case 'critical':
          score += 20;
          break;
        case 'warning':
          score += 10;
          break;
        case 'info':
          score += 3;
          break;
      }
    });

    return Math.min(100, score);
  }
}
```

**File:** `src/rules/anti-patterns/hooks-rules.ts`

```typescript
export const HOOKS_ANTI_PATTERNS: AntiPatternRule[] = [
  {
    id: 'conditional-hook-call',
    name: 'Conditional Hook Call',
    category: 'hooks',
    check: (ast: t.File) => {
      const violations: AntiPattern[] = [];

      traverse(ast, {
        CallExpression: path => {
          const callee = path.node.callee;

          // Check if it's a hook call
          if (t.isIdentifier(callee) && callee.name.startsWith('use')) {
            // Check if inside conditional/loop
            let parent = path.parentPath;
            while (parent) {
              if (
                parent.isIfStatement() ||
                parent.isConditionalExpression() ||
                parent.isForStatement() ||
                parent.isWhileStatement()
              ) {
                violations.push({
                  id: 'conditional-hook-call',
                  name: 'Conditional Hook Call',
                  category: 'hooks',
                  severity: 'critical',
                  location: getLocation(path),
                  description: `Hook ${callee.name} is called conditionally`,
                  impact: 'Violates Rules of Hooks, causes crashes',
                  fix: 'Move hook call to component top level',
                  autoFixable: false,
                  references: ['https://react.dev/reference/rules/rules-of-hooks'],
                });
                break;
              }
              parent = parent.parentPath;
            }
          }
        },
      });

      return violations;
    },
  },

  {
    id: 'missing-effect-dependencies',
    name: 'Missing Effect Dependencies',
    category: 'hooks',
    check: (ast: t.File) => {
      const violations: AntiPattern[] = [];

      traverse(ast, {
        CallExpression: path => {
          const callee = path.node.callee;

          if (t.isIdentifier(callee) && callee.name === 'useEffect') {
            const [callback, deps] = path.node.arguments;

            // Extract variables used in effect
            const usedVariables = extractUsedVariables(callback);

            // Check if deps array exists
            if (!deps) {
              violations.push({
                id: 'missing-effect-dependencies',
                name: 'Missing Effect Dependencies',
                category: 'hooks',
                severity: 'warning',
                location: getLocation(path),
                description: 'useEffect has no dependency array - runs every render',
                impact: 'Performance issue, potential infinite loops',
                fix: 'Add dependency array with required dependencies',
                autoFixable: true,
                references: [
                  'https://react.dev/reference/react/useEffect#my-effect-runs-after-every-re-render',
                ],
              });
            } else if (t.isArrayExpression(deps)) {
              // Check for missing dependencies
              const declaredDeps = deps.elements
                .filter(el => t.isIdentifier(el))
                .map(el => (el as t.Identifier).name);

              const missing = usedVariables.filter(v => !declaredDeps.includes(v));

              if (missing.length > 0) {
                violations.push({
                  id: 'missing-effect-dependencies',
                  name: 'Missing Effect Dependencies',
                  category: 'hooks',
                  severity: 'warning',
                  location: getLocation(path),
                  description: `useEffect is missing dependencies: ${missing.join(', ')}`,
                  impact: 'Stale closure bug, incorrect behavior',
                  fix: `Add to dependency array: [${[...declaredDeps, ...missing].join(', ')}]`,
                  autoFixable: true,
                  references: [
                    'https://react.dev/reference/react/useEffect#specifying-reactive-dependencies',
                  ],
                });
              }
            }
          }
        },
      });

      return violations;
    },
  },
];
```

### Milestone 1.3: Performance Analysis (2-3 weeks)

**File:** `src/analyzers/performance-analyzer.ts`

```typescript
export class PerformanceAnalyzer {
  analyze(ast: t.File): PerformanceMetrics {
    const issues: PerformanceIssue[] = [];

    // 1. Find missing memoization opportunities
    issues.push(...this.findMissingMemoization(ast));

    // 2. Find expensive operations in render
    issues.push(...this.findExpensiveRenderOperations(ast));

    // 3. Find list rendering issues
    issues.push(...this.findListRenderingIssues(ast));

    // 4. Find inline style/object creation
    issues.push(...this.findInlineObjectCreation(ast));

    // Calculate scores
    const score = this.calculatePerformanceScore(issues);
    const optimizationOpportunities = this.generateOptimizations(issues);

    return {
      score,
      issues,
      optimizationOpportunities,
      estimatedImpact: this.estimateImpact(issues),
    };
  }

  private findMissingMemoization(ast: t.File): PerformanceIssue[] {
    const issues: PerformanceIssue[] = [];

    traverse(ast, {
      // Find components receiving object/array props without memo
      JSXElement: path => {
        const openingElement = path.node.openingElement;

        // Check each prop
        openingElement.attributes.forEach(attr => {
          if (t.isJSXAttribute(attr) && t.isJSXExpressionContainer(attr.value)) {
            const expr = attr.value.expression;

            // Inline object or array
            if (t.isObjectExpression(expr) || t.isArrayExpression(expr)) {
              // Check if component is memoized
              const componentName = t.isJSXIdentifier(openingElement.name)
                ? openingElement.name.name
                : '';

              if (!this.isComponentMemoized(componentName, ast)) {
                issues.push({
                  type: 'missing-memo',
                  severity: 'warning',
                  location: getLocation(path),
                  description: `Component ${componentName} receives inline ${expr.type === 'ObjectExpression' ? 'object' : 'array'} but is not memoized`,
                  impact: 'Unnecessary re-renders on parent updates',
                  fix: `Wrap ${componentName} with React.memo() or extract prop to constant`,
                  estimatedImpact: 'medium',
                });
              }
            }
          }
        });
      },
    });

    return issues;
  }

  private findExpensiveRenderOperations(ast: t.File): PerformanceIssue[] {
    const issues: PerformanceIssue[] = [];

    traverse(ast, {
      FunctionDeclaration: path => {
        // Check if it's a component
        if (this.isComponent(path.node)) {
          // Look for expensive operations in render
          traverse(path.node, {
            CallExpression: callPath => {
              // Check for operations that should be memoized
              if (this.isExpensiveOperation(callPath.node)) {
                // Check if it's wrapped in useMemo
                if (!this.isInUseMemo(callPath)) {
                  issues.push({
                    type: 'expensive-render-operation',
                    severity: 'warning',
                    location: getLocation(callPath),
                    description: 'Expensive operation in render without useMemo',
                    impact: 'Performance degradation on every render',
                    fix: 'Wrap in useMemo with appropriate dependencies',
                    estimatedImpact: 'high',
                  });
                }
              }
            },
          });
        }
      },
    });

    return issues;
  }

  private findListRenderingIssues(ast: t.File): PerformanceIssue[] {
    const issues: PerformanceIssue[] = [];

    traverse(ast, {
      // Find .map() calls in JSX
      JSXExpressionContainer: path => {
        if (t.isCallExpression(path.node.expression)) {
          const callExpr = path.node.expression;

          if (
            t.isMemberExpression(callExpr.callee) &&
            t.isIdentifier(callExpr.callee.property) &&
            callExpr.callee.property.name === 'map'
          ) {
            // Check for key prop in mapped elements
            const callback = callExpr.arguments[0];

            if (t.isArrowFunctionExpression(callback) || t.isFunctionExpression(callback)) {
              const body = callback.body;

              if (t.isJSXElement(body)) {
                const keyProp = this.findKeyProp(body);

                if (!keyProp) {
                  issues.push({
                    type: 'missing-key-prop',
                    severity: 'warning',
                    location: getLocation(path),
                    description: 'List items rendered without key prop',
                    impact: 'React cannot efficiently update list',
                    fix: 'Add unique key prop to each element',
                    estimatedImpact: 'high',
                  });
                } else if (this.isIndexUsedAsKey(keyProp)) {
                  issues.push({
                    type: 'index-as-key',
                    severity: 'info',
                    location: getLocation(path),
                    description: 'Array index used as key (anti-pattern)',
                    impact: 'May cause issues when list items reorder',
                    fix: 'Use stable unique identifier as key',
                    estimatedImpact: 'medium',
                  });
                }
              }
            }
          }
        }
      },
    });

    return issues;
  }
}
```

---

## Phase 2: VS Code Extension Foundation

### Milestone 2.1: LSP Server Setup (2 weeks)

**File:** `vipr-vscode/src/server/server.ts`

```typescript
import {
  createConnection,
  TextDocuments,
  Diagnostic,
  DiagnosticSeverity,
  ProposedFeatures,
  InitializeParams,
  DidChangeConfigurationNotification,
  CompletionItem,
  CompletionItemKind,
  TextDocumentPositionParams,
  TextDocumentSyncKind,
  InitializeResult,
} from 'vscode-languageserver/node';

import { TextDocument } from 'vscode-languageserver-textdocument';
import { analyzeReactComplexity } from '@analyzers/react';

// Create LSP connection
const connection = createConnection(ProposedFeatures.all);
const documents: TextDocuments<TextDocument> = new TextDocuments(TextDocument);

let hasConfigurationCapability = false;
let hasWorkspaceFolderCapability = false;
let hasDiagnosticRelatedInformationCapability = false;

connection.onInitialize((params: InitializeParams) => {
  const capabilities = params.capabilities;

  hasConfigurationCapability = !!(capabilities.workspace && !!capabilities.workspace.configuration);
  hasWorkspaceFolderCapability = !!(
    capabilities.workspace && !!capabilities.workspace.workspaceFolders
  );
  hasDiagnosticRelatedInformationCapability = !!(
    capabilities.textDocument &&
    capabilities.textDocument.publishDiagnostics &&
    capabilities.textDocument.publishDiagnostics.relatedInformation
  );

  const result: InitializeResult = {
    capabilities: {
      textDocumentSync: TextDocumentSyncKind.Incremental,
      completionProvider: {
        resolveProvider: true,
      },
      codeActionProvider: true,
      hoverProvider: true,
    },
  };

  if (hasWorkspaceFolderCapability) {
    result.capabilities.workspace = {
      workspaceFolders: {
        supported: true,
      },
    };
  }

  return result;
});

connection.onInitialized(() => {
  if (hasConfigurationCapability) {
    connection.client.register(DidChangeConfigurationNotification.type, undefined);
  }
  if (hasWorkspaceFolderCapability) {
    connection.workspace.onDidChangeWorkspaceFolders(_event => {
      connection.console.log('Workspace folder change event received.');
    });
  }
});

// Analyze document and send diagnostics
async function analyzeDocument(textDocument: TextDocument): Promise<void> {
  const text = textDocument.getText();
  const uri = textDocument.uri;

  // Only analyze React/TypeScript files
  if (!uri.endsWith('.tsx') && !uri.endsWith('.jsx')) {
    return;
  }

  try {
    const result = analyzeReactComplexity(text);

    // Convert insights to diagnostics
    const diagnostics: Diagnostic[] = result.insights.map(insight => {
      const severity =
        insight.severity === 'critical'
          ? DiagnosticSeverity.Error
          : insight.severity === 'warning'
            ? DiagnosticSeverity.Warning
            : DiagnosticSeverity.Information;

      const diagnostic: Diagnostic = {
        severity,
        range: {
          start: textDocument.positionAt(insight.line ? insight.line - 1 : 0),
          end: textDocument.positionAt(insight.line ? insight.line : 0),
        },
        message: insight.message,
        source: 'vipr-react',
        code: insight.category,
      };

      if (insight.suggestion && hasDiagnosticRelatedInformationCapability) {
        diagnostic.relatedInformation = [
          {
            location: {
              uri: textDocument.uri,
              range: diagnostic.range,
            },
            message: insight.suggestion,
          },
        ];
      }

      return diagnostic;
    });

    // Send diagnostics to client
    connection.sendDiagnostics({ uri: textDocument.uri, diagnostics });
  } catch (error) {
    connection.console.error(`Error analyzing ${uri}: ${error}`);
  }
}

// Analyze on document open/change
documents.onDidOpen(e => {
  analyzeDocument(e.document);
});

documents.onDidChangeContent(e => {
  analyzeDocument(e.document);
});

// Make the text document manager listen on the connection
documents.listen(connection);

// Listen on the connection
connection.listen();
```

### Milestone 2.2: VS Code Extension Client (2 weeks)

**File:** `vipr-vscode/src/extension.ts`

```typescript
import * as path from 'path';
import { workspace, ExtensionContext, window, commands, StatusBarAlignment } from 'vscode';
import {
  LanguageClient,
  LanguageClientOptions,
  ServerOptions,
  TransportKind,
} from 'vscode-languageclient/node';

let client: LanguageClient;

export function activate(context: ExtensionContext) {
  // Server options
  const serverModule = context.asAbsolutePath(path.join('server', 'out', 'server.js'));

  const debugOptions = { execArgv: ['--nolazy', '--inspect=6009'] };

  const serverOptions: ServerOptions = {
    run: { module: serverModule, transport: TransportKind.ipc },
    debug: {
      module: serverModule,
      transport: TransportKind.ipc,
      options: debugOptions,
    },
  };

  // Client options
  const clientOptions: LanguageClientOptions = {
    documentSelector: [
      { scheme: 'file', language: 'typescriptreact' },
      { scheme: 'file', language: 'javascriptreact' },
    ],
    synchronize: {
      fileEvents: workspace.createFileSystemWatcher('**/.{tsx,jsx}'),
    },
  };

  // Create language client
  client = new LanguageClient(
    'viprReactAnalyzer',
    'Vipr React Analyzer',
    serverOptions,
    clientOptions
  );

  // Start client
  client.start();

  // Status bar
  const statusBar = window.createStatusBarItem(StatusBarAlignment.Right, 100);
  statusBar.text = '$(symbol-misc) Vipr';
  statusBar.command = 'vipr.showComplexity';
  statusBar.show();
  context.subscriptions.push(statusBar);

  // Commands
  context.subscriptions.push(
    commands.registerCommand('vipr.analyzeWorkspace', () => {
      // Analyze entire workspace
      window.showInformationMessage('Analyzing workspace...');
      // Implementation
    }),

    commands.registerCommand('vipr.showComplexity', () => {
      // Show complexity for current file
      // Implementation
    }),

    commands.registerCommand('vipr.checkMigrationReadiness', () => {
      // Check migration readiness
      // Implementation
    })
  );
}

export function deactivate(): Thenable<void> | undefined {
  if (!client) {
    return undefined;
  }
  return client.stop();
}
```

---

## Technical Specifications

### Performance Targets

| Metric                      | Target  | Notes                 |
| --------------------------- | ------- | --------------------- |
| Analysis Time (single file) | < 500ms | For files < 500 LOC   |
| Analysis Time (large file)  | < 2s    | For files < 2000 LOC  |
| Memory Usage                | < 100MB | Per analyzer instance |
| VSCode Extension Startup    | < 1s    | Initial activation    |
| Real-time Diagnostics       | < 200ms | After file change     |

### API Surface

```typescript
// Main analyzer API
export function analyzeReactComplexity(
  source: string,
  options?: AnalyzerOptions
): ReactComplexityResult;

// Specialized analyzers
export function analyzeMigration(source: string, targetVersion: string): MigrationMetrics;

export function analyzePerformance(source: string): PerformanceMetrics;

export function analyzeAntiPatterns(source: string): AntiPatternMetrics;

// Configuration
export interface AnalyzerOptions {
  enabledAnalyzers?: string[]; // Which analyzers to run
  thresholds?: ThresholdConfig;
  reactVersion?: string;
  ignorePatterns?: string[];
  customRules?: CustomRule[];
}
```

---

## Testing Strategy

### Unit Tests

```typescript
// Example: migration-analyzer.test.ts
describe('MigrationAnalyzer', () => {
  describe('version detection', () => {
    it('detects React 19 from use() hook', () => {
      const code = `
        import { use } from 'react';
        export function MyComponent() {
          const data = use(promise);
          return <div>{data}</div>;
        }
      `;

      const analyzer = new MigrationAnalyzer('19.0.0');
      const result = analyzer.analyze(parseCode(code));

      expect(result.versionInfo.detectedVersion).toBe('19.0.0');
      expect(result.versionInfo.confidenceLevel).toBe('high');
    });
  });

  describe('deprecated API detection', () => {
    it('detects componentWillMount', () => {
      const code = `
        class MyComponent extends React.Component {
          componentWillMount() {
            // ...
          }
        }
      `;

      const analyzer = new MigrationAnalyzer('19.0.0');
      const result = analyzer.analyze(parseCode(code));

      expect(result.blockers).toHaveLength(1);
      expect(result.blockers[0].type).toBe('deprecated-api');
      expect(result.blockers[0].description).toContain('componentWillMount');
    });
  });
});
```

### Integration Tests

```typescript
// Test full analysis pipeline
describe('Full Analysis Pipeline', () => {
  it('analyzes real-world component', async () => {
    const componentPath = path.join(__dirname, '../sample-components/DataTable.tsx');
    const code = fs.readFileSync(componentPath, 'utf-8');

    const result = analyzeReactComplexity(code);

    expect(result).toHaveProperty('total');
    expect(result).toHaveProperty('grade');
    expect(result).toHaveProperty('migration');
    expect(result).toHaveProperty('performance');
    expect(result).toHaveProperty('antiPatterns');
  });
});
```

### VS Code Extension Tests

```typescript
// Test extension activation
describe('Extension Activation', () => {
  it('activates successfully', async () => {
    const ext = vscode.extensions.getExtension('vipr.react-analyzer');
    await ext.activate();
    expect(ext.isActive).toBe(true);
  });

  it('provides diagnostics for React components', async () => {
    const doc = await vscode.workspace.openTextDocument({
      language: 'typescriptreact',
      content: 'export function MyComponent() { return <div>Hello</div>; }',
    });

    await vscode.languages.getDiagnostics(doc.uri);
    // Assertions on diagnostics
  });
});
```

---

## Performance Considerations

### 1. AST Caching

```typescript
class ASTCache {
  private cache = new Map<string, { ast: t.File; hash: string }>();

  get(filePath: string, content: string): t.File | null {
    const hash = this.hashContent(content);
    const cached = this.cache.get(filePath);

    if (cached && cached.hash === hash) {
      return cached.ast;
    }

    return null;
  }

  set(filePath: string, content: string, ast: t.File): void {
    const hash = this.hashContent(content);
    this.cache.set(filePath, { ast, hash });
  }

  private hashContent(content: string): string {
    return crypto.createHash('md5').update(content).digest('hex');
  }
}
```

### 2. Incremental Analysis

Only re-analyze changed sections when possible:

```typescript
function incrementalAnalyze(
  previousResult: ReactComplexityResult,
  changes: TextDocumentContentChangeEvent[]
): ReactComplexityResult {
  // If changes are minor, only re-run affected analyzers
  if (areChangesMinor(changes)) {
    return {
      ...previousResult,
      structural: reanalyzeStructural(changes),
    };
  }

  // Otherwise, full re-analysis
  return fullAnalyze();
}
```

### 3. Web Worker for VS Code

```typescript
// Offload heavy analysis to web worker
const worker = new Worker('./analyzer-worker.js');

worker.postMessage({ code, options });

worker.onmessage = event => {
  const result = event.data;
  updateDiagnostics(result);
};
```

---

## Next Steps

1. **Review and approve** this roadmap
2. **Set up Phase 1 project structure** with new directories
3. **Implement Milestone 1.1** (Migration Analysis)
4. **Create test suite** for migration analyzer
5. **Iterate based on feedback**

Each milestone should include:

- ✅ Implementation
- ✅ Unit tests (>80% coverage)
- ✅ Integration tests
- ✅ Documentation updates
- ✅ Schema updates
- ✅ Example outputs

---

**Document Version:** 1.0  
**Last Updated:** January 9, 2026  
**Status:** Draft - Awaiting Approval
