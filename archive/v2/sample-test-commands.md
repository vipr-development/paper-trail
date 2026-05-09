# Sample Test Commands

## React Analyzer

Comprehensive catalog of fixture components and recommended commands for testing the React analyzer.

### Quick Reference

| Fixture                        | Primary Analyses             | Complexity  | Key Issues                    |
| ------------------------------ | ---------------------------- | ----------- | ----------------------------- |
| SimpleComponent.tsx            | Structural, Hook             | Low         | Basic hooks usage             |
| SearchInput.tsx                | Hook, Temporal               | Medium      | Debouncing, cleanup           |
| DataTable.tsx                  | Structural, Types            | Medium-High | Complex props, generics       |
| InaccessibleComponent.tsx      | Accessibility                | High        | ARIA violations, keyboard nav |
| InsecureComponent.tsx          | Security                     | High        | XSS, injection risks          |
| ProblematicComponent.tsx       | Anti-Pattern, Technical Debt | High        | Multiple anti-patterns        |
| HookPatternComponent.tsx       | Hook, Temporal               | High        | 33 hooks, missing cleanup     |
| DataFlowComponent.tsx          | Dataflow, Coupling           | Critical    | 10 state vars, prop drilling  |
| IdentityPatternsComponent.tsx  | Identity, Performance        | Medium-High | Unstable references           |
| CouplingComponent.tsx          | Coupling                     | High        | 25+ props, 5 contexts         |
| PerformanceIssuesComponent.tsx | Performance                  | Critical    | No memoization, O(n³)         |
| ReliabilityComponent.tsx       | Reliability                  | Medium-High | 25+ crash risks               |
| TypeComplexityComponent.tsx    | Types                        | High        | 30+ complex types             |
| AntiPatternShowcase.tsx        | Anti-Pattern                 | High        | 36+ violations                |

### By Analysis Type

#### Structural Analysis

Tests component structure, nesting, and complexity.

```bash
# Simple component - Grade A expected
pnpm analyze analyzers/react/src/fixtures/SimpleComponent.tsx

# Medium complexity data table
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx

# High complexity component
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# Extremely complex nested structure
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx
```

#### Dataflow Analysis

Tests state management, data transformations, and data flow patterns.

```bash
# Best showcase: 10 state variables, prop drilling, transform chains
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Moderate: data table with sorting/filtering
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx

# Complex: multiple state updates and data transformations
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx
```

**DataFlowComponent.tsx** demonstrates:

- 10 separate state variables
- 5-level deep prop drilling
- Multiple chained array operations (filter → map → reduce)
- Complex derived computations without memoization
- Transform chains in JSX rendering

#### Anti-Pattern Analysis

Tests for React anti-patterns and violations.

```bash
# Comprehensive showcase: 36+ anti-pattern violations
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx

# Multiple anti-patterns with technical debt
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# Inline functions and object literals
pnpm analyze analyzers/react/src/fixtures/IdentityPatternsComponent.tsx

# Performance anti-patterns
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx
```

**AntiPatternShowcase.tsx** demonstrates:

- Conditional hook calls (Rules of Hooks violations)
- Missing effect dependencies
- Direct state mutation
- Props in initial state
- Excessive useState calls (8+)
- Array index as key
- Inline function and object props

#### Hook Analysis

Tests hook usage patterns, complexity, and violations.

```bash
# Best showcase: 33 total hooks with various patterns
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# Moderate: proper hook usage with cleanup
pnpm analyze analyzers/react/src/fixtures/SearchInput.tsx

# Complex: multiple hooks with state management
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Anti-patterns: conditional hooks, violations
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx
```

**HookPatternComponent.tsx** demonstrates:

- 8 useState calls
- 4 useRef calls
- 2 useContext calls
- 9 useEffect calls (including useLayoutEffect)
- 5 useCallback calls
- 6 useMemo calls
- Effects without cleanup (memory leaks)
- Effects with missing dependencies
- Effects without dependency array

#### Temporal Analysis

Tests effect lifecycle, cleanup, and timing issues.

```bash
# Best showcase: multiple effect patterns and temporal issues
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# Effects without cleanup causing memory leaks
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx

# Proper effect cleanup patterns
pnpm analyze analyzers/react/src/fixtures/SearchInput.tsx
```

