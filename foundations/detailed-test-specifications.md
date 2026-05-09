# Detailed Test Specifications

**Comprehensive test case specifications for P0 files**

## temporal-analysis.test.ts

**File:** `/analyzers/react/src/analyses/temporal-analysis.ts`
**Test Count:** 50-55 tests
**Priority:** P0 (CRITICAL)

### Test Structure

```typescript
describe('TemporalAnalysis', () => {
  describe('metadata', () => {
    // 4 tests
  });

  describe('execute', () => {
    describe('effect detection', () => {
      // 12 tests
    });

    describe('cleanup detection', () => {
      // 8 tests
    });

    describe('risk assessment', () => {
      // 15 tests
    });

    describe('pattern detection', () => {
      // 10 tests
    });

    describe('insight generation', () => {
      // 8 tests
    });
  });
});
```

### Specific Test Cases

#### Metadata (4 tests)

```typescript
✓ should have correct id 'react-temporal'
✓ should have correct category 'technical-debt'
✓ should be enabled by default
✓ should have execution cost of 3
```

#### Effect Detection (12 tests)

```typescript
✓ should detect useEffect hooks
✓ should detect useLayoutEffect hooks
✓ should detect useInsertionEffect hooks
✓ should count total effects correctly
✓ should identify mount-only effects (empty deps array)
✓ should identify every-render effects (no deps array)
✓ should count effects with 1-5 dependencies
✓ should count effects with 10+ dependencies
✓ should handle effects without callback argument
✓ should handle effects with inline arrow function
✓ should handle effects with function expression
✓ should handle effects with function reference
```

#### Cleanup Detection (8 tests)

```typescript
✓ should detect cleanup function (return function in block)
✓ should detect cleanup function (return arrow function)
✓ should not detect non-function return values
✓ should handle multiple return statements (only last matters)
✓ should count cleanup functions correctly
✓ should detect conditional cleanup (within if statement)
✓ should handle cleanup that returns another function
✓ should handle missing cleanup when expected
```

#### Risk Assessment (15 tests)

```typescript
✓ should assign low risk to simple mount-only effect
✓ should assign medium risk to effect with dependencies
✓ should assign high risk to every-render effect
✓ should assign high risk to setInterval without cleanup
✓ should assign high risk to addEventListener without cleanup
✓ should assign medium risk to setTimeout without cleanup
✓ should assign medium risk to async operations
✓ should assign medium risk to state update with dependencies
✓ should mitigate risk with proper cleanup (high -> medium)
✓ should accumulate multiple risk factors
✓ should handle effects with 10+ dependencies (warning threshold)
✓ should correctly identify risky effects count
✓ should generate risk reason string
✓ should handle conflicting risk factors
✓ should test risk level boundary cases (medium/high edge)
```

#### Pattern Detection (10 tests)

```typescript
✓ should detect setTimeout in effect callback
✓ should detect setInterval in effect callback
✓ should detect addEventListener in effect callback
✓ should detect removeEventListener in cleanup
✓ should detect async/await in effect callback
✓ should detect promise .then() in effect callback
✓ should detect state update calls (setState pattern)
✓ should detect nested async operations
✓ should detect fetch calls
✓ should not false-positive on non-effect functions
```

#### Insight Generation (8 tests)

```typescript
✓ should generate insight for setTimeout without cleanup
✓ should generate critical insight for setInterval without cleanup
✓ should generate critical insight for addEventListener without cleanup
✓ should generate warning for async operations
✓ should generate critical insight for missing deps array
✓ should generate warning for 10+ dependencies
✓ should generate warning for high effect count (threshold)
✓ should not generate insights for clean effects
```

#### Edge Cases (5 tests)

```typescript
✓ should handle effects with destructured arguments
✓ should handle effects in conditional rendering
✓ should handle nested effects (effect calling function with effect)
✓ should handle effects with complex dependency arrays
✓ should handle component with 20+ effects
```

