---
id: 11-react-analyzer-gap-analysis-and-phased-plan
---

# React Analyzer Gap Analysis and Phased Implementation Plan

**Date**: 2026-01-25

**Purpose**: Comprehensive audit of React analyses against sophisticated patterns documented in naive-vs-sophisticated.md, with prioritized, iterative implementation phases.

## Executive Summary

The current React analyzer implementation includes substantial analysis capabilities but relies primarily on **syntactic pattern matching** rather than **semantic analysis**. This audit identifies gaps where sophisticated analysis (as defined in the reference documents) would reduce false positives and provide actionable, high-value insights.

**Key Findings**:

- **14 major gaps** identified across 4 pattern categories
- **3 areas of naive implementation** requiring semantic enhancement
- **Estimated total effort**: 8-10 weeks across 7 phases
- **Highest ROI**: Effect dependency analysis, memoization necessity analysis, index-as-key sophistication

## Gap Analysis by Pattern Category

### Category 1: useMemo/useCallback Opportunities

#### Current Implementation

**File**: `analyzers/react/src/analyses/performance-analysis.ts`

**Lines 1086-1135**: Naive useCallback detection

```typescript
private analyzeUseCallbackInstance(node: Node, ...): void {
  // Flags ALL simple callbacks as potentially unnecessary
  if (isSimple && referencedIdentifiers.length === 0) {
    unnecessary.push({
      type: 'useCallback',
      reason: 'Memoizing a simple callback...'
    });
  }
}
```

**Lines 308-401**: Inline function prop detection

```typescript
// Flags ALL inline functions without checking consumer memoization
if (Node.isArrowFunction(expr) || Node.isFunctionExpression(expr)) {
  if (attrName.startsWith('on') && /^on[A-Z]/.test(attrName)) {
    opportunities.push({
      type: 'add-useCallback',
      reason: `Inline callback in "${attrName}" prop`,
    });
  }
}
```

#### Gaps Identified

**Gap 1.1**: No consumer memoization analysis

- **Reference**: naive-vs-sophisticated.md lines 42-109
- **Impact**: High false positive rate - flags callbacks passed to unmemoized children
- **Sophistication needed**:
  - Trace prop to receiving component
  - Check if component wrapped in React.memo
  - Check if component uses prop in useMemo/useCallback deps
  - Only flag if memoized consumer exists

**Gap 1.2**: No render cost estimation

- **Reference**: naive-vs-sophisticated.md lines 132-176
- **Impact**: Suggests memoization for cheap operations (overhead > benefit)
- **Sophistication needed**:
  - JSX tree depth measurement
  - Child component counting
  - Computational complexity detection (loops, recursion)
  - Threshold-based recommendations

**Gap 1.3**: No parent re-render frequency analysis

- **Reference**: naive-vs-sophisticated.md lines 80-89
- **Impact**: Recommends optimization for rarely re-rendering parents
- **Sophistication needed**:
  - State update frequency heuristics
  - Hook dependency volatility analysis
  - Context consumer detection

**Effort Estimate**: 2-3 weeks
**Value**: Very High (eliminates most common false positives)

---

### Category 2: useEffect Dependencies

#### Current Implementation

**File**: `analyzers/react/src/analyses/anti-pattern-analysis.ts`

**Lines 212-283**: Basic stale closure detection

```typescript
// Finds referenced vars not in deps, but doesn't check stability
const referencedVars = findReferencedVariables(callback);
const missingDeps: string[] = [];
referencedVars.forEach(varName => {
  if (declaredDeps.has(varName)) return;
  if (isStableReference(varName, sourceFile)) return; // Basic check
  if (setterNames.has(varName)) return;
  missingDeps.push(varName);
});
```

**File**: `analyzers/react/src/utils/react-helpers.ts` (isStableReference)

Basic stability detection exists but is limited to syntactic patterns (variable names, module-level consts).

#### Gaps Identified

**Gap 2.1**: No library-aware stability analysis

