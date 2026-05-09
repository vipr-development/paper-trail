# Code Complexity Analysis and Testing Priorities

**Analysis Date:** 2026-01-12
**Scope:** @analyzers/core and @analyzers/react packages

## Executive Summary

This analysis evaluates code complexity across the analyzer codebase to identify critical areas requiring comprehensive testing. Using cyclomatic complexity, cognitive complexity, and structural analysis, I've prioritized files based on:

- Control flow complexity (decision points, nesting depth)
- Integration risks (dependencies, side effects)
- Business criticality (core functionality)
- Current test coverage gaps

### Priority Legend

- **P0 (Critical):** Must have comprehensive tests before production
- **P1 (High):** Should have tests in current sprint
- **P2 (Medium):** Add tests in next sprint
- **P3 (Low):** Basic coverage sufficient

---

## @analyzers/core Package Analysis

### P0 - Critical Priority

#### 1. analysis-engine.ts

**Location:** `/analyzers/core/src/engine/analysis-engine.ts`
**Lines:** 781
**Cyclomatic Complexity:** ~45-50 (CRITICAL)

**Complexity Metrics:**

- **Decision Points:** 35+ conditional branches
- **Nesting Depth:** Up to 5 levels
- **Method Count:** 25+ methods with complex orchestration
- **State Management:** Multiple maps, caches, concurrent execution

**Critical Code Paths:**

1. **`analyzeFile()` (lines 227-263)** - Cyclomatic: 8
   - Cache management logic
   - Plugin selection and orchestration
   - Error handling for file operations
   - EDGE CASE: File doesn't exist, invalid source file
   - EDGE CASE: All plugins reject the file

2. **`runPluginsInParallel()` (lines 643-703)** - Cyclomatic: 12
   - Dual execution paths (legacy vs. enhanced)
   - Complex promise orchestration
   - Error boundary per plugin
   - EDGE CASE: Plugin throws during canHandle()
   - EDGE CASE: Plugin hangs/timeouts
   - EDGE CASE: Mixed plugin versions

3. **`executePluginAnalyses()` (lines 307-331)** - Cyclomatic: 7
   - Analysis sorting and scheduling
   - Concurrency limit management
   - Cost-based scheduling
   - EDGE CASE: Zero analyses registered
   - EDGE CASE: All analyses fail

4. **`executeAnalysis()` (lines 336-404)** - Cyclomatic: 10
   - Cache hit/miss logic
   - Timeout handling
   - Error recovery
   - Result normalization
   - EDGE CASE: Analysis throws non-Error object
   - EDGE CASE: Timeout during cache write
   - EDGE CASE: Analysis returns malformed result

5. **`aggregateResults()` (lines 705-759)** - Cyclomatic: 9
   - Score weighting algorithm
   - Insight deduplication
   - Error consolidation
   - EDGE CASE: All plugins fail
   - EDGE CASE: Conflicting scores
   - EDGE CASE: Missing required fields in PluginResult

**Testing Requirements:**

- [ ] Unit tests for each public method (12 methods)
- [ ] Integration tests for plugin lifecycle
- [ ] Performance tests for concurrency limits (1, 5, 10, Infinity)
- [ ] Error scenario tests (15+ scenarios)
- [ ] Cache invalidation edge cases (6 scenarios)
- [ ] Memory leak tests under load
- [ ] Concurrent file analysis stress test

**Recommended Test Count:** 50-60 tests

**Performance-Critical Sections:**

- Lines 495-519: `executeWithConcurrencyLimit()` - Needs benchmarking
- Lines 606-615: `computeContentHash()` - Should use crypto hash in production
- Lines 329: Analysis sorting - Profile with 50+ analyses

---

#### 2. analysis-cache.ts

**Location:** `/analyzers/core/src/engine/analysis-cache.ts`
**Lines:** 112
**Cyclomatic Complexity:** ~18-20

**Complexity Metrics:**

