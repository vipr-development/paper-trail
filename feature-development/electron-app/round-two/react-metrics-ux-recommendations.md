---
id: react-metrics-ux-recommendations
title: UX Recommendations for React-Specific Metrics
phase: 2
dependencies:
  - 04-architectural-anti-patterns-detection
  - 07-dependency-cascade-analysis
  - 17-adaptive-visualizations-scale
  - 18-cognitive-load-halstead-heatmaps
---

# UX Recommendations for React-Specific Metrics

## Overview

Vipr tracks comprehensive React-specific metrics that go beyond standard complexity measurements. This document provides UX recommendations for presenting these metrics in ways that are accessible, actionable, and meaningful to React developers.

## React Metric Categories

### 1. Hook Complexity

- **Metrics**: Hook count, temporal complexity weight, custom hook usage
- **Challenges**: Developers need to understand hook composition costs without premature optimization
- **Key Insight**: Not all hooks are equal (useEffect is more complex than useState)

### 2. Temporal Complexity

- **Metrics**: Effect count, dependency tracking, cleanup patterns, mount/render timing
- **Challenges**: Temporal reasoning is cognitively demanding ("when does this run?")
- **Key Insight**: Missing cleanup and stale closures are common bugs

### 3. Coupling Complexity

- **Metrics**: Props count, context consumers, callback props, ref forwarding, children patterns
- **Challenges**: Coupling can be good (composition) or bad (tight coupling)
- **Key Insight**: Context depth and unstable values matter more than count

### 4. Identity Stability

- **Metrics**: useCallback/useMemo usage, unstable references, dependency complexity
- **Challenges**: Over-memoization is as bad as under-memoization
- **Key Insight**: Memoization only helps when downstream consumers check referential equality

### 5. Dataflow Complexity

- **Metrics**: Prop drilling depth/chains, state update paths, derived state, transform chains
- **Challenges**: Visualizing data flow through component trees
- **Key Insight**: Prop drilling depth matters more than breadth

### 6. Anti-Patterns

- **Metrics**: Conditional hooks, inline functions, missing dependencies, SRP violations
- **Challenges**: Distinguishing intentional patterns from mistakes
- **Key Insight**: Severity varies dramatically by context

---

## Document 04: Architectural AntiPatterns Detection

### React-Specific AntiPatterns to Present

#### 1. God Component (React Edition)

**Visual Treatment:**

```
God Component: Dashboard.tsx
================================================================================
Complexity: 82 | Imports: 24 | Concerns: 6 | Hooks: 15 | Props: 12

DETECTED CONCERNS:
 [📊 State Management]    useState: 8, useReducer: 1
 [⏱️  Effects]             useEffect: 4 (3 missing cleanup)
 [🌐 Data Fetching]        3 API calls, 2 cache hooks
 [🎨 Rendering]           847 LOC JSX, 6 conditional branches
 [📡 Context Consumers]   useContext: 3 (Auth, Theme, Data)
 [🔐 Authorization]       Role checks, permission guards

HOOK COMPLEXITY BREAKDOWN:
  useEffect (4×)         Weight: 16  "Temporal reasoning overhead"
  useState (8×)          Weight: 8   "Multiple state variables"
  useContext (3×)        Weight: 9   "Cross-cutting concerns"
  Custom hooks (2×)      Weight: 4   "Abstraction layers"
  Total Hook Weight: 37

TEMPORAL COMPLEXITY:
  4 effects, 3 missing cleanup
  2 effects run every render (empty deps)
  1 effect has 8 dependencies (stale closure risk)

RECOMMENDED REFACTORING:
  ├─ DashboardLayout.tsx           (UI structure, hooks: 2)
  ├─ useDashboardData.ts          (data fetching, hooks: 4)
  ├─ useDashboardAuth.ts          (auth state, hooks: 3)
  ├─ useDashboardAnalytics.ts     (tracking, hooks: 2)
  └─ DashboardWidgets.tsx         (widget rendering, hooks: 4)

Expected outcome: 5 files with 2-4 hooks each, clear responsibility
Temporal complexity reduced by 70% (isolated effects)

[View Hook Dependencies] [Generate Refactoring Prompt] [Mark as Accepted]
================================================================================
```

**Key UX Principles:**

- **Hook-aware scoring**: Traditional complexity doesn't capture React-specific overhead
- **Concern detection**: Identify responsibilities by hook patterns (data hooks vs UI hooks)
- **Temporal visibility**: Effects are the #1 source of bugs, show them prominently
- **Actionable splits**: Suggest specific extractions with expected hook counts

#### 2. Prop Drilling Chain Visualization

**Visual Treatment:**

```
Prop Drilling: theme, setTheme (5 levels deep)
================================================================================

App.tsx
  └─ theme, setTheme (passed to Layout)
      └─ Layout.tsx
          └─ theme, setTheme (forwarded to Dashboard)
              └─ Dashboard.tsx
                  └─ theme, setTheme (forwarded to Widget)
                      └─ Widget.tsx
                          └─ theme, setTheme (forwarded to WidgetHeader)
                              └─ WidgetHeader.tsx
                                  └─ theme, setTheme (forwarded to ThemeToggle)
                                      └─ ThemeToggle.tsx
                                          ✓ Actually uses theme, setTheme

WASTED COUPLING:
  4 components forward props without using them
  Each component must update when props change (even if not used)

REFACTORING OPTIONS:

Option A: Context API (Recommended for < 5 consumers)
  ┌─────────────────────────────────────────┐
  │ <ThemeProvider>                         │
  │   └─ App                                │
  │       └─ Layout                         │
  │           └─ Dashboard                  │
  │               └─ Widget                 │
  │                   └─ WidgetHeader       │
  │                       └─ ThemeToggle    │
  │                           useTheme() ←──┤
  └─────────────────────────────────────────┘

  Eliminates: 4 prop declarations, 4 forwarding patterns
  Adds: 1 context provider, 1 useTheme hook

Option B: State Management (If >10 consumers or complex updates)
  Use Zustand/Redux for global theme state

[Generate Context Refactoring] [Generate Zustand Refactoring] [Show All Drilling Chains]
================================================================================
```