- **Reference**: naive-vs-sophisticated.md lines 378-411
- **Impact**: Flags stable library exports (useNavigate, useDispatch) as missing deps
- **Sophistication needed**:
  - Known stable import registry (react-router, redux, zustand)
  - Import declaration tracking
  - Named export matching

**Gap 2.2**: No useReducer dispatch detection

- **Reference**: naive-vs-sophisticated.md lines 350-369
- **Impact**: Flags dispatch as missing dependency
- **Current**: Partial detection via `setterNames` but misses array destructuring patterns
- **Sophistication needed**:
  - Array binding pattern analysis
  - Type-based detection (dispatch function signature)

**Gap 2.3**: No ref.current vs ref object distinction

- **Reference**: naive-vs-sophisticated.md lines 371-376
- **Impact**: Incorrect guidance on when to include refs in deps
- **Sophistication needed**:
  - Property access expression analysis
  - Intent detection (ref.current intentionally excluded to avoid re-runs)

**Gap 2.4**: No "functional update" suggestion

- **Reference**: problematic-patterns.md lines 330-357
- **Impact**: Missing actionable fix for stale closure issues
- **Sophistication needed**:
  - Detect `setState(value + 1)` patterns
  - Suggest `setState(v => v + 1)` alternative
  - Identify when functional update eliminates dependency

**Effort Estimate**: 1.5-2 weeks
**Value**: Very High (prevents infinite loops and subtle bugs)

---

### Category 3: Expensive Renders / React.memo Candidates

#### Current Implementation

**File**: `analyzers/react/src/analyses/performance-analysis.ts`

**Lines 659-759**: Basic React.memo opportunity detection

```typescript
// Flags ALL components with props that aren't memoized
if (hasProps) {
  opportunities.push({
    type: 'add-memo',
    estimatedImpact: 'low',
    reason: `Component "${name}" receives props but is not wrapped in React.memo`,
  });
}
```

**Lines 1250-1284**: Simple component check

```typescript
// Only checks if component is single-expression
if (body && !Node.isBlock(body)) {
  unnecessary.push({
    type: 'memo',
    reason: 'Simple single-expression components may not benefit...',
  });
}
```

#### Gaps Identified

**Gap 3.1**: No props stability analysis

- **Reference**: naive-vs-sophisticated.md lines 496-522, 550-578
- **Impact**: Recommends memo for components receiving unstable props (memo never prevents re-renders)
- **Sophistication needed**:
  - Find all component usages
  - Analyze prop values for stability
  - Detect inline object/function props
  - Flag if props are `always-new`

**Gap 3.2**: No parent update pattern detection

- **Reference**: naive-vs-sophisticated.md lines 466-474
- **Impact**: Missing key decision factor - does parent re-render frequently with unchanged props?
- **Sophistication needed**:
  - Parent component state analysis
  - Update frequency estimation
  - Prop derivation tracking

**Gap 3.3**: No cost-benefit scoring

- **Reference**: naive-vs-sophisticated.md lines 447-548
- **Impact**: All recommendations treated equally
- **Sophistication needed**:
  - Positive factor scoring (expensive render, frequent parent updates, pure component)
  - Negative factor scoring (unstable props, simple component, leaf component)
  - Net score calculation with thresholds

**Effort Estimate**: 2-3 weeks
**Value**: High (actionable memo recommendations, reduced noise)

---

### Category 4: Index as Key Detection

#### Current Implementation

**File**: `analyzers/react/src/analyses/anti-pattern-analysis.ts`

**Lines 655-725**: Array index detection (syntactic)

```typescript
// Flags ALL uses of index parameter as key
if (params.length >= 2) {
  const indexParam = params[1];
  if (indexParamName === keyVarName) {
    // Check parent is .map()
    if (methodName === 'map') {
      antiPatterns.push({
        id: 'array-index-key',
        severity: 'warning',
        description: 'Array index used as React key',
      });
    }
  }
}
```

**Lines 728-783**: Unstable key detection (good!)