#### Score Calculation (3 tests)

```typescript
✓ should calculate score for simple component
✓ should calculate higher score for complex temporal patterns
✓ should apply correct weights (verify against constants)
```

---

## coupling-analysis.test.ts

**File:** `/analyzers/react/src/analyses/coupling-analysis.ts`
**Test Count:** 30-35 tests
**Priority:** P0

### Test Structure

```typescript
describe('CouplingAnalysis', () => {
  describe('metadata', () => {
    // 4 tests
  });

  describe('execute', () => {
    describe('props counting', () => {
      // 8 tests
    });

    describe('context consumers', () => {
      // 6 tests
    });

    describe('children patterns', () => {
      // 8 tests
    });

    describe('scoring', () => {
      // 6 tests
    });
  });
});
```

### Specific Test Cases

#### Metadata (4 tests)

```typescript
✓ should have correct id 'react-coupling'
✓ should have correct category 'technical-debt'
✓ should be enabled by default
✓ should have execution cost of 2
```

#### Props Counting (8 tests)

```typescript
✓ should count props from object destructuring
✓ should count props from function parameter
✓ should identify callback props (starts with 'on')
✓ should identify callback props (ends with 'Handler')
✓ should identify callback props (ends with 'Callback')
✓ should handle props with rest parameters
✓ should handle components with no props
✓ should handle components with 20+ props
```

#### Context Consumers (6 tests)

```typescript
✓ should detect useContext calls
✓ should count multiple context consumers
✓ should generate warning for 3+ contexts (threshold)
✓ should handle Context.Consumer pattern (legacy)
✓ should handle nested context consumers
✓ should not count non-context hooks
```

#### Children Patterns (8 tests)

```typescript
✓ should detect simple children usage (childrenUsage: 'simple')
✓ should detect render props pattern (childrenUsage: 'render-props')
✓ should detect render prop with 'render' attribute
✓ should detect function-as-child pattern
✓ should default to 'none' when no children
✓ should distinguish between simple and render props
✓ should detect forwardRef usage
✓ should handle compound patterns (children + forwardRef)
```

#### Scoring (6 tests)

```typescript
✓ should calculate base score with props count
✓ should add context multiplier to score
✓ should add callback multiplier to score
✓ should add refForwarding base score
✓ should add children score based on pattern
✓ should generate insight for high coupling (threshold)
```

#### Edge Cases (3 tests)

```typescript
✓ should handle component with all coupling patterns
✓ should handle minimal component (no coupling)
✓ should handle renamed prop destructuring
```

---

## identity-analysis.test.ts

**File:** `/analyzers/react/src/analyses/identity-analysis.ts`
**Test Count:** 35-40 tests
**Priority:** P0

### Test Structure

```typescript
describe('IdentityAnalysis', () => {
  describe('metadata', () => {
    // 4 tests
  });

  describe('execute', () => {
    describe('memoization hooks', () => {
      // 8 tests
    });

    describe('unstable references', () => {
      // 12 tests
    });

    describe('React.memo detection', () => {
      // 6 tests
    });

    describe('insight generation', () => {
      // 8 tests
    });
  });
});
```

### Specific Test Cases

#### Metadata (4 tests)

```typescript
✓ should have correct id 'react-identity'
✓ should have correct category 'performance'
✓ should be enabled by default
✓ should have execution cost of 2
```

#### Memoization Hooks (8 tests)

```typescript
✓ should count useCallback hooks
✓ should count useMemo hooks
✓ should track dependencies for useCallback
✓ should track dependencies for useMemo
✓ should sum total dependencies across all hooks
✓ should handle empty dependency arrays
✓ should handle hooks without dependency arrays (not counted)
✓ should handle nested useCallback/useMemo
```

#### Unstable References (12 tests)