**Key UX Principles:**

- **Show the actual chain**: Component paths with depth indentation
- **Highlight waste**: Make it obvious which components are just forwarding
- **Multiple solutions**: Context for simple cases, state management for complex
- **Impact estimate**: Show exactly what gets removed vs added

#### 3. Temporal Complexity Issues

**Visual Treatment:**

```
Temporal Complexity: UserProfile.tsx (Critical)
================================================================================

TEMPORAL ISSUES DETECTED: 5

❌ CRITICAL: Missing Cleanup in useEffect (Line 45)
   useEffect(() => {
     const subscription = api.subscribe(userId);
     return subscription.unsubscribe; // ← Missing! Memory leak!
   }, [userId]);

   Impact: Memory leak, grows with each userId change
   Fix: Add cleanup function: return () => subscription.unsubscribe()

⚠️  HIGH: Stale Closure in useEffect (Line 67)
   const handleUpdate = useCallback(() => {
     saveData(formData); // ← formData captured, may be stale
   }, []); // ← Empty deps, closure over formData

   useEffect(() => {
     socket.on('update', handleUpdate);
     return () => socket.off('update', handleUpdate);
   }, [handleUpdate]);

   Impact: Saves old formData when socket triggers
   Fix: Include formData in useCallback deps

⚠️  HIGH: Effect Runs Every Render (Line 89)
   useEffect(() => {
     logAnalytics('profile_view', { userId, timestamp: Date.now() });
   }); // ← No dependency array! Runs every render!

   Impact: Logs analytics on every tiny state change
   Fix: Add [userId] to deps, or move to mount effect

⚠️  MEDIUM: Complex Dependency Array (Line 112)
   useEffect(() => {
     fetchProfile(userId, settings.locale, theme.mode, auth.token);
   }, [userId, settings.locale, theme.mode, auth.token,
       settings.timezone, user.preferences]); // ← 8 dependencies

   Impact: Runs frequently, hard to predict when
   Fix: Consider useMemo for derived computation

⚠️  MEDIUM: Missing Dependencies (Line 145)
   useEffect(() => {
     if (user.isAdmin) {
       loadAdminData(currentProject); // ← currentProject not in deps!
     }
   }, [user.isAdmin]); // ← Misses currentProject

   Impact: Won't reload when project changes
   Fix: Add currentProject to dependencies

TEMPORAL COMPLEXITY SCORE: 47 (Critical)
  4 effects with issues
  1 missing cleanup
  2 effects with timing issues
  1 stale closure risk

[Fix All Auto-Fixable] [Generate Cleanup Functions] [Show Effect Timeline]
================================================================================
```

**Key UX Principles:**

- **Categorize by impact**: Memory leaks > stale closures > missing deps
- **Show the code**: Developers need to see the actual pattern
- **Explain the bug**: Not everyone understands temporal bugs
- **Auto-fix where possible**: Generate cleanup functions, fix deps arrays

#### 4. Missing Error Boundaries

**Visual Treatment:**

```
Missing Error Boundaries
================================================================================

RISKY COMPONENT TREE DETECTED:

App.tsx (no error boundary)
  └─ Dashboard.tsx (no error boundary)
      ├─ UserProfile.tsx (RISKY: API calls, 3 effects)
      ├─ DataTable.tsx (RISKY: Complex rendering, 847 LOC)
      └─ ChartWidget.tsx (RISKY: External library, crashes observed)

CRASH RADIUS:
  If ChartWidget crashes → Entire Dashboard unmounts
  If DataTable crashes → User loses entire view
  If UserProfile crashes → App-level crash

RECOMMENDED BOUNDARIES:

  App.tsx
    └─ <ErrorBoundary fallback={<AppError />}>
        └─ Dashboard.tsx
            ├─ <ErrorBoundary fallback={<WidgetError />}>
            │   └─ UserProfile.tsx
            ├─ <ErrorBoundary fallback={<TableError />}>
            │   └─ DataTable.tsx
            └─ <ErrorBoundary fallback={<ChartError />}>
                └─ ChartWidget.tsx

BLAST RADIUS REDUCTION:
  Before: 1 boundary (app-level)
  After: 4 boundaries (isolated failures)
  Crash scope reduced by 75%

[Generate Error Boundaries] [Show Crash History] [Configure Sentry]
================================================================================
```

**Key UX Principles:**

- **Visualize crash radius**: Show component tree with risk indicators
- **Quantify impact**: "If X crashes, Y unmounts"
- **Suggest boundaries**: Don't just flag, recommend where to add them
- **Show before/after**: Make the improvement tangible

#### 5. Poor Null Safety Patterns

**Visual Treatment:**