- Detects Math.random(), Date.now(), etc.
- Detects likely non-unique keys

#### Gaps Identified

**Gap 4.1**: No array mutation analysis

- **Reference**: naive-vs-sophisticated.md lines 636-700
- **Impact**: Flags index keys for static/append-only lists (false positives)
- **Sophistication needed**:
  - Symbol tracking for array variable
  - findReferences() to detect mutations
  - Method call detection (sort, reverse, splice, filter, unshift)
  - Reassignment pattern detection
  - Distinguish append-only (push) from reordering operations

**Gap 4.2**: No component statefulness analysis

- **Reference**: naive-vs-sophisticated.md lines 671-682, 702-751
- **Impact**: Warns about index keys for stateless components (false positives)
- **Sophistication needed**:
  - Rendered component detection from .map() callback
  - Check component for useState/useReducer/useRef
  - Check for uncontrolled inputs (defaultValue without value)
  - Only flag critical if both: reordering mutations AND stateful component

**Effort Estimate**: 1.5-2 weeks
**Value**: Medium (reduces false positives in common patterns)

---

## Existing Strengths (No Gaps)

The following patterns are **well-implemented** and do not require enhancement:

1. **Nested component definitions** (performance-analysis.ts:833-992)
   - Comprehensive detection with library pattern exclusions (Radix UI, Headless UI)
   - Render prop pattern recognition
   - Iterator callback exclusions

2. **Unstable dependency literals** (anti-pattern-analysis.ts:285-344)
   - Object/array literal detection in dependency arrays
   - Infinite loop risk detection

3. **Direct state mutations** (anti-pattern-analysis.ts:381-458)
   - Semantic state variable tracking via extractStateVariables
   - Property mutation detection
   - Array mutation method detection

4. **Effect cleanup patterns** (temporal-analysis.ts:95-250)
   - Comprehensive resource detection (timers, listeners, WebSockets, Observers)
   - Cleanup method validation

5. **Context provider inline values** (performance-analysis.ts:771-830)
   - Detects inline objects in Context.Provider value prop

6. **Server Component violations** (anti-pattern-analysis.ts:896-1057)
   - Detects client hooks in server components
   - Browser API usage detection

## Phased Implementation Plan

### Phase 1: Effect Dependency Sophistication (2 weeks)

**Goal**: Eliminate false positives in useEffect dependency detection

**Deliverables**:

1. Library-aware stability registry
   - Create `STABLE_LIBRARY_EXPORTS` constant mapping
   - Implement import declaration tracking
   - Match named imports against known-stable exports
2. Enhanced useReducer dispatch detection
   - Array binding pattern analysis for `const [_, dispatch] = useReducer(...)`
   - Type signature detection
3. ref.current vs ref distinction
   - Property access expression analysis
   - Context-aware ref dependency guidance
4. Functional update suggestions
   - Detect `setState(stateVar + ...)` patterns
   - Provide `setState(s => s + ...)` fix suggestion

**Files to modify**:

- `analyzers/react/src/utils/react-helpers.ts`: Add `isStableLibraryImport()`, `isUseReducerDispatch()`
- `analyzers/react/src/analyses/anti-pattern-analysis.ts`: Enhance dependency checking (lines 212-283)
- `analyzers/react/src/constants/stable-exports.ts`: New file with registry

**Testing**:

- Add test fixtures with react-router, redux, zustand
- Test ref.current exclusion scenarios
- Test functional update detection

**Success Criteria**:

- Zero false positives for known stable imports
- Correct guidance on ref dependencies
- Actionable functional update suggestions

---

### Phase 2: Memoization Necessity Analysis (3 weeks)

**Goal**: Sophisticated useCallback/useMemo recommendations based on consumer analysis and render cost

**Deliverables**:

1. Consumer memoization detection
   - Component definition finder (across files)
   - React.memo wrapper detection
   - Dependency array prop usage analysis
