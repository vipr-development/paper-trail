# Phase 02: Anti-Pattern Detection

**Phase Duration:** 5-7 days  
**Priority:** High - Developer experience and code quality  
**Complexity:** High  
**Dependencies:** Phase 00 (Foundation) ✅  
**Status:** Ready for implementation

All code examples use the `ts-morph` API for AST parsing and traversal.

## Overview

This phase implements comprehensive detection of React anti-patterns, including hooks violations, performance anti-patterns, state management issues, and JSX problems. The system uses a modular rules engine that can be extended with custom rules.

## Business Value

- Catch bugs before they reach production
- Enforce React best practices automatically
- Reduce code review burden
- Improve team consistency
- Foundation for IDE integration (quick fixes)

## Agent Assignments

| Agent                    | Role                                       | Capacity  |
| ------------------------ | ------------------------------------------ | --------- |
| react-engineer           | Lead implementer, React patterns expertise | Primary   |
| typescript-engineer      | Type system, rule engine design            | Secondary |
| code-complexity-analyzer | Algorithm design for pattern matching      | Advisory  |

## Execution Strategy

### Milestone 2.1: Rules Engine Foundation (Day 1-2)

**Synchronous Tasks:**

1. Design rule interface (typescript-engineer)
2. Implement rule registry (typescript-engineer)
3. Review with react-engineer

**Parallel Tasks:**

- Rule type definitions (Haiku)
- Test infrastructure (Sonnet)

### Milestone 2.2: Hooks Anti-Patterns (Day 2-4)

**Parallel Tasks (can run concurrently):**

- Conditional hook detection (Sonnet)
- Missing dependencies detection (Opus - complex)
- Stale closure detection (Opus - complex)
- Effect cleanup detection (Sonnet)

### Milestone 2.3: Performance Anti-Patterns (Day 4-5)

**Parallel Tasks:**

- Inline function/object props (Sonnet)
- Index as key detection (Sonnet)
- Component in render detection (Opus - complex)

### Milestone 2.4: State & JSX Anti-Patterns (Day 5-6)

**Parallel Tasks:**

- Derived state detection (Sonnet)
- Prop drilling detection (Sonnet)
- JSX spreading detection (Sonnet)

### Milestone 2.5: Integration & Testing (Day 7)

**Synchronous Tasks:**

1. Integrate with main analyzer (react-engineer)
2. Schema updates (Haiku)
3. Documentation (Haiku)

## Detailed Tasks

### Task 2.1: Anti-Pattern Type Definitions

**Model:** Opus (architectural, foundational)
**File:** `src/types/anti-patterns.ts` (NEW FILE)

**Note:** This builds on the existing rule types in `src/types/rules.ts`. We create anti-pattern-specific types that integrate with the existing rule system.

```typescript
import { SourceFile } from 'ts-morph';
import type { CodeLocation, Severity, CodeFix } from './index';

/**
 * Anti-pattern categories for organization
 */
export type AntiPatternCategory =
  | 'hooks'
  | 'performance'
  | 'state-management'
  | 'lifecycle'
  | 'jsx'
  | 'props'
  | 'security'
  | 'testing'
  | 'accessibility';

/**
 * Detected anti-pattern instance
 */
export interface AntiPattern {
  /** Unique rule ID */
  id: string;
  /** Human-readable name */
  name: string;
  /** Category for grouping */
  category: AntiPatternCategory;
  /** Issue severity (using existing Severity type) */
  severity: Severity;
  /** Code location */
  location: CodeLocation;
  /** Description of the problem */
  description: string;
  /** Impact on code quality/performance */
  impact: string;
  /** How to fix */
  fix: string;
  /** Can be auto-fixed */
  autoFixable: boolean;
  /** Auto-fix details if available */
  autoFix?: CodeFix;
  /** Code snippet showing the issue */
  codeSnippet?: string;
  /** Reference documentation */
  references: string[];
}

/**
 * Anti-pattern rule interface
 *
 * Simplified rule interface specifically for anti-pattern detection.
 * This is separate from the more complex Rule interface in src/types/rules.ts
 * to keep anti-pattern rules simple and focused.
 */
export interface AntiPatternRule {
  /** Unique rule identifier (kebab-case) */
  id: string;
  /** Human-readable name */
  name: string;
  /** Rule category */
  category: AntiPatternCategory;
  /** Default severity */
  defaultSeverity: Severity;
  /** Rule description */
  description: string;
  /** Documentation links */
  references: string[];
  /** Whether rule supports auto-fix */
  fixable: boolean;

  /**
   * Check source file for violations
   *
   * @param sourceFile - ts-morph SourceFile
   * @returns Array of detected anti-patterns
   */
  check(sourceFile: SourceFile): AntiPattern[];

  /**
   * Optional: Generate fix for violation
   *
   * @param antiPattern - The detected issue
   * @param source - Original source code
   * @returns Fixed source or null
   */
  fix?(antiPattern: AntiPattern, source: string): string | null;
}

/**
 * Rule configuration for enabling/disabling and severity override
 */
export interface AntiPatternRuleConfig {
  /** Enable or disable rule */
  enabled: boolean;
  /** Override default severity */
  severity?: Severity;
  /** Rule-specific options */
  options?: Record<string, unknown>;
}

/**
 * Complete anti-pattern rules configuration
 */
export interface AntiPatternRulesConfig {
  /** Per-rule configuration */
  rules: Record<string, AntiPatternRuleConfig>;
  /** Categories to ignore */
  ignoreCategories?: AntiPatternCategory[];
  /** File patterns to ignore */
  ignorePatterns?: string[];
}
```