```
Null Safety Issues: 8 detected
================================================================================

❌ CRITICAL: Unsafe Property Access (Line 34)
   <div>{user.profile.avatar.url}</div>
                 ↑       ↑
   Chain of nullable properties without guards

   Risk: "Cannot read property 'url' of undefined"
   Fix: Optional chaining or null checks

   Suggested:
   <div>{user?.profile?.avatar?.url ?? <DefaultAvatar />}</div>

⚠️  HIGH: Unsafe Array Access (Line 67)
   const firstItem = items[0].name;
                     ↑
   Array might be empty

   Risk: "Cannot read property 'name' of undefined"
   Fix: Check array length or use optional chaining

   Suggested:
   const firstItem = items[0]?.name ?? 'No items';

⚠️  MEDIUM: Null Callback Invocation (Line 89)
   onClick={onSubmit}
            ↑
   Callback might be undefined (optional prop)

   Risk: "onSubmit is not a function"
   Fix: Guard with null check or provide default

   Suggested:
   onClick={onSubmit ?? (() => {})}
   // or
   onClick={() => onSubmit?.()}

NULL SAFETY SCORE: 34/100 (Poor)
  8 unsafe accesses detected
  3 optional props without guards
  2 array accesses without length checks

[Enable TypeScript Strict Mode] [Add Runtime Guards] [Generate Type Guards]
================================================================================
```

**Key UX Principles:**

- **Show the pattern**: Highlight exactly what's unsafe
- **Explain the runtime error**: Many devs don't understand the crash
- **Provide fixes**: Show the safe pattern next to unsafe one
- **TypeScript integration**: Recommend strict mode for compile-time safety

---

## Document 07: Dependency Cascade Analysis

### React-Specific Cascade Visualizations

#### 1. Prop Drilling Cascade

**Visual Treatment:**

```
Prop Drilling Cascade: userData
================================================================================

App.tsx (source)
  │
  ├─ props: { userData }
  └─ Navbar.tsx (forwards)
      │
      ├─ props: { userData }
      └─ UserMenu.tsx (forwards)
          │
          ├─ props: { userData }
          └─ UserProfile.tsx (forwards)
              │
              ├─ props: { userData }
              └─ Avatar.tsx (CONSUMER)
                  │
                  └─ Uses: userData.avatar, userData.name

DRILLING METRICS:
  Depth: 4 levels
  Forwarding components: 3
  Total component re-renders on userData change: 5
  Actual consumers: 1 (Avatar.tsx)

  Wasted re-renders: 4 components (80%)

ALTERNATIVE ARCHITECTURES:

[Context Solution]
  <UserContext.Provider value={userData}>
    └─ App
        └─ Navbar
            └─ UserMenu
                └─ UserProfile
                    └─ Avatar (useContext(UserContext))

  Re-renders on change: 2 (Provider + Avatar)
  Performance improvement: 60%

[Component Composition]
  <Navbar avatar={<Avatar user={userData} />}>
    └─ Navbar accepts pre-rendered avatar
    └─ No prop drilling through intermediate layers

  Re-renders on change: 2 (App + Avatar)
  Performance improvement: 60%

[Generate Context Code] [Generate Composition Code] [Show Performance Comparison]
================================================================================
```

**Key UX Principles:**

- **Show wasted re-renders**: Make the performance cost visible
- **Multiple solutions**: Context vs composition vs state management
- **Quantify improvement**: Show exact re-render reduction
- **Generate code**: Don't just recommend, provide implementation

#### 2. Context Propagation Paths

**Visual Treatment:**

```
Context Propagation: ThemeContext
================================================================================

<ThemeProvider value={{ theme, setTheme, colors, spacing }}>  ← SOURCE
  │
  ├─ App.tsx (doesn't consume)
  │   └─ Layout.tsx (doesn't consume)
  │       └─ Sidebar.tsx (CONSUMER: useContext)
  │           Uses: theme, colors
  │
  └─ Dashboard.tsx (doesn't consume)
      └─ Widget.tsx (CONSUMER: useContext)
          Uses: colors

CONTEXT HEALTH CHECK:

✓ Good: Only 2 consumers (focused usage)
❌ Bad: Value is unstable object (causes unnecessary re-renders)
⚠️  Warning: Provider passes 4 values (consider splitting)

VALUE STABILITY ANALYSIS:
  const value = { theme, setTheme, colors, spacing };
                ↑
  Creates new object every render!

  Fix:
  const value = useMemo(
    () => ({ theme, setTheme, colors, spacing }),
    [theme, setTheme, colors, spacing]
  );

SPLITTING RECOMMENDATION:
  Current: ThemeContext (4 values)
  Better:  ThemeContext (theme, setTheme)  ← Core theme
           ColorContext (colors)            ← Derived values
           LayoutContext (spacing)          ← Layout concerns

  Benefit: Widget only re-renders on color changes, not theme/spacing

[Fix Value Stability] [Generate Split Contexts] [Show Re-render Impact]
================================================================================
```

**Key UX Principles:**

- **Health check pattern**: Quick visual scan (✓/❌/⚠️)
- **Explain re-renders**: Many devs don't understand context pitfalls
- **Value stability**: Highlight the #1 context performance issue
- **Splitting guidance**: When to split, how to split

#### 3. Hook Dependency Chains

**Visual Treatment:**