```typescript
✓ should detect inline arrow function in JSX
✓ should detect inline function expression in JSX
✓ should detect inline object literal in JSX
✓ should detect inline array literal in JSX
✓ should count event handlers (props starting with 'on')
✓ should detect style prop inline objects
✓ should not count stable references (variables)
✓ should detect unstable refs in nested JSX
✓ should handle multiple unstable refs in one component
✓ should distinguish between inline and stable references
✓ should handle conditional JSX with inline refs
✓ should detect unstable refs in spread attributes
```

#### React.memo Detection (6 tests)

```typescript
✓ should detect memo() wrapper (import from 'react')
✓ should detect React.memo() wrapper
✓ should set hasMemoComponents flag correctly
✓ should not false-positive on other 'memo' identifiers
✓ should detect multiple memoized components in file
✓ should handle memo with comparison function
```

#### Insight Generation (8 tests)

```typescript
✓ should generate insight for inline function in event handler
✓ should prioritize insight when memo components present
✓ should generate insight for inline style object
✓ should generate warning for inline object in non-style prop
✓ should generate insight for inline array
✓ should suggest useCallback for event handlers (memo context)
✓ should suggest useMemo for objects (memo context)
✓ should not generate insights below threshold (no memo)
```

#### Scoring (4 tests)

```typescript
✓ should calculate score with useCallback weight
✓ should calculate score with useMemo weight
✓ should calculate score with dependency weight
✓ should calculate score with unstable reference weight
```

#### Edge Cases (3 tests)

```typescript
✓ should handle component with 10+ unstable references
✓ should handle component with no identity issues
✓ should handle memo component with perfect memoization
```

---

## analysis-cache.test.ts

**File:** `/analyzers/core/src/engine/analysis-cache.ts`
**Test Count:** 25-30 tests
**Priority:** P0

### Test Structure

```typescript
describe('AnalysisCacheManagerImpl', () => {
  describe('get and set', () => {
    // 8 tests
  });

  describe('invalidate', () => {
    // 8 tests
  });

  describe('getStats', () => {
    // 6 tests
  });

  describe('edge cases', () => {
    // 6 tests
  });
});
```

### Specific Test Cases

#### Get and Set (8 tests)

```typescript
✓ should set and get cache entry
✓ should return null for missing entry
✓ should increment hit count on cache hit
✓ should increment miss count on cache miss
✓ should handle multiple entries
✓ should update existing entry
✓ should handle concurrent get operations
✓ should handle concurrent set operations
```

#### Invalidate (8 tests)

```typescript
✓ should invalidate by cache key
✓ should invalidate all entries for a file path
✓ should invalidate all entries for a plugin ID
✓ should invalidate all entries for an analysis ID
✓ should handle invalidation of non-existent keys
✓ should clear all entries
✓ should reset counters on clear
✓ should handle partial path matches correctly
```

#### GetStats (6 tests)

```typescript
✓ should return correct total entries
✓ should calculate hit rate correctly
✓ should estimate memory usage
✓ should calculate oldest entry age
✓ should handle empty cache
✓ should handle cache with 10,000 entries
```

#### Edge Cases (6 tests)

```typescript
✓ should handle file paths with special characters
✓ should handle file paths with colons (Windows)
✓ should handle plugin IDs containing colons
✓ should handle unicode in cache keys
✓ should handle very long file paths (1000+ chars)
✓ should prevent cache key collisions
```

#### Performance (4 tests)

```typescript
✓ should perform 10,000 get operations in < 100ms
✓ should perform 10,000 set operations in < 100ms
✓ should invalidate by file with 10,000 entries in < 50ms
✓ should calculate stats with 10,000 entries in < 200ms
```

---

## react-helpers.test.ts

**File:** `/analyzers/react/src/utils/react-helpers.ts`
**Test Count:** 60-70 tests
**Priority:** P1

### Test Structure (High-Level)