### Task 2.2: Anti-Pattern Rule Registry

**Model:** Sonnet (well-defined patterns)
**File:** `src/rules/anti-pattern-registry.ts` (NEW FILE)

**Note:** This is separate from the existing `RuleRegistry` in `src/rules/index.ts` to keep anti-pattern rules focused and simple.

```typescript
import type { AntiPatternRule, AntiPatternRulesConfig, AntiPattern } from '../types/anti-patterns';
import { SourceFile } from 'ts-morph';

// Import rule modules (to be created)
import { HOOKS_RULES } from './anti-patterns/hooks-rules';
import { PERFORMANCE_RULES } from './anti-patterns/performance-rules';
import { STATE_RULES } from './anti-patterns/state-rules';
import { JSX_RULES } from './anti-patterns/jsx-rules';

/**
 * Anti-Pattern Rule Registry
 *
 * Manages registration, configuration, and execution of anti-pattern detection rules.
 * This is a focused registry specifically for anti-pattern rules, separate from
 * the more general rule system.
 */
export class AntiPatternRegistry {
  private rules: Map<string, AntiPatternRule> = new Map();
  private config: AntiPatternRulesConfig;

  constructor(config?: AntiPatternRulesConfig) {
    this.config = config ?? { rules: {} };
    this.registerBuiltinRules();
  }

  /**
   * Register built-in anti-pattern rules
   */
  private registerBuiltinRules(): void {
    const allRules = [...HOOKS_RULES, ...PERFORMANCE_RULES, ...STATE_RULES, ...JSX_RULES];

    allRules.forEach(rule => this.register(rule));
  }

  /**
   * Register a single anti-pattern rule
   */
  register(rule: AntiPatternRule): void {
    if (this.rules.has(rule.id)) {
      throw new Error(`Anti-pattern rule with id "${rule.id}" already registered`);
    }
    this.rules.set(rule.id, rule);
  }

  /**
   * Get all registered rules
   */
  getAllRules(): AntiPatternRule[] {
    return Array.from(this.rules.values());
  }

  /**
   * Get enabled rules based on config
   */
  getEnabledRules(): AntiPatternRule[] {
    return this.getAllRules().filter(rule => {
      const ruleConfig = this.config.rules[rule.id];

      // If not explicitly configured, enabled by default
      if (!ruleConfig) return true;

      return ruleConfig.enabled;
    });
  }

  /**
   * Run all enabled rules against SourceFile
   */
  analyze(sourceFile: SourceFile): AntiPattern[] {
    const results: AntiPattern[] = [];
    const enabledRules = this.getEnabledRules();

    for (const rule of enabledRules) {
      try {
        const violations = rule.check(sourceFile);

        // Apply severity overrides
        const configuredViolations = violations.map(v => {
          const ruleConfig = this.config.rules[rule.id];
          if (ruleConfig?.severity) {
            return { ...v, severity: ruleConfig.severity };
          }
          return v;
        });

        results.push(...configuredViolations);
      } catch (error) {
        console.error(`Error running anti-pattern rule ${rule.id}:`, error);
      }
    }

    return results;
  }

  /**
   * Get rules by category
   */
  getRulesByCategory(category: string): AntiPatternRule[] {
    return this.getAllRules().filter(rule => rule.category === category);
  }
}

// Export default registry instance
export const defaultAntiPatternRegistry = new AntiPatternRegistry();
```

### Task 2.3: Hooks Anti-Pattern Rules

**Model:** Opus (complex pattern matching, accuracy critical)
**File:** `src/rules/anti-patterns/hooks-rules.ts` (NEW FILE)

