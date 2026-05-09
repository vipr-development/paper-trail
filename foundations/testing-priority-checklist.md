# Testing Priority Checklist

**Generated:** 2026-01-12
**Status Tracking:** Use this checklist to track testing progress

## Sprint 1: Critical Tests (P0)

### Missing Test Files (URGENT)

- [ ] **temporal-analysis.test.ts** (50-55 tests)
  - Effect detection (useEffect, useLayoutEffect, useInsertionEffect)
  - Cleanup function analysis
  - Risk level calculation (low, medium, high)
  - Pattern detection: timeout, interval, listeners, async, state updates
  - Dependency array scenarios: missing, empty, 1-5, 10+
  - Edge cases: nested effects, conditional cleanup, multiple returns

- [ ] **coupling-analysis.test.ts** (30-35 tests)
  - Props counting with various component patterns
  - Context consumer detection
  - Callback prop identification
  - forwardRef detection
  - Children usage patterns (none, simple, render-props)
  - Scoring algorithm validation
  - Threshold-based insight generation

- [ ] **identity-analysis.test.ts** (35-40 tests)
  - useCallback/useMemo counting
  - Dependency tracking and counting
  - Unstable reference detection (inline functions, objects, arrays in JSX)
  - React.memo detection (both `memo()` and `React.memo()` patterns)
  - Context-aware insights (with/without memoized components)
  - Event handler pattern detection
  - Style prop inline object detection

- [ ] **react-helpers.test.ts** (60-70 tests)
  - Component detection (function, arrow, variable declarations)
  - JSX helpers (depth, nesting, conditional rendering)
  - Hook detection (all 17 built-in hooks)
  - Dependency extraction from hooks
  - Effect pattern detection functions (5 detectors)
  - Props analysis (info extraction, type analysis)
  - Type helpers (any detection, untyped hooks)
  - Edge cases: default exports, HOCs, generic types

### Extend Existing Test Coverage

- [ ] **analysis-engine.ts** - Add 30+ tests
  - [ ] analyzeFile() edge cases
    - [ ] Empty source file
    - [ ] Syntax error in source
    - [ ] File with 10,000+ nodes
    - [ ] No applicable plugins
  - [ ] runPluginsInParallel() scenarios
    - [ ] Plugin throws during canHandle()
    - [ ] Plugin timeout
    - [ ] Mixed plugin API versions
  - [ ] executeAnalysis() error handling
    - [ ] Analysis throws non-Error
    - [ ] Timeout during execution
    - [ ] Malformed result object
  - [ ] Cache invalidation
    - [ ] Invalidate by file path
    - [ ] Invalidate during active analysis
  - [ ] Concurrency tests
    - [ ] Limits: 1, 5, 10, Infinity
    - [ ] Analysis cost-based scheduling
  - [ ] Memory and performance
    - [ ] 100 sequential analyses (memory leak check)
    - [ ] Cache memory growth

- [ ] **analysis-cache.ts** - Add 15+ tests
  - [ ] Cache key collision scenarios
  - [ ] Special characters in file paths (unicode, spaces, colons)
  - [ ] invalidateByFile() with 10,000 entries (performance)
  - [ ] invalidateByPlugin() with plugin IDs containing colons
  - [ ] getStats() with large objects
  - [ ] Concurrent read/write operations
  - [ ] Memory estimation accuracy

- [ ] **dataflow-analysis.ts** - Verify existing, add if needed
  - [ ] Verify useState tracking with edge cases
  - [ ] Verify prop drilling detection at various depths (2, 5, 10, 20)
  - [ ] Verify transform chain analysis (1-20 levels)
  - [ ] Add: malformed destructuring (< 2 elements)
  - [ ] Add: setState before useState declaration
  - [ ] Add: circular prop references
  - [ ] Add: spread attributes in JSX

### Integration Tests

- [ ] **analysis-engine.integration.test.ts** - Add 10+ scenarios
  - [ ] Multi-plugin analysis (core + react)
  - [ ] Plugin failure isolation (one fails, others succeed)
  - [ ] Cache coordination across plugins
  - [ ] Concurrent file analysis (5 files simultaneously)
  - [ ] Large file handling (1MB+, 10MB+)
  - [ ] Timeout enforcement across analyses
  - [ ] Result aggregation with partial failures
  - [ ] Memory leak prevention (100 file analyses)