```
Hook Dependency Chain: useOrderSummary
================================================================================

useOrderSummary(orderId)
  │
  ├─ Depends on: orderId
  │
  └─ Calls: useOrderData(orderId)
      │
      ├─ Depends on: orderId, userId, authToken
      │
      └─ Calls: useApiClient()
          │
          ├─ Depends on: authToken, baseUrl
          │
          └─ Calls: useAuth()
              │
              └─ Depends on: sessionId, refreshToken

DEPENDENCY COMPLEXITY:
  Total unique dependencies: 6
  Dependency depth: 4 levels
  Re-execution triggers: 6 possible values

  If authToken changes:
    → useAuth re-executes
    → useApiClient re-executes
    → useOrderData re-executes
    → useOrderSummary re-executes

  Cascade depth: 4 hooks

OPTIMIZATION OPPORTUNITIES:

⚠️  Dependency Duplication:
   authToken appears in 3 hooks
   → Consider lifting to parent or using context

⚠️  Deep Chain:
   4-level hook chain creates tight coupling
   → Consider flattening: useOrderSummary directly calls useAuth

✓ Good: Each hook has focused responsibility

RECOMMENDED REFACTORING:

  Current:
    useOrderSummary → useOrderData → useApiClient → useAuth

  Flattened:
    useOrderSummary → useAuth (directly)
                   → fetch with auth token

  Benefit:
    - Reduced chain length (4 → 2)
    - Fewer intermediate re-executions
    - Clearer dependencies

[Show Dependency Graph] [Flatten Chain] [Extract Context]
================================================================================
```

**Key UX Principles:**

- **Visualize the chain**: Show the call hierarchy
- **Explain cascades**: "If X changes, then Y, then Z"
- **Highlight duplication**: Repeated deps across hooks
- **Suggest flattening**: Balance DRY with shallow dependencies

---

## Document 17: Adaptive Visualizations for Scale

### React-Specific Visualizations

#### 1. Hook Complexity Distribution

**Small Scale (1-50 components):**

```
Hook Complexity Distribution
================================================================================

Component                      Hooks  Weight  Visualization
─────────────────────────────────────────────────────────────
App.tsx                          3      9     ███
Dashboard.tsx                    8     32     ████████████████
UserProfile.tsx                 12     47     ███████████████████████
DataTable.tsx                    5     15     ███████
Button.tsx                       0      0
Card.tsx                         1      3     █
...

Legend: Each █ = 2 complexity points
Color: Green (0-10) | Yellow (11-25) | Orange (26-40) | Red (41+)
```

**Medium Scale (51-500 components):**

```
Hook Complexity Heatmap (Directory Level)
================================================================================

src/
├─ components/     (28 files, avg: 12 hooks, max: 47)  [ORANGE]
├─ pages/          (15 files, avg: 18 hooks, max: 32)  [ORANGE]
├─ hooks/          (12 files, avg: 3 hooks, max: 8)    [GREEN]
└─ utils/          (8 files, avg: 0 hooks, max: 0)     [GREEN]

Click directory to expand to file view
```

**Large Scale (500+ components):**

```
Hook Complexity Distribution Histogram
================================================================================

Components by Hook Complexity:

  0-5 hooks   ████████████████████████ 124 components (62%)
  6-10 hooks  ████████████ 48 components (24%)
  11-20 hooks █████ 18 components (9%)
  21-40 hooks ██ 8 components (4%)
  41+ hooks   █ 2 components (1%) ← REVIEW THESE

Top 5 Most Complex:
  1. UserProfile.tsx       47 hooks  [VIEW]
  2. Dashboard.tsx         32 hooks  [VIEW]
  3. CheckoutFlow.tsx      28 hooks  [VIEW]
  4. AdminPanel.tsx        24 hooks  [VIEW]
  5. ReportBuilder.tsx     22 hooks  [VIEW]

[Filter by Directory] [Show Only Complex] [Compare to Industry Benchmarks]
```

**Key UX Principles:**

- **Scale-appropriate detail**: File list → Directory → Histogram
- **Always show outliers**: Top N most complex always visible
- **Distribution context**: "Most components are simple, these are outliers"
- **Actionable thresholds**: Color coding based on maintainability research

#### 2. Temporal Complexity Patterns

**Visualization:**

```
Temporal Complexity Heatmap
================================================================================

Component                Effect Count  Missing Cleanup  Stale Closure Risk
──────────────────────────────────────────────────────────────────────────
UserProfile.tsx              4             2                 1        [RED]
Dashboard.tsx                3             1                 0     [ORANGE]
DataLoader.tsx               2             1                 0     [YELLOW]
Analytics.tsx                1             0                 0     [GREEN]
...

PATTERN DETECTION:

⚠️  Missing Cleanup Pattern (8 components):
   Components with effects but no cleanup function
   Risk: Memory leaks, subscription buildup
   [Show All] [Generate Cleanup Templates]

⚠️  Every-Render Effects (5 components):
   Effects with no dependency array (run every render)
   Risk: Performance degradation
   [Show All] [Add Dependency Arrays]

⚠️  Complex Dependency Arrays (12 components):
   Effects with 5+ dependencies
   Risk: Hard to predict when effects run
   [Show All] [Suggest useMemo]

[Export Temporal Audit] [Track Over Time]
```

**Key UX Principles:**

- **Pattern grouping**: Don't just list, categorize by type
- **Risk-based ordering**: Most dangerous patterns first
- **Batch actions**: Fix all instances of a pattern
- **Trend tracking**: "Getting better or worse?"

#### 3. Prop Drilling Depth Heatmap

**Visualization:**