- **Decision Points:** 12 conditionals
- **Data Structure:** Map with string keys (composite)
- **Algorithm Complexity:** O(n) invalidation patterns

**Critical Code Paths:**

1. **`get()` (lines 26-35)** - Cyclomatic: 2
   - Simple but critical for performance
   - EDGE CASE: Malformed cache key

2. **`invalidateByFile()` (lines 47-53)** - Cyclomatic: 3
   - String prefix matching - potential performance bottleneck
   - EDGE CASE: Thousands of entries with similar prefixes
   - EDGE CASE: Special characters in file paths

3. **`invalidateByPlugin()` (lines 55-62)** - Cyclomatic: 3
   - String splitting and comparison
   - EDGE CASE: Plugin ID contains colons

4. **`getStats()` (lines 79-106)** - Cyclomatic: 5
   - JSON serialization for memory estimation (expensive)
   - Reduce operation over all entries
   - EDGE CASE: Circular references in cached results
   - EDGE CASE: Very large cached objects

**Testing Requirements:**

- [ ] Cache key collision tests
- [ ] Invalidation correctness (by file, plugin, analysis)
- [ ] Memory usage tracking with 1000+ entries
- [ ] Stats calculation performance test
- [ ] Concurrent read/write scenarios
- [ ] Edge cases: special characters, long paths, unicode

**Recommended Test Count:** 25-30 tests

**Performance Concerns:**

- `invalidateByFile()` with 10,000+ cache entries
- `getStats()` called frequently could cause performance issues
- Memory estimation via JSON.stringify is expensive

---

### P1 - High Priority

#### 3. ast-helpers.ts

**Location:** `/analyzers/core/src/utils/ast-helpers.ts`
**Lines:** 299
**Cyclomatic Complexity:** ~25-28

**Complexity Metrics:**

- **Function Count:** 19 exported functions
- **Decision Points:** 20+ across all functions
- **AST Traversal:** Heavy use of recursive descent

**Critical Functions:**

1. **`calculateCyclomaticComplexity()` (lines 140-172)** - Cyclomatic: 8
   - Core algorithm for complexity calculation
   - Complex set-based matching
   - Nested forEachDescendant
   - EDGE CASE: Deeply nested expressions (100+ levels)
   - EDGE CASE: Optional chaining in complex expressions
   - EDGE CASE: Binary expressions with multiple logical operators

2. **`shouldIgnore()` in base-analyzer.ts (lines 167-180)** - Cyclomatic: 5
   - Glob pattern matching with regex conversion
   - EDGE CASE: Malicious regex patterns (ReDoS)
   - EDGE CASE: Patterns with multiple wildcards

**Testing Requirements:**

- [ ] Cyclomatic complexity calculation accuracy (compare with known tools)
- [ ] Edge cases: empty files, syntax errors, huge files
- [ ] Performance: files with 10,000+ nodes
- [ ] Each helper function needs 3-5 test cases
- [ ] Naming convention validators (PascalCase, camelCase, SCREAMING_SNAKE_CASE)

**Recommended Test Count:** 40-45 tests

---

#### 4. scoring.ts

**Location:** `/analyzers/core/src/utils/scoring.ts`
**Lines:** 197
**Cyclomatic Complexity:** ~15-18

**Complexity Metrics:**

- **Decision Points:** 12 conditionals
- **Algorithm Complexity:** Weighted averaging, normalization
- **Validation:** Extensive input validation with warnings

**Critical Functions:**

1. **`calculateCompositeScore()` (lines 44-73)** - Cyclomatic: 6
   - Weight validation and alignment checking
   - Console warnings for misconfigurations
   - EDGE CASE: Weights sum to 0
   - EDGE CASE: NaN or Infinity in scores
   - EDGE CASE: Missing dimensions vs weights

2. **`normalizeScore()` (lines 135-157)** - Cyclomatic: 7
   - Input validation for finite numbers
   - Warning for misconfigured references
   - EDGE CASE: referenceScore = 0 (division by zero)
   - EDGE CASE: rawScore = Infinity
   - EDGE CASE: Negative scores