```typescript
describe('React Helpers', () => {
  describe('Component Detection', () => {
    // 12 tests - findReactComponents, isReactComponentFunction
  });

  describe('JSX Helpers', () => {
    // 10 tests - isInsideJSX, isJsxConditionalRender, getJsxDepth, countJsxElements
  });

  describe('Hook Helpers', () => {
    // 15 tests - isHookCall, getHookName, extractHookDependencies, getDependencyCount
  });

  describe('Effect Detectors', () => {
    // 10 tests - hasCleanupFunction, detect* functions
  });

  describe('Props Helpers', () => {
    // 10 tests - extractPropsInfo, analyzePropsType
  });

  describe('Type Helpers', () => {
    // 8 tests - isAnyType, countAnyUsage, countUntypedUseState, etc.
  });
});
```

### Key Test Cases (Sampling)

#### Component Detection (12 tests)

```typescript
✓ should detect function declaration component
✓ should detect arrow function component
✓ should detect const + arrow function component
✓ should detect function expression component
✓ should require PascalCase name
✓ should require JSX in return
✓ should not detect non-component functions
✓ should detect multiple components in file
✓ should handle default exported component
✓ should handle component without explicit return type
✓ should detect component with generic type parameters
✓ should handle higher-order component patterns
```

#### Hook Helpers (15 tests)

```typescript
✓ should identify all 17 built-in hooks
✓ should identify custom hooks (starts with 'use')
✓ should extract dependencies from useEffect
✓ should extract dependencies from useCallback
✓ should extract dependencies from useMemo
✓ should handle empty dependency array
✓ should handle missing dependency array (return -1)
✓ should extract property access dependencies (object.prop)
✓ should extract array access dependencies
✓ should handle inline function in deps (not extracted)
✓ should count dependencies correctly
✓ should handle nested hooks
✓ should handle conditional hooks (not valid but should parse)
✓ should handle hooks in custom hook functions
✓ should validate hook naming convention
```

---

## analysis-engine (additions).test.ts

**File:** `/analyzers/core/src/engine/analysis-engine.ts`
**Test Count:** 30+ additional tests
**Priority:** P0

### Test Areas Needing Coverage

#### analyzeFile Edge Cases (10 tests)

```typescript
✓ should handle empty source file
✓ should handle source file with syntax errors
✓ should handle file with 10,000+ AST nodes
✓ should handle case where no plugins apply
✓ should handle case where cache is disabled
✓ should handle concurrent analyzeFile calls on same file
✓ should handle cache hit correctly
✓ should invalidate cache on file change
✓ should handle file path with special characters
✓ should handle non-existent file path
```

#### Plugin Execution Error Handling (8 tests)

```typescript
✓ should isolate plugin errors (one plugin fails, others succeed)
✓ should handle plugin throwing during canHandle()
✓ should handle plugin throwing during analyze()
✓ should handle plugin returning null
✓ should handle plugin returning malformed PluginResult
✓ should handle plugin timeout
✓ should collect errors in result
✓ should not crash engine on plugin error
```

#### Concurrency and Performance (6 tests)

```typescript
✓ should execute analyses with concurrency limit = 1
✓ should execute analyses with concurrency limit = 5
✓ should execute analyses with concurrency limit = Infinity
✓ should respect analysis timeout setting
✓ should schedule analyses by cost when enabled
✓ should handle 100 sequential analyzeFile calls without memory leak
```

#### Cache Coordination (6 tests)

```typescript
✓ should cache analysis results per plugin
✓ should invalidate cache by file path
✓ should invalidate cache by plugin ID
✓ should respect cache TTL
✓ should recompute on content hash mismatch
✓ should handle cache write failure gracefully
```

---

## Integration Tests (analysis-engine.integration.test.ts)

**Additional Test Count:** 10-15 tests
**Priority:** P0

### Test Cases