```
Prop Drilling Heatmap (Component Tree View)
================================================================================

App.tsx
  ├─ Navbar.tsx            [depth: 0]  ░
  │   └─ UserMenu.tsx      [depth: 1]  ▒ (forwards: userData, theme)
  │       └─ Profile.tsx   [depth: 2]  ▓ (forwards: userData, theme)
  │           └─ Avatar    [depth: 3]  █ (forwards: userData)
  │
  └─ Dashboard.tsx         [depth: 0]  ░
      ├─ Widget.tsx        [depth: 1]  ▒ (forwards: theme, settings)
      │   └─ Chart.tsx     [depth: 2]  ▓ (forwards: theme)
      └─ Sidebar.tsx       [depth: 1]  ▒ (forwards: theme, userData)

Legend: Darkness = drilling depth
  ░ No drilling
  ▒ 1-2 levels (acceptable)
  ▓ 3 levels (concerning)
  █ 4+ levels (critical)

DRILLING HOTSPOTS:

❌ Critical: Avatar (4 levels deep, forwards userData)
   App → Navbar → UserMenu → Profile → Avatar
   Recommendation: Use UserContext

⚠️  High: Chart (3 levels deep, forwards theme)
   Dashboard → Widget → Chart
   Recommendation: Use ThemeContext

[Show All Chains] [Generate Context Solutions] [Compare to Best Practices]
```

**Key UX Principles:**

- **Tree visualization**: Preserve component hierarchy
- **Visual density**: Darker = deeper = worse
- **Identify hotspots**: List the worst cases with recommendations
- **Solution generation**: Provide refactoring code

#### 4. Component Size vs Responsibility Count

**Visualization:**

```
Component Size vs Responsibility Scatter Plot
================================================================================
                    │
        100 LOC     │
                    │                    ● UserProfile.tsx
         90         │                     (92 LOC, 6 responsibilities)
                    │                     [GOD COMPONENT]
         80         │
                    │
         70         │         ● DataTable.tsx
                    │          (72 LOC, 4 responsibilities)
         60         │
                    │
         50         │                              ● Dashboard.tsx
                    │                               (125 LOC, 3 responsibilities)
         40         │    ● Form.tsx                 [LARGE BUT FOCUSED]
                    │     (38 LOC, 2 responsibilities)
         30         │
                    │ ● Button  ● Card  ● Input
         20         │  (20 LOC)  (18)    (22)
                    │
         10         │  ● Icon ● Badge ● Avatar
                    │   (8)    (12)     (15)
          0         └────────────────────────────────────────────────
                         0    1    2    3    4    5    6    7  Responsibilities

CONCERN DETECTION:
  Responsibilities = Distinct concerns in component
  - State management (useState, useReducer)
  - Data fetching (useEffect with fetch/API)
  - UI rendering (JSX complexity)
  - Event handling (callbacks, handlers)
  - Validation (form validation, input checking)
  - Analytics (tracking calls)

QUADRANT ANALYSIS:

✓ Bottom Left (Small, Focused): Ideal components
  Examples: Button, Icon, Badge

⚠️  Bottom Right (Small, Many Concerns): Doing too much
  Example: UserProfile (92 LOC but 6 concerns)
  Action: Split by concern

⚠️  Top Left (Large, Few Concerns): Acceptable (legitimate complexity)
  Example: Dashboard (125 LOC, 3 concerns)
  Action: Monitor, consider splitting at 150+ LOC

❌ Top Right (Large, Many Concerns): God components
  Action: Urgent refactoring needed

[Filter by Quadrant] [Show Responsibility Breakdown] [Generate Split Suggestions]
```

**Key UX Principles:**

- **Quadrant framework**: Make interpretation obvious
- **Size isn't everything**: Distinguish size from complexity
- **Explain responsibilities**: What counts as a concern?
- **Prioritize action**: Urgent vs monitor vs acceptable

---

## Document 18: Cognitive Load and Halstead Effort Heatmaps

### React-Specific Cognitive Factors

#### 1. Hook Complexity Weight

**Cognitive Load Formula (React Edition):**

```
React Cognitive Load =
  Base Cognitive Complexity +
  Hook Complexity Weight +
  Temporal Reasoning Overhead +
  Identity Stability Complexity

Where:
  Hook Complexity Weight = Σ(hook_type × count × weight)
  Temporal Reasoning = effect_count × (5 + dependency_count)
  Identity Stability = (useCallback + useMemo) × 2
```

**Hook Weight Table:**

```
Hook Type          Base Weight  Reasoning
─────────────────────────────────────────────────────────────
useState                1      Simple mental model
useReducer              3      Action/reducer indirection
useEffect               4      Temporal reasoning required
useLayoutEffect         5      Timing + paint understanding
useContext              3      Cross-cutting concern tracking
useCallback             2      Dependency + identity management
useMemo                 2      Dependency + computation caching
useRef                  1      Simple mutable container
useImperativeHandle     4      Complex ref manipulation
Custom hooks          1-5      Depends on hook composition
```

**Visualization:**

```
Cognitive Load Breakdown: UserProfile.tsx
================================================================================

Total Cognitive Load: 67 (Critical)

BREAKDOWN:

Base Cognitive Complexity:        18  ██████
  - Nested conditions:            12
  - Loops:                         4
  - Error handling:                2

Hook Complexity:                  24  ████████
  - useState (5×, weight 1):       5
  - useEffect (3×, weight 4):     12
  - useContext (2×, weight 3):     6
  - useCallback (1×, weight 2):    2

Temporal Reasoning:               20  ███████
  - Effect 1: 8 deps              13  (4 + 8)
  - Effect 2: 2 deps               6  (4 + 2)
  - Effect 3: 0 deps               4  (mount-only, but cleanup)

Identity Stability:                5  ██
  - useCallback deps tracking:     2
  - useMemo deps tracking:         3

COMPARISON TO SIMILAR COMPONENTS:
  Average profile component: 28
  This component: 67 (2.4× worse)

  Contributing factors:
  - Too many effects (avg: 1.2, this: 3)
  - Complex effect dependencies (avg: 3, this: 8)
  - Multiple state variables (avg: 3, this: 5)

REFACTORING IMPACT ESTIMATE:
  Extract effects to custom hooks:       -16 (24%)
  Reduce effect dependencies:            -8  (12%)
  Consolidate state with useReducer:     -4  (6%)

  Projected cognitive load: 39 (acceptable)

[Show Hook Timeline] [Extract Custom Hooks] [Consolidate State]
================================================================================
```