2. Render cost estimation
   - JSX tree depth measurement
   - Child component counting
   - Computational complexity detection (loops, recursion, array methods)
   - Scoring algorithm (0-100)
3. Parent re-render frequency heuristics
   - State count in parent
   - Hook dependency volatility
   - Context consumption
4. Cost-benefit recommendation engine
   - Combine consumer memoization + render cost + parent frequency
   - Graduated confidence levels (high/medium/low)
   - Threshold-based flagging

**Files to modify**:

- `analyzers/react/src/analyses/performance-analysis.ts`: Enhance useCallback/useMemo analysis (lines 308-401, 1086-1135)
- `analyzers/react/src/utils/component-graph.ts`: New file for cross-component analysis
- `analyzers/react/src/utils/render-cost.ts`: New file for cost estimation

**Testing**:

- Fixtures with memoized vs unmemoized children
- Fixtures with expensive vs cheap components
- Fixtures with high/low parent update frequency

**Success Criteria**:

- No false positives for callbacks passed to unmemoized children
- Accurate render cost scoring
- Actionable memoization recommendations with reasoning

---

### Phase 3: React.memo Candidate Sophistication (2 weeks)

**Goal**: Accurate React.memo recommendations based on props stability and component characteristics

**Deliverables**:

1. Props stability analysis
   - Find all component usages (jsx tag matching)
   - Analyze prop value expressions for stability
   - Detect inline object/function props
   - Calculate stability score
2. Parent update pattern detection
   - Parent state/hook analysis
   - Update frequency estimation
3. Positive/negative factor scoring
   - Positive: expensive render, frequent parent updates, pure component
   - Negative: unstable props, simple component, leaf component, always-new props
   - Net score calculation
4. Graduated recommendations
   - High confidence (net > 40): strong recommendation
   - Medium confidence (net 20-40): suggest investigation
   - Low confidence (net < 20): mention possibility

**Files to modify**:

- `analyzers/react/src/analyses/performance-analysis.ts`: Enhance memo detection (lines 659-759)
- `analyzers/react/src/utils/props-stability.ts`: New file for props analysis
- `analyzers/react/src/utils/component-graph.ts`: Extend with parent-child relationships

**Testing**:

- Fixtures with stable vs unstable props
- Fixtures with expensive vs cheap components
- Fixtures with pure vs stateful components

**Success Criteria**:

- No recommendations for components with always-new props
- Accurate confidence levels
- Reasoning clearly explains positive/negative factors

---

### Phase 4: Index-as-Key Sophistication (2 weeks)

**Goal**: Context-aware index key warnings based on array mutations and component statefulness

**Deliverables**:

1. Array mutation tracking
   - Symbol.findReferences() to track array variable
   - Detect mutating method calls (sort, reverse, splice, filter, unshift)
   - Detect reassignment with filtering/sorting
   - Classify as append-only (push only) vs reordering
2. Component statefulness detection
   - Extract rendered component from .map() callback
   - Check for useState/useReducer/useRef
   - Check for uncontrolled inputs (defaultValue without value)
3. Risk-based severity
   - Low/info: static or append-only + stateless component
   - High/critical: reordering mutations + stateful component

**Files to modify**:

- `analyzers/react/src/analyses/anti-pattern-analysis.ts`: Enhance index key detection (lines 655-725)
- `analyzers/react/src/utils/array-mutation-tracker.ts`: New file for mutation analysis
- `analyzers/react/src/utils/react-helpers.ts`: Add `findRenderedComponentInMap()`, `componentHasState()`

**Testing**:

- Fixtures with static arrays, append-only, reordering mutations
- Fixtures with stateful vs stateless components
- Fixtures with uncontrolled inputs

**Success Criteria**:

- No warnings for static/append-only arrays with stateless components
- Critical severity only for truly problematic cases
- Reasoning explains mutation pattern and statefulness

---

### Phase 5: Enhanced Hook Dependency Analysis (1 week)

**Goal**: Improve existing dependency analysis with additional semantic checks

**Deliverables**:

1. Object property dependency detection
   - Detect `props.field` in dependencies
   - Warn if entire object not stable
2. Function dependency detection
   - Detect function identifiers in deps
   - Check if memoized with useCallback
3. Dependency change frequency heuristics
   - Primitive deps = stable
   - Object deps without memo = volatile
   - Scoring for effect re-run frequency

**Files to modify**:

- `analyzers/react/src/analyses/temporal-analysis.ts`: Enhance dependency analysis (lines 311-342)
- `analyzers/react/src/utils/react-helpers.ts`: Add dependency classification helpers

**Testing**:

- Fixtures with object property dependencies
- Fixtures with function dependencies

**Success Criteria**:

- Warns about unmemoized object dependencies
- Identifies high-frequency re-run effects

---

### Phase 6: Cross-File Component Graph (1.5 weeks)

**Goal**: Build component relationship graph for advanced analysis

**Deliverables**:

1. Component graph builder
   - Find all components across project
   - Extract child component references from JSX
   - Build parent-child relationships
   - Detect memoization status
2. Prop flow tracking
   - Track props passed from parent to child
   - Detect prop drilling (3+ levels)
   - Identify pass-through components
3. Integration with existing analyses
   - Use graph in memoization analysis
   - Use graph in memo candidate detection

**Files to modify**:

- `analyzers/react/src/utils/component-graph.ts`: New comprehensive graph builder
- `analyzers/react/src/analyses/performance-analysis.ts`: Integrate graph
- `analyzers/react/src/analyses/coupling-analysis.ts`: Use for prop drilling detection

**Testing**:

- Multi-file component fixtures
- Prop drilling scenarios
- Complex component hierarchies

**Success Criteria**:

- Accurate parent-child relationships
- Prop flow tracking across multiple levels
- Memoization status correctly detected

---

### Phase 7: Documentation and Refinement (1 week)

**Goal**: Document new capabilities and refine based on real-world testing

**Deliverables**:

1. Update metric documentation
   - New insights and their meanings
   - Confidence level interpretation
   - Cost-benefit reasoning
2. Add examples to fixtures
   - Sophisticated pattern examples
   - False positive prevention examples
3. Refinement based on testing
   - Tune thresholds
   - Adjust confidence calculations
   - Fix edge cases

**Files to modify**:

- `documentation/docs/analyzers/react.md`: Document new capabilities
- `packages/fixtures/src/react/`: Add sophisticated pattern examples
- Various analyzer files: Threshold tuning

**Success Criteria**:

- Clear documentation of all new analyses
- Examples demonstrate sophistication
- Thresholds tuned for minimal false positives

---

## Implementation Guidelines

### Conformity Requirements

All implementations must:

1. **Use existing types and interfaces**
   - `IAnalysis<Config, ComplexityData>`
   - `AnalysisResult<ComplexityData>`
   - `ComplexityInsight`
   - Do not create new top-level analysis types

2. **Follow plugin architecture**
   - Analyses live in `analyzers/react/src/analyses/`
   - Helpers in `analyzers/react/src/utils/`
   - Wire up through existing analysis classes
   - Report through existing presenters

3. **Maintain scoring consistency**
   - Use `roundScore()` from `@vipr/common`
   - Follow existing weight/threshold patterns
   - Document scoring rationale

4. **Testing requirements**
   - Add test fixtures in `packages/fixtures/src/react/`
   - Update corresponding `.test.ts` files
   - Test both positive and negative cases

5. **CLI and VSCode integration**
   - Insights automatically appear in CLI formatters
   - Insights automatically appear in VSCode problems panel
   - No special wiring needed (uses existing infrastructure)

### ts-morph Patterns to Use

Refer to `/Users/jamesleebaker/Codespace/vipr/documentation/docs/feature-development/in-depth-react-analysis/01c-tsmorph-analysis.md` for:

- Symbol.findReferences() for mutation tracking
- Type-aware analysis patterns
- Data flow tracing
- Control flow analysis
- Cross-file analysis with Project API