**Testing Requirements:**

- [ ] Score to grade boundary tests (exact boundaries)
- [ ] Normalization with edge values (0, Infinity, NaN, negative)
- [ ] Composite score calculation with various weight distributions
- [ ] Warning message generation tests
- [ ] Round-off error accumulation in large calculations

**Recommended Test Count:** 30-35 tests

**Existing Coverage:** `/analyzers/core/src/utils/scoring.test.ts` exists - review and extend

---

### P2 - Medium Priority

#### 5. plugin.ts (core)

**Location:** `/analyzers/core/src/plugin.ts`
**Lines:** 172
**Cyclomatic Complexity:** ~12-15

**Complexity Metrics:**

- **Decision Points:** 9 conditionals
- **Integration:** Coordinates multiple analyses
- **Data Transformation:** Maps ComplexityInsight to PluginInsight

**Critical Methods:**

1. **`analyze()` (lines 86-104)** - Cyclomatic: 3
   - Fallback path for legacy compatibility
   - Sequential execution (performance concern)

2. **`aggregateResults()` (lines 113-163)** - Cyclomatic: 8
   - Score calculation across analyses
   - Insight transformation
   - Analysis breakdown map construction

**Testing Requirements:**

- [ ] Plugin registration and lifecycle
- [ ] Analysis execution (enabled/disabled)
- [ ] Result aggregation correctness
- [ ] Integration with AnalysisEngine

**Recommended Test Count:** 20-25 tests

---

#### 6. base-analyzer.ts

**Location:** `/analyzers/core/src/base-analyzer.ts`
**Lines:** 196
**Cyclomatic Complexity:** ~8-10

**Complexity Metrics:**

- **Decision Points:** 6 conditionals
- **Abstraction:** Template method pattern
- **Timing:** Performance measurement built-in

**Testing Requirements:**

- [ ] Timing accuracy
- [ ] Location extraction correctness
- [ ] Threshold override behavior
- [ ] Ignore pattern matching (including ReDoS vulnerabilities)

**Recommended Test Count:** 15-20 tests

---

## @analyzers/react Package Analysis

### P0 - Critical Priority

#### 7. dataflow-analysis.ts

**Location:** `/analyzers/react/src/analyses/dataflow-analysis.ts`
**Lines:** 359
**Cyclomatic Complexity:** ~35-40 (CRITICAL)

**Complexity Metrics:**

- **Decision Points:** 28+ conditionals
- **Nesting Depth:** Up to 6 levels
- **Algorithm Complexity:** Graph traversal for prop drilling

**Critical Methods:**

1. **`analyzeDataflow()` (lines 66-198)** - Cyclomatic: 14
   - Multiple nested forEachDescendant calls
   - Complex state tracking (useState, useMemo, useCallback, useRef)
   - EDGE CASE: Destructured array bindings with < 2 elements
   - EDGE CASE: setState called before declaration

2. **`analyzePropDrilling()` (lines 291-357)** - Cyclomatic: 12
   - JSX nesting depth calculation
   - Prop usage tracking across components
   - EDGE CASE: Circular component references
   - EDGE CASE: JSX spread attributes
   - EDGE CASE: Component names with special characters

3. **`getArrayMethodChainLength()` (lines 243-286)** - Cyclomatic: 8
   - Iterative chain traversal
   - JSX context detection
   - EDGE CASE: Recursive method calls
   - EDGE CASE: Method chains > 20 levels

**Testing Requirements:**

- [ ] State tracking accuracy (useState declarations and setter calls)
- [ ] Prop drilling detection (various nesting levels: 2, 5, 10, 20)
- [ ] Transform chain analysis (map/filter/reduce combinations)
- [ ] Derived value tracking (useMemo/useCallback)
- [ ] Edge cases: malformed destructuring, missing setters
- [ ] Performance: components with 100+ state variables

**Recommended Test Count:** 45-50 tests