**Key UX Principles:**

- **Visual breakdown**: Show where cognitive load comes from
- **React-specific weights**: Not all code is equal
- **Comparative context**: "Is this component worse than similar ones?"
- **Actionable improvements**: Specific refactoring with impact estimate

#### 2. Temporal Reasoning Cognitive Cost

**Visualization:**

```
Temporal Reasoning: When Does This Run?
================================================================================

Component: OrderProcessor.tsx
Cognitive Temporal Load: 32 (High)

EFFECT TIMELINE:

Effect 1 (Line 23) - Cleanup: YES - Deps: [orderId]
  ┌────────────────────────────────────────────────┐
  │ When: orderId changes                          │
  │ What: Fetch order data                         │
  │ Cleanup: Cancel previous fetch                 │
  │ Cognitive Load: 9 (moderate)                   │
  │   - Dependency tracking: +4                    │
  │   - Cleanup reasoning: +3                      │
  │   - Race condition awareness: +2               │
  └────────────────────────────────────────────────┘

Effect 2 (Line 45) - Cleanup: NO - Deps: [orderId, status, user.id]
  ┌────────────────────────────────────────────────┐
  │ When: orderId OR status OR user.id changes     │
  │ What: Log analytics event                      │
  │ Cleanup: MISSING!                              │
  │ Cognitive Load: 15 (high)                      │
  │   - 3 dependencies to track: +8                │
  │   - Object property dependency: +4             │
  │   - No cleanup (memory leak risk): +3          │
  └────────────────────────────────────────────────┘

  ⚠️  ISSUE: user.id changes frequently
      Effect runs on every user update, even if orderId/status same
      Recommendation: Extract to separate effect or use ref

Effect 3 (Line 67) - Cleanup: YES - Deps: []
  ┌────────────────────────────────────────────────┐
  │ When: Mount only                               │
  │ What: Subscribe to websocket                   │
  │ Cleanup: Unsubscribe                           │
  │ Cognitive Load: 8 (low-moderate)               │
  │   - Mount-only is simple: +0                   │
  │   - Cleanup required: +3                       │
  │   - Async subscription: +2                     │
  │   - Stale closure risk: +3                     │
  └────────────────────────────────────────────────┘

  ⚠️  ISSUE: Callback references may be stale
      Event handlers capture props/state at mount time
      Recommendation: Use ref for latest values

TOTAL TEMPORAL COGNITIVE LOAD: 32
  Average for 3 effects: 10.7
  Industry benchmark: 6-8 per effect

  Your component is 33% more cognitively complex than average

COGNITIVE REDUCTION STRATEGIES:
  1. Split effects by concern (-12 load)
  2. Extract to custom hooks (-8 load)
  3. Use refs for stable callbacks (-5 load)

  Projected load: 7 per effect (within benchmark)

[Visualize Effect Execution] [Generate Custom Hooks] [Add Cleanup Templates]
================================================================================
```

**Key UX Principles:**

- **Timeline metaphor**: "When does this run?" is the key question
- **Explain each effect**: What, when, why, cleanup?
- **Highlight issues**: Missing cleanup, stale closures, complex deps
- **Cognitive scoring**: Quantify the mental effort
- **Reduction strategies**: Show how to simplify

#### 3. Identity Stability Complexity

**Visualization:**

```
Identity Stability Analysis: ProductList.tsx
================================================================================

MEMOIZATION HEALTH:

useCallback instances: 3
useMemo instances: 2
Unstable references: 4

CALLBACK ANALYSIS:

✓ GOOD: handleSort (Line 34)
  const handleSort = useCallback((field) => {
    setSortField(field);
  }, []);

  ├─ Dependencies: [] (stable)
  ├─ Passed to: <DataTable /> (memoized child)
  ├─ Benefit: HIGH (prevents DataTable re-render)
  └─ Cognitive Load: 2 (simple dependency tracking)

❌ BAD: handleFilter (Line 42)
  const handleFilter = useCallback((value) => {
    applyFilter(products, value, selectedCategory, sortOrder);
  }, [products, selectedCategory, sortOrder]);

  ├─ Dependencies: [products, selectedCategory, sortOrder]
  ├─ Problem: products is array (changes every render due to parent)
  ├─ Result: handleFilter identity changes every render anyway
  ├─ Benefit: NONE (memoization doesn't help)
  └─ Cognitive Load: 8 (complex dependencies, no benefit)

  Recommendation:
    Don't memoize OR fix parent to stabilize products OR
    use product IDs instead of full objects

⚠️  QUESTIONABLE: calculateTotal (Line 56)
  const calculateTotal = useMemo(() => {
    return products.reduce((sum, p) => sum + p.price, 0);
  }, [products]);

  ├─ Dependencies: [products]
  ├─ Computation: O(n) sum operation
  ├─ Benefit: MAYBE (depends on render frequency)
  ├─ Cognitive Load: 3

  Analysis:
    If ProductList renders < 5 times/sec: Skip memoization (simpler)
    If ProductList renders >10 times/sec: Keep memoization

  [Show Render Frequency] [Profile Performance]

UNSTABLE REFERENCES:

❌ Inline function (Line 89)
  <ProductCard onClick={() => selectProduct(product)} />
                       ↑
  Creates new function every render
  Causes ProductCard to re-render even if product unchanged

  Fix: Extract to useCallback with [product] deps

❌ Inline object (Line 103)
  <FilterPanel config={{ showPrice: true, showStock: true }} />
                      ↑
  Creates new object every render
  Causes FilterPanel to re-render unnecessarily

  Fix: Extract config to useMemo or move outside component

IDENTITY COGNITIVE LOAD: 18
  Necessary memoization: 2 points (1 callback)
  Unnecessary memoization: 8 points (1 callback with no benefit)
  Unstable references: 8 points (2 inline creations)

OPTIMIZATION POTENTIAL:
  Remove unnecessary callback: -8
  Fix unstable references: -8
  Net cognitive improvement: -16 (89% reduction)

[Auto-fix Inline Functions] [Remove Bad Memoization] [Measure Re-renders]
================================================================================
```

