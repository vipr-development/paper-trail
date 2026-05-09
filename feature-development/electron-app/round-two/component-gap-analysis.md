# Component Gap Analysis

## Overview

During the UI component integration audit of Phase Documents 07-21, we identified **6 component gaps** where desired functionality does not exist in the current `@vipr/ui` library. This document consolidates all gaps with priority rankings, options, and interim solutions.

**Summary:**

- **2 HIGH PRIORITY** gaps: Graph Visualization, Scatter Plot
- **2 MEDIUM PRIORITY** gaps: Wizard/Stepper, Range Slider
- **2 LOW PRIORITY** gaps: Time Picker, Continuous Color Legend

---

## Gap 1: Graph/Network Visualization ⚠️ HIGH PRIORITY

### Description

Force-directed or hierarchical graph layout for showing dependency relationships and type propagation chains.

### Affected Phases

- **Phase 07:** Dependency Cascade Analysis (dependency graphs)
- **Phase 21:** TypeScript Type Complexity (type propagation chains)

### Options

1. **Add React Flow library** (~50KB gzipped)
   - **Pros:** Mature, feature-complete, Electron-compatible, good TypeScript support
   - **Cons:** Large dependency, learning curve, potential performance issues with 1000+ nodes
   - **Effort:** ~1-2 weeks integration + styling

2. **Build custom D3-force implementation**
   - **Pros:** Full control, smaller bundle, optimized for our use case
   - **Cons:** Significant development effort, reinventing wheel, maintenance burden
   - **Effort:** ~3-4 weeks initial build + ongoing maintenance

3. **Table-only approach (RECOMMENDED INTERIM)**
   - **Pros:** Zero new dependencies, uses existing CardTable, immediate implementation
   - **Cons:** Loses spatial/visual representation of relationships
   - **Effort:** Already implemented in phase documents

### Recommendation

**DEFER** graph visualization to future enhancement. Use table-only approach initially:

**Phase 07 (Dependencies):**

```tsx
<Tabs>
  <Tab label="Upstream">
    <CardTable
      columns={[
        { key: 'file', label: 'File' },
        { key: 'direction', label: 'Direction' },
        { key: 'depth', label: 'Depth' },
        { key: 'coupling', label: 'Coupling' },
      ]}
      data={upstreamDeps}
    />
  </Tab>
  <Tab label="Downstream">...</Tab>
  <Tab label="Circular">
    <Alert variant="banner" type="error">
      Circular dependency detected: {circularChain.join(' → ')}
    </Alert>
  </Tab>
</Tabs>
```

**When to Revisit:**

- User feedback explicitly requests graph visualization
- Dependencies exceed 100 files (table becomes unwieldy)
- Budget allows for React Flow integration (~2 weeks)

---

## Gap 2: Scatter Plot / Bubble Chart 📊 HIGH PRIORITY

### Description

XY scatter chart to visualize correlation between two metrics (e.g., type complexity vs code complexity).

### Affected Phases

- **Phase 21:** TypeScript Type Complexity (correlation analysis)

### Options

1. **Extend Chart.js wrapper** to support scatter type
   - **Pros:** Chart.js has native scatter support, consistent with existing charts
   - **Cons:** Requires updating MetricChart component wrapper
   - **Effort:** ~2-3 days

2. **Use Recharts library**
   - **Pros:** Excellent scatter plot support, React-native
   - **Cons:** Adds new dependency (~90KB), different API from Chart.js
   - **Effort:** ~1 week integration

3. **Table with two metric columns (RECOMMENDED INTERIM)**
   - **Pros:** Zero effort, uses existing CardTable
   - **Cons:** Loses visual correlation insight
   - **Effort:** Already implemented

### Recommendation

**DEFER** scatter plot. Use table approach initially:

```tsx
<CardTable
  columns={[
    { key: 'file', label: 'File', sortable: true },
    { key: 'typeComplexity', label: 'Type Complexity', sortable: true },
    { key: 'codeComplexity', label: 'Code Complexity', sortable: true },
    { key: 'delta', label: 'Δ (TC - CC)', sortable: true },
  ]}
  data={files}
  defaultSort={{ key: 'delta', direction: 'desc' }}
/>
```

