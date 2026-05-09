# Problematic React Patterns: Research-Backed Analysis Requirements

This document catalogs the most impactful anti-patterns in React codebases, with specific guidance on what static analysis must understand to detect them accurately.

## Category 1: Memoization Anti-Patterns

### 1.1 Premature Memoization

**Pattern**: Applying `useMemo` or `useCallback` without measurable benefit

**Why naive detection fails**:

```tsx
// Naive rule: "Function passed as prop should use useCallback"
// This flags BOTH cases below identically:

// Case A: Genuinely problematic (causes child re-renders)
function Parent() {
  const handleClick = () => doExpensiveOperation();
  return <MemoizedExpensiveChild onClick={handleClick} />;
}

// Case B: Not problematic (child is cheap, no memoization chain)
function Parent() {
  const handleClick = () => setCount(c => c + 1);
  return <button onClick={handleClick}>Click</button>;
}
```

**What analysis must determine**:

1. **Consumer memoization status**
   - Is the receiving component wrapped in `React.memo`?
   - Does the receiving component use `useMemo`/`useCallback` on props?
2. **Render cost of consumer**
   - How deep is the subtree?
   - Does it perform expensive computations?
   - How many DOM nodes does it manage?

3. **Update frequency**
   - How often does the parent re-render?
   - Would memoization prevent meaningful re-renders?

**ts-morph analysis approach**:

```typescript
// Trace the prop to its consumer
const propUsages = findPropConsumers(functionNode, propName);
for (const usage of propUsages) {
  const component = getContainingComponent(usage);
  const isMemoized = hasReactMemoWrapper(component) || usesMemoDependingOnProp(component, propName);
  const renderCost = estimateRenderCost(component);
  // Only flag if memoized consumer AND significant render cost
}
```

### 1.2 Unstable useMemo Dependencies

**Pattern**: `useMemo` with dependencies that change every render, defeating memoization

```tsx
// Object created inline - new reference every render
const config = useMemo(
  () => processConfig(settings),
  [{ theme: 'dark', locale: 'en' }] // New object every render!
);

// Function created inline
const handler = useCallback(
  () => doSomething(value),
  [() => getValue()] // New function every render!
);
```

**What analysis must determine**:

1. **Dependency stability**
   - Is the dependency a primitive, stable reference, or recreated each render?
   - Track value origin through assignments and function parameters

2. **Object/array literal detection in deps**
   - Direct literals in dependency array
   - Variables assigned to literals in same render scope

**ts-morph analysis approach**:

```typescript
function analyzeDepStability(dep: Expression): Stability {
  // Primitives are stable
  if (isPrimitiveLiteral(dep)) return 'stable';

  // Check if identifier traces to stable source
  if (dep.isKind(SyntaxKind.Identifier)) {
    const definition = traceToDefinition(dep);
    if (isModuleScopeConst(definition)) return 'stable';
    if (isUseRefCurrent(definition)) return 'stable';
    if (isUseStateDispatch(definition)) return 'stable';
    if (isRenderScopeObjectLiteral(definition)) return 'unstable';
  }

  // Object/array literals are unstable
  if (isObjectOrArrayLiteral(dep)) return 'unstable';

  return 'unknown';
}
```

### 1.3 useMemo for Non-Expensive Computations

**Pattern**: Memoizing cheap operations where overhead exceeds benefit

```tsx
// Memoization overhead likely exceeds computation cost
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

// vs. genuinely expensive
const sortedAndFilteredData = useMemo(
  () => data.filter(predicate).sort(comparator).map(transform),
  [data, predicate, comparator, transform]
);
```

**Computation cost heuristics**:

| Operation                   | Cost Level | Memoize? |
| --------------------------- | ---------- | -------- |
| String concatenation        | Trivial    | No       |
| Simple arithmetic           | Trivial    | No       |
| Object property access      | Trivial    | No       |
| Array.map (small, &lt; 100) | Low        | Maybe    |
| Array.filter + sort         | Medium     | Usually  |
| Recursive computation       | High       | Yes      |
| Complex transformations     | High       | Yes      |
| Nested loops O(n²)          | High       | Yes      |

**What analysis must determine**:

1. **Computational complexity**
   - Loop detection and nesting level
   - Recursive call detection
   - Array method chains

2. **Data size inference**
   - Type annotations indicating array/collection size
   - Upstream data source analysis

## Category 2: Effect Anti-Patterns

### 2.1 Missing Effect Cleanup

**Pattern**: Effects that create subscriptions/timers without cleanup

```tsx
// Memory leak: interval never cleared
useEffect(() => {
  const id = setInterval(tick, 1000);
  // Missing: return () => clearInterval(id);
}, []);

// Memory leak: subscription never removed
useEffect(() => {
  eventEmitter.on('event', handler);
  // Missing: return () => eventEmitter.off('event', handler);
}, []);
```