**Existing Coverage:** `/analyzers/react/src/analyses/dataflow-analysis.test.ts` exists - **verify coverage**

---

#### 8. temporal-analysis.ts

**Location:** `/analyzers/react/src/analyses/temporal-analysis.ts`
**Lines:** 270
**Cyclomatic Complexity:** ~32-35 (CRITICAL)

**Complexity Metrics:**

- **Decision Points:** 25+ conditionals
- **Risk Assessment:** Multi-factor risk level calculation
- **Pattern Detection:** 7 different temporal patterns

**Critical Methods:**

1. **`analyzeTemporal()` (lines 69-264)** - Cyclomatic: 18
   - Effect detection (useEffect, useLayoutEffect, useInsertionEffect)
   - Cleanup function analysis
   - Risk factor accumulation
   - Pattern detection (timeout, interval, event listeners, async, state updates)
   - EDGE CASE: Effect without callback argument
   - EDGE CASE: Multiple cleanup returns in effect body
   - EDGE CASE: Cleanup that doesn't clear subscriptions

2. **Risk Level Calculation (lines 154-209)** - Cyclomatic: 15
   - Complex conditional logic for risk assessment
   - Risk mitigation via cleanup detection
   - EDGE CASE: Conflicting risk factors
   - EDGE CASE: Edge between medium/high risk

**Testing Requirements:**

- [ ] Effect detection accuracy (all 3 types)
- [ ] Cleanup function detection (various patterns)
- [ ] Risk level calculation (all combinations of factors)
- [ ] Pattern detection: timeout, interval, listeners, async, state updates
- [ ] Dependency array analysis (-1, 0, 1-5, 10+)
- [ ] Insight generation for each risky pattern
- [ ] Edge cases: missing cleanup, nested effects, conditional effects

**Recommended Test Count:** 50-55 tests

**Test Coverage Gap:** NO EXISTING TEST FILE - **URGENT**

---

#### 9. hook-analysis.ts

**Location:** `/analyzers/react/src/analyses/hook-analysis.ts`
**Lines:** 214
**Cyclomatic Complexity:** ~22-25

**Complexity Metrics:**

- **Decision Points:** 18 conditionals
- **Hook Tracking:** 17+ built-in hooks + custom hooks
- **Dependency Analysis:** Complex dependency counting and validation

**Critical Methods:**

1. **`analyzeHooks()` (lines 60-154)** - Cyclomatic: 10
   - Hook call identification
   - Dependency counting with special cases
   - Custom hook detection
   - EDGE CASE: Hooks with missing dependency arrays
   - EDGE CASE: Unstable dependencies (objects, functions)

2. **`generateHookInsights()` (lines 156-208)** - Cyclomatic: 8
   - Pattern-based insight generation
   - Premature optimization detection
   - EDGE CASE: Many optimizations with little state

**Testing Requirements:**

- [ ] All 17 built-in hooks detected correctly
- [ ] Custom hook identification
- [ ] Dependency counting (empty, missing, 1-10+ deps)
- [ ] Unstable dependency detection
- [ ] Insight generation for various patterns
- [ ] Scoring algorithm verification

**Recommended Test Count:** 40-45 tests

**Existing Coverage:** `/analyzers/react/src/analyses/hook-analysis.test.ts` exists - **verify coverage**

---

### P1 - High Priority

#### 10. react-helpers.ts

**Location:** `/analyzers/react/src/utils/react-helpers.ts`
**Lines:** 685
**Cyclomatic Complexity:** ~45-50 (HIGH)

**Complexity Metrics:**

- **Function Count:** 35+ exported functions
- **Decision Points:** 35+ conditionals
- **AST Traversal:** Extensive React-specific pattern matching

**Critical Functions:**

1. **`findReactComponents()` (lines 71-104)** - Cyclomatic: 6
   - Multiple component patterns (function, arrow, variable)
   - EDGE CASE: Components without JSX (render props)
   - EDGE CASE: Default exported components