**Key UX Principles:**

- **Traffic light system**: Good/bad/questionable memoization
- **Explain the benefit**: Does memoization actually help here?
- **Show the cost**: Cognitive overhead of dependency tracking
- **Quantify renders**: Is optimization premature?
- **Auto-fix patterns**: Remove bad memoization, fix unstable refs

#### 4. Coupling Through Props and Context

**Visualization:**

```
Coupling Complexity: DataTable.tsx
================================================================================

COUPLING ANALYSIS:

Props Coupling:               Score: 24 (High)
  ├─ Total props: 12
  ├─ Required props: 8
  ├─ Optional props: 4
  ├─ Callback props: 5
  └─ Object props: 3

Context Coupling:             Score: 9 (Moderate)
  ├─ Contexts consumed: 3
  │   ├─ ThemeContext (uses: theme, colors)
  │   ├─ UserContext (uses: user.permissions)
  │   └─ DataContext (uses: filters, sort)
  └─ Unstable context values: 1 (DataContext)

PROP COMPLEXITY BREAKDOWN:

High Complexity Props (cognitive load: 3-4 each):

  ❌ onRowClick: (row, event, metadata) => void
     - 3 parameters
     - Called in multiple places
     - Interaction with shift/ctrl requires understanding
     Cognitive Load: 4

  ❌ columns: Column[]
     - Array of complex objects
     - Each column has: id, label, accessor, formatter, sortable
     - 8 properties per column, 12 columns = 96 pieces of info
     Cognitive Load: 4

  ⚠️  data: T[]
     - Generic array
     - Type varies by usage
     - Requires understanding generic constraints
     Cognitive Load: 3

Medium Complexity Props (cognitive load: 2 each):
  - sortBy: string | undefined
  - sortOrder: 'asc' | 'desc'
  - onSort: (field: string) => void
  - selectedRows: Set<string>
  - onSelectionChange: (rows: Set<string>) => void

Low Complexity Props (cognitive load: 1 each):
  - loading: boolean
  - emptyMessage: string
  - maxHeight: number

CONTEXT COMPLEXITY:

⚠️  DataContext Issue:
  const { filters, sort, updateFilters } = useContext(DataContext);

  Problem: Context value is unstable
  const value = { filters, sort, updateFilters }; // ← New object every render

  Impact:
    - DataTable re-renders on every parent render
    - Even if filters/sort haven't changed
    - 12 columns × 50 rows = 600 cells re-rendered unnecessarily

  Cognitive Load: 9
    - Understanding context re-render rules: +4
    - Tracking context value stability: +3
    - Debugging unnecessary renders: +2

TOTAL COUPLING COGNITIVE LOAD: 42

REFACTORING RECOMMENDATIONS:

1. Simplify onRowClick signature (-2 cognitive load)
   Pass event only, component extracts metadata internally

2. Stabilize DataContext value (-9 cognitive load)
   Wrap in useMemo at provider

3. Reduce prop count (-6 cognitive load)
   Group related props: selection={{ rows, onChange }}

4. Extract column config (-4 cognitive load)
   Move to external config file, import as const

Projected coupling load: 21 (50% reduction)

[Measure Re-render Frequency] [Fix Context Stability] [Simplify Props]
================================================================================
```

**Key UX Principles:**

- **Categorize by complexity**: High/medium/low cognitive cost
- **Explain coupling**: Why does this prop add mental load?
- **Context pitfalls**: Unstable values are the #1 issue
- **Quantify renders**: Show the actual performance cost
- **Refactoring ROI**: Cognitive load reduction estimate

---

## General UX Recommendations Across All Documents

### 1. Visual Language for React Concepts

**Icons and Symbols:**

```
📊  State (useState, useReducer)
⏱️   Effects (useEffect, useLayoutEffect)
🎨  Rendering/JSX
🌐  Context (useContext)
🔄  Callbacks (useCallback)
💾  Memoization (useMemo)
📡  Data Fetching
🔐  Authentication/Authorization
🎯  Refs (useRef, useImperativeHandle)
🧩  Custom Hooks
```

**Color Coding:**

```
Green:   0-20   Simple, maintainable
Yellow:  21-40  Moderate, acceptable
Orange:  41-60  Complex, review recommended
Red:     61+    Critical, refactor needed
```