**What analysis must determine**:

1. **Resource creation detection**
   - `setInterval`, `setTimeout` calls
   - `addEventListener` calls
   - Subscription method patterns (`on`, `subscribe`, `addListener`)
   - WebSocket/EventSource creation

2. **Cleanup pairing**
   - Does return function call corresponding cleanup?
   - Are the same identifiers used in setup and cleanup?

**ts-morph analysis approach**:

```typescript
const SETUP_CLEANUP_PAIRS = {
  setInterval: 'clearInterval',
  setTimeout: 'clearTimeout',
  addEventListener: 'removeEventListener',
  on: 'off',
  subscribe: 'unsubscribe',
};

function analyzeEffectCleanup(effectCallback: ArrowFunction) {
  const setupCalls = findCallsMatching(effectCallback, Object.keys(SETUP_CLEANUP_PAIRS));
  const returnStatement = effectCallback
    .getStatements()
    .find(s => s.isKind(SyntaxKind.ReturnStatement));

  if (!returnStatement && setupCalls.length > 0) {
    return { issue: 'missing-cleanup', setupCalls };
  }

  const cleanupFn = returnStatement?.getExpression();
  for (const setupCall of setupCalls) {
    const expectedCleanup = SETUP_CLEANUP_PAIRS[setupCall.name];
    if (!containsCall(cleanupFn, expectedCleanup)) {
      return { issue: 'incomplete-cleanup', missing: expectedCleanup };
    }
  }
}
```

### 2.2 useEffect for Derived State

**Pattern**: Using effects to synchronize state that could be computed

```tsx
// Anti-pattern: Effect for derived state
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter(item => item.active));
}, [items]);

// Correct: Compute during render
const [items, setItems] = useState([]);
const filteredItems = items.filter(item => item.active);
// Or with useMemo if expensive
const filteredItems = useMemo(() => items.filter(item => item.active), [items]);
```

**Detection criteria**:

- Effect body contains only setState call(s)
- setState value derived entirely from dependency values
- No side effects (API calls, DOM manipulation, subscriptions)

**What analysis must determine**:

1. **Effect body purity**
   - Does it only call setState?
   - Is the new state value a pure transformation of deps?

2. **Side effect absence**
   - No fetch/axios/API calls
   - No DOM manipulation
   - No console.log/external writes

### 2.3 Object/Function Dependencies Causing Infinite Loops

**Pattern**: Effects with unstable dependencies triggering themselves

```tsx
// Infinite loop: options recreated each render
function Component({ id }) {
  const options = { id, timestamp: Date.now() };

  useEffect(() => {
    fetchData(options);
  }, [options]); // options is new object every render!
}
```

**What analysis must determine**:

1. **Dependency stability** (same as 1.2)
2. **Self-triggering potential**
   - Does effect body cause re-render?
   - Does re-render recreate dependency?

## Category 3: State Management Anti-Patterns

### 3.1 Redundant State

**Pattern**: State that duplicates or derives from other state/props

```tsx
// Anti-pattern: State duplicating props
function UserProfile({ user }) {
  const [name, setName] = useState(user.name); // Redundant!
  // Now name can drift from user.name
}

// Anti-pattern: Derived state stored separately
function ItemList({ items }) {
  const [itemCount, setItemCount] = useState(items.length);
  // itemCount is always items.length - why store it?
}
```

**What analysis must determine**:

1. **Initial value source**
   - Is useState initialized from props?
   - Is the value computable from other state/props?

2. **Update patterns**
   - Is the state always updated when source changes?
   - Can the values diverge?

### 3.2 State Updates in Render

**Pattern**: Calling setState during render (outside effects/handlers)

```tsx
// Anti-pattern: setState in render body
function Counter({ value }) {
  const [count, setCount] = useState(0);

  if (value > count) {
    setCount(value); // Triggers re-render during render!
  }

  return <div>{count}</div>;
}
```

**What analysis must determine**:

1. **Call location**
   - Is setState called in component body (not in callback/effect)?
   - Is it conditionally called? (still problematic)

2. **Legitimate exceptions**
   - Lazy initialization patterns
   - Error boundary state

### 3.3 Stale Closure State

**Pattern**: Callbacks capturing stale state values

```tsx
// Stale closure: count captured at creation time
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // Always logs initial count!
      setCount(count + 1); // Always sets to 1!
    }, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps = closure never updates
}

// Fix: Use functional update
setCount(c => c + 1);
```

**What analysis must determine**:

1. **Closure capture analysis**
   - What variables does the callback close over?
   - Are closed-over values state/props that change?

