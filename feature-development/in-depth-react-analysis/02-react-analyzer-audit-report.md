---
id: 02-react-analyzer-audit-report
---

# React Analyzer Audit Report

**Date**: 2026-01-25
**Auditor**: react-static-analysis-auditor
**Scope**: @vipr/react analyzer implementation vs. research findings
**Status**: Multi-phase improvement plan created

## Executive Summary

The React analyzer provides foundational coverage of common anti-patterns and performance issues but has significant opportunities for sophistication improvements. This audit compares the current implementation against the comprehensive anti-pattern research documented in `01-react-anti-patterns-research.md` and identifies gaps where detection is either missing, naive, or could be significantly enhanced through deeper semantic analysis.

### Overall Assessment

| Category                  | Current State       | Target State                | Gap Level    |
| ------------------------- | ------------------- | --------------------------- | ------------ |
| Performance Anti-Patterns | 60% Coverage, Naive | 95% Coverage, Sophisticated | **HIGH**     |
| Maintainability Issues    | 40% Coverage        | 90% Coverage                | **CRITICAL** |
| Hook Misuse Detection     | 50% Coverage, Basic | 95% Coverage, Semantic      | **HIGH**     |
| Reliability Concerns      | 30% Coverage        | 85% Coverage                | **CRITICAL** |
| Modern React (19+)        | 60% Coverage        | 90% Coverage                | **MEDIUM**   |

## Detailed Findings

### 1. Performance Anti-Patterns (⚡)

#### 1.1 Key Prop Issues

**Current**: ✅ Detects array index as key
**Gap**: Missing detection of:

- Random/unstable keys (Math.random(), Date.now(), UUID generated in render)
- Missing keys entirely in list rendering
- Non-unique keys (using fields that might duplicate)
- Keys that don't match data identity

**Example of undetected issue**:

```tsx
// Currently NOT detected
items.map(item => <div key={Math.random()}>{item.name}</div>);
items.map(item => <div key={item.type}>{item.name}</div>); // type not unique
```

#### 1.2 Inline Function/Component Detection

**Current**: ✅ Detects inline functions in props, nested component definitions
**Gap**: Could enhance with:

- Detection of components defined in renderX helper functions
- Identification of HOC patterns that recreate components
- Better filtering of legitimate render props vs anti-patterns

#### 1.3 Heavy Calculations During Render

**Current**: ⚠️ **NAIVE** - Only checks for expensive array methods on dynamic data
**Gap**: Missing:

- Nested loop detection (O(n²) complexity)
- Recursive function calls in render
- Complex object/array transformations
- Heavy regex operations
- Large data set processing
- Synchronous file/storage operations

**Example of undetected issue**:

```tsx
// Currently NOT detected
function Component({ data }) {
  // Nested loops in render - O(n²)
  const processed = data.map(item => otherData.filter(other => other.id === item.id));

  // Complex recursive transformation
  const nested = deeplyNestData(data);

  return <div>{processed.length}</div>;
}
```

**Improvement Strategy**: Implement AST-based complexity detection:

- Track nested loops within component render
- Detect recursive calls
- Measure transformation chain depth
- Identify synchronous blocking operations

#### 1.4 Context Anti-Patterns

**Current**: ⚠️ Detects inline context values only
**Gap**: Missing:

- Multiple context providers in single component (overuse indicator)
- Contexts with frequent updates (should be split)
- Contexts consumed in many components (high coupling)
- Context used for non-global data (should be props/composition)

**Example of undetected issues**:

```tsx
// Currently NOT detected
function App() {
  // Multiple contexts = overuse signal
  return (
    <ThemeContext.Provider value={theme}>
      <UserContext.Provider value={user}>
        <SettingsContext.Provider value={settings}>
          <DataContext.Provider value={data}>{children}</DataContext.Provider>
        </SettingsContext.Provider>
      </UserContext.Provider>
    </ThemeContext.Provider>
  );
}
```

