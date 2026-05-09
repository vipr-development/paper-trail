# Code Complexity Matrix

**Visual reference for testing priorities based on complexity metrics**

## Complexity Heat Map

| File                     | LOC | Cyclomatic | Methods | Tests   | Priority | Risk     |
| ------------------------ | --- | ---------- | ------- | ------- | -------- | -------- |
| **@analyzers/core**      |
| analysis-engine.ts       | 781 | 45-50      | 25+     | Partial | P0       | HIGH     |
| analysis-cache.ts        | 112 | 18-20      | 10      | None    | P0       | MEDIUM   |
| ast-helpers.ts           | 299 | 25-28      | 19      | None    | P1       | MEDIUM   |
| scoring.ts               | 197 | 15-18      | 9       | Yes     | P1       | LOW      |
| plugin.ts                | 172 | 12-15      | 8       | None    | P2       | LOW      |
| base-analyzer.ts         | 196 | 8-10       | 7       | None    | P2       | LOW      |
| **@analyzers/react**     |
| dataflow-analysis.ts     | 359 | 35-40      | 7       | Yes     | P0       | HIGH     |
| temporal-analysis.ts     | 270 | 32-35      | 3       | None    | P0       | CRITICAL |
| hook-analysis.ts         | 214 | 22-25      | 4       | Yes     | P0       | MEDIUM   |
| react-helpers.ts         | 685 | 45-50      | 35+     | None    | P1       | HIGH     |
| coupling-analysis.ts     | 161 | 15-18      | 3       | None    | P1       | MEDIUM   |
| identity-analysis.ts     | 192 | 18-20      | 3       | None    | P1       | MEDIUM   |
| plugin.ts                | 713 | 40-45      | 10      | Partial | P2       | MEDIUM   |
| accessibility-helpers.ts | 179 | 12-15      | 7       | None    | P2       | LOW      |

## Complexity Score Legend

### Cyclomatic Complexity

- **1-10:** Low complexity (maintainable)
- **11-20:** Moderate complexity (needs attention)
- **21-30:** High complexity (requires refactoring consideration)
- **31+:** Critical complexity (urgent testing/refactoring)

### Risk Assessment

- **LOW:** Simple logic, few dependencies
- **MEDIUM:** Moderate complexity, some integration points
- **HIGH:** Complex logic, multiple integration points, or missing tests
- **CRITICAL:** Very high complexity + no tests + critical functionality

## Top 5 Most Complex Functions

### 1. analysis-engine.ts::executeAnalysis()

**Cyclomatic:** 10 | **Lines:** 68 | **Risk:** HIGH

```
Decision points: 10
Nesting depth: 4
Integration: Cache, timeout, error handling
Edge cases: 7+
```

### 2. dataflow-analysis.ts::analyzeDataflow()

**Cyclomatic:** 14 | **Lines:** 132 | **Risk:** HIGH

```
Decision points: 14
Nesting depth: 6
Integration: AST traversal, state tracking
Edge cases: 8+
```

### 3. temporal-analysis.ts::analyzeTemporal()

**Cyclomatic:** 18 | **Lines:** 195 | **Risk:** CRITICAL

```
Decision points: 18
Nesting depth: 5
Integration: Multiple hook types, risk assessment
Edge cases: 12+
```

### 4. dataflow-analysis.ts::analyzePropDrilling()

**Cyclomatic:** 12 | **Lines:** 66 | **Risk:** HIGH

```
Decision points: 12
Nesting depth: 6
Integration: JSX traversal, prop tracking
Edge cases: 6+
```

### 5. analysis-engine.ts::aggregateResults()

**Cyclomatic:** 9 | **Lines:** 54 | **Risk:** MEDIUM

```
Decision points: 9
Nesting depth: 3
Integration: Score weighting, insight merging
Edge cases: 5+
```

## Complexity Distribution

### @analyzers/core

```
Files analyzed: 6
Total LOC: 1,757
Avg cyclomatic: 20.5
Files > 20 complexity: 2 (33%)
Files without tests: 4 (67%)
```

### @analyzers/react

```
Files analyzed: 8
Total LOC: 2,773
Avg cyclomatic: 27.9
Files > 20 complexity: 6 (75%)
Files without tests: 4 (50%)
```

### Combined Statistics

```
Total files: 14
Total LOC: 4,530
Total estimated test count needed: 405
Currently missing tests: ~255
Coverage gap: ~63%
```

## Critical Path Analysis

### Engine Execution Flow (Risk: HIGH)

```
AnalysisEngine.analyzeFile()
  ↓
  getCachedResult() [Cache miss potential]
  ↓
  findApplicablePlugins() [Plugin rejection potential]
  ↓
  runPluginsInParallel() [Concurrency + error handling]
    ↓
    executePluginAnalyses() [Per-plugin execution]
      ↓
      executeAnalysis() [Timeout + cache + errors]
        ↓
        analysis.execute() [Analysis-specific logic]
  ↓
  aggregateResults() [Score weighting]
  ↓
  setCachedResult()
```