2. **Dependency array completeness**
   - Are captured values in effect deps?
   - Would functional update avoid the issue?

## Category 4: Component Structure Anti-Patterns

### 4.1 Components Defined Inside Components

**Pattern**: Component definitions inside other components

```tsx
// Anti-pattern: InnerComponent recreated every render
function OuterComponent() {
  // New component identity each render = full remount
  function InnerComponent() {
    return <div>Inner</div>;
  }

  return <InnerComponent />;
}
```

**Why this is always wrong**:

- New component identity each render
- React unmounts and remounts, losing all state
- Performance disaster

**Detection**: Straightforward AST analysis - function declarations/expressions returning JSX inside component bodies.

### 4.2 Index as Key in Dynamic Lists

**Pattern**: Using array index as key when list can change

```tsx
// Problematic when items can be reordered/inserted/removed
{
  items.map((item, index) => <Item key={index} data={item} />);
}
```

**When it's actually fine**:

- Static lists that never change
- Lists only appended to (never reordered/filtered)
- No component state in list items

**What analysis must determine**:

1. **List dynamism**
   - Is the array ever filtered/sorted/reversed?
   - Are items inserted/removed from middle?

2. **Item component statefulness**
   - Does Item component have useState/useRef?
   - Does it have uncontrolled form inputs?

### 4.3 Prop Drilling Through Many Levels

**Pattern**: Props passed through many intermediate components unchanged

```tsx
// Prop drilling: theme passed through 5 components
<App>
  <Layout theme={theme}>
    <Sidebar theme={theme}>
      <Menu theme={theme}>
        <MenuItem theme={theme}>
          <Icon theme={theme} />
        </MenuItem>
      </Menu>
    </Sidebar>
  </Layout>
</App>
```

**What analysis must determine**:

1. **Drilling depth**
   - How many levels is the prop passed unchanged?
   - Threshold: typically 3+ levels is problematic

2. **Usage pattern**
   - Is prop used by intermediate components or just passed through?
   - Would Context be more appropriate?

## Category 5: Performance Anti-Patterns

### 5.1 Large Component Trees Without Boundaries

**Pattern**: State changes causing re-renders of large subtrees

```tsx
// One state change re-renders entire app
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState([]);
  const [cart, setCart] = useState([]);
  // ... many more states

  return (
    <div>
      <Header user={user} notifications={notifications} />
      <Sidebar theme={theme} />
      <MainContent cart={cart} />
      <Footer />
    </div>
  );
}
```

**What analysis must determine**:

1. **State update impact**
   - Which components depend on which state?
   - What is the re-render blast radius?

2. **Optimization opportunities**
   - Where would memo boundaries help?
   - Where should state be colocated?

### 5.2 Inline Object/Array Props

**Pattern**: Object/array literals passed as props causing unnecessary re-renders

```tsx
// New object every render, even if values unchanged
<Child style={{ color: 'red' }} />
<Child items={[1, 2, 3]} />
<Child config={{ theme, locale }} />
```

**When it matters**:

- Child is memoized (React.memo)
- Child does expensive operations based on these props
- Parent re-renders frequently

**When it doesn't matter**:

- Child is simple/cheap
- Parent rarely re-renders
- Props don't trigger expensive operations

### 5.3 Unnecessary Context Re-renders

**Pattern**: Context value changes causing all consumers to re-render

```tsx
// Every context consumer re-renders when ANY value changes
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  // New object every render!
  const value = { user, setUser, theme, setTheme };

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}
```

**Solutions to detect/suggest**:

- Split contexts by update frequency
- Memoize context value
- Use state management library

## Analysis Complexity Ratings

| Pattern                  | Detection Complexity | Accuracy Without Types | Accuracy With Types |
| ------------------------ | -------------------- | ---------------------- | ------------------- |
| Components in components | Low                  | High                   | High                |
| Missing effect cleanup   | Medium               | Medium                 | High                |
| Unstable deps            | High                 | Low                    | Medium              |
| Premature memoization    | Very High            | Very Low               | Medium              |
| Stale closures           | Very High            | Low                    | Medium              |
| Redundant state          | High                 | Low                    | Medium              |
| Prop drilling            | Medium               | Medium                 | High                |

## Implementation Priority

Based on impact and feasibility:

**High Priority (High impact, achievable)**:

1. Components defined inside components
2. Missing effect cleanup
3. Index as key in dynamic lists
4. setState in render

**Medium Priority (High impact, complex)**:

1. Unstable dependencies
2. useEffect for derived state
3. Redundant state

**Lower Priority (Needs profiling data)**:

1. Premature memoization
2. Inline object props
3. Context re-render optimization