**Improvement Strategy**:

- Count context providers per component
- Track context consumer count across codebase
- Analyze context value update frequency patterns
- Suggest context splitting when value contains unrelated data

#### 1.5 Unstable Props Detection

**Current**: ⚠️ Detects inline objects/functions in props
**Gap**: Missing:

- Object/array created outside JSX but still in render (unstable)
- Props that reference unstable parent state
- Callback props that close over unstable references

**Example of undetected issue**:

```tsx
function Parent() {
  const [items, setItems] = useState([]);

  // Creates new object every render - NOT detected
  const config = { items, onChange: setItems };

  return <Child config={config} />;
}
```

### 2. Maintainability & Complexity Pitfalls (🧩)

#### 2.1 Prop Drilling Detection

**Current**: ⚠️ **NAIVE** - Counts JSX nesting depth only
**Gap**: Missing:

- Detection of props that are passed through without being used
- Identification of "router components" (accept many props, use few)
- Measurement of actual prop threading depth
- Suggestion of appropriate solutions (context, composition, co-location)

**Example of undetected issue**:

```tsx
// Currently does NOT detect this as prop drilling
function GrandParent({ userId }) {
  return <Parent userId={userId} />;
}

function Parent({ userId }) {
  // userId not used here, just forwarded
  return <Child userId={userId} />;
}

function Child({ userId }) {
  // userId not used here, just forwarded
  return <GrandChild userId={userId} />;
}

function GrandChild({ userId }) {
  // Finally used here!
  return <div>{userId}</div>;
}
```

**Improvement Strategy**:

- Build symbol table of prop usage per component
- Detect props that are received but only passed down (not used locally)
- Calculate drilling score: (number of pass-through components) × (drilling depth)
- Suggest context when 3+ components drill same prop
- Suggest composition patterns for co-locating

#### 2.2 Context Overuse

**Current**: ❌ **NOT DETECTED**
**Status**: No analysis for context overuse patterns

**Needed**: Comprehensive context overuse detection
**Improvement Strategy**: See section 1.4 above

#### 2.3 God Components

**Current**: ⚠️ Partially covered (excessive useState detection)
**Gap**: Missing:

- Lines of code threshold
- Responsibility count (SRP violation)
- Complexity metrics (cyclomatic, JSX nesting)
- Mixed concerns detection (data + UI + business logic)

**Example of undetected issue**:

```tsx
// 300+ line component doing everything - NOT flagged as problematic
function Dashboard({ userId }) {
  // Data fetching
  const [user, setUser] = useState();
  const [posts, setPosts] = useState();
  const [comments, setComments] = useState();

  useEffect(() => { /* fetch user */ }, []);
  useEffect(() => { /* fetch posts */ }, [user]);
  useEffect(() => { /* fetch comments */ }, [posts]);

  // Form handling
  const [formData, setFormData] = useState({});
  const handleChange = (e) => { /* ... */ };
  const handleSubmit = (e) => { /* ... */ };

  // Business logic
  const processedData = /* complex transformation */;
  const validationErrors = /* validation logic */;

  // 200 lines of JSX rendering
  return ( /* ... massive JSX ... */ );
}
```

**Improvement Strategy**:

- Measure component LOC and flag >150 lines
- Count distinct responsibilities (data fetch, form, render, effects)
- Calculate JSX complexity (nesting, conditionals)
- Detect mixed business logic and rendering
- Suggest extraction into custom hooks and sub-components

#### 2.4 Business Logic in Components

**Current**: ❌ **NOT DETECTED**
**Status**: No detection of embedded business logic

**Needed**: Detection of:

- Complex calculations in component body
- Data transformation logic (should be pure functions)
- Validation logic (should be extracted)
- Formatting/parsing (should be utilities)

**Example**:

```tsx
function OrderForm({ items }) {
  // Complex business logic embedded in component - NOT detected
  const calculateTotal = () => {
    return items.reduce((sum, item) => {
      const basePrice = item.price * item.quantity;
      const discount = item.discount || 0;
      const tax = basePrice * 0.08;
      const shipping = item.weight > 10 ? 15 : 5;
      return sum + (basePrice - discount + tax + shipping);
    }, 0);
  };

  return <div>Total: ${calculateTotal()}</div>;
}
```

**Improvement Strategy**:

- Detect complex function declarations inside components
- Identify calculations with multiple steps/branches
- Flag validation/parsing/formatting logic
- Suggest extraction into pure utility functions

#### 2.5 Duplicated Code

**Current**: ❌ **NOT DETECTED** (single-file analyzer)
**Status**: Cannot detect cross-file duplication

**Note**: This requires cross-file analysis which is beyond single-file analyzer scope. Could be addressed at engine/project level.

#### 2.6 Poor State Management

**Current**: ⚠️ Basic detection (derived state in effect, props in initial state)
**Gap**: Missing:

- Redundant state (one state always derives from another)
- State that could be computed from props
- State used in only one place (should be local)
- Complex state that should use useReducer

**Example of undetected issue**:

```tsx
function Component({ items }) {
  const [items, setItems] = useState(props.items); // DETECTED
  const [count, setCount] = useState(0);
  const [isEmpty, setIsEmpty] = useState(true); // NOT detected - derives from count

  useEffect(() => {
    setIsEmpty(count === 0); // Redundant state update
  }, [count]);

  return <div>{isEmpty ? 'Empty' : count}</div>;
}
```

**Improvement Strategy**:

- Track state variables and their update locations
- Identify state only written in effects (likely derived)
- Detect state that's a pure function of props/other state
- Suggest direct computation or useMemo instead

#### 2.7 Excessive Hook Usage

**Current**: ✅ Threshold-based detection
**Status**: Basic coverage adequate

### 3. React Hooks Misuse & Anti-Patterns (🔄)

#### 3.1 Overusing useEffect for State Synchronization

**Current**: ⚠️ Basic detection (setState-only effects)
**Gap**: Missing:

- Effects that derive one state from another
- Effects that should be in event handlers
- Effects performing synchronous state updates

**Example of undetected issue**:

```tsx
function Search({ query }) {
  const [results, setResults] = useState([]);

  // Should be direct computation, NOT effect - not detected
  useEffect(() => {
    const filtered = data.filter(item => item.name.includes(query));
    setResults(filtered);
  }, [query]);

  // Should be in onClick handler, NOT effect - not detected
  useEffect(() => {
    if (buttonClicked) {
      setModalOpen(true);
    }
  }, [buttonClicked]);
}
```

**Improvement Strategy**:

- Detect effects with only setState calls (no side effects)
- Identify synchronous computations in effects
- Flag effects responding to event-like state changes
- Suggest direct computation or event handler placement

#### 3.2 Missing Effect Cleanup

**Current**: ⚠️ Partial detection (setTimeout, setInterval, addEventListener)
**Gap**: Missing detection of:

- WebSocket cleanup
- fetch/axios cancellation (AbortController)
- Observer cleanup (IntersectionObserver, ResizeObserver, MutationObserver)
- Third-party library subscriptions
- Custom event emitter cleanup

**Example of undetected issues**:

```tsx
useEffect(() => {
  // WebSocket - NOT detected
  const ws = new WebSocket('...');
  ws.onmessage = handleMessage;
  // Missing: return () => ws.close();
}, []);

useEffect(() => {
  // Fetch - NOT detected
  fetch('/api/data').then(setData);
  // Missing: AbortController cleanup
}, []);

useEffect(() => {
  // IntersectionObserver - NOT detected
  const observer = new IntersectionObserver(callback);
  observer.observe(ref.current);
  // Missing: return () => observer.disconnect();
}, []);
```

**Improvement Strategy**:

- Expand detection to cover all async/subscription patterns
- Check for AbortController usage with fetch
- Detect observer patterns
- Verify cleanup actually cancels the subscription
- Suggest specific cleanup for each pattern type

#### 3.3 Stale Closures & Dependency Issues

**Current**: ⚠️ Basic detection (missing dependencies via simple variable tracking)
**Gap**: Missing:

- Object/array literals causing infinite loops
- Unstable references in dependencies
- Dependencies that should be refs
- Effects/callbacks closing over stale values

**Example of undetected issues**:

```tsx
useEffect(() => {
  doSomething(user.id);
  // Missing dependency: user.id - may be detected
}, []);

// Infinite loop - NOT detected
useEffect(() => {
  const config = { userId }; // Creates new object every time
  fetchData(config);
}, [{ userId }]); // Object literal in deps = new reference

// Stale closure - NOT detected
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // Always logs 0 (stale)
  }, 1000);
  return () => clearInterval(id);
}, []); // Should have [count] or use setCount(c => c + 1)
```

**Improvement Strategy**:

- Build comprehensive symbol table of component scope
- Track variable types (ref, state, prop, const, function)
- Detect object/array literals in dependency arrays
- Identify closure captures and compare with deps
- Handle stable references correctly (refs, setState, imports)
- Suggest fixes: add missing deps, use functional updates, memoize objects

#### 3.4 Derived State Anti-Pattern

**Current**: ✅ Detects setState-only effects
**Status**: Basic coverage adequate, but see 3.1 for enhancements

#### 3.5 Excessive Memoization

**Current**: ⚠️ Detects some unnecessary memoization (primitives, simple functions)
**Gap**: Missing:

- Premature optimization (memoizing before profiling)
- Memoization that doesn't actually prevent re-renders
- useMemo/useCallback with wrong/incomplete deps

**Example of undetected issue**:

```tsx
// Premature optimization - NOT detected
function SimpleComponent({ name }) {
  // Memoizing trivial operation
  const uppercased = useMemo(() => name.toUpperCase(), [name]);

  // This useCallback doesn't help - NOT detected
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  // Child is not memoized, so this useCallback is pointless
  return <NonMemoChild onClick={handleClick} />;
}
```

**Improvement Strategy**:

- Detect memoization of trivial operations (single property access, simple arithmetic)
- Check if memoized values are actually used in dependencies
- Verify child components are memoized when passing memoized callbacks
- Suggest profiling before memoization
- Calculate memoization effectiveness score

#### 3.6 useLayoutEffect Misuse

**Current**: ⚠️ Basic detection (warns if no DOM measurement APIs)
**Status**: Adequate coverage

#### 3.7 Missing Error Boundaries

**Current**: ⚠️ Only checks Suspense components
**Gap**: Missing:

- General error boundary coverage analysis
- Async operation error boundaries
- Component tree risk assessment

**Improvement Strategy**:

- Track error boundary presence in file
- Flag components with async operations lacking boundaries
- Suggest error boundary placement for high-risk areas

### 4. Reliability Concerns (🛠️)

#### 4.1 Direct State Mutation

**Current**: ⚠️ **NAIVE** - Uses variable name heuristics
**Gap**: Need semantic analysis:

- Track actual useState declarations
- Detect mutations on state arrays (.push, .pop, .splice, .sort)
- Detect property assignments on state objects
- Use type information when available

**Example showing current naive detection**:

```tsx
function Component() {
  const [users, setUsers] = useState([]);
  const [config, setConfig] = useState({});

  // Currently NOT detected (doesn't contain "state" in name)
  users.push(newUser); // MUTATION!
  config.theme = 'dark'; // MUTATION!

  // Currently MAY be detected (has "state" in name)
  const myState = {};
  myState.value = 5; // False positive - not React state
}
```

**Improvement Strategy**:

- Build map of useState destructuring assignments
- Track state variable names precisely
- Detect mutation methods on tracked state arrays
- Detect property assignments on tracked state objects
- Reduce false positives from non-state variables