2. **`extractPropsInfo()` (lines 423-462)** - Cyclomatic: 8
   - Object destructuring pattern matching
   - Callback prop detection
   - EDGE CASE: Rest parameters
   - EDGE CASE: Renamed props

3. **`analyzePropsType()` (lines 536-585)** - Cyclomatic: 10
   - Type coverage calculation
   - Optional vs required prop detection
   - EDGE CASE: Generic props types
   - EDGE CASE: Union/intersection types

4. **Effect Detection Functions (lines 298-400)** - Cyclomatic: 15 total
   - Multiple pattern detectors for temporal analysis
   - EDGE CASE: Nested function calls

**Testing Requirements:**

- [ ] Component detection (all 3 patterns)
- [ ] JSX helpers (depth, nesting, conditional rendering)
- [ ] Hook detection and dependency extraction
- [ ] Effect pattern detection (cleanup, timeout, interval, listeners, async)
- [ ] Props analysis (counting, types, callbacks)
- [ ] Type analysis helpers
- [ ] Edge cases for each function

**Recommended Test Count:** 60-70 tests

**Test Coverage Gap:** NO EXISTING TEST FILE - **URGENT**

---

#### 11. coupling-analysis.ts

**Location:** `/analyzers/react/src/analyses/coupling-analysis.ts`
**Lines:** 161
**Cyclomatic Complexity:** ~15-18

**Complexity Metrics:**

- **Decision Points:** 12 conditionals
- **Pattern Detection:** Props, context, callbacks, children, forwardRef

**Critical Methods:**

1. **`analyzeCoupling()` (lines 55-155)** - Cyclomatic: 10
   - Multiple coupling indicators
   - Children usage pattern detection
   - EDGE CASE: Render props vs simple children
   - EDGE CASE: Multiple context consumers

**Testing Requirements:**

- [ ] Props counting accuracy
- [ ] Context consumer detection
- [ ] Callback prop identification
- [ ] forwardRef detection
- [ ] Children usage patterns (none, simple, render-props)
- [ ] Scoring algorithm verification
- [ ] Insight generation thresholds

**Recommended Test Count:** 30-35 tests

**Test Coverage Gap:** NO EXISTING TEST FILE - **URGENT**

---

#### 12. identity-analysis.ts

**Location:** `/analyzers/react/src/analyses/identity-analysis.ts`
**Lines:** 192
**Cyclomatic Complexity:** ~18-20

**Complexity Metrics:**

- **Decision Points:** 14 conditionals
- **Pattern Detection:** Inline functions, objects, arrays in JSX

**Critical Methods:**

1. **`analyzeIdentity()` (lines 55-186)** - Cyclomatic: 12
   - React.memo detection
   - Unstable reference detection in JSX
   - Context-aware insight generation
   - EDGE CASE: Nested JSX expressions
   - EDGE CASE: Conditional inline values

**Testing Requirements:**

- [ ] useCallback/useMemo counting
- [ ] Dependency tracking
- [ ] Unstable reference detection (functions, objects, arrays)
- [ ] React.memo detection (both patterns)
- [ ] Context-aware insights (with/without memo components)
- [ ] Scoring algorithm

**Recommended Test Count:** 35-40 tests

**Test Coverage Gap:** NO EXISTING TEST FILE - **URGENT**

---

### P2 - Medium Priority

#### 13. plugin.ts (react)

**Location:** `/analyzers/react/src/plugin.ts`
**Lines:** 713
**Cyclomatic Complexity:** ~40-45

**Complexity Metrics:**

- **Decision Points:** 30+ conditionals
- **Integration:** Coordinates 14 analyses
- **Data Transformation:** Extensive result aggregation

**Critical Methods:**

1. **`analyze()` (lines 163-187)** - Cyclomatic: 3
   - Orchestrates all analyses
   - Backward compatibility handling

2. **`aggregateResults()` (lines 196-579)** - Cyclomatic: 25
   - Extracts data from 14 analyses
   - Normalizes scores
   - Calculates composite score
   - Builds comprehensive metrics object
   - EDGE CASE: Missing analysis results
   - EDGE CASE: Malformed analysis data