### 2. Severity and Priority Framework

**React-Specific Severity:**

```
Critical (Fix Immediately):
  - Missing effect cleanup (memory leaks)
  - Conditional hook calls (breaks rules of hooks)
  - Infinite render loops
  - Unsafe null access in render

High (Fix This Sprint):
  - Stale closure bugs in effects
  - God components (>15 hooks or >6 responsibilities)
  - Prop drilling >3 levels
  - Missing error boundaries in risky trees

Medium (Plan Refactoring):
  - Complex effect dependencies (>5 deps)
  - Over-memoization (no benefit)
  - Large components (>500 LOC)
  - Poor null safety patterns

Low (Monitor):
  - Hook count approaching thresholds
  - Potential optimization opportunities
  - Code style inconsistencies
```

### 3. Actionable Next Steps

**Always Provide:**

1. **What's wrong**: Specific issue with code snippet
2. **Why it matters**: Impact on bugs, performance, or maintainability
3. **How to fix**: Concrete refactoring suggestion
4. **Expected improvement**: Quantified benefit (cognitive load, re-renders, etc.)

**Generate Code:**

- Context implementations
- Custom hook extractions
- Cleanup function templates
- Memoization wrappers
- Error boundary boilerplate

### 4. Progressive Disclosure

**Three Levels:**

1. **At-a-glance**: Traffic light colors, scores, counts
2. **On hover**: Tooltips with metric explanation and thresholds
3. **On click**: Full detail panel with code, recommendations, and actions

**Example:**

```
Component Card (Level 1):
  UserProfile.tsx [RED] 67

Hover (Level 2):
  Cognitive Load: 67 (Critical)
  - Hook complexity: 24
  - Temporal: 20
  - Base: 18
  Click for breakdown

Click (Level 3):
  [Full detail panel as shown in previous sections]
```

### 5. Educational Content

**Embed Learning:**

- **Tooltips**: Explain why metric matters
- **Examples**: Show good vs bad patterns
- **Documentation links**: React docs, best practices articles
- **Videos**: Short explanations of concepts (temporal complexity, etc.)

**Example Tooltip:**

```
[?] Temporal Complexity

Temporal complexity measures how hard it is to understand
WHEN code runs in React. Effects with many dependencies or
missing cleanup are cognitively demanding.

Higher scores indicate:
- More effects to track mentally
- More dependencies to reason about
- More potential bugs (stale closures, memory leaks)

Learn more: [docs.react.dev/learn/synchronizing-with-effects]
```

### 6. Comparison and Benchmarking

**Always Show Context:**

- Component vs similar components
- Project vs industry benchmarks
- Current vs previous snapshots
- Before vs after refactoring

**Example:**

```
Your Component: 47 hooks, cognitive load 67
Similar components (avg): 12 hooks, cognitive load 28
Industry benchmark: 8 hooks, cognitive load 22

You are: 2.4× more complex than similar components
         3.0× more complex than industry average
```

### 7. Filtering and Search

**React-Specific Filters:**

- By hook type (components using useEffect)
- By metric threshold (cognitive load >40)
- By anti-pattern (missing cleanup, stale closures)
- By responsibility count (god components)
- By temporal complexity (complex effects)

### 8. Bulk Actions

**Allow Batch Operations:**

- Generate cleanup for all missing cleanups
- Add error boundaries to risky components
- Fix all unstable context values
- Remove all unnecessary memoization
- Add TypeScript strict mode fixes

---

## Implementation Priority

### Phase 1: Core React Metrics (MVP)

1. Hook complexity scoring and visualization
2. Temporal complexity detection (missing cleanup, stale closures)
3. God component detection with responsibility breakdown
4. Prop drilling chain visualization

### Phase 2: Advanced Analysis

5. Context propagation and stability analysis
6. Identity stability and memoization recommendations
7. Cognitive load calculation (React-aware)
8. Anti-pattern detection (comprehensive)

### Phase 3: Optimization Guidance

9. Performance profiling integration
10. Automated refactoring suggestions
11. Code generation (contexts, custom hooks, error boundaries)
12. Historical trend tracking

---

## Success Metrics

### User Understanding

- **Target**: 80% of developers understand React metrics within 5 minutes
- **Measure**: User testing, comprehension surveys

### Actionability

- **Target**: 60% of flagged issues result in code changes
- **Measure**: Track issue resolution rates

### False Positive Rate

- **Target**: < 10% of flagged issues marked as "acceptable"
- **Measure**: Track "Mark as Accepted" usage

### Performance Impact

- **Target**: Reduce average component cognitive load by 30%
- **Measure**: Track before/after refactoring metrics

### Developer Satisfaction

- **Target**: 4.5/5 satisfaction rating
- **Measure**: Post-refactoring surveys

---

## Open Questions for User Research

1. **Cognitive Load Weights**: Are the hook weights accurate? Do developers agree useState is simpler than useEffect?

2. **Severity Thresholds**: Are the red/yellow/green thresholds calibrated correctly for your team?

3. **Visualization Preferences**: Tree vs graph vs table for prop drilling chains?

4. **Detail Level**: How much code context is helpful vs overwhelming?

5. **Automated Fixes**: Which refactorings are safe to auto-generate vs require human review?

6. **Team Benchmarks**: Should we compare to industry averages or internal team averages?

7. **Learning Curve**: Do junior developers need different visualizations than senior developers?

8. **Integration**: Should recommendations integrate with IDE (VSCode), linter (ESLint), or both?