```typescript
import { SourceFile, Node, SyntaxKind } from 'ts-morph';
import type { AntiPatternRule, AntiPattern } from '../../types/anti-patterns';
import type { CodeLocation } from '../../types';

/**
 * Helper to get code location from node
 */
function getLocation(node: Node): CodeLocation {
  const sourceFile = node.getSourceFile();
  const start = node.getStart();
  const lineStarts = sourceFile.getLineStarts();
  const lineNumber = node.getStartLineNumber();

  return {
    line: lineNumber,
    column: start - lineStarts[lineNumber - 1],
  };
}

/**
 * Conditional Hook Call Detection
 *
 * Hooks must be called at the top level of a component,
 * not inside conditionals, loops, or nested functions.
 */
const conditionalHookRule: AntiPatternRule = {
  id: 'conditional-hook-call',
  name: 'Conditional Hook Call',
  category: 'hooks',
  defaultSeverity: 'critical',
  description: 'Hooks must be called at the top level of a React function component',
  references: [
    'https://react.dev/reference/rules/rules-of-hooks',
    'https://react.dev/warnings/invalid-hook-call-warning',
  ],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isIdentifier(expr)) return;

      const hookName = expr.getText();
      if (!hookName.startsWith('use')) return;

      // Walk up the tree to find problematic parents
      let current = node.getParent();
      let componentFunction: Node | undefined = undefined;

      while (current) {
        // Track the component function boundary
        if (
          Node.isFunctionDeclaration(current) ||
          Node.isFunctionExpression(current) ||
          Node.isArrowFunction(current)
        ) {
          if (!componentFunction) {
            componentFunction = current;
          } else {
            // Hook is inside a nested function
            violations.push({
              id: 'conditional-hook-call',
              name: 'Hook in Nested Function',
              category: 'hooks',
              severity: 'critical',
              location: getLocation(node),
              description: `${hookName} is called inside a nested function`,
              impact: 'Violates Rules of Hooks, may cause crashes or incorrect behavior',
              fix: "Move the hook call to the component's top level",
              autoFixable: false,
              references: ['https://react.dev/reference/rules/rules-of-hooks'],
            });
            return;
          }
        }

        // Check for conditional parents
        if (Node.isIfStatement(current)) {
          violations.push({
            id: 'conditional-hook-call',
            name: 'Hook in Conditional',
            category: 'hooks',
            severity: 'critical',
            location: getLocation(node),
            description: `${hookName} is called inside an if statement`,
            impact: 'Violates Rules of Hooks - hooks must be called in the same order every render',
            fix: 'Move the hook outside the conditional, use conditional logic inside the hook instead',
            autoFixable: false,
            references: ['https://react.dev/reference/rules/rules-of-hooks'],
          });
          return;
        }

        // Check for loop parents
        if (
          Node.isForStatement(current) ||
          Node.isWhileStatement(current) ||
          Node.isDoStatement(current) ||
          Node.isForOfStatement(current) ||
          Node.isForInStatement(current)
        ) {
          violations.push({
            id: 'conditional-hook-call',
            name: 'Hook in Loop',
            category: 'hooks',
            severity: 'critical',
            location: getLocation(node),
            description: `${hookName} is called inside a loop`,
            impact: 'Violates Rules of Hooks - number of hook calls must be constant',
            fix: 'Create a separate component for each item that needs the hook',
            autoFixable: false,
            references: ['https://react.dev/reference/rules/rules-of-hooks'],
          });
          return;
        }

        // Check for ternary expression
        if (Node.isConditionalExpression(current)) {
          violations.push({
            id: 'conditional-hook-call',
            name: 'Hook in Ternary',
            category: 'hooks',
            severity: 'critical',
            location: getLocation(node),
            description: `${hookName} is called inside a ternary expression`,
            impact: 'Violates Rules of Hooks - hooks must be called unconditionally',
            fix: 'Move the hook outside the ternary expression',
            autoFixable: false,
            references: ['https://react.dev/reference/rules/rules-of-hooks'],
          });
          return;
        }

        current = current.getParent();
      }
    });

    return violations;
  },
};

/**
 * Missing useEffect Dependencies
 *
 * Detects when useEffect uses variables that aren't in the dependency array.
 * This is a simplified heuristic - full analysis requires scope tracking.
 */
const missingDepsRule: AntiPatternRule = {
  id: 'missing-effect-deps',
  name: 'Missing Effect Dependencies',
  category: 'hooks',
  defaultSeverity: 'medium',
  description: 'useEffect dependency array should include all reactive values used inside',
  references: [
    'https://react.dev/reference/react/useEffect#specifying-reactive-dependencies',
    'https://react.dev/learn/removing-effect-dependencies',
  ],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isIdentifier(expr)) return;

      const hookName = expr.getText();
      if (!['useEffect', 'useLayoutEffect', 'useInsertionEffect'].includes(hookName)) return;

      const args = node.getArguments();
      if (args.length === 0) return;

      const callback = args[0];
      const depsArg = args[1];

      // No dependency array at all
      if (!depsArg) {
        violations.push({
          id: 'missing-effect-deps',
          name: 'No Dependency Array',
          category: 'hooks',
          severity: 'medium',
          location: getLocation(node),
          description: `${hookName} has no dependency array - runs after every render`,
          impact:
            'Effect runs on every render, potentially causing performance issues or infinite loops',
          fix: 'Add a dependency array. Use [] for mount-only effects, or list all reactive dependencies.',
          autoFixable: false,
          references: ['https://react.dev/reference/react/useEffect#parameters'],
        });
        return;
      }

      // Check if deps argument is an array
      if (!Node.isArrayLiteralExpression(depsArg)) return;

      // Collect identifiers used inside the callback (simplified heuristic)
      const usedIdentifiers = new Set<string>();
      const declaredIdentifiers = new Set<string>();

      if (Node.isArrowFunction(callback) || Node.isFunctionExpression(callback)) {
        // Track parameters
        callback.getParameters().forEach(param => {
          const name = param.getName();
          if (name) declaredIdentifiers.add(name);
        });

        // Find all identifiers used
        callback.forEachDescendant(desc => {
          if (Node.isIdentifier(desc)) {
            const name = desc.getText();
            const parent = desc.getParent();

            // Skip if it's a property access key
            if (Node.isPropertyAccessExpression(parent) && parent.getNameNode() === desc) {
              return;
            }

            // Skip if it's a declaration
            if (Node.isVariableDeclaration(parent) && parent.getNameNode() === desc) {
              declaredIdentifiers.add(name);
              return;
            }

            // Skip if it's a function name
            if (Node.isFunctionDeclaration(parent) && parent.getNameNode() === desc) {
              declaredIdentifiers.add(name);
              return;
            }

            usedIdentifiers.add(name);
          }
        });
      }

      // Get declared dependencies
      const declaredDeps = new Set<string>();
      depsArg.getElements().forEach(el => {
        if (Node.isIdentifier(el)) {
          declaredDeps.add(el.getText());
        }
      });

      // Find potentially missing dependencies (heuristic)
      const potentiallyMissing: string[] = [];
      const globals = [
        'console',
        'window',
        'document',
        'setTimeout',
        'setInterval',
        'clearTimeout',
        'clearInterval',
        'fetch',
        'Promise',
        'Math',
        'JSON',
        'Array',
        'Object',
        'String',
        'Number',
        'Boolean',
        'Date',
        'Error',
        'undefined',
        'null',
        'React',
        'process',
      ];

      usedIdentifiers.forEach(name => {
        // Skip if declared inside callback
        if (declaredIdentifiers.has(name)) return;

        // Skip common globals
        if (globals.includes(name)) return;

        // Skip if already in deps
        if (declaredDeps.has(name)) return;

        // Skip if it looks like a constant (ALL_CAPS)
        if (/^[A-Z_]+$/.test(name)) return;

        potentiallyMissing.push(name);
      });

      if (potentiallyMissing.length > 0) {
        violations.push({
          id: 'missing-effect-deps',
          name: 'Potentially Missing Dependencies',
          category: 'hooks',
          severity: 'medium',
          location: getLocation(node),
          description: `${hookName} may be missing dependencies: ${potentiallyMissing.join(', ')}`,
          impact: 'May cause stale closure bugs where the effect uses outdated values',
          fix: `Add missing dependencies to the array, or use a ref if the value shouldn't trigger re-runs`,
          autoFixable: true,
          references: ['https://react.dev/learn/removing-effect-dependencies'],
        });
      }
    });

    return violations;
  },
};