#### 4.2 Non-Idiomatic Class Patterns

**Current**: ❌ **NOT DETECTED**
**Status**: No detection of legacy pattern carryover

**Needed**:

- Manual componentWillMount-style patterns in hooks
- ref.current forced re-renders
- Lifecycle method thinking in hooks

#### 4.3 Side Effects in Render

**Current**: ❌ **NOT DETECTED**
**Status**: No detection of impure render functions

**Needed**: Detect:

- setState calls during render
- DOM manipulation during render (document.getElementById, etc.)
- Network requests in component body
- console.log/debugging (low severity)
- localStorage/sessionStorage access
- Date.now() calls (non-deterministic)

**Example**:

```tsx
function Component({ userId }) {
  // All NOT detected:

  // setState in render
  if (userId) {
    setUser(userId); // Infinite loop!
  }

  // DOM manipulation
  document.title = `User ${userId}`;

  // Network request
  fetch(`/api/user/${userId}`);

  // Non-deterministic
  const timestamp = Date.now();

  return <div>{timestamp}</div>;
}
```

**Improvement Strategy**:

- Detect function calls in component body (not in useEffect/handlers)
- Flag specific APIs: setState, document, window, fetch, Date.now
- Track render phase vs effect phase
- Suggest moving to useEffect or event handlers

#### 4.4 Accessibility Issues

**Current**: ✅ Has dedicated accessibility analyzer
**Status**: Adequate, but integration with anti-patterns could improve

#### 4.5 Security Issues

**Current**: ✅ Has dedicated security analyzer
**Status**: Adequate coverage

## Anti-Pattern Coverage Matrix

| Anti-Pattern           | Research Priority | Current Status | Detection Quality | Gap Severity |
| ---------------------- | ----------------- | -------------- | ----------------- | ------------ |
| **Performance**        |                   |                |                   |              |
| Poor Keys              | HIGH              | Partial        | Naive             | HIGH         |
| Inline Functions       | HIGH              | Good           | Good              | LOW          |
| Heavy Calculations     | HIGH              | Partial        | Naive             | **CRITICAL** |
| Context Misuse         | MEDIUM            | Partial        | Naive             | HIGH         |
| Unstable Props         | MEDIUM            | Partial        | Basic             | MEDIUM       |
| **Maintainability**    |                   |                |                   |              |
| Prop Drilling          | HIGH              | Partial        | **Naive**         | **CRITICAL** |
| Context Overuse        | MEDIUM            | None           | N/A               | HIGH         |
| God Components         | HIGH              | Partial        | Basic             | **CRITICAL** |
| Business Logic         | MEDIUM            | None           | N/A               | HIGH         |
| Redundant State        | MEDIUM            | Partial        | Basic             | MEDIUM       |
| **Hooks**              |                   |                |                   |              |
| useEffect Overuse      | HIGH              | Partial        | Basic             | HIGH         |
| Missing Cleanup        | HIGH              | Partial        | Basic             | HIGH         |
| Stale Closures         | HIGH              | Partial        | **Naive**         | **CRITICAL** |
| Derived State          | HIGH              | Good           | Good              | LOW          |
| Excessive Memo         | MEDIUM            | Partial        | Basic             | MEDIUM       |
| useLayoutEffect        | LOW               | Good           | Good              | LOW          |
| **Reliability**        |                   |                |                   |              |
| State Mutation         | HIGH              | Partial        | **Naive**         | **CRITICAL** |
| Side Effects in Render | MEDIUM            | None           | N/A               | HIGH         |
| Class Patterns         | LOW               | None           | N/A               | LOW          |

## Implementation Gaps Summary

### Critical Gaps (Must Fix)