3. **`analyzeTraditional()` (lines 592-711)** - Cyclomatic: 12
   - Legacy Halstead metrics calculation
   - Should be extracted to analysis

**Testing Requirements:**

- [ ] Plugin lifecycle (register, analyze, unregister)
- [ ] Analysis coordination (enabled/disabled)
- [ ] Result aggregation with all 14 analyses
- [ ] Result aggregation with partial analysis failures
- [ ] Score normalization and weighting
- [ ] Backward compatibility paths

**Recommended Test Count:** 40-45 tests

**Existing Coverage:** `/analyzers/react/src/plugin.test.ts` exists - **verify coverage**

---

#### 14. accessibility-helpers.ts

**Location:** `/analyzers/react/src/utils/accessibility-helpers.ts`
**Lines:** 179
**Cyclomatic Complexity:** ~12-15

**Complexity Metrics:**

- **Decision Points:** 10 conditionals
- **Data Structures:** WCAG criterion mapping
- **Algorithm Complexity:** O(1) lookups

**Critical Functions:**

1. **`getWCAGLevelForCriterion()` (lines 62-128)** - Cyclomatic: 3
   - Large mapping of WCAG criteria to levels
   - EDGE CASE: Unknown criterion
   - EDGE CASE: WCAG 2.2 vs 2.1 criteria

2. **`toA11yViolation()` (lines 133-166)** - Cyclomatic: 5
   - Severity mapping
   - Element extraction from description
   - EDGE CASE: Malformed anti-pattern description

**Testing Requirements:**

- [ ] WCAG criterion mapping accuracy (all criteria)
- [ ] URL generation for all criteria
- [ ] Principle mapping correctness
- [ ] Level classification (A, AA, AAA)
- [ ] Anti-pattern to violation conversion
- [ ] Severity mapping

**Recommended Test Count:** 25-30 tests

**Test Coverage Gap:** NO EXISTING TEST FILE

---

## Integration Testing Requirements

### Critical Integration Points

1. **AnalysisEngine ↔ Plugins** (P0)
   - Plugin registration and lifecycle
   - Multi-plugin execution
   - Error isolation between plugins
   - Cache coordination

2. **AnalysisEngine ↔ Individual Analyses** (P0)
   - Analysis execution and result collection
   - Timeout handling
   - Cache hit/miss scenarios
   - Concurrent analysis execution

3. **Plugin ↔ Multiple Analyses** (P1)
   - Result aggregation across analyses
   - Score normalization consistency
   - Insight deduplication

4. **Analysis ↔ Helper Functions** (P1)
   - AST traversal correctness
   - Pattern detection accuracy
   - Edge case handling

### Integration Test Scenarios

#### AnalysisEngine Integration (15-20 tests)

```
✓ Register multiple plugins and analyze file
✓ Plugin failure doesn't crash engine
✓ Cache invalidation across plugins
✓ Concurrent file analysis (5 files in parallel)
✓ Large file handling (10MB+)
✓ Analysis timeout enforcement
✓ Memory leak prevention (100 sequential analyses)
```

#### React Plugin Integration (12-15 tests)

```
✓ All 14 analyses execute successfully
✓ Partial analysis failure handling
✓ Score calculation with missing analyses
✓ Insight aggregation from all analyses
✓ Real-world component analysis (small, medium, large)
```

**Existing Integration Test:** `/analyzers/core/src/engine/analysis-engine.integration.test.ts` - **verify and extend**

---

## Performance Benchmarking Requirements

### Critical Performance Tests (P0)

1. **analysis-engine.ts**
   - Benchmark: Analyze 100 files sequentially (target: < 30s)
   - Benchmark: Analyze 100 files with concurrency=5 (target: < 10s)
   - Benchmark: Cache hit rate with 1000 files (target: > 90% on repeated analysis)
   - Benchmark: Memory usage over 500 file analyses (target: < 500MB)