/**
 * Effect Cleanup Missing
 *
 * Detects effects with subscriptions, timers, or listeners without cleanup.
 */
const missingCleanupRule: AntiPatternRule = {
  id: 'missing-effect-cleanup',
  name: 'Missing Effect Cleanup',
  category: 'hooks',
  defaultSeverity: 'medium',
  description: 'Effects that create subscriptions or timers should return cleanup functions',
  references: ['https://react.dev/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed'],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isIdentifier(expr) || expr.getText() !== 'useEffect') return;

      const args = node.getArguments();
      if (args.length === 0) return;

      const callback = args[0];
      if (!Node.isArrowFunction(callback) && !Node.isFunctionExpression(callback)) return;

      // Check if callback has return statement (cleanup)
      let hasCleanup = false;
      let hasSubscription = false;
      let subscriptionType: string | null = null;

      // Check for return statements
      callback.forEachDescendant(desc => {
        if (Node.isReturnStatement(desc)) {
          const returnExpr = desc.getExpression();
          if (
            returnExpr &&
            (Node.isArrowFunction(returnExpr) ||
              Node.isFunctionExpression(returnExpr) ||
              Node.isIdentifier(returnExpr))
          ) {
            hasCleanup = true;
          }
        }

        // Check for addEventListener
        if (Node.isCallExpression(desc)) {
          const callExpr = desc.getExpression();
          if (Node.isPropertyAccessExpression(callExpr)) {
            const methodName = callExpr.getName();
            if (methodName === 'addEventListener') {
              hasSubscription = true;
              subscriptionType = 'event listener';
            }
          }

          // Check for setInterval
          if (Node.isIdentifier(callExpr) && callExpr.getText() === 'setInterval') {
            hasSubscription = true;
            subscriptionType = 'setInterval';
          }

          // Check for setTimeout
          if (Node.isIdentifier(callExpr) && callExpr.getText() === 'setTimeout') {
            hasSubscription = true;
            subscriptionType = 'setTimeout';
          }
        }

        // Check for WebSocket, EventSource
        if (Node.isNewExpression(desc)) {
          const identifier = desc.getExpression();
          if (Node.isIdentifier(identifier)) {
            const name = identifier.getText();
            if (['WebSocket', 'EventSource'].includes(name)) {
              hasSubscription = true;
              subscriptionType = name;
            }
          }
        }
      });

      if (hasSubscription && !hasCleanup) {
        const severity =
          subscriptionType === 'setInterval' ? ('critical' as const) : ('medium' as const);

        violations.push({
          id: 'missing-effect-cleanup',
          name: `Missing Cleanup for ${subscriptionType}`,
          category: 'hooks',
          severity,
          location: getLocation(node),
          description: `useEffect creates ${subscriptionType} but has no cleanup function`,
          impact:
            subscriptionType === 'setInterval'
              ? 'Memory leak - interval continues running after component unmounts'
              : 'Potential memory leak or state updates on unmounted component',
          fix: `Return a cleanup function that removes/clears the ${subscriptionType}`,
          autoFixable: false,
          references: [
            'https://react.dev/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed',
          ],
        });
      }
    });

    return violations;
  },
};

/**
 * Async Effect Callback
 *
 * useEffect callback cannot be async directly.
 */
const asyncEffectRule: AntiPatternRule = {
  id: 'async-effect-callback',
  name: 'Async Effect Callback',
  category: 'hooks',
  defaultSeverity: 'critical',
  description: 'useEffect callback cannot be an async function directly',
  references: ['https://react.dev/reference/react/useEffect#fetching-data-with-effects'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isIdentifier(expr) || expr.getText() !== 'useEffect') return;

      const args = node.getArguments();
      if (args.length === 0) return;

      const callback = args[0];

      if (
        (Node.isArrowFunction(callback) || Node.isFunctionExpression(callback)) &&
        callback.isAsync()
      ) {
        violations.push({
          id: 'async-effect-callback',
          name: 'Async Effect Callback',
          category: 'hooks',
          severity: 'critical',
          location: getLocation(node),
          description:
            'useEffect callback is async, but effects must return undefined or a cleanup function',
          impact: "Async functions return Promises, which useEffect doesn't handle correctly",
          fix: 'Define an async function inside the effect and call it immediately',
          autoFixable: true,
          codeSnippet: `useEffect(() => {
  async function fetchData() {
    // your async code
  }
  fetchData();
}, [deps]);`,
          references: ['https://react.dev/reference/react/useEffect#fetching-data-with-effects'],
        });
      }
    });

    return violations;
  },
};