---

## Sprint 2: High Priority Tests (P1)

### Core Package Tests

- [ ] **ast-helpers.test.ts** (40-45 tests)
  - [ ] calculateCyclomaticComplexity accuracy
    - [ ] Simple functions (complexity 1-5)
    - [ ] Complex functions (10-20 decision points)
    - [ ] Edge: deeply nested expressions (100+ levels)
    - [ ] Edge: optional chaining in complex expressions
    - [ ] Edge: multiple logical operators in binary expressions
  - [ ] LOC utilities (getLOC, getNodeLOC)
  - [ ] Function detection (getFunctions, isFunctionLike)
  - [ ] Naming convention validators
    - [ ] isPascalCase (valid, invalid, edge cases)
    - [ ] isCamelCase (valid, invalid, edge cases)
    - [ ] isScreamingSnakeCase (valid, invalid, edge cases)
  - [ ] Node depth calculation
  - [ ] Async operation detection
  - [ ] Performance: 10,000 node file

- [ ] **scoring.test.ts** - Extend existing (add 15+ tests)
  - [ ] scoreToGrade boundary tests (exact boundaries: 25, 45, 65, 80)
  - [ ] normalizeScore edge values
    - [ ] referenceScore = 0 (should handle gracefully)
    - [ ] rawScore = Infinity
    - [ ] rawScore = NaN
    - [ ] Negative scores
  - [ ] calculateCompositeScore validation
    - [ ] Weights sum to 0
    - [ ] Missing dimensions in weights
    - [ ] Extra weights not in dimensions
  - [ ] Round-off error accumulation (100 calculations)
  - [ ] normalizeDimensions with various reference values

- [ ] **plugin.ts (core)** (20-25 tests)
  - [ ] Plugin lifecycle (onRegister, onUnregister)
  - [ ] Analysis registration and retrieval
  - [ ] canHandle() logic
  - [ ] analyze() fallback path
  - [ ] aggregateResults() correctness
  - [ ] Score calculation with missing scores
  - [ ] Insight transformation (ComplexityInsight → PluginInsight)

### React Package Tests

- [ ] **plugin.ts (react)** - Verify existing, extend
  - [ ] Verify lifecycle tests
  - [ ] Verify 14 analysis orchestration
  - [ ] Add: partial analysis failure handling
  - [ ] Add: score normalization edge cases
  - [ ] Add: missing analysis result handling
  - [ ] Add: aggregateResults with all 14 analyses
  - [ ] Add: backward compatibility scenarios

- [ ] **hook-analysis.test.ts** - Verify coverage
  - [ ] Verify all 17 built-in hooks tested
  - [ ] Verify custom hook detection
  - [ ] Verify dependency counting (empty, missing, 1-10+)
  - [ ] Verify unstable dependency detection
  - [ ] Add: premature optimization detection
  - [ ] Add: insight generation validation

- [ ] **accessibility-helpers.test.ts** (25-30 tests)
  - [ ] WCAG criterion mapping for all rules
  - [ ] getWCAGReferenceUrl() for all criteria
  - [ ] getPrincipleForCriterion() (1, 2, 3, 4)
  - [ ] getWCAGLevelForCriterion() (A, AA, AAA)
  - [ ] toA11yViolation() transformation
  - [ ] Severity mapping correctness
  - [ ] Element extraction from description
  - [ ] Edge: unknown criterion
  - [ ] Edge: malformed anti-pattern description

---

## Sprint 3: Medium Priority (P2)

### Core Package

- [ ] **base-analyzer.ts** (15-20 tests)
  - [ ] analyze() timing accuracy
  - [ ] getLocation() correctness (line/column)
  - [ ] getThreshold() with config overrides
  - [ ] shouldIgnore() pattern matching
    - [ ] Simple patterns (substring matching)
    - [ ] Glob patterns with wildcards
    - [ ] Edge: malicious regex patterns (ReDoS protection)
  - [ ] debug() logging behavior

### Additional Analyses Tests

Review and add tests for remaining React analyses as needed:

- [ ] structural-analysis.test.ts - Verify coverage
- [ ] anti-pattern-analysis.test.ts - Verify coverage
- [ ] security-analysis.test.ts - Verify coverage
- [ ] performance-analysis.test.ts - Verify coverage
- [ ] reliability-analysis.test.ts - Verify coverage
- [ ] technical-debt-analysis.test.ts - Verify coverage
- [ ] migration-analysis.test.ts - Verify coverage
- [ ] types-analysis.test.ts - Verify coverage
- [ ] accessibility-analysis.test.ts - Verify coverage

---

## Performance Benchmarking

### Create Benchmark Files

- [ ] **analysis-engine.bench.ts**
  - [ ] Benchmark: 100 files sequentially (target: < 30s)
  - [ ] Benchmark: 100 files with concurrency=5 (target: < 10s)
  - [ ] Benchmark: Cache hit rate with 1000 files (target: > 90%)
  - [ ] Benchmark: Memory usage over 500 analyses (target: < 500MB)

- [ ] **analysis-cache.bench.ts**
  - [ ] Benchmark: 10,000 get/set operations (target: < 100ms)
  - [ ] Benchmark: invalidateByFile with 10,000 entries (target: < 50ms)
  - [ ] Benchmark: getStats with 10,000 entries (target: < 200ms)

- [ ] **ast-helpers.bench.ts**
  - [ ] Benchmark: calculateCyclomaticComplexity on 10,000 node file (target: < 500ms)
  - [ ] Benchmark: getFunctions on 500 function file (target: < 100ms)

- [ ] **dataflow-analysis.bench.ts**
  - [ ] Benchmark: 50 state variables (target: < 200ms)
  - [ ] Benchmark: 10-level JSX nesting (target: < 300ms)

- [ ] **temporal-analysis.bench.ts**
  - [ ] Benchmark: 20 effects (target: < 150ms)

---

## Test Quality Metrics

### Coverage Targets

- [ ] P0 files: 90%+ line, 85%+ branch coverage
- [ ] P1 files: 80%+ line, 75%+ branch coverage
- [ ] P2 files: 70%+ line, 65%+ branch coverage

### Test Standards

- [ ] All tests use `@vipr/testing` utilities
- [ ] Complex results use snapshot testing
- [ ] Timing tests mock `performance.now()`
- [ ] Test file structure follows template (metadata, happy path, edge cases, errors)
- [ ] Reusable fixtures created for common patterns

---

## Progress Tracking

**Sprint 1 Progress:** 0/185 tests (0%)
**Sprint 2 Progress:** 0/180 tests (0%)
**Sprint 3 Progress:** 0/40 tests (0%)

**Total Progress:** 0/405 core tests (0%)
**Benchmark Progress:** 0/11 benchmarks (0%)

**Estimated Effort:**

- Sprint 1 (Critical): 40-60 hours
- Sprint 2 (High): 40-50 hours
- Sprint 3 (Medium): 10-15 hours
- Benchmarks: 8-12 hours

**Total Estimate:** 98-137 hours

---

## Notes

- Run full test suite after each batch: `npm test`
- Check coverage: `npm run test:coverage`
- Run benchmarks: `npm run test:bench`
- Update this checklist as tests are completed
- Flag any tests that reveal bugs for immediate fix

## Quick Reference: Files Without Tests

1. `/analyzers/react/src/analyses/temporal-analysis.ts` - 270 lines, cyclomatic ~35 (CRITICAL)
2. `/analyzers/react/src/analyses/coupling-analysis.ts` - 161 lines, cyclomatic ~18
3. `/analyzers/react/src/analyses/identity-analysis.ts` - 192 lines, cyclomatic ~20
4. `/analyzers/react/src/utils/react-helpers.ts` - 685 lines, cyclomatic ~50 (HIGH)
5. `/analyzers/react/src/utils/accessibility-helpers.ts` - 179 lines, cyclomatic ~15
6. `/analyzers/core/src/plugin.ts` - 172 lines, cyclomatic ~15
7. `/analyzers/core/src/base-analyzer.ts` - 196 lines, cyclomatic ~10
8. `/analyzers/core/src/utils/ast-helpers.ts` - 299 lines (partial coverage)