### Avoiding Common Pitfalls

1. **Don't over-optimize prematurely**
   - Start with correctness, optimize later
   - Use caching for expensive operations

2. **Don't break existing analyses**
   - Run full test suite after each phase
   - Ensure backward compatibility

3. **Don't introduce new dependencies**
   - Use ts-morph APIs exclusively
   - No external AST tools

4. **Don't skip documentation**
   - Update docs as you implement
   - Add JSDoc comments for new functions

---

## Success Metrics

### Quantitative Goals

- **Reduce false positive rate by 60%** for memoization suggestions
- **Reduce false positive rate by 80%** for dependency warnings
- **Increase actionable insights ratio to >90%** (insights user can act on without profiler)
- **Zero regressions** in existing test suite

### Qualitative Goals

- Users understand **why** the analysis flagged an issue
- Users receive **confidence levels** and can prioritize accordingly
- Users get **actionable suggestions**, not just warnings
- Analysis output is **credible** - users trust the analyzer

---

## Risk Mitigation

### Risk 1: Complexity Explosion

**Mitigation**:

- Phased approach allows evaluation after each phase
- Can stop/adjust if complexity becomes unmaintainable
- Use helper functions to keep analysis logic readable

### Risk 2: Performance Degradation

**Mitigation**:

- Benchmark after each phase
- Use caching for expensive operations (component graph, symbol lookups)
- Execution cost rating guides user expectations

### Risk 3: False Negatives

**Mitigation**:

- Comprehensive test fixtures
- Prefer false negatives over false positives (sophisticated analysis should be conservative)
- Document known limitations

---

## Appendix: Reference Document Alignment

### Patterns Covered

| Pattern                       | Reference Doc Lines               | Phases Addressing |
| ----------------------------- | --------------------------------- | ----------------- |
| useMemo/useCallback necessity | naive-vs-sophisticated.md:5-188   | Phase 2           |
| useEffect dependencies        | naive-vs-sophisticated.md:190-424 | Phase 1, 5        |
| Expensive renders / memo      | naive-vs-sophisticated.md:426-606 | Phase 3           |
| Index as key                  | naive-vs-sophisticated.md:608-774 | Phase 4           |

### Analysis Techniques Required

| Technique           | Reference Doc Lines         | Implementation Phase |
| ------------------- | --------------------------- | -------------------- |
| Type-aware analysis | tsmorph-analysis.md:26-142  | Phase 1, 2           |
| Data flow tracing   | tsmorph-analysis.md:144-201 | Phase 2, 4           |
| Cross-file analysis | tsmorph-analysis.md:586-726 | Phase 6              |
| Symbol references   | tsmorph-analysis.md:203-246 | Phase 4              |

### Problematic Patterns Addressed

| Pattern                  | problematic-patterns.md Lines | Phase                 |
| ------------------------ | ----------------------------- | --------------------- |
| Premature memoization    | 7-56                          | Phase 2               |
| Unstable deps            | 57-106                        | Phase 1               |
| Missing cleanup          | 148-208                       | (Already implemented) |
| Derived state in effects | 210-246                       | (Already implemented) |
| Stale closures           | 326-357                       | Phase 1               |
| Index as key             | 387-411                       | Phase 4               |

---

## Conclusion

This phased approach provides:

1. **Iterative delivery** - Value delivered every 1-2 weeks
2. **Risk management** - Can stop/adjust based on results
3. **Focused effort** - Each phase has clear scope
4. **Measurable progress** - Concrete deliverables and success criteria
5. **Technical excellence** - Sophisticated analysis reduces noise

**Recommended starting point**: Phase 1 (Effect Dependency Sophistication)

- Highest impact on reducing false positives
- Foundation for later phases
- Relatively straightforward implementation
- Immediate user value

**Total estimated effort**: 8-10 weeks for full implementation
**Estimated value**: Very High - transforms analyzer from pattern matcher to semantic analysis tool