// Export all hooks rules
export const HOOKS_RULES: AntiPatternRule[] = [
  conditionalHookRule,
  missingDepsRule,
  missingCleanupRule,
  asyncEffectRule,
];
```

### Task 2.4: Performance Anti-Pattern Rules

**Model:** Sonnet (patterns are well-defined)
**File:** `src/rules/anti-patterns/performance-rules.ts` (NEW FILE)

```typescript
import { SourceFile, Node, SyntaxKind } from 'ts-morph';
import type { AntiPatternRule, AntiPattern } from '../../types/anti-patterns';
import type { CodeLocation } from '../../types';

function getLocation(node: Node): CodeLocation {
  const sourceFile = node.getSourceFile();
  const start = node.getStart();
  const lineStarts = sourceFile.getLineStarts();
  const lineNumber = node.getStartLineNumber();

  return {
    line: lineNumber,
    column: start - lineStarts[lineNumber - 1],
  };
}

/**
 * Inline Function Props
 */
const inlineFunctionPropsRule: AntiPatternRule = {
  id: 'inline-function-props',
  name: 'Inline Function Props',
  category: 'performance',
  defaultSeverity: 'low',
  description: 'Inline functions in JSX props create new references on every render',
  references: ['https://react.dev/reference/react/useCallback'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    // First, check if there are any memoized components (makes this more relevant)
    let hasMemoizedComponents = false;
    sourceFile.forEachDescendant(node => {
      if (Node.isCallExpression(node)) {
        const expr = node.getExpression();
        if (Node.isIdentifier(expr) && expr.getText() === 'memo') {
          hasMemoizedComponents = true;
        }
        if (Node.isPropertyAccessExpression(expr)) {
          const name = expr.getName();
          if (name === 'memo') {
            hasMemoizedComponents = true;
          }
        }
      }
    });

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxAttribute(node)) return;

      const nameNode = node.getNameNode();
      if (!Node.isIdentifier(nameNode)) return;

      const attrName = nameNode.getText();

      // Only check callback props (on* handlers)
      if (!attrName.startsWith('on')) return;

      const initializer = node.getInitializer();
      if (!Node.isJsxExpression(initializer)) return;

      const expr = initializer.getExpression();
      if (Node.isArrowFunction(expr) || Node.isFunctionExpression(expr)) {
        violations.push({
          id: 'inline-function-props',
          name: 'Inline Function Prop',
          category: 'performance',
          severity: hasMemoizedComponents ? 'medium' : 'low',
          location: getLocation(node),
          description: `Inline function in "${attrName}" creates new reference each render`,
          impact: hasMemoizedComponents
            ? 'Breaks memoization of child components'
            : 'Creates garbage on each render (minor performance impact)',
          fix: 'Extract to useCallback or define outside component if no deps',
          autoFixable: true,
          references: ['https://react.dev/reference/react/useCallback'],
        });
      }
    });

    return violations;
  },
};

/**
 * Index as Key
 */
const indexAsKeyRule: AntiPatternRule = {
  id: 'index-as-key',
  name: 'Index as Key',
  category: 'performance',
  defaultSeverity: 'medium',
  description: 'Using array index as key can cause issues when list items are reordered',
  references: ['https://react.dev/learn/rendering-lists#rules-of-keys'],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxAttribute(node)) return;

      const nameNode = node.getNameNode();
      if (!Node.isIdentifier(nameNode) || nameNode.getText() !== 'key') return;

      const initializer = node.getInitializer();
      if (!Node.isJsxExpression(initializer)) return;

      const expr = initializer.getExpression();

      // Check for direct index usage: key={index} or key={i}
      if (Node.isIdentifier(expr)) {
        const name = expr.getText();
        // Common index variable names
        if (['index', 'i', 'idx', 'key', 'k'].includes(name)) {
          // Check if inside a map callback
          let isInMap = false;
          let current = node.getParent();

          while (current) {
            if (Node.isCallExpression(current)) {
              const callExpr = current.getExpression();
              if (Node.isPropertyAccessExpression(callExpr)) {
                const methodName = callExpr.getName();
                if (methodName === 'map') {
                  isInMap = true;
                  break;
                }
              }
            }
            current = current.getParent();
          }

          if (isInMap) {
            violations.push({
              id: 'index-as-key',
              name: 'Index Used as Key',
              category: 'performance',
              severity: 'medium',
              location: getLocation(node),
              description: `Using "${name}" as key in .map() can cause issues`,
              impact:
                'React may not correctly update items when they are reordered, filtered, or sorted',
              fix: 'Use a unique, stable identifier from the data (e.g., id, uuid)',
              autoFixable: false,
              references: ['https://react.dev/learn/rendering-lists#rules-of-keys'],
            });
          }
        }
      }
    });

    return violations;
  },
};

/**
 * Component Definition in Render
 */