**When to Revisit:**

- After Chart.js wrapper is refactored for other needs
- User feedback requests correlation visualization
- Implementing Phase 21 and scatter plot is must-have

**Future Implementation Path:** Extend Chart.js wrapper (~2-3 days effort)

---

## Gap 3: Wizard/Stepper Progress Indicator 🧭 MEDIUM PRIORITY

### Description

Numbered step indicator showing completed/active/pending states for multi-step flows.

### Affected Phases

- **Phase 13:** Initial Analysis wizard (5-step flow)
- **Phase 16:** Complexity Budget creation wizard (4-step flow)

### Options

1. **Build dedicated Stepper component** (~50-100 LOC)
   - **Pros:** Reusable, clean abstraction, proper accessibility
   - **Cons:** New component to maintain
   - **Effort:** ~1-2 days

2. **Use Breadcrumb with numbered labels**
   - **Pros:** Zero new components, existing Breadcrumb has navigation
   - **Cons:** Breadcrumb semantics don't quite match wizard flow
   - **Effort:** ~1 hour styling tweaks

3. **Custom inline implementation (RECOMMENDED)**
   - **Pros:** Specific to use case, no abstraction overhead
   - **Cons:** Not reusable (but only 2 use cases)
   - **Effort:** ~2-3 hours per wizard

### Recommendation

**BUILD custom wizard container** (~80 LOC) as documented in Phase 13:

```tsx
const WizardContainer: React.FC<WizardContainerProps> = ({ steps, onComplete, onSkip }) => {
  const [currentStepIndex, setCurrentStepIndex] = useState(0);
  const currentStep = steps[currentStepIndex];

  return (
    <div className="flex flex-col h-screen">
      {/* Step breadcrumb with numbered badges */}
      <div className="flex items-center gap-2 p-4 border-b">
        {steps.map((step, index) => (
          <div
            key={index}
            className={cn(
              'flex items-center gap-2',
              index < steps.length - 1 && 'after:content-["→"] after:ml-2 after:text-gray-400'
            )}
          >
            <div
              className={cn(
                'w-8 h-8 rounded-full flex items-center justify-center text-sm font-semibold',
                index < currentStepIndex && 'bg-green-500 text-white',
                index === currentStepIndex && 'bg-violet-500 text-white',
                index > currentStepIndex && 'bg-gray-200 dark:bg-gray-700 text-gray-500'
              )}
            >
              {index < currentStepIndex ? '✓' : index + 1}
            </div>
            <span
              className={cn(
                'text-sm',
                index === currentStepIndex && 'font-semibold',
                index !== currentStepIndex && 'text-gray-600 dark:text-gray-400'
              )}
            >
              {step.title}
            </span>
          </div>
        ))}
      </div>

      {/* Step content */}
      <div className="flex-1 overflow-auto p-6">{currentStep.content}</div>

      {/* Navigation */}
      <div className="flex items-center justify-between p-4 border-t">
        <Button
          appearance="tertiary"
          onClick={() => setCurrentStepIndex(i => i - 1)}
          disabled={currentStepIndex === 0}
        >
          Back
        </Button>
        <Button
          appearance="primary"
          onClick={() => {
            if (currentStepIndex === steps.length - 1) {
              onComplete();
            } else {
              setCurrentStepIndex(i => i + 1);
            }
          }}
        >
          {currentStepIndex === steps.length - 1 ? 'Complete' : 'Next'}
        </Button>
      </div>
    </div>
  );
};
```

**Effort:** ~2-3 hours
**Status:** **Implemented in Phase 13 document**, ready to build

---

## Gap 4: Range Slider 🎚️ MEDIUM PRIORITY

### Description

Reusable range slider input component. Heatmap has internal one but it's not exported.

### Affected Phases