2. **analysis-cache.ts**
   - Benchmark: 10,000 cache entries get/set operations (target: < 100ms)
   - Benchmark: invalidateByFile with 10,000 entries (target: < 50ms)
   - Benchmark: getStats with 10,000 entries (target: < 200ms)

3. **ast-helpers.ts**
   - Benchmark: calculateCyclomaticComplexity on 10,000 node file (target: < 500ms)
   - Benchmark: getFunctions on file with 500 functions (target: < 100ms)

4. **dataflow-analysis.ts**
   - Benchmark: Component with 50 state variables (target: < 200ms)
   - Benchmark: Component with 10-level JSX nesting (target: < 300ms)

5. **temporal-analysis.ts**
   - Benchmark: Component with 20 effects (target: < 150ms)

### Performance Test Tooling

- Use Vitest's `bench` API
- Track performance over time (regression detection)
- Profile hot paths with V8 profiler

---

## Edge Case and Boundary Condition Testing

### Critical Edge Cases by File

#### analysis-engine.ts (20+ edge cases)

```typescript
✓ Empty source file
✓ Source file with syntax errors
✓ File with 10,000+ nodes
✓ All plugins reject file (no applicable plugins)
✓ All analyses fail for a plugin
✓ Analysis returns null/undefined
✓ Analysis throws Error vs other types
✓ Timeout during cache write
✓ Concurrent modification of plugin registry
✓ Plugin.analyze() returns malformed PluginResult
✓ Cache key collision (different files, same hash)
✓ Memory pressure during analysis (OOM scenarios)
```

#### dataflow-analysis.ts (15+ edge cases)

```typescript
✓ useState with < 2 destructured elements
✓ useState without setter name
✓ setState called before useState declaration
✓ Circular prop drilling references
✓ Transform chains > 20 levels deep
✓ JSX nesting > 50 levels
✓ Props with unicode characters
✓ Spread attributes in prop drilling detection
```

#### temporal-analysis.ts (12+ edge cases)

```typescript
✓ useEffect without callback
✓ useEffect with multiple return statements
✓ useEffect with conditional cleanup
✓ Nested effects (useEffect inside useEffect)
✓ Effect with missing dependency array vs empty array
✓ Cleanup that throws error
```

#### react-helpers.ts (25+ edge cases)

```typescript
✓ Component without JSX (render props)
✓ Default exported components
✓ Generic component with complex type parameters
✓ Props with rest parameters
✓ Props with renamed destructuring
✓ Nested function components
✓ Higher-order components
✓ Memo components with multiple wrapping patterns
```

---

## Test Organization and Standards

### Test File Structure

```typescript
describe('FileName', () => {
  describe('metadata', () => {
    // Verify IAnalysis interface compliance
  });

  describe('methodName', () => {
    describe('happy path', () => {
      // Normal operation tests
    });

    describe('edge cases', () => {
      // Boundary conditions
    });

    describe('error handling', () => {
      // Failure scenarios
    });
  });
});
```

### Testing Utilities

- Use `@vipr/testing` utilities (TestUtils, TestFixtures)
- Create reusable fixtures for common patterns
- Mock performance.now() for timing tests
- Use snapshot testing for complex result objects

### Code Coverage Targets

- **P0 files:** 90%+ line coverage, 85%+ branch coverage
- **P1 files:** 80%+ line coverage, 75%+ branch coverage
- **P2 files:** 70%+ line coverage, 65%+ branch coverage

---

## Summary: Testing Priorities

### Immediate Action (This Sprint)

1. **Create missing test files** (4 files, ~180 tests total):
   - `temporal-analysis.test.ts` (50-55 tests)
   - `coupling-analysis.test.ts` (30-35 tests)
   - `identity-analysis.test.ts` (35-40 tests)
   - `react-helpers.test.ts` (60-70 tests)