const componentInRenderRule: AntiPatternRule = {
  id: 'component-in-render',
  name: 'Component Definition in Render',
  category: 'performance',
  defaultSeverity: 'critical',
  description: 'Defining components inside other components causes them to remount on every render',
  references: ['https://react.dev/learn/your-first-component#nesting-and-organizing-components'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    // Helper to check if a function returns JSX
    function hasJSXReturn(node: Node): boolean {
      let hasJSX = false;

      // Check for arrow function with JSX body
      if (Node.isArrowFunction(node)) {
        const body = node.getBody();
        if (
          Node.isJsxElement(body) ||
          Node.isJsxSelfClosingElement(body) ||
          Node.isJsxFragment(body)
        ) {
          return true;
        }
      }

      node.forEachDescendant(desc => {
        if (Node.isReturnStatement(desc)) {
          const returnExpr = desc.getExpression();
          if (
            returnExpr &&
            (Node.isJsxElement(returnExpr) ||
              Node.isJsxFragment(returnExpr) ||
              Node.isJsxSelfClosingElement(returnExpr))
          ) {
            hasJSX = true;
          }
        }
        if (Node.isJsxElement(desc) || Node.isJsxSelfClosingElement(desc)) {
          hasJSX = true;
        }
      });

      return hasJSX;
    }

    // Check function declarations
    sourceFile.getFunctions().forEach(fn => {
      // Skip if not inside another function component
      let parent = fn.getParent();
      let foundParentComponent = false;

      while (parent) {
        if (
          (Node.isFunctionDeclaration(parent) ||
            Node.isArrowFunction(parent) ||
            Node.isFunctionExpression(parent)) &&
          hasJSXReturn(parent)
        ) {
          foundParentComponent = true;
          break;
        }
        parent = parent.getParent();
      }

      if (foundParentComponent && hasJSXReturn(fn)) {
        const name = fn.getName() || 'anonymous';
        violations.push({
          id: 'component-in-render',
          name: 'Nested Component Definition',
          category: 'performance',
          severity: 'critical',
          location: getLocation(fn),
          description: `Component "${name}" is defined inside another component`,
          impact: 'Component remounts on every parent render, losing all state',
          fix: 'Move the component definition outside, or use useMemo if it depends on parent props',
          autoFixable: true,
          references: [
            'https://react.dev/learn/your-first-component#nesting-and-organizing-components',
          ],
        });
      }
    });

    // Check variable declarations (arrow functions)
    sourceFile.getVariableDeclarations().forEach(varDecl => {
      const init = varDecl.getInitializer();
      if (!init || (!Node.isArrowFunction(init) && !Node.isFunctionExpression(init))) {
        return;
      }

      // Check if this looks like a component (returns JSX)
      if (!hasJSXReturn(init)) return;

      const name = varDecl.getName();

      // Skip if doesn't look like a component name (PascalCase)
      if (!/^[A-Z]/.test(name)) return;

      // Check if inside another component
      let parent = varDecl.getParent();
      let foundParentComponent = false;

      while (parent) {
        if (
          (Node.isFunctionDeclaration(parent) ||
            Node.isArrowFunction(parent) ||
            Node.isFunctionExpression(parent)) &&
          hasJSXReturn(parent)
        ) {
          foundParentComponent = true;
          break;
        }
        parent = parent.getParent();
      }

      if (foundParentComponent) {
        violations.push({
          id: 'component-in-render',
          name: 'Nested Component Definition',
          category: 'performance',
          severity: 'critical',
          location: getLocation(varDecl),
          description: `Component "${name}" is defined inside another component`,
          impact: 'Component remounts on every parent render, losing all state',
          fix: 'Move the component definition outside the parent component',
          autoFixable: true,
          references: [
            'https://react.dev/learn/your-first-component#nesting-and-organizing-components',
          ],
        });
      }
    });

    return violations;
  },
};

// Export all performance rules
export const PERFORMANCE_RULES: AntiPatternRule[] = [
  inlineFunctionPropsRule,
  indexAsKeyRule,
  componentInRenderRule,
];
```

### Task 2.5: Anti-Pattern Analyzer

**Model:** Opus (integration logic)
**File:** `src/analyzers/anti-pattern-analyzer.ts` (NEW FILE)

```typescript
import { SourceFile } from 'ts-morph';
import { BaseAnalyzer, BaseAnalyzerResult } from './base-analyzer-tsmorph';
import { AntiPatternRegistry } from '../rules/anti-pattern-registry';
import type { AntiPattern, AntiPatternCategory } from '../types/anti-patterns';

export interface AntiPatternMetrics extends BaseAnalyzerResult {
  /** Total anti-patterns detected */
  totalAntiPatterns: number;
  /** Anti-patterns grouped by category */
  byCategory: Record<AntiPatternCategory, AntiPattern[]>;
  /** Count of critical issues */
  criticalCount: number;
  /** Count of high severity issues */
  highCount: number;
  /** Count of auto-fixable issues */
  autoFixableCount: number;
  /** Impact score (0-100, higher is worse) */
  impactScore: number;
  /** All detected anti-patterns */
  antiPatterns: AntiPattern[];
}

/**
 * Anti-Pattern Analyzer
 *
 * Analyzes React code for common anti-patterns using a rules-based system.
 * Extends BaseAnalyzer to integrate with the existing analyzer infrastructure.
 */
export class AntiPatternAnalyzer extends BaseAnalyzer<AntiPatternMetrics> {
  private registry: AntiPatternRegistry;

  constructor(sourceFile: SourceFile, registry?: AntiPatternRegistry) {
    super(sourceFile);
    this.registry = registry ?? new AntiPatternRegistry();
  }