- **Phase 07:** Dependency depth slider (1-10 levels)
- **Phase 14:** Alert sensitivity slider
- **Phase 16:** Threshold adjustment (if implemented as slider)

### Options

1. **Extract and export Heatmap's slider**
   - **Pros:** Already exists, proven implementation
   - **Cons:** Heatmap's slider may be tightly coupled to its use case
   - **Effort:** ~1-2 hours refactoring

2. **Use Dropdown with discrete options (RECOMMENDED)**
   - **Pros:** Zero new components, accessible, works for limited ranges
   - **Cons:** Not suitable for continuous ranges (but our use cases are discrete)
   - **Effort:** Already implemented

3. **Build dedicated Slider component**
   - **Pros:** Proper UX for ranges, accessible
   - **Cons:** ~100-150 LOC, accessibility considerations (keyboard nav, ARIA)
   - **Effort:** ~1-2 days

### Recommendation

**Use Dropdown (select variant)** with discrete options:

```tsx
<Dropdown
  variant="select"
  label="Depth"
  options={[
    { value: 1, label: '1 level' },
    { value: 2, label: '2 levels' },
    { value: 3, label: '3 levels' },
    { value: 5, label: '5 levels' },
    { value: 10, label: '10 levels (max)' },
  ]}
  value={depth}
  onChange={setDepth}
/>
```

**Alternative (HTML native):**

```tsx
<Input
  type="range"
  min={1}
  max={10}
  value={depth}
  onChange={e => setDepth(parseInt(e.target.value))}
  className="w-full h-2 bg-gray-200 rounded-lg"
/>
```

**When to Revisit:** If >3 use cases emerge, consider extracting Heatmap's slider

---

## Gap 5: Time Picker ⏰ LOW PRIORITY

### Description

Time-of-day picker (hours:minutes). DatePicker only selects dates.

### Affected Phases

- **Phase 20:** Scheduled analysis time-of-day selection

### Options

1. **Build custom TimePicker component**
   - **Pros:** Tailored UX, custom styling
   - **Cons:** ~150-200 LOC, accessibility considerations, time zone handling
   - **Effort:** ~2-3 days

2. **Use HTML `<input type="time">` (RECOMMENDED)**
   - **Pros:** Native browser support, zero effort, accessible by default
   - **Cons:** Styling limited, appearance varies by browser
   - **Effort:** Already implemented

3. **Use Dropdown with hour/minute selects**
   - **Pros:** Fully styleable, consistent across browsers
   - **Cons:** More complex UI (two dropdowns), less intuitive
   - **Effort:** ~1-2 hours

### Recommendation

**Use HTML `<input type="time">`** with Input component styling:

```tsx
<Input
  type="time"
  label="Analysis Time"
  value={scheduleTime} // "HH:MM" format (e.g., "02:00")
  onChange={e => setScheduleTime(e.target.value)}
  className="font-mono w-32"
/>
```

**Browser Support:** Excellent in modern browsers (95%+ coverage)

**Status:** **Implemented in Phase 20 document**, no action needed

---

## Gap 6: Continuous Color Scale Legend 🎨 LOW PRIORITY

### Description

Gradient legend showing min-to-max continuous value mapping. Current MetricsHeatmap has discrete violation count legend.

### Affected Phases

- **Phase 18:** Cognitive load heatmaps (complexity gradients)
- **Phase 21:** Type complexity heatmaps

### Options

1. **Build custom gradient legend component** (~50-100 LOC)
   - **Pros:** Precise visual mapping, aesthetically clean
   - **Cons:** New SVG component, limited utility (only 2 use cases)
   - **Effort:** ~1-2 days

2. **Extend MetricsHeatmap** to support continuous mode
   - **Pros:** Native integration with heatmap
   - **Cons:** Significant refactor, changes core heatmap logic
   - **Effort:** ~3-5 days

3. **Use existing discrete legend** with threshold-based coloring (RECOMMENDED)
   - **Pros:** Zero effort, existing pattern, actionable thresholds
   - **Cons:** Less precise for subtle variations
   - **Effort:** Already implemented