**HookPatternComponent.tsx** demonstrates:

- 4 effects without cleanup (setTimeout, setInterval, event listeners)
- 1 effect without dependency array (runs every render)
- Effects with async operations without cancellation
- Effects with proper cleanup patterns
- Complex dependency tracking

#### Coupling Analysis

Tests component coupling through props, contexts, and callbacks.

```bash
# Best showcase: 52 total coupling points
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx

# Deep prop drilling through 5 levels
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Multiple render props and high prop count
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx
```

**CouplingComponent.tsx** demonstrates:

- 25+ props
- 13+ callback props
- 5 context consumers (Theme, Auth, Settings, Notification, Locale)
- Ref forwarding with useImperativeHandle (6 methods)
- 3 render props (renderHeader, renderFooter, renderItem)
- Children composition patterns

#### Identity Analysis

Tests reference stability and memoization patterns.

```bash
# Best showcase: 15+ inline functions, unstable references
pnpm analyze analyzers/react/src/fixtures/IdentityPatternsComponent.tsx

# Missing memoization for event handlers
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# Proper useCallback/useMemo usage (good patterns)
pnpm analyze analyzers/react/src/fixtures/SearchInput.tsx
```

**IdentityPatternsComponent.tsx** demonstrates:

- Inline arrow functions in JSX props
- Inline object literals in style props
- Inline array literals passed to components
- Unstable references to memoized components
- Missing useCallback for event handlers
- Missing useMemo for computed objects
- Side-by-side comparison of good vs bad patterns

#### Accessibility Analysis

Tests ARIA, semantic HTML, and accessibility patterns.

```bash
# Best showcase: comprehensive accessibility violations
pnpm analyze analyzers/react/src/fixtures/InaccessibleComponent.tsx

# Interactive elements without proper accessibility
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx
```

**InaccessibleComponent.tsx** demonstrates:

- Missing ARIA labels
- Non-semantic HTML elements
- Missing keyboard navigation
- Insufficient color contrast
- Missing focus management
- Interactive elements without proper roles

#### Security Analysis

Tests XSS vulnerabilities, injection risks, and security patterns.

```bash
# Best showcase: comprehensive security vulnerabilities
pnpm analyze analyzers/react/src/fixtures/InsecureComponent.tsx

# Potential security issues with user input
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx
```

**InsecureComponent.tsx** demonstrates:

- XSS vulnerabilities with dangerouslySetInnerHTML
- Unsafe user input handling
- Missing input sanitization
- Potential injection attacks
- Insecure external link handling

#### Migration Analysis

Tests deprecated APIs and migration readiness.

```bash
# Class component with lifecycle methods
pnpm analyze analyzers/react/src/fixtures/migration/ClassComponentLegacy.tsx

# Deprecated string refs
pnpm analyze analyzers/react/src/fixtures/migration/StringRefsExample.tsx

# PropTypes to TypeScript migration
pnpm analyze analyzers/react/src/fixtures/migration/PropTypesExample.tsx

# Legacy context API
pnpm analyze analyzers/react/src/fixtures/migration/LegacyContextExample.tsx

# Modern React 19 patterns
pnpm analyze analyzers/react/src/fixtures/migration/React19Features.tsx

# Migration-ready component
pnpm analyze analyzers/react/src/fixtures/migration/ModernComponent.tsx

# Analyze all migration fixtures
pnpm analyze analyzers/react/src/fixtures/migration/
```

#### Performance Analysis

Tests memoization, expensive operations, and performance patterns.

```bash
# Best showcase: 7 expensive operations without memoization
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# Unstable references causing re-renders
pnpm analyze analyzers/react/src/fixtures/IdentityPatternsComponent.tsx

# Transform chains without memoization
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Heavy computations in render
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx
```

**PerformanceIssuesComponent.tsx** demonstrates:

- 7 expensive computations per render (no useMemo)
- Multiple chained array operations in render
- Complex filtering, mapping, sorting without memoization
- Expensive derived statistics on every render
- Large list rendering without virtualization
- O(n³) complexity with nested loops
- Inline transformations in JSX