**Total Complexity:** ~60 decision points across flow
**Critical Tests Needed:** 40-50 tests covering this path

### React Analysis Flow (Risk: HIGH)

```
ReactPlugin.analyze()
  ↓
  getEnabledAnalyses() [Configuration]
  ↓
  For each analysis:
    execute(sourceFile)
      ↓
      forEachDescendant() [AST traversal - expensive]
        ↓
        Pattern matching [Hook detection, JSX analysis]
        ↓
        Score calculation
  ↓
  aggregateResults() [14 analyses, normalization]
```

**Total Complexity:** ~120 decision points across 14 analyses
**Critical Tests Needed:** 150-200 tests covering all analyses

## Complexity Trends

### High-Risk Patterns Identified

1. **Nested forEachDescendant() calls** (4 occurrences)
   - Files: dataflow-analysis.ts, temporal-analysis.ts
   - Risk: O(n²) performance, high cognitive load
   - Mitigation: Performance tests, profiling

2. **Complex conditional logic in loops** (6 occurrences)
   - Files: analysis-engine.ts, temporal-analysis.ts, dataflow-analysis.ts
   - Risk: Hard to test all branches
   - Mitigation: Extract to pure functions

3. **Cache management with state** (3 occurrences)
   - Files: analysis-engine.ts, analysis-cache.ts
   - Risk: Race conditions, memory leaks
   - Mitigation: Concurrent access tests, memory profiling

4. **Error handling without recovery** (8 occurrences)
   - Files: analysis-engine.ts (multiple methods)
   - Risk: Cascading failures
   - Mitigation: Error isolation tests

## Testing ROI Analysis

### Highest Testing ROI (Test Coverage / Complexity)

1. **temporal-analysis.ts** - ROI: 9.5/10
   - No tests, critical complexity (35)
   - 270 LOC, 50-55 tests needed
   - High bug probability due to risk assessment logic

2. **analysis-engine.ts** - ROI: 9/10
   - Partial tests, highest complexity (50)
   - 781 LOC, 50-60 tests needed
   - Critical path for all analyses

3. **react-helpers.ts** - ROI: 8.5/10
   - No tests, high complexity (50)
   - 685 LOC, 60-70 tests needed
   - Used by all React analyses

4. **dataflow-analysis.ts** - ROI: 7.5/10
   - Has tests, high complexity (40)
   - 359 LOC, verify/extend tests
   - Complex graph traversal algorithms

5. **analysis-cache.ts** - ROI: 7/10
   - No tests, moderate complexity (20)
   - 112 LOC, 25-30 tests needed
   - Performance-critical component

## Recommended Test Distribution

### Sprint 1 (Critical - 40-60 hours)

```
temporal-analysis.test.ts:      55 tests (20%)
analysis-engine additions:      30 tests (11%)
coupling-analysis.test.ts:      35 tests (13%)
identity-analysis.test.ts:      40 tests (14%)
react-helpers.test.ts:          70 tests (25%)
analysis-cache.test.ts:         30 tests (11%)
Integration tests:              15 tests (6%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total:                         275 tests (100%)
```

### Sprint 2 (High - 40-50 hours)

```
ast-helpers.test.ts:            45 tests (28%)
scoring.test.ts additions:      15 tests (9%)
plugin.ts (core):               25 tests (15%)
dataflow-analysis verify:       10 tests (6%)
hook-analysis verify:           10 tests (6%)
plugin.ts (react) verify:       15 tests (9%)
accessibility-helpers.test.ts:  30 tests (19%)
Integration additions:          10 tests (6%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total:                         160 tests (100%)
```

## Complexity Reduction Recommendations

While not part of the testing task, these refactoring opportunities would reduce complexity:

1. **analysis-engine.ts**
   - Extract `executeWithConcurrencyLimit()` to separate utility
   - Extract `computeContentHash()` to crypto utility with proper hashing
   - Split into multiple files: engine, orchestration, aggregation

2. **temporal-analysis.ts**
   - Extract risk calculation to separate function
   - Extract pattern detectors to react-helpers

3. **dataflow-analysis.ts**
   - Extract prop drilling algorithm to separate function
   - Extract transform chain analysis to separate function

4. **react-helpers.ts**
   - Split into multiple files: component-helpers, hook-helpers, jsx-helpers, type-helpers
   - Reduce from 685 LOC to 4 files of ~170 LOC each

## Conclusion

The complexity analysis reveals:

- **3 files with critical complexity** (>30 cyclomatic) without adequate tests
- **63% test coverage gap** across core and React analyzers
- **~405 additional tests needed** for comprehensive coverage
- **temporal-analysis.ts is the highest-risk file** (complexity 35, no tests, critical functionality)
- **analysis-engine.ts is the most complex file** (complexity 50, 781 LOC, partial tests)

Recommended approach: **Follow the 3-sprint plan in testing-priority-checklist.md** to systematically address these gaps.