### Recommendation

**Use threshold-based approach** with discrete legend:

```tsx
<MetricsHeatmap
  metrics={['cognitive']} // Single metric
  files={files}
  thresholds={{
    cognitive: metricThresholds.cognitive.concerning, // "Concerning" level = 30
  }}
  colorScheme={{
    healthy: 'green', // Below threshold (0-20)
    moderate: 'yellow', // Near threshold (20-30)
    concerning: 'orange', // Above threshold (30-50)
    critical: 'red', // Well above threshold (50+)
  }}
/>;

{
  /* Discrete legend */
}
<div className="flex items-center gap-4 mt-4">
  <div className="flex items-center gap-2">
    <div className="w-4 h-4 rounded bg-green-500/20" />
    <span className="text-xs">Healthy (0-20)</span>
  </div>
  <div className="flex items-center gap-2">
    <div className="w-4 h-4 rounded bg-yellow-500/20" />
    <span className="text-xs">Moderate (20-30)</span>
  </div>
  <div className="flex items-center gap-2">
    <div className="w-4 h-4 rounded bg-orange-500/20" />
    <span className="text-xs">Concerning (30-50)</span>
  </div>
  <div className="flex items-center gap-2">
    <div className="w-4 h-4 rounded bg-red-500/20" />
    <span className="text-xs">Critical (50+)</span>
  </div>
</div>;
```

**Benefits:**

- Clear action thresholds (healthy vs. concerning)
- Aligns with industry-standard complexity thresholds
- Simpler mental model: "above threshold = needs attention"

**When to Revisit:**

- User feedback indicates confusion with discrete thresholds
- Need to visualize subtle differences within same threshold band
- Implementing "before/after" refactoring comparisons

**Status:** **Implemented in Phases 18 & 21**, no action needed

---

## Gap Priority Matrix

| Gap                     | Priority | Effort          | Recommendation             | Status             |
| ----------------------- | -------- | --------------- | -------------------------- | ------------------ |
| **Graph Visualization** | HIGH     | 1-4 weeks       | Defer, use table           | Interim documented |
| **Scatter Plot**        | HIGH     | 2 days - 1 week | Defer, use table           | Interim documented |
| **Wizard/Stepper**      | MEDIUM   | 2-3 hours       | Build custom (~80 LOC)     | Ready to implement |
| **Range Slider**        | MEDIUM   | 1-2 hours       | Use Dropdown or HTML range | Interim documented |
| **Time Picker**         | LOW      | 0 (use HTML)    | Use `<input type="time">`  | **Implemented**    |
| **Continuous Legend**   | LOW      | 1-5 days        | Use discrete thresholds    | **Implemented**    |

---

## Implementation Roadmap

### Immediate (Week 1)

1. ✅ **Time Picker** - Already implemented with HTML `<input type="time">`
2. ✅ **Continuous Legend** - Already implemented with threshold approach
3. ⏳ **Wizard/Stepper** - Build WizardContainer component (~80 LOC, 2-3 hours)

### Short-term (Weeks 2-4)

4. **Range Slider** - Use Dropdown or HTML native (decision per use case)
5. **Scatter Plot** - Evaluate Chart.js scatter extension if Phase 21 is prioritized

### Long-term (Months 2-3)

6. **Graph Visualization** - Add React Flow only if user demand justifies

### Deferred (Future Enhancement)

- Continuous color legend (build if threshold approach proves insufficient)

---

## Conclusion

Of 6 identified gaps:

- **2 are already resolved** with HTML native or threshold approaches (Time Picker, Continuous Legend)
- **1 is ready to build** with minimal effort (Wizard/Stepper, ~80 LOC)
- **2 use interim solutions** that defer complexity (Graph, Scatter Plot)
- **1 has flexible options** based on use case (Range Slider)

**Total new component effort:** ~2-3 hours (WizardContainer only)

**Key Principle:** Use simple, existing solutions first. Add complexity only when user demand or feature requirements justify it.