#### Reliability Analysis

Tests error handling, null safety, and crash prevention.

```bash
# Best showcase: 25+ crash risk points
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx

# Multiple anti-patterns with reliability risks
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx
```

**ReliabilityComponent.tsx** demonstrates:

- Missing null checks (crash risks)
- Unsafe property access without optional chaining
- Missing error handling in promises (no .catch())
- Missing try-catch blocks
- Effects without cleanup (memory leaks)
- Unhandled async operations
- Division by zero risks
- Side-by-side comparison of good vs bad patterns

#### Technical Debt Analysis

Tests code quality, maintainability, and technical debt indicators.

```bash
# High technical debt with multiple issues
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# Extreme complexity contributing to technical debt
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# High coupling as technical debt
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx

# Complex type system debt
pnpm analyze analyzers/react/src/fixtures/TypeComplexityComponent.tsx
```

#### Type Analysis

Tests TypeScript type complexity and type health.

```bash
# Best showcase: 30+ complex type definitions
pnpm analyze analyzers/react/src/fixtures/TypeComplexityComponent.tsx

# Complex generics and interfaces
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx

# Any types and poor type usage
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# Deep generic nesting
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx
```

**TypeComplexityComponent.tsx** demonstrates:

- Deep generic nesting (4+ levels)
- Conditional types with multiple branches
- Mapped types (Readonly, Partial, DeepPartial, Required)
- Recursive types (Json, DeepReadonly)
- Complex union types (15+ members)
- Large intersection types
- Template literal types
- Infer keyword usage (8+ examples)
- Complex utility types

### Complexity Progression

Commands ordered by increasing complexity for demonstrations:

```bash
# 1. Simple - Grade A (minimal complexity)
pnpm analyze analyzers/react/src/fixtures/SimpleComponent.tsx

# 2. Basic - Grade B (basic hooks with cleanup)
pnpm analyze analyzers/react/src/fixtures/SearchInput.tsx

# 3. Moderate - Grade C (data table with features)
pnpm analyze analyzers/react/src/fixtures/DataTable.tsx

# 4. High - Grade D (multiple hook patterns)
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# 5. Very High - Grade F (extreme coupling)
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx

# 6. Critical - Grade F (performance issues)
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# 7. Critical - Grade F (data flow complexity)
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx
```

### Issue-Focused Testing

#### Memory Leak Detection

```bash
# Effects without cleanup
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# Unhandled subscriptions
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx
```

#### Performance Issues

```bash
# Missing memoization
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# Unstable references
pnpm analyze analyzers/react/src/fixtures/IdentityPatternsComponent.tsx

# Transform chains
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx
```

#### Rules of Hooks Violations

```bash
# Conditional hook calls and other violations
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx
```

#### Crash Risks

```bash
# Unsafe property access, missing null checks
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx
```

#### Architecture Issues

```bash
# High coupling
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx

# Deep prop drilling
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Complex type system
pnpm analyze analyzers/react/src/fixtures/TypeComplexityComponent.tsx
```

### Batch Analysis

#### All Fixtures

```bash
# Analyze all fixtures with basic metrics
pnpm analyze analyzers/react/src/fixtures/

# Full analysis with all metrics
pnpm analyze analyzers/react/src/fixtures/ --json

# Specific analysis types on all fixtures
pnpm analyze analyzers/react/src/fixtures/ --anti-patterns
```

#### Migration Analysis

```bash
# All migration fixtures
pnpm analyze analyzers/react/src/fixtures/migration/

# Migration with JSON output
pnpm analyze analyzers/react/src/fixtures/migration/ --json
```

#### High Complexity Fixtures Only

```bash
# Critical complexity fixtures
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx \
  analyzers/react/src/fixtures/DataFlowComponent.tsx \
  analyzers/react/src/fixtures/CouplingComponent.tsx
```

### JSON Output for CI/CD

```bash
# Single file with JSON output
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx --json

# Directory analysis with JSON
pnpm analyze analyzers/react/src/fixtures/ --json

# Specific issues with JSON output
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx --json

# Migration report in JSON
pnpm analyze analyzers/react/src/fixtures/migration/ --json
```