  protected analyzeAST(): Omit<AntiPatternMetrics, 'metadata'> {
    const antiPatterns = this.registry.analyze(this.ast);

    // Group by category
    const byCategory = this.groupByCategory(antiPatterns);

    // Calculate counts
    const criticalCount = antiPatterns.filter(ap => ap.severity === 'critical').length;
    const highCount = antiPatterns.filter(ap => ap.severity === 'high').length;
    const autoFixableCount = antiPatterns.filter(ap => ap.autoFixable).length;

    // Calculate impact score
    const impactScore = this.calculateImpactScore(antiPatterns);

    // Calculate overall score (inverse of impact - higher is better)
    const score = Math.max(0, 100 - impactScore);

    return {
      score,
      totalAntiPatterns: antiPatterns.length,
      byCategory,
      criticalCount,
      highCount,
      autoFixableCount,
      impactScore,
      antiPatterns,
    };
  }

  private groupByCategory(antiPatterns: AntiPattern[]): Record<AntiPatternCategory, AntiPattern[]> {
    const groups: Partial<Record<AntiPatternCategory, AntiPattern[]>> = {};

    antiPatterns.forEach(ap => {
      if (!groups[ap.category]) {
        groups[ap.category] = [];
      }
      groups[ap.category]!.push(ap);
    });

    return groups as Record<AntiPatternCategory, AntiPattern[]>;
  }

  private calculateImpactScore(antiPatterns: AntiPattern[]): number {
    let score = 0;

    antiPatterns.forEach(ap => {
      switch (ap.severity) {
        case 'critical':
          score += 20;
          break;
        case 'high':
          score += 12;
          break;
        case 'medium':
          score += 6;
          break;
        case 'low':
          score += 2;
          break;
      }
    });

    return Math.min(100, score);
  }
}

/**
 * Convenience function to analyze anti-patterns in source code
 */
export function analyzeAntiPatterns(source: string): AntiPatternMetrics {
  const { Project } = require('ts-morph');
  const project = new Project({ useInMemoryFileSystem: true });
  const sourceFile = project.createSourceFile('temp.tsx', source);
  const analyzer = new AntiPatternAnalyzer(sourceFile);
  return analyzer.analyze();
}
```

### Task 2.6: State & JSX Rules (Placeholder)

**Files:**

- `src/rules/anti-patterns/state-rules.ts` (NEW FILE)
- `src/rules/anti-patterns/jsx-rules.ts` (NEW FILE)

**Note:** These rule files need to be created following the same ts-morph patterns as hooks and performance rules. Example structure:

```typescript
import { SourceFile, Node } from 'ts-morph';
import type { AntiPatternRule } from '../../types/anti-patterns';

// TODO: Implement state management anti-pattern rules
// - Derived state detection
// - Direct state mutation
// - Prop drilling detection

export const STATE_RULES: AntiPatternRule[] = [
  // Rules to be implemented
];
```

```typescript
import { SourceFile, Node } from 'ts-morph';
import type { AntiPatternRule } from '../../types/anti-patterns';

// TODO: Implement JSX anti-pattern rules
// - JSX spreading detection
// - Missing key props
// - Dangerous HTML usage