2. **Comprehensive test coverage for critical files**:
   - `analysis-engine.ts` - Add 30+ tests (50-60 total)
   - `analysis-cache.ts` - Add 15+ tests (25-30 total)
   - `dataflow-analysis.ts` - Verify and extend existing tests (45-50 total)

3. **Integration tests**:
   - Extend `analysis-engine.integration.test.ts` with 10+ scenarios

### Next Sprint

4. **Extend existing test files**:
   - `ast-helpers.ts` - Create comprehensive test file (40-45 tests)
   - `scoring.ts` - Extend existing tests (30-35 total)
   - `plugin.ts` (core) - Create test file (20-25 tests)

5. **React plugin tests**:
   - Verify `hook-analysis.test.ts` coverage
   - Verify `dataflow-analysis.test.ts` coverage
   - Verify `plugin.test.ts` coverage

6. **Helper function tests**:
   - `accessibility-helpers.test.ts` (25-30 tests)

### Performance and Load Testing

7. **Benchmarking suite** (create `*.bench.ts` files):
   - `analysis-engine.bench.ts`
   - `analysis-cache.bench.ts`
   - `dataflow-analysis.bench.ts`

---

## Appendix: Complexity Calculation Methodology

### Cyclomatic Complexity

McCabe's cyclomatic complexity: `M = E - N + 2P`

- E = edges in control flow graph
- N = nodes
- P = connected components

For practical purposes, counting decision points:

- Each `if`, `else if`, `for`, `while`, `case`, `catch`, `&&`, `||`, `??`, `?:` adds 1

### Cognitive Complexity

Considers human readability:

- Nesting depth penalty (each level adds weight)
- Break-in-flow statements (continue, break, return)
- Recursive calls

### Risk Assessment Factors

Files prioritized by:

1. Cyclomatic complexity > 30 = Critical
2. Integration points (caches, concurrency, I/O)
3. Error-prone patterns (recursive descent, complex conditions)
4. Business criticality (core engine, analysis orchestration)

---

## Appendix: Test Count Justification

### Total Recommended Tests

**@analyzers/core:**

- analysis-engine.ts: 50-60 tests
- analysis-cache.ts: 25-30 tests
- ast-helpers.ts: 40-45 tests
- scoring.ts: 30-35 tests (10-15 new)
- plugin.ts: 20-25 tests
- base-analyzer.ts: 15-20 tests
- **Subtotal: ~200-235 tests**

**@analyzers/react:**

- dataflow-analysis.ts: 45-50 tests (verify existing)
- temporal-analysis.ts: 50-55 tests (NEW)
- hook-analysis.ts: 40-45 tests (verify existing)
- react-helpers.ts: 60-70 tests (NEW)
- coupling-analysis.ts: 30-35 tests (NEW)
- identity-analysis.ts: 35-40 tests (NEW)
- plugin.ts: 40-45 tests (verify existing)
- accessibility-helpers.ts: 25-30 tests (NEW)
- **Subtotal: ~325-370 tests**

**Integration tests: 25-35 tests**

**Performance benchmarks: 15-20 benchmarks**

**Grand Total: ~565-660 tests**

Current test count estimate: ~150-200 tests
**Gap: ~365-460 additional tests needed**

---

## Conclusion

The analysis reveals **4 critical files** requiring immediate comprehensive testing:

1. `analysis-engine.ts` - Complex orchestration, concurrency, caching
2. `dataflow-analysis.ts` - Complex graph traversal and state tracking
3. `temporal-analysis.ts` - Multi-factor risk assessment
4. `react-helpers.ts` - Extensive pattern matching library

Additionally, **3 analysis files have no tests** and need complete test coverage:

- `temporal-analysis.ts`
- `coupling-analysis.ts`
- `identity-analysis.ts`

The recommended testing effort prioritizes these files based on:

- Code complexity (cyclomatic > 30)
- Integration risk (concurrency, caching, I/O)
- Missing coverage (no existing tests)
- Business criticality (core functionality)

**Estimated effort:** 40-60 hours for P0 tests, 80-120 hours for comprehensive coverage.