### Demo Sequences

#### Quick Demo (5 minutes)

```bash
# 1. Simple component - Grade A
pnpm analyze analyzers/react/src/fixtures/SimpleComponent.tsx

# 2. Problematic component - Grade F
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# 3. Anti-patterns showcase
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx
```

#### Comprehensive Demo (15 minutes)

```bash
# 1. Start simple
pnpm analyze analyzers/react/src/fixtures/SimpleComponent.tsx

# 2. Show proper patterns
pnpm analyze analyzers/react/src/fixtures/SearchInput.tsx

# 3. Hook complexity
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# 4. Performance issues
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# 5. Anti-patterns
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx

# 6. Reliability issues
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx

# 7. Migration needs
pnpm analyze analyzers/react/src/fixtures/migration/ClassComponentLegacy.tsx

# 8. Full suite
pnpm analyze analyzers/react/src/fixtures/ --json
```

#### Analysis-Focused Demo

Demonstrating each analysis type:

```bash
# Structural
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Dataflow
pnpm analyze analyzers/react/src/fixtures/DataFlowComponent.tsx

# Anti-Pattern
pnpm analyze analyzers/react/src/fixtures/AntiPatternShowcase.tsx

# Hook
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# Temporal
pnpm analyze analyzers/react/src/fixtures/HookPatternComponent.tsx

# Coupling
pnpm analyze analyzers/react/src/fixtures/CouplingComponent.tsx

# Identity
pnpm analyze analyzers/react/src/fixtures/IdentityPatternsComponent.tsx

# Accessibility
pnpm analyze analyzers/react/src/fixtures/InaccessibleComponent.tsx

# Security
pnpm analyze analyzers/react/src/fixtures/InsecureComponent.tsx

# Migration
pnpm analyze analyzers/react/src/fixtures/migration/ClassComponentLegacy.tsx

# Performance
pnpm analyze analyzers/react/src/fixtures/PerformanceIssuesComponent.tsx

# Reliability
pnpm analyze analyzers/react/src/fixtures/ReliabilityComponent.tsx

# Technical Debt
pnpm analyze analyzers/react/src/fixtures/ProblematicComponent.tsx

# Types
pnpm analyze analyzers/react/src/fixtures/TypeComplexityComponent.tsx
```

### Fixture Coverage Summary

| Analysis       | Primary Fixture            | Secondary Fixtures              | Lines of Code | Issue Count     |
| -------------- | -------------------------- | ------------------------------- | ------------- | --------------- |
| Structural     | DataFlowComponent          | ProblematicComponent, DataTable | 436           | Multiple        |
| Dataflow       | DataFlowComponent          | HookPatternComponent            | 436           | 10 state vars   |
| Anti-Pattern   | AntiPatternShowcase        | ProblematicComponent            | 274           | 36+ violations  |
| Hook           | HookPatternComponent       | DataFlowComponent               | 264           | 33 hooks        |
| Temporal       | HookPatternComponent       | ReliabilityComponent            | 264           | 9 issues        |
| Coupling       | CouplingComponent          | DataFlowComponent               | 336           | 52 points       |
| Identity       | IdentityPatternsComponent  | PerformanceIssuesComponent      | 311           | 30+ unstable    |
| Accessibility  | InaccessibleComponent      | -                               | -             | Multiple        |
| Security       | InsecureComponent          | -                               | -             | Multiple        |
| Migration      | migration/                 | 7 files                         | -             | Multiple        |
| Performance    | PerformanceIssuesComponent | IdentityPatternsComponent       | 318           | 7 major issues  |
| Reliability    | ReliabilityComponent       | -                               | 349           | 25+ crash risks |
| Technical Debt | ProblematicComponent       | Multiple complex fixtures       | -             | Multiple        |
| Types          | TypeComplexityComponent    | DataTable, CouplingComponent    | 284           | 30+ types       |

### Notes

- All fixtures use TypeScript for type analysis
- Fixtures include both anti-patterns and good patterns where appropriate
- Each fixture is designed to trigger multiple related issues for its target analysis
- Complexity scores range from Grade A (5 points) to Grade F (50+ points)
- All fixtures are realistic React components, not contrived examples