```typescript
describe('AnalysisEngine Integration', () => {
  describe('multi-plugin scenarios', () => {
    ✓ should run core + react plugins together
    ✓ should merge insights from multiple plugins
    ✓ should calculate weighted scores across plugins
    ✓ should handle plugin priority ordering
  });

  describe('real-world analysis', () => {
    ✓ should analyze small component (< 100 LOC)
    ✓ should analyze medium component (100-500 LOC)
    ✓ should analyze large component (500-1000 LOC)
    ✓ should analyze file with multiple components
    ✓ should analyze file with complex hooks
  });

  describe('error scenarios', () => {
    ✓ should handle analysis failure in one of 14 React analyses
    ✓ should aggregate results when some analyses fail
    ✓ should handle timeout in one analysis
  });

  describe('performance', () => {
    ✓ should analyze 100 files in < 30s (sequential)
    ✓ should analyze 100 files in < 10s (concurrent = 5)
    ✓ should achieve > 90% cache hit rate on re-analysis
  });
});
```

---

## Test Fixtures to Create

### Reusable Test Fixtures

```typescript
// test-fixtures.ts

export const simpleComponent = `
import React, { useState } from 'react';

function SimpleComponent() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}
`;

export const componentWithEffects = `
import React, { useState, useEffect } from 'react';

function ComponentWithEffects() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('/api/data').then(setData);
  }, []);

  return <div>{data}</div>;
}
`;

export const componentWithCleanup = `
import React, { useEffect } from 'react';

function ComponentWithCleanup() {
  useEffect(() => {
    const timer = setTimeout(() => {}, 1000);
    return () => clearTimeout(timer);
  }, []);

  return <div>Hello</div>;
}
`;

export const componentWithManyHooks = `
import React, { useState, useEffect, useCallback, useMemo } from 'react';

function ComponentWithManyHooks() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  const [c, setC] = useState(0);

  const value = useMemo(() => a + b + c, [a, b, c]);
  const handler = useCallback(() => setA(a + 1), [a]);

  useEffect(() => {
    console.log('effect 1');
  }, [a]);

  useEffect(() => {
    console.log('effect 2');
  }, [b]);

  return <div onClick={handler}>{value}</div>;
}
`;

export const componentWithHighCoupling = `
import React, { useContext } from 'react';

function HighlyC oupled({
  prop1, prop2, prop3, prop4, prop5,
  onEvent1, onEvent2, onEvent3,
  children
}) {
  const context1 = useContext(Context1);
  const context2 = useContext(Context2);
  const context3 = useContext(Context3);

  return <div>{children}</div>;
}
`;
```

---

## Test Utilities to Create

```typescript
// test-utils.ts

export function createTestSourceFile(code: string): SourceFile {
  const project = new Project({
    compilerOptions: {
      allowJs: true,
      jsx: 2, // React
    },
    useInMemoryFileSystem: true,
  });
  return project.createSourceFile('test.tsx', code);
}

export function expectScoreBetween(score: number, min: number, max: number) {
  expect(score).toBeGreaterThanOrEqual(min);
  expect(score).toBeLessThanOrEqual(max);
}

export function expectInsightWithSeverity(
  insights: ComplexityInsight[],
  severity: 'info' | 'warning' | 'critical',
  message?: string
) {
  const found = insights.find(
    i => i.severity === severity && (!message || i.message.includes(message))
  );
  expect(found).toBeDefined();
}
```

---

## Summary

This document provides detailed specifications for ~275 critical tests across 6 files. The specifications include:

- **50-55 tests** for temporal-analysis.ts
- **30-35 tests** for coupling-analysis.ts
- **35-40 tests** for identity-analysis.ts
- **25-30 tests** for analysis-cache.ts
- **60-70 tests** for react-helpers.ts
- **30 additional tests** for analysis-engine.ts
- **10-15 integration tests**

Each test specification includes:

- Test structure (describe blocks)
- Specific test cases with checkboxes
- Edge cases to cover
- Performance requirements where applicable
- Reusable fixtures
- Custom test utilities

**Next step:** Begin implementing these tests following the specifications, starting with temporal-analysis.test.ts.