export const JSX_RULES: AntiPatternRule[] = [
  // Rules to be implemented
];
```

## Implementation Checklist

Use this checklist to track implementation progress:

### Step 1: Create Type Definitions (Priority: HIGH)

- [ ] Create `src/types/anti-patterns.ts`
- [ ] Define `AntiPatternCategory` type
- [ ] Define `AntiPattern` interface
- [ ] Define `AntiPatternRule` interface
- [ ] Define `AntiPatternRuleConfig` and `AntiPatternRulesConfig`
- [ ] Export types from `src/types/index.ts`

### Step 2: Create Registry (Priority: HIGH)

- [ ] Create `src/rules/anti-pattern-registry.ts`
- [ ] Implement `AntiPatternRegistry` class
- [ ] Implement `register()` method
- [ ] Implement `analyze()` method with severity overrides
- [ ] Implement `getEnabledRules()` filtering
- [ ] Export default registry instance
- [ ] Create tests in `src/rules/anti-pattern-registry.test.ts`

### Step 3: Implement Hooks Rules (Priority: HIGH)

- [ ] Create `src/rules/anti-patterns/` directory
- [ ] Create `hooks-rules.ts` file
- [ ] Implement `conditionalHookRule` (critical)
- [ ] Implement `missingDepsRule` (medium)
- [ ] Implement `missingCleanupRule` (medium)
- [ ] Implement `asyncEffectRule` (critical)
- [ ] Export `HOOKS_RULES` array
- [ ] Create tests in `src/rules/anti-patterns/hooks-rules.test.ts`

### Step 4: Implement Performance Rules (Priority: MEDIUM)

- [ ] Create `performance-rules.ts` file
- [ ] Implement `inlineFunctionPropsRule` (low)
- [ ] Implement `indexAsKeyRule` (medium)
- [ ] Implement `componentInRenderRule` (critical)
- [ ] Export `PERFORMANCE_RULES` array
- [ ] Create tests in `src/rules/anti-patterns/performance-rules.test.ts`

### Step 5: Implement State & JSX Rules (Priority: MEDIUM)

- [ ] Create `state-rules.ts` file with placeholder
- [ ] Create `jsx-rules.ts` file with placeholder
- [ ] Export empty arrays for now
- [ ] Plan detailed implementation for future phase

### Step 6: Create Anti-Pattern Analyzer (Priority: HIGH)

- [ ] Create `src/analyzers/anti-pattern-analyzer.ts`
- [ ] Implement `AntiPatternMetrics` interface
- [ ] Implement `AntiPatternAnalyzer` class extending `BaseAnalyzer`
- [ ] Implement `groupByCategory()` method
- [ ] Implement `calculateImpactScore()` method
- [ ] Export convenience function `analyzeAntiPatterns()`
- [ ] Create tests in `src/analyzers/anti-pattern-analyzer.test.ts`

### Step 7: Integration & Testing (Priority: MEDIUM)

- [ ] Update `src/types/index.ts` to export anti-pattern types
- [ ] Update `schema.json` with `AntiPatternMetrics` definition
- [ ] Add CLI flag `--anti-patterns` support
- [ ] Create integration tests with sample components
- [ ] Test each rule category independently
- [ ] Document in README.md

## Dependencies

- Phase 00 must be complete (BaseAnalyzer, types structure)
- `ts-morph` for AST analysis
- Existing `BaseAnalyzer` class from `base-analyzer-tsmorph.ts`
- Existing `Severity` and `CodeLocation` types from `src/types/index.ts`

## Acceptance Criteria

### Rule Coverage

- [ ] Conditional hook call detection (100% accuracy)
- [ ] Missing effect dependencies detection (>70% accuracy with heuristics)
- [ ] Effect cleanup detection for timers, listeners
- [ ] Async effect callback detection
- [ ] Inline function props detection
- [ ] Index as key detection
- [ ] Component in render detection

### Performance

- [ ] Analysis < 300ms for typical component
- [ ] No false positives on common patterns
- [ ] Low false negative rate for critical issues

### Integration

- [ ] Works with existing BaseAnalyzer pattern
- [ ] Schema extended with AntiPatternMetrics
- [ ] CLI `--anti-patterns` flag for rule selection
- [ ] Uses ts-morph exclusively (no Babel)

## Testing Instructions

### Manual Testing

1. **Test Conditional Hook Detection**

   ```bash
   cat > /tmp/ConditionalHook.tsx << 'EOF'
   function Component({ enabled }) {
     if (enabled) {
       const [count, setCount] = useState(0);
     }
     return <div>Test</div>;
   }
   EOF

   npm run analyze -- /tmp/ConditionalHook.tsx --verbose
   # Expected: Critical - conditional-hook-call detected
   ```

2. **Test Missing Deps Detection**

   ```bash
   cat > /tmp/MissingDeps.tsx << 'EOF'
   function Component({ userId }) {
     const [user, setUser] = useState(null);

     useEffect(() => {
       fetch(`/api/users/${userId}`).then(r => r.json()).then(setUser);
     }, []);

     return <div>{user?.name}</div>;
   }
   EOF

   npm run analyze -- /tmp/MissingDeps.tsx --verbose
   # Expected: Warning - missing userId in deps
   ```

3. **Test Component in Render**

   ```bash
   cat > /tmp/NestedComponent.tsx << 'EOF'
   function ParentComponent() {
     const ChildComponent = () => <div>Child</div>;
     return <ChildComponent />;
   }
   EOF

   npm run analyze -- /tmp/NestedComponent.tsx --verbose
   # Expected: Critical - component-in-render detected
   ```

## Schema Updates

Add to `schema.json`:

```json
{
  "definitions": {
    "AntiPatternMetrics": {
      "type": "object",
      "properties": {
        "totalAntiPatterns": { "type": "number" },
        "criticalCount": { "type": "number" },
        "autoFixableCount": { "type": "number" },
        "impactScore": { "type": "number" },
        "byCategory": {
          "type": "object",
          "additionalProperties": {
            "type": "array",
            "items": { "$ref": "#/definitions/AntiPattern" }
          }
        }
      }
    },
    "AntiPattern": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "category": { "type": "string" },
        "severity": { "enum": ["critical", "warning", "info"] },
        "location": { "$ref": "#/definitions/CodeLocation" },
        "description": { "type": "string" },
        "impact": { "type": "string" },
        "fix": { "type": "string" },
        "autoFixable": { "type": "boolean" }
      },
      "required": ["id", "name", "category", "severity", "description"]
    }
  }
}
```

## Estimated Effort

| Task                            | Estimated Time  | Notes                                            |
| ------------------------------- | --------------- | ------------------------------------------------ |
| 2.1 Anti-Pattern Types          | 1-2 hours       | Define interfaces, integrate with existing types |
| 2.2 Anti-Pattern Registry       | 2-3 hours       | Separate from general rule system                |
| 2.3 Hooks Rules (4 rules)       | 5-6 hours       | Complex pattern matching with ts-morph           |
| 2.4 Performance Rules (3 rules) | 3-4 hours       | Well-defined patterns                            |
| 2.5 State/JSX Placeholders      | 1 hour          | Basic structure for future work                  |
| 2.6 Analyzer Integration        | 2-3 hours       | Extend BaseAnalyzer, implement scoring           |
| Unit Tests (Hooks)              | 3 hours         | Test all 4 hook rules                            |
| Unit Tests (Performance)        | 2 hours         | Test all 3 performance rules                     |
| Unit Tests (Registry)           | 1 hour          | Test registration and execution                  |
| Integration Tests               | 2 hours         | End-to-end anti-pattern detection                |
| Schema Updates                  | 30 minutes      | Add AntiPatternMetrics                           |
| CLI Integration                 | 1 hour          | Add --anti-patterns flag                         |
| Documentation                   | 1 hour          | Update README                                    |
| **Total**                       | **24-28 hours** | Comprehensive ts-morph implementation            |

---

**Document Version:** 2.0  
**Updated:** January 10, 2026  
**Status:** Ready for Implementation  
**Foundation:** Uses ts-morph exclusively, builds on Phase 00