1. **State Mutation Detection** - Current naive approach has false positives/negatives
2. **Heavy Calculation Detection** - Missing nested loops, recursion, complex operations
3. **Prop Drilling Intelligence** - Current depth-only approach misses actual drilling
4. **God Component Detection** - No LOC threshold, responsibility analysis
5. **Stale Closure Analysis** - Missing object literal deps, comprehensive variable tracking

### High Priority Gaps (Should Fix)

1. **Context Overuse Detection** - Completely missing
2. **Business Logic Detection** - No analysis of embedded logic
3. **useEffect Overuse** - Basic detection, needs semantic analysis
4. **Comprehensive Cleanup Detection** - Missing many async patterns
5. **Side Effects in Render** - Completely missing

### Medium Priority Gaps (Nice to Have)

1. **Memoization Effectiveness** - Detect premature optimization
2. **Unstable Props** - Detect object creation outside JSX
3. **Error Boundary Coverage** - Expand beyond Suspense
4. **Redundant State Detection** - Identify state deriving from other state

## Recommended Implementation Order

The multi-phase plan in the task list provides a structured approach:

### **Phase 1: Enhance Anti-Pattern Detection** (Weeks 1-2)

Focus on fixing naive implementations with semantic analysis:

- State mutation semantic tracking
- Key prop comprehensive analysis
- Heavy calculation detection (nested loops, recursion)
- Context anti-pattern detection
- Derived state improvements

### **Phase 2: Hook Dependency Analysis** (Weeks 3-4)

Sophisticated dependency array analysis:

- Stale closure semantic detection
- Dependency stability analysis
- Effect optimization detection
- Comprehensive cleanup validation

### **Phase 3: Component Complexity Analysis** (Weeks 5-6)

Add complexity metrics:

- Component size thresholds
- Responsibility analysis (SRP)
- JSX conditional complexity
- Business logic detection

### **Phase 4: Prop Drilling Intelligence** (Week 7)

Smart prop threading detection:

- Pass-through prop detection
- Prop usage analysis
- Solution suggestions

### **Phase 5: Performance Pattern Detection** (Weeks 8-9)

Advanced performance analysis:

- Re-render cascade detection
- Memoization effectiveness
- Render phase purity
- List rendering optimization

### **Phase 6: Modern React Patterns** (Week 10)

React 19+ analysis:

- Server/Client component validation
- Concurrent features
- Error boundary coverage
- Suspense best practices

### **Phase 7: Testing & Validation** (Week 11)

Comprehensive testing:

- Anti-pattern test cases
- Real-world validation
- Performance benchmarking
- Integration testing

### **Phase 8: Documentation** (Week 12)

Documentation and examples:

- Analysis documentation
- Metric descriptions
- Developer guide
- Code examples

## Success Metrics

### Coverage Metrics

- **Anti-Pattern Detection**: 60% → 95%
- **False Positive Rate**: &lt; 5%
- **False Negative Rate**: &lt; 10%

### Quality Metrics

- **Detection Sophistication**: Naive → Semantic
- **Actionability**: Provide specific fix suggestions
- **Performance**: &lt; 500ms per file analysis

### Validation Metrics

- **Community Alignment**: Match ESLint react-hooks findings
- **Real-World Accuracy**: Validate against known problematic repos
- **User Satisfaction**: Helpful insights, not noise

## References

1. Research Document: `01-react-anti-patterns-research.md`
2. Current Implementation: `analyzers/react/src/analyses/`
3. React Documentation: https://react.dev/
4. ESLint Plugin React Hooks: https://github.com/facebook/react/tree/main/packages/eslint-plugin-react-hooks

## Conclusion

The React analyzer has a solid foundation but significant opportunity for improvement through more sophisticated, semantic analysis. The naive implementations (particularly in state mutation, prop drilling, and heavy calculation detection) should be prioritized for enhancement. The phased approach ensures systematic improvement while maintaining backward compatibility and test coverage.

The end goal is to capture real-world, non-trivial React problems through intelligent static analysis, providing developers with actionable insights that improve code quality, performance, and maintainability.
