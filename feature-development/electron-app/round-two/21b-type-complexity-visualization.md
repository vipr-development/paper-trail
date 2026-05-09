---
id: 21b-type-complexity-visualization
title: TypeScript Type Complexity Visualization
phase: 21B
dependencies:
  - 21a-typescript-analyzer-plugin
  - 01-blast-radius-hotspot-view
  - 02-complexity-velocity-dashboard
  - 04-architectural-smells-detection
  - 17-adaptive-visualizations-scale
  - 18-cognitive-load-halstead-heatmaps
status: not-started
---

# TypeScript Type Complexity Visualization

> **Note:** This document supersedes Phase 21. The original Phase 21 sourced type metrics from the React analyzer (`plugin_id = 'react'`). Phase 21A introduced a dedicated TypeScript analyzer plugin (`@vipr/typescript`, `plugin_id = 'typescript'`). This document updates the visualization layer to use that plugin as its exclusive data source and exposes the eight additional analyses it provides beyond type complexity (nine analyses total).

## User Story

**As a TypeScript developer**, I want to visualize type-level complexity separately from runtime code complexity, so that I can identify where type gymnastics are adding cognitive load and maintenance burden to the codebase.

## User Need

TypeScript's type system enables powerful compile-time guarantees, but type complexity creates its own cognitive load that traditional code metrics miss. A component with cyclomatic complexity 15 might seem reasonable until you discover it uses deeply nested generics with conditional types that take 20 minutes to understand.

Developers need to answer:

- "Where is type complexity making code harder to maintain?"
- "Which types propagate complexity through the codebase?"
- "When does type safety become type burden?"
- "Where should we simplify types vs. accept essential complexity?"

Type complexity is distinct from code complexity but equally impacts maintainability. The type system creates its own cognitive overhead that compounds runtime complexity.

---

## Core Insight: Type Complexity as Cognitive Tax

TypeScript types exist only at compile time, but they affect:

- **Comprehension Time** - How long to understand what a type accepts
- **Modification Risk** - How easily breaking changes propagate through types
- **IDE Performance** - How fast autocomplete and type checking run
- **Onboarding Friction** - How quickly new developers can contribute

Unlike runtime complexity, type complexity can be avoided by using `any` or loosening constraints. This creates a tension: strict types catch bugs but complex types slow development.

---

## UX Flow

### Entry Points

1. **Primary:** Navigation sidebar "Type Complexity" under Technical Debt section
2. **Secondary:** Cognitive Load Heatmap shows type complexity as separate layer
3. **Contextual:** File detail shows type metrics alongside code metrics
4. **Alert:** Type complexity regression notification
5. **Search:** "complex types" or "type gymnastics" in global search

### User Journey

```mermaid
flowchart TD
    A[Enter Type Complexity View] --> B[View Type Heatmap]
    B --> C{Select Dimension}

    C -->|Generic Depth| D[Show Deep Generic Nesting]
    C -->|Union Size| E[Show Large Union Types]
    C -->|Conditional Types| F[Show Type-Level Logic]
    C -->|Overall Complexity| G[Show Combined Type Score]

    D --> H[Click Hotspot]
    E --> H
    F --> H
    G --> H

    H --> I[View Type Detail]
    I --> J[See Type Definition]
    J --> K[View Type Dependencies]
    K --> L[See Propagation Impact]

    I --> M[Compare with Code Complexity]
    M --> N{Complexity Mismatch?}
    N -->|High Type, Low Code| O[Type Gymnastics Anti-Pattern]
    N -->|Both High| P[Essential Complexity]
    N -->|Low Type, High Code| Q[Runtime Complexity]

    O --> R[Generate Simplification Prompt]
    P --> S[Document Type Rationale]
    Q --> T[No Type Action Needed]

    style A fill:#3b82f6,color:#fff
    style I fill:#8b5cf6,color:#fff
    style O fill:#ef4444,color:#fff
    style R fill:#f59e0b,color:#000
```

### Exit Points

1. **To File Detail:** Inspect specific file's type complexity
2. **To Type Dependency Graph:** Visualize type propagation
3. **To AI Prompt:** Generate type simplification suggestions
4. **To Comparison:** Compare type complexity across snapshots
5. **To Cognitive Heatmap:** See types in broader complexity context

---

## Information Architecture

### Type Complexity Metrics (from TypeScript Analyzer)

All metrics in this section are sourced from the `@vipr/typescript` plugin (`plugin_id = 'typescript'`). The primary type complexity analysis is stored under the `ts-type-complexity` key in the metrics JSON.

| Metric                          | Analysis Key       | What It Measures                     | Interpretation                           |
| ------------------------------- | ------------------ | ------------------------------------ | ---------------------------------------- |
| **Generic Depth**               | ts-type-complexity | Nesting levels of generic parameters | `Array<Promise<Result<T, E>>>` = depth 3 |
| **Conditional Branches**        | ts-type-complexity | Count of conditional type branches   | Ternary conditional type patterns        |
| **Union Size**                  | ts-type-complexity | Maximum members in union type        | `type Status = 'a' \| 'b' \| 'c' \| ...` |
| **Intersection Size**           | ts-type-complexity | Maximum members in intersection type | Type intersection count                  |
| **Mapped Type Count**           | ts-type-complexity | Count of mapped types                | `{ [K in keyof T]: ... }`                |
| **Recursive Types**             | ts-type-complexity | Self-referencing types               | `type Node = { children: Node[] }`       |
| **Template Literal Complexity** | ts-type-complexity | Template literal type sophistication | Template literal patterns                |
| **Type Parameter Count**        | ts-type-complexity | Total generic type parameters        | Count of type parameters                 |
| **Infer Usage**                 | ts-type-complexity | Count of `infer` keywords            | Type extraction patterns                 |

The composite type health score comes from the TypeScript plugin's `buildCompositeScore()` method and is stored at `analyses.score` for the plugin row. The nine analyses are weighted as follows (defined in `analyzers/typescript/src/constants/weights.ts`):

| Analysis                 | Weight |
| ------------------------ | ------ |
| `ts-type-complexity`     | 0.22   |
| `ts-type-safety`         | 0.20   |
| `ts-generics`            | 0.13   |
| `ts-structural-quality`  | 0.10   |
| `ts-declaration-shape`   | 0.10   |
| `ts-type-guards`         | 0.08   |
| `ts-utility-types`       | 0.07   |
| `ts-import-discipline`   | 0.06   |
| `ts-module-augmentation` | 0.04   |

### Pre-requisite: TypeScriptPluginMetrics Export

> **Implementation note:** The `TypeScriptPluginMetrics` container type must be exported from `analyzers/typescript/src/types/index.ts` before this phase can begin. This type provides the canonical shape for all nine analysis result objects referenced below. Without this export, the desktop renderer and IPC layer cannot type the data flowing from the main process.

### Additional Analyses from the TypeScript Plugin

Phase 21A exposes eight additional analyses beyond type complexity. These enrich the file detail view and power the extended metrics panels described below.

| Analysis ID              | Data Available                                                                                                                                                                                                                                                                                                             | Primary UI Use                           |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `ts-type-safety`         | `anyCount`, `typeAssertionCount`, `doubleAssertionCount`, `tsIgnoreCount`, `tsExpectErrorCount`, `tsNoCheckCount`, `nonNullAssertionCount`, `unknownCount`, `narrowingPatternCount`, `satisfiesCount`, `brandedTypeCount`, `implicitAnyCatchCount`, `indexSignatureAbuseCount`, `readonlyPropertyCount`, `resultTypeCount` | Type Safety panel in file detail         |
| `ts-declaration-shape`   | `interfaceCount`, `typeAliasCount`, `enumCount`, `functionOverloadCount`, `maxInheritanceDepth`, `emptyInterfaceCount`, `declarationMergingCount`, `namespaceCount`                                                                                                                                                        | Declaration Shape summary in file detail |
| `ts-generics`            | `unconstrainedParamCount`, `broadConstraintCount`, `varianceAnnotationCount`, `maxArity`, `unusedTypeParamCount`, `singleUseTypeParamCount`, `constTypeParamCount`                                                                                                                                                         | Generics Health panel in file detail     |
| `ts-utility-types`       | `builtinUtilityCount`, `uniqueUtilitiesUsed`, `customMappedTypeCount`, `redundantCustomTypeCount`, `utilityUsageBreakdown`                                                                                                                                                                                                 | Utility Types panel in file detail       |
| `ts-import-discipline`   | `importTypeRatio`, `typeOnlyExportCount`, `typeOnlyImportCount`, `namespaceImportCount`, `valueImportsCarryingTypes`                                                                                                                                                                                                       | Import Discipline score in file detail   |
| `ts-type-guards`         | `typePredicateCount`, `assertionFunctionCount`, `discriminatedUnionCount`, `exhaustivenessCheckCount`, `typeofGuardCount`, `instanceofGuardCount`, `nullishCheckCount`                                                                                                                                                     | Type Guard Coverage panel in file detail |
| `ts-module-augmentation` | `declareModuleCount`, `declareGlobalCount`, `ambientModuleCount`, `augmentationTargets`, `globalPollutionScore`                                                                                                                                                                                                            | Module Augmentation panel in file detail |
| `ts-structural-quality`  | `emptyCatchCount`, `untypedThrowCount`, `globalStateAccessCount`, `singletonPatternCount`, `constructorInjectionCount`, `decoratorCount`, `usingDeclarationCount`, `typeToRuntimeRatio`                                                                                                                                    | Structural Quality panel in file detail  |

### Data Displayed

**Level 1: Type Heatmap (Repository View)**

- Treemap of files colored by type complexity
- Cell size = lines of TypeScript code
- Cell color = type complexity score (0-100)
- Overlayable with code complexity for comparison

**Level 2: Type Dimension View**

- Filter heatmap by specific type metric
- Show only generic depth, or only union size, etc.
- Identify files high on one dimension

**Level 3: Type Detail Panel**

- Specific complex types in the file
- Type dependency graph (what depends on this type)
- Type propagation impact (how far does it spread)
- Suggested simplifications
- Extended panels for all nine TypeScript analyses (see File Detail View Enhancement)

**Level 4: Type vs. Code Complexity Matrix**

- Two-metric table: X dimension = code complexity, Y dimension = type complexity
- Quadrants reveal different patterns:
  - High Type, Low Code = Type gymnastics
  - High Both = Essential complexity
  - Low Type, High Code = Runtime complexity
  - Low Both = Simple code

### Progressive Disclosure Strategy

| Visible By Default    | Revealed on Hover        | Revealed on Click      |
| --------------------- | ------------------------ | ---------------------- |
| Type complexity score | Breakdown by dimension   | Specific complex types |
| File name             | Type vs. code comparison | Type definitions       |
| Color intensity       | Propagation count        | Dependency graph       |
| Relative size         | Exact metrics            | Historical trend       |

---

## Interaction Patterns

### Primary Actions

| Action              | Trigger            | Result                                     |
| ------------------- | ------------------ | ------------------------------------------ |
| Switch dimensions   | Dropdown selection | Re-color heatmap by selected metric        |
| Compare with code   | Toggle overlay     | Show code complexity alongside types       |
| Filter by threshold | Slider adjustment  | Hide files below type complexity threshold |
| View type detail    | Click file         | Open type analysis panel                   |

### Secondary Actions

| Action             | Trigger         | Result                              |
| ------------------ | --------------- | ----------------------------------- |
| Export type report | Toolbar button  | Download type complexity summary    |
| Show propagation   | Click type name | Visualize where type is used        |
| Generate prompt    | AI button       | Create type simplification prompt   |
| Mark as justified  | Context menu    | Accept type complexity as necessary |

### Micro-interactions

**Heatmap Hover:**

- Show type vs. code complexity comparison
- Display top 3 complex types in file
- Indicate propagation impact badge

**Type Badge:**

- Color codes by severity:
  - Green: Simple types (0-20)
  - Yellow: Moderate types (21-40)
  - Yellow-600: Complex types (41-60)
  - Red: Very complex types (61+)

**Comparison Toggle:**

- Smooth transition between type-only and split view
- Legend updates to show both scales
- Files can be colored by type or code independently

---

## Component Map

This section provides explicit `@vipr/ui` component specifications for TypeScript type complexity visualization.

### Primary Components

| Component      | Import Path                | Configuration                     | Usage in Phase 21B                                                                                       |
| -------------- | -------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------- |
| MetricsHeatmap | @vipr/ui/heatmap           | files, metricDefs, onFileClick    | Type complexity visualization by file                                                                    |
| Treemap        | @vipr/ui/treemap           | data, width, height               | Alternative hierarchical view of type complexity                                                         |
| CardTable      | @vipr/ui/card-table        | data, columns, onRowClick         | **Interim scatter plot replacement** - two metric columns                                                |
| MetricBarChart | @vipr/ui/components/charts | label, value, min, max, direction | Type metric details (generic depth, union size, etc.) — no dedicated path export; import from the barrel |
| MetricGroup    | @vipr/ui/metric-group      | data, defaultExpanded             | Grouped type metrics in file detail                                                                      |
| InsightCard    | @vipr/ui/insight-card      | insight, defaultExpanded          | Type complexity insights and recommendations                                                             |
| Tabs           | @vipr/ui/tabs              | tabs, variant="underline"         | Switch between Heatmap/Table/Detail views                                                                |
| Dropdown       | @vipr/ui/dropdown          | variant="select", options         | Dimension filter, metric selection                                                                       |
| Badge          | @vipr/ui/badge             | variant, size                     | Type complexity severity indicators                                                                      |
| Button         | @vipr/ui/button            | appearance, size, onClick         | Generate simplification prompt, view details                                                             |
| Alert          | @vipr/ui/alert             | variant="banner", type="warning"  | Type gymnastics anti-pattern warnings                                                                    |

### Color Tokens

**Type Complexity Severity:**

- `green-500` / `green-500/20` - Simple (0-20)
- `yellow-500` / `yellow-500/20` - Moderate (21-40)
- `yellow-600` / `yellow-600/20` - Complex (41-60)
- `red-500` / `red-500/20` - Very Complex (61+)

**Type vs Code Quadrants:**

- `violet-500` - Type gymnastics (high type, low code)
- `red-500` - Essential complexity (high both)
- `sky-500` - Runtime complexity (low type, high code)
- `green-500` - Simple (low both)

**Code Styling:**

- `bg-gray-50 dark:bg-gray-900` - Code block backgrounds
- `border-gray-200 dark:border-gray-700` - Code block borders

### Typography Tokens

**Type Signatures:**

- `font-mono text-xs text-gray-800 dark:text-gray-100` - Type definitions
- `bg-gray-50 dark:bg-gray-900 rounded px-2 py-1` - Inline type code

**Metrics:**

- `text-sm font-medium text-gray-700 dark:text-gray-300` - Metric labels
- `text-lg font-semibold text-gray-800 dark:text-gray-100` - Metric values

**Severity Labels:**

- `text-xs font-semibold uppercase tracking-wide` - Severity badges (SIMPLE/MODERATE/COMPLEX)

### Layout Patterns

**Page Container:**

```tsx
className = 'px-4 sm:px-6 lg:px-8 py-8';
```

**Heatmap Container:**

```tsx
className = 'bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6 mb-6';
```

**Type Detail Panel:**

```tsx
className = 'grid grid-cols-1 lg:grid-cols-2 gap-6';
```

### Data Access Patterns

Data for this view is sourced exclusively from the TypeScript analyzer plugin. The old pattern read from the React plugin; the new pattern reads from the `typescript` plugin.

```tsx
// OLD (React plugin - Phase 21, do not use)
const typeMetrics = reactPluginResult.metrics.types;

// NEW (TypeScript plugin - Phase 21B)
const tsPluginResult = analysisResults.get('typescript');
const typeComplexity = tsPluginResult?.metrics?.['ts-type-complexity'];
const typeSafety = tsPluginResult?.metrics?.['ts-type-safety'];
const declarationShape = tsPluginResult?.metrics?.['ts-declaration-shape'];
const generics = tsPluginResult?.metrics?.['ts-generics'];
const utilityTypes = tsPluginResult?.metrics?.['ts-utility-types'];
const importDiscipline = tsPluginResult?.metrics?.['ts-import-discipline'];
const typeGuards = tsPluginResult?.metrics?.['ts-type-guards'];
const moduleAugmentation = tsPluginResult?.metrics?.['ts-module-augmentation'];
const structuralQuality = tsPluginResult?.metrics?.['ts-structural-quality'];
```

### Data Fetching Hook

The view requires a `useTypescriptComplexityData()` custom hook following the established `useAdaptiveHotspotData.ts` pattern:

```tsx
function useTypescriptComplexityData() {
  const [data, setData] = useState<TypescriptComplexityData | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchData = useCallback(async () => {
    try {
      setLoading(true);
      const result = await window.api.getTypescriptComplexity();
      setData(result);
      setError(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load data');
      addToast('error', 'Failed to load TypeScript complexity data');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  // Re-fetch when new analysis completes
  useEffect(() => {
    const unsubscribe = window.api.on('batch-complete', () => {
      fetchData();
    });
    return unsubscribe;
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}
```

### Loading, Error, and Empty States

Before rendering the main dashboard content, apply the standard three-state guard:

```tsx
const { data, loading, error } = useTypescriptComplexityData();

if (loading) {
  return (
    <div className="flex items-center justify-center h-64">
      <Spinner size="lg" />
    </div>
  );
}

if (error) {
  return (
    <Alert variant="banner" type="error">
      {error}
    </Alert>
  );
}

if (!data || data.files.length === 0) {
  return (
    <Alert variant="banner" type="info">
      No TypeScript files analyzed yet. Run an analysis to see type complexity data.
    </Alert>
  );
}

// Main dashboard render follows...
```

### Composition Patterns

#### Type Complexity Dashboard

```tsx
const [activeTab, setActiveTab] = useState<'heatmap' | 'table' | 'detail'>('heatmap');

<div className="px-4 sm:px-6 lg:px-8 py-8">
  {/* Page header */}
  <div className="flex items-center justify-between mb-6">
    <div>
      <h1 className="text-2xl font-semibold text-gray-800 dark:text-gray-100">
        TypeScript Type Complexity
      </h1>
      <p className="text-sm text-gray-600 dark:text-gray-300 mt-1">
        Identify where type-level complexity adds cognitive load
      </p>
    </div>

    <div className="flex items-center gap-3">
      <Dropdown
        variant="select"
        label="Dimension"
        options={[
          { id: 'all', label: 'All Metrics' },
          { id: 'generic', label: 'Generic Depth' },
          { id: 'conditional', label: 'Conditional Types' },
          { id: 'union', label: 'Union Size' },
          { id: 'intersection', label: 'Intersection Size' },
        ]}
        selected={selectedDimension}
        onSelect={setSelectedDimension}
      />

      <Button appearance="secondary" onClick={exportReport}>
        Export Report
      </Button>
    </div>
  </div>

  {/* Type gymnastics warning */}
  {hasTypeGymnastics && (
    <Alert variant="banner" type="warning" className="mb-6">
      <div className="space-y-2">
        <p className="font-semibold">Type Gymnastics Detected</p>
        <p className="text-sm">
          {typeGymnasticsCount} files have high type complexity with low code complexity, indicating
          over-engineered types that may add more burden than value.
        </p>
        <Button appearance="tertiary" size="sm" onClick={viewTypeGymnastics}>
          View Files
        </Button>
      </div>
    </Alert>
  )}

  {/* Tabs for different views */}
  <Tabs variant="underline">
    <Tab active={activeTab === 'heatmap'} onClick={() => setActiveTab('heatmap')}>Heatmap</Tab>
    <Tab active={activeTab === 'table'} onClick={() => setActiveTab('table')}>Table View</Tab>
    <Tab active={activeTab === 'detail'} onClick={() => setActiveTab('detail')}>Type Detail</Tab>
  </Tabs>

  {activeTab === 'heatmap' && (
    <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
            <div className="flex items-center justify-between mb-4">
              <h2 className="text-lg font-semibold text-gray-800 dark:text-gray-100">
                Type Complexity by File
              </h2>
              <div className="flex items-center gap-4">
                <div className="flex items-center gap-2 text-sm text-gray-600 dark:text-gray-400">
                  <span>Color Scale:</span>
                  <div className="flex items-center gap-1">
                    <div className="w-12 h-4 bg-green-500 rounded-l" />
                    <div className="w-12 h-4 bg-yellow-500" />
                    <div className="w-12 h-4 bg-yellow-600" />
                    <div className="w-12 h-4 bg-red-500 rounded-r" />
                  </div>
                  <span className="text-xs">0 → 100</span>
                </div>
              </div>
            </div>

            {/* @todo MetricsHeatmap uses threshold-based coloring, not custom color
                functions. If custom color scaling is needed, switch to Treemap
                or reformulate as threshold violations. */}
            <MetricsHeatmap
              files={typescriptFiles}
              metricDefs={[
                {
                  key: 'typeComplexity',
                  label: 'Type Complexity',
                  threshold: 60,
                  description: 'Composite type complexity score (0-100)',
                },
              ]}
              onFileClick={(filePath) => openFileDetail(filePath)}
              maxFiles={500}
            />

            {/* Top complex types */}
            <div className="mt-6 border-t border-gray-200 dark:border-gray-700 pt-6">
              <h3 className="text-sm font-semibold text-gray-700 dark:text-gray-300 mb-3">
                Top Complex Types
              </h3>
              <div className="space-y-2">
                {topComplexTypes.slice(0, 5).map((type, index) => (
                  <div
                    key={index}
                    className="flex items-center justify-between p-3 bg-gray-50 dark:bg-gray-900 rounded hover:bg-gray-100 dark:hover:bg-gray-800 cursor-pointer"
                    onClick={() => openTypeDetail(type)}
                  >
                    <div className="flex-1">
                      <code className="text-xs font-mono text-violet-600 dark:text-violet-400">
                        {type.name}
                      </code>
                      <span className="text-xs text-gray-600 dark:text-gray-400 ml-2">
                        - {type.file}
                      </span>
                    </div>
                    <Badge
                      variant={type.score >= 61 ? 'red' : type.score >= 41 ? 'yellow' : 'gray'}
                      size="sm"
                    >
                      {type.score}
                    </Badge>
                  </div>
                ))}
              </div>
            </div>
          </div>
  )}

  {activeTab === 'table' && (
          <CardTable
            title="Type Complexity Ranking"
            description="Files sorted by type complexity. Click to view details."
            {/* Note: CardTable sortable is a no-op. Sorting must be managed by
                the parent — pre-sort data before passing to CardTable or use a
                useSortedData wrapper hook. */}
            columns={[
              { key: 'file', label: 'File' },
              { key: 'typeComplexity', label: 'Type Complexity' },
              { key: 'codeComplexity', label: 'Code Complexity' },
              { key: 'delta', label: 'Delta' },
              { key: 'pattern', label: 'Pattern' },
            ]}
            data={typescriptFiles.map(file => ({
              id: file.id,
              file: (
                <div className="space-y-1">
                  <div className="text-sm font-medium text-gray-800 dark:text-gray-100">
                    {file.name}
                  </div>
                  <div className="text-xs font-mono text-gray-600 dark:text-gray-400">
                    {file.path}
                  </div>
                </div>
              ),
              typeComplexity: (
                <div className="flex items-center gap-2">
                  <span
                    className={cn(
                      'text-sm font-semibold',
                      file.typeComplexity >= 61 && 'text-red-600 dark:text-red-400',
                      file.typeComplexity >= 41 &&
                        file.typeComplexity < 61 &&
                        'text-yellow-600 dark:text-yellow-400',
                      file.typeComplexity < 41 && 'text-green-600 dark:text-green-400'
                    )}
                  >
                    {file.typeComplexity}
                  </span>
                </div>
              ),
              codeComplexity: (
                <span className="text-sm text-gray-700 dark:text-gray-300">
                  {file.codeComplexity}
                </span>
              ),
              delta: (
                <span
                  className={cn(
                    'text-sm font-medium',
                    file.typeComplexity - file.codeComplexity > 20 &&
                      'text-violet-600 dark:text-violet-400',
                    Math.abs(file.typeComplexity - file.codeComplexity) <= 20 &&
                      'text-gray-600 dark:text-gray-400'
                  )}
                >
                  {file.typeComplexity > file.codeComplexity ? '+' : ''}
                  {file.typeComplexity - file.codeComplexity}
                </span>
              ),
              pattern: (
                <Badge
                  variant={
                    file.typeComplexity > file.codeComplexity + 20
                      ? 'violet'
                      : file.typeComplexity > 40 && file.codeComplexity > 40
                        ? 'red'
                        : 'gray'
                  }
                  size="sm"
                >
                  {file.typeComplexity > file.codeComplexity + 20
                    ? 'Type Gymnastics'
                    : file.typeComplexity > 40 && file.codeComplexity > 40
                      ? 'Essential'
                      : 'Balanced'}
                </Badge>
              ),
            }))}
            onRowClick={(row) => openFileDetail(row.id)}
            keyExtractor={(row) => row.id}
          />
  )}

  {activeTab === 'detail' && (
          <div className="space-y-6">
            {selectedFile && (
              <>
                {/* File header */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h2 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-1">
                    {selectedFile.name}
                  </h2>
                  <p className="text-sm font-mono text-gray-600 dark:text-gray-400 mb-4">
                    {selectedFile.path}
                  </p>

                  {/* Summary comparison */}
                  <div className="grid grid-cols-2 gap-4 mb-4">
                    <div>
                      <div className="text-xs text-gray-600 dark:text-gray-400 mb-1">
                        Type Complexity
                      </div>
                      <div className="text-2xl font-bold text-violet-600 dark:text-violet-400">
                        {selectedFile.typeComplexity}
                      </div>
                    </div>
                    <div>
                      <div className="text-xs text-gray-600 dark:text-gray-400 mb-1">
                        Code Complexity
                      </div>
                      <div className="text-2xl font-bold text-gray-800 dark:text-gray-100">
                        {selectedFile.codeComplexity}
                      </div>
                    </div>
                  </div>

                  {/* Pattern identification */}
                  {selectedFile.typeComplexity > selectedFile.codeComplexity + 20 && (
                    <Alert variant="notification" type="warning">
                      <p className="text-sm font-semibold mb-1">Type Gymnastics Anti-Pattern</p>
                      <p className="text-xs">
                        This file has significantly higher type complexity than code complexity,
                        suggesting over-engineered types that may add more cognitive burden than
                        value.
                      </p>
                    </Alert>
                  )}
                </div>

                {/* Type metrics breakdown - sourced from ts-type-complexity */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Type Metrics Breakdown
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Type Complexity Metrics',
                      score: selectedFile.typeComplexity,
                      scoreLabel: `${selectedFile.typeComplexity}/100`,
                      scoreLevel: getTypeSeverity(selectedFile.typeComplexity),
                      metrics: [
                        {
                          label: 'Generic Depth',
                          value: selectedFile.metrics['ts-type-complexity'].genericDepth,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'Nesting levels of generic parameters',
                        },
                        {
                          label: 'Conditional Types',
                          value: selectedFile.metrics['ts-type-complexity'].conditionalBranches,
                          min: 0,
                          max: 20,
                          direction: 'lower-is-better',
                          description: 'Count of conditional type branches',
                        },
                        {
                          label: 'Union Size',
                          value: selectedFile.metrics['ts-type-complexity'].unionSize,
                          min: 0,
                          max: 15,
                          direction: 'lower-is-better',
                          description: 'Maximum members in union type',
                        },
                        {
                          label: 'Mapped Types',
                          value: selectedFile.metrics['ts-type-complexity'].mappedTypeCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'Count of mapped type constructs',
                        },
                      ],
                    }}
                    defaultExpanded
                  />
                </div>

                {/* Type Safety panel - sourced from ts-type-safety */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Type Safety
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Type Safety Signals',
                      metrics: [
                        {
                          label: 'any Count',
                          value: selectedFile.metrics['ts-type-safety'].anyCount,
                          min: 0,
                          max: 20,
                          direction: 'lower-is-better',
                          description: 'Explicit any annotations in this file',
                        },
                        {
                          label: 'Type Assertions (as)',
                          value: selectedFile.metrics['ts-type-safety'].typeAssertionCount,
                          min: 0,
                          max: 20,
                          direction: 'lower-is-better',
                          description: 'as T cast expressions',
                        },
                        {
                          label: '@ts-ignore',
                          value: selectedFile.metrics['ts-type-safety'].tsIgnoreCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'Suppressed TypeScript diagnostics',
                        },
                        {
                          label: 'Non-null Assertions',
                          value: selectedFile.metrics['ts-type-safety'].nonNullAssertionCount,
                          min: 0,
                          max: 20,
                          direction: 'lower-is-better',
                          description: 'Postfix ! assertions',
                        },
                        {
                          label: 'unknown Usage',
                          value: selectedFile.metrics['ts-type-safety'].unknownCount,
                          min: 0,
                          max: 20,
                          direction: 'higher-is-better',
                          description: 'unknown (preferred over any for safe typing)',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Declaration Shape summary - sourced from ts-declaration-shape */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Declaration Shape
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Declaration Counts',
                      metrics: [
                        {
                          label: 'Interfaces',
                          value: selectedFile.metrics['ts-declaration-shape'].interfaceCount,
                          min: 0,
                          max: 30,
                          description: 'Named interface declarations',
                        },
                        {
                          label: 'Type Aliases',
                          value: selectedFile.metrics['ts-declaration-shape'].typeAliasCount,
                          min: 0,
                          max: 30,
                          description: 'type = ... declarations',
                        },
                        {
                          label: 'Enums',
                          value: selectedFile.metrics['ts-declaration-shape'].enumCount,
                          min: 0,
                          max: 10,
                          description: 'Enum declarations',
                        },
                        {
                          label: 'Overloads',
                          value: selectedFile.metrics['ts-declaration-shape'].functionOverloadCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'Function overload signatures',
                        },
                        {
                          label: 'Inheritance Depth',
                          value: selectedFile.metrics['ts-declaration-shape'].maxInheritanceDepth,
                          min: 0,
                          max: 8,
                          direction: 'lower-is-better',
                          description: 'Maximum extends chain depth',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Generics Health panel - sourced from ts-generics */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Generics Health
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Generic Parameters',
                      metrics: [
                        {
                          label: 'Unconstrained Params',
                          value: selectedFile.metrics['ts-generics'].unconstrainedParamCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: '<T> with no extends constraint',
                        },
                        {
                          label: 'Broad Constraints',
                          value: selectedFile.metrics['ts-generics'].broadConstraintCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'Constraints extending object or {}',
                        },
                        {
                          label: 'Variance Annotations',
                          value: selectedFile.metrics['ts-generics'].varianceAnnotationCount,
                          min: 0,
                          max: 10,
                          direction: 'higher-is-better',
                          description: 'in / out / inout annotations (TS 4.7+)',
                        },
                        {
                          label: 'Max Arity',
                          value: selectedFile.metrics['ts-generics'].maxArity,
                          min: 0,
                          max: 8,
                          direction: 'lower-is-better',
                          description: 'Highest type parameter count in one declaration',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Import Discipline score - sourced from ts-import-discipline */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Import Discipline
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Import Type Usage',
                      metrics: [
                        {
                          label: 'import type Ratio',
                          value: selectedFile.metrics['ts-import-discipline'].importTypeRatio,
                          min: 0,
                          max: 1,
                          direction: 'higher-is-better',
                          description: 'Fraction of type-only imports using import type',
                        },
                        {
                          label: 'Type-only Exports',
                          value: selectedFile.metrics['ts-import-discipline'].typeOnlyExportCount,
                          min: 0,
                          max: 20,
                          direction: 'higher-is-better',
                          description: 'export type declarations (reduces bundler impact)',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Type Guard Coverage - sourced from ts-type-guards */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Type Guard Coverage
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Narrowing Patterns',
                      metrics: [
                        {
                          label: 'Type Predicates',
                          value: selectedFile.metrics['ts-type-guards'].typePredicateCount,
                          min: 0,
                          max: 10,
                          description: 'value is T guard functions',
                        },
                        {
                          label: 'Assertion Functions',
                          value: selectedFile.metrics['ts-type-guards'].assertionFunctionCount,
                          min: 0,
                          max: 10,
                          description: 'asserts value is T functions',
                        },
                        {
                          label: 'Discriminated Unions',
                          value: selectedFile.metrics['ts-type-guards'].discriminatedUnionCount,
                          min: 0,
                          max: 10,
                          direction: 'higher-is-better',
                          description: 'Unions with a discriminant literal property',
                        },
                        {
                          label: 'Exhaustiveness Checks',
                          value: selectedFile.metrics['ts-type-guards'].exhaustivenessCheckCount,
                          min: 0,
                          max: 10,
                          direction: 'higher-is-better',
                          description: 'never assignments used for exhaustive switches',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Structural Quality panel - sourced from ts-structural-quality */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Structural Quality
                  </h3>

                  <MetricGroup
                    data={{
                      title: 'Structural Patterns',
                      metrics: [
                        {
                          label: 'Empty Catch Blocks',
                          value: selectedFile.metrics['ts-structural-quality'].emptyCatchCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'catch blocks that silently swallow errors',
                        },
                        {
                          label: 'Untyped Throws',
                          value: selectedFile.metrics['ts-structural-quality'].untypedThrowCount,
                          min: 0,
                          max: 10,
                          direction: 'lower-is-better',
                          description: 'throw expressions without typed error objects',
                        },
                        {
                          label: 'Global State Accesses',
                          value: selectedFile.metrics['ts-structural-quality'].globalStateAccessCount,
                          min: 0,
                          max: 20,
                          direction: 'lower-is-better',
                          description: 'window/globalThis/global property accesses',
                        },
                        {
                          label: 'using Declarations',
                          value: selectedFile.metrics['ts-structural-quality'].usingDeclarationCount,
                          min: 0,
                          max: 10,
                          direction: 'higher-is-better',
                          description: 'Explicit resource management (TS 5.2+)',
                        },
                      ],
                    }}
                    defaultExpanded={false}
                  />
                </div>

                {/* Complex type definitions */}
                <div className="bg-white dark:bg-gray-800 rounded-xl shadow-xs p-6">
                  <h3 className="text-lg font-semibold text-gray-800 dark:text-gray-100 mb-4">
                    Complex Type Definitions
                  </h3>

                  <div className="space-y-4">
                    {selectedFile.complexTypes.map((type, index) => (
                      <div
                        key={index}
                        className="border border-gray-200 dark:border-gray-700 rounded-lg p-4"
                      >
                        <div className="flex items-start justify-between mb-2">
                          <code className="text-sm font-mono text-violet-600 dark:text-violet-400">
                            {type.name}
                          </code>
                          <Badge
                            variant={
                              type.complexity >= 60
                                ? 'red'
                                : type.complexity >= 40
                                  ? 'yellow'
                                  : 'gray'
                            }
                            size="sm"
                          >
                            Complexity: {type.complexity}
                          </Badge>
                        </div>

                        <div className="bg-gray-50 dark:bg-gray-900 rounded border border-gray-200 dark:border-gray-700 p-3 mb-3">
                          <pre className="text-xs font-mono text-gray-800 dark:text-gray-100 overflow-x-auto">
                            {type.definition}
                          </pre>
                        </div>

                        <div className="flex items-center gap-2">
                          <Button
                            appearance="secondary"
                            size="sm"
                            onClick={() => generateSimplificationPrompt(type)}
                          >
                            Generate Simplification Prompt
                          </Button>
                          <Button
                            appearance="tertiary"
                            size="sm"
                            onClick={() => viewTypeDependencies(type)}
                          >
                            View Dependencies
                          </Button>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>

                {/* Insights and recommendations */}
                {selectedFile.insights && selectedFile.insights.length > 0 && (
                  <div>
                    {selectedFile.insights.map((insight, index) => (
                      <InsightCard
                        key={index}
                        insight={insight}
                        defaultExpanded={index === 0}
                        onAction={action => handleInsightAction(action)}
                      />
                    ))}
                  </div>
                )}
              </>
            )}
          </div>
  )}
</div>
```

### Component Decomposition

Split the main view into focused components for maintainability and testability:

| Component File                    | Responsibility                                               |
| --------------------------------- | ------------------------------------------------------------ |
| `TypescriptComplexityHeatmap.tsx` | Heatmap tab content: MetricsHeatmap + top complex types list |
| `TypescriptComplexityTable.tsx`   | Table tab content: CardTable with sorting wrapper            |
| `TypescriptFileDetailPanel.tsx`   | Detail tab content: all MetricGroup panels + insights        |
| `useTypescriptComplexityData.ts`  | Data fetching hook (see Data Fetching Hook section)          |

Each component receives only the props it needs from the parent page component, which owns the `activeTab` state and calls `useTypescriptComplexityData()`.

### Severity Utility

Define a `getTypeSeverity(score: number)` utility for consistent color and label mapping:

```tsx
type SeverityLevel = 'success' | 'warning' | 'error' | 'critical';

function getTypeSeverity(score: number): SeverityLevel {
  if (score >= 61) return 'critical';
  if (score >= 41) return 'error';
  if (score >= 21) return 'warning';
  return 'success';
}
```

This function should live alongside the view components and reference the canonical `TYPE_COMPLEXITY_SCORE_BANDS` constants from `analyzers/typescript/src/constants/thresholds.ts`.

### Implementation Notes

**Treemap colorScale callback**: If the Treemap component is used for custom color scaling (see MetricsHeatmap note above), the inline arrow function passed to `colorScale` must be wrapped in `useCallback` to prevent unnecessary re-renders of the Treemap's internal canvas/SVG.

**Keyboard accessibility for top complex types list**: The `<div onClick>` elements in the "Top Complex Types" list must be keyboard accessible. Add `role="button"`, `tabIndex={0}`, and an `onKeyDown` handler that triggers on Enter/Space:

```tsx
<div
  role="button"
  tabIndex={0}
  onClick={() => openTypeDetail(type)}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      openTypeDetail(type);
    }
  }}
  className="..."
>
```

### Type Signature Styling Pattern

**For inline type code:**

```tsx
<code className="font-mono text-xs bg-gray-50 dark:bg-gray-900 text-violet-600 dark:text-violet-400 px-2 py-1 rounded">
  {typeName}
</code>
```

**For type definition blocks:**

```tsx
<div className="bg-gray-50 dark:bg-gray-900 rounded border border-gray-200 dark:border-gray-700 p-3">
  <pre className="text-xs font-mono text-gray-800 dark:text-gray-100 overflow-x-auto whitespace-pre">
    {typeDefinition}
  </pre>
</div>
```

### Responsive Behavior

**Mobile (< 640px):**

- Heatmap uses reduced grid size (max 20x20 instead of 60x60)
- Type detail panel collapses to single column
- Table hides Delta and Pattern columns
- Type definition code blocks scroll horizontally

**Tablet (640px - 1024px):**

- Heatmap uses medium grid (30x30)
- Type detail shows 1 column
- Table shows all columns

**Desktop (1024px+):**

- Full heatmap grid (up to 60x60)
- Type detail uses 2-column layout
- All table columns visible

### Dark Mode Considerations

All components adapt automatically:

- Code blocks: `bg-gray-50` -> `bg-gray-900`, `border-gray-200` -> `border-gray-700`
- Type names: `text-violet-600` -> `text-violet-400`
- MetricBarChart: Uses alpha variants for fill colors
- InsightCard: Background and text adapt seamlessly

---

## Design System Gaps

### Gap 1: Scatter Plot / Bubble Chart - MEDIUM PRIORITY

**Description:** XY scatter chart to visualize correlation between type complexity and code complexity.

**Current Impact on Phase 21B:**

- Cannot visualize the four quadrants spatially:
  - Type Gymnastics (high type, low code)
  - Essential Complexity (high both)
  - Runtime Complexity (low type, high code)
  - Simple (low both)
- Cannot show correlation pattern visually
- Missing spatial clustering that reveals patterns

**Interim Solution (Implemented Above):**

Use **CardTable with two metric columns:**

```tsx
<CardTable
  columns={[
    { key: 'file', label: 'File' },
    { key: 'typeComplexity', label: 'Type Complexity' },
    { key: 'codeComplexity', label: 'Code Complexity' },
    { key: 'delta', label: 'Delta' },
    { key: 'pattern', label: 'Pattern' },
  ]}
  // Delta column = typeComplexity - codeComplexity
  // Pattern column = Badge showing quadrant classification
/>
```

**Why this works:**

- Provides all the same data (two metrics per file)
- Sortable by either metric to find extremes
- Delta column shows relationship strength
- Pattern Badge classifies files into quadrants
- Searchable and filterable
- Accessible (screen readers)

**Missing from table approach:**

- Visual clustering of similar files
- Quadrant boundaries (invisible in table)
- Spatial correlation visualization
- Outlier detection by visual inspection

**Future Enhancement Options:**

**Option 1: Chart.js Scatter Plugin**

- Use Chart.js built-in scatter chart type
- Pros: Already have Chart.js as dependency, simple to add
- Cons: Limited customization, basic quadrant support
- Timeline: 1-2 days

**Option 2: Recharts Library**

- Add Recharts for more sophisticated scatter charts
- Pros: Better customization, React-native, responsive
- Cons: Additional dependency (~60KB), different API than Chart.js
- Timeline: 3-5 days

**Option 3: Custom D3 Scatter**

- Build custom scatter plot with D3
- Pros: Full control, can add quadrant lines, labels
- Cons: Significant development effort, state management
- Timeline: 1-2 weeks

**Recommendation:** Start with table-only approach (implemented above). Add Chart.js scatter only if:

1. Users explicitly request spatial visualization
2. Pattern detection requires visual clustering
3. Team has bandwidth for 1-2 day implementation

**User Impact:** Low-Medium - table provides 80% of value, scatter adds polish

---

## Visual Concepts

**NOTE:** Visual concepts updated to reflect table-based implementation for type vs. code correlation. Scatter plot deferred to future enhancement (see Design System Gaps).

### Type Complexity Heatmap

```
TypeScript Type Complexity
================================================================================

View: [Type Complexity v]    Dimension: [All Metrics v]    [Compare with Code]

+------------------------------------------------------------------+
|  src/types/                                                       |
|  +------------------------+  +----------------------------------+ |
|  | api-types.ts          |  | utility-types.ts                 | |
|  | Generic Depth: 5      |  | Conditional: 12                  | |
|  | Union Size: 8         |  | Mapped: 7                        | |
|  |                        |  |                                  | |
|  |      YELLOW-600         |  |         RED                      | |
|  +------------------------+  +----------------------------------+ |
|                              +----------------------------------+ |
|  +------------------------+  | domain-types.ts                  | |
|  | component-props.ts     |  | Union Size: 3                    | |
|  | Generic Depth: 2       |  | Generic Depth: 2                 | |
|  |                        |  |                                  | |
|  |      GREEN             |  |      GREEN                       | |
|  +------------------------+  +----------------------------------+ |
+------------------------------------------------------------------+

Color Scale (Type Complexity):
[  GREEN  |  YELLOW  |  YELLOW-600  |   RED   ]
    0-20      21-40      41-60     61+

Top Complex Types:
1. `ApiResponse<T, E>` - utility-types.ts - Score: 78
2. `RouteConfig` - api-types.ts - Score: 65
3. `ComponentProps<T>` - component-props.ts - Score: 42

================================================================================
```

### Type vs. Code Complexity Matrix

```
Type vs. Code Complexity Scatter
================================================================================

Type
Complexity
100 |
    |                    [domain-types.ts]
 80 |         [api-types.ts]
    |                                    TYPE GYMNASTICS
 60 |                                    (Simplify types)
    |  [utils.ts]
 40 |            [component-props.ts]   ESSENTIAL COMPLEXITY
    |                                    (Accept or refactor both)
 20 |  [Button.tsx]
    |               [hooks.ts]          RUNTIME COMPLEXITY
  0 +----------------------------------------> Code Complexity
    0      20      40      60      80     100

QUADRANTS:

Bottom-Left (Low Type, Low Code): Simple, maintainable code
Bottom-Right (Low Type, High Code): Runtime complexity, types are fine
Top-Left (High Type, Low Code): TYPE GYMNASTICS - simplify types
Top-Right (High Type, High Code): Essential complexity or needs refactoring

================================================================================
```

### Type Detail Panel

```
+------------------------------------------------------------------+
| Type Complexity: api-types.ts                                     |
+------------------------------------------------------------------+
| Overall Score: 65 (Complex)                                       |
|                                                                   |
| Breakdown:                                                        |
| - Generic Depth: 5         [========--------] High              |
| - Conditional Branches: 3  [=====-----------] Moderate          |
| - Union Size: 8            [=======---------] High              |
| - Mapped Types: 2          [===--------------] Low              |
|                                                                   |
| -- Type Safety --------------------------------------------------  |
| - any Count: 0             [----------------] None              |
| - as Assertions: 2         [===--------------] Low              |
| - @ts-ignore: 0            [----------------] None              |
|                                                                   |
| -- Generics Health -----------------------------------------------
| - Unconstrained Params: 1  [==--------------] Low              |
| - Max Arity: 3             [====-----------] Moderate           |
| - Variance Annotations: 0  [----------------] None              |
|                                                                   |
| -- Complex Types ----------------------------------------------- |
|                                                                   |
| 1. ApiResponse<T, E = Error>                         Score: 78   |
|    Location: line 24                                             |
|    Used by: 47 files                                             |
|                                                                   |
|    type ApiResponse<T, E = Error> =                              |
|      | { status: 'success'; data: T }                             |
|      | { status: 'error'; error: E }                              |
|      | { status: 'loading' }                                      |
|      | { status: 'idle' };                                        |
|                                                                   |
|    Complexity Factors:                                           |
|    - Discriminated union (4 branches)                            |
|    - Generic with default parameter                              |
|    - Used extensively (high propagation)                         |
|                                                                   |
|    Suggestion: Consider simpler alternatives for common cases    |
|    [Generate Simplification Prompt]   [View Propagation]         |
|                                                                   |
+------------------------------------------------------------------+
| [Export Type Report]   [Compare with Code Complexity]            |
+------------------------------------------------------------------+
```

### Type Propagation Graph

```
Type Propagation: ApiResponse<T, E>
================================================================================

ApiResponse<T, E> (api-types.ts)
  |
  +-- DIRECT USAGE (12 files)
  |   |
  |   +-- hooks/useApi.ts
  |   +-- services/user-service.ts
  |   +-- services/product-service.ts
  |   +-- ... (9 more)
  |
  +-- INDIRECT USAGE (35 files)
      |
      +-- hooks/useUser.ts (via useApi)
      +-- hooks/useProducts.ts (via useApi)
      +-- components/UserProfile.tsx (via useUser)
      +-- pages/Dashboard.tsx (via useUser, useProducts)
      +-- ... (31 more)

IMPACT ANALYSIS:

Total Affected Files: 47
Propagation Depth: 4 levels
Breaking Change Risk: HIGH

If you change ApiResponse, you must update:
- 12 direct consumers
- Review 35 indirect consumers
- Consider compatibility wrappers

[Show Full Dependency Tree]   [Generate Migration Prompt]

================================================================================
```

---

## Complexity Analysis Methodology

The TypeScript analyzer plugin (`@vipr/typescript`) performs all complexity calculations. The score formula below reflects the weights used in the plugin's `buildCompositeScore()` method. Phase 21B reads the pre-computed score from the plugin; it does not recalculate it in the renderer.

### Type Complexity Score Calculation

```
TypeComplexity = (
  genericDepth × 2.0 +
  conditionalBranches × 3.0 +
  unionSize × 0.5 +
  intersectionSize × 0.8 +
  mappedTypeCount × 2.5 +
  recursiveTypes × 5.0 +
  templateLiteralComplexity × 1.5 +
  typeParameterCount × 1.0 +
  inferKeywordCount × 2.0
)

Where weights reflect cognitive load per unit.
```

These weights are defined in `analyzers/typescript/src/constants/weights.ts` as `TYPE_COMPLEXITY_WEIGHTS`. Recursive types carry the highest penalty (5.0) because they can cause IDE slowdown and are the hardest to reason about. Conditional branches (3.0) are weighted above generic depth (2.0) because nested conditionals compound exponentially.

### Meaningful Thresholds

| Type Score | Classification | Interpretation                         | Action                    |
| ---------- | -------------- | -------------------------------------- | ------------------------- |
| 0-20       | Simple Types   | Easy to understand, low cognitive load | None needed               |
| 21-40      | Moderate Types | Requires attention but manageable      | Monitor                   |
| 41-60      | Complex Types  | Significant cognitive overhead         | Review for simplification |
| 61+        | Very Complex   | High maintenance burden, IDE slowdown  | Priority refactoring      |

### Threshold Justification

**Generic Depth Threshold: 3 levels**

- Depth 1: `Array<T>` - Trivial
- Depth 2: `Promise<Result<T>>` - Common pattern
- Depth 3: `Array<Promise<Result<T>>>` - Cognitive limit
- Depth 4+: `Map<K, Array<Promise<Result<T>>>>` - Difficult to reason about

**Union Size Threshold: 10 members**

- 2-4 members: Discriminated union pattern (good)
- 5-10 members: State machine or enum-like (acceptable)
- 11-20 members: Difficult to exhaustively handle
- 21+ members: Consider alternative approaches

**Conditional Types Threshold: 5 in file**

- 1-2: Common utility types (good)
- 3-5: Type-level logic emerging (moderate)
- 6-10: Significant type-level programming (complex)
- 11+: Type system being abused (refactor)

### Pattern Recognition

**Type Anti-Patterns That Indicate Problems:**

1. **Generic Soup**
   - Pattern: `<T, U, V, W, X, Y, Z>` - Many type parameters
   - Problem: Impossible to track relationships
   - Detection: typeParameterCount > 5 in single type
   - Example: `function complex<T, U, V, W, X>(a: T, b: U, c: V, d: W, e: X): ...`
   - Solution: Extract to multiple simpler functions

2. **Conditional Type Maze**
   - Pattern: Nested conditional types 3+ levels deep
   - Problem: Mental stack overflow
   - Detection: conditionalBranches > 5, nesting > 2
   - Example: Nested conditional types with multiple branches
   - Solution: Extract to helper types with descriptive names

3. **Union Explosion**
   - Pattern: 15+ member union type
   - Problem: Exhaustive handling is tedious
   - Detection: unionSize > 15
   - Example: `type Status = 'a' | 'b' | 'c' | ... | 'o' | 'p'`
   - Solution: Group into discriminated unions or use string literals with validation

4. **Recursive Type Hell**
   - Pattern: Deeply recursive type without clear termination
   - Problem: Type checking slowness, IDE hangs
   - Detection: recursiveTypes > 0, genericDepth > 4
   - Example: `type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> }`
   - Solution: Limit recursion depth or use library utilities

5. **Type Gymnastics**
   - Pattern: High type complexity, low code complexity
   - Problem: Types doing work that could be simpler
   - Detection: typeScore > 60, cyclomaticComplexity < 20
   - Example: Complex mapped types for simple transformations
   - Solution: Use runtime validation or simpler types

---

## Detection Algorithms

### Type Complexity vs. Code Complexity Correlation

```
FOR each file:
  typeScore = calculateTypeComplexity(file)
  codeScore = calculateCodeComplexity(file)

  IF typeScore > 60 AND codeScore < 30:
    FLAG as "type_gymnastics"
    SUGGEST: Simplify types, consider runtime validation

  ELSE IF typeScore > 50 AND codeScore > 50:
    FLAG as "essential_complexity"
    SUGGEST: Accept or refactor both type and code

  ELSE IF typeScore < 30 AND codeScore > 60:
    FLAG as "runtime_complexity"
    SUGGEST: Types are fine, focus on code refactoring

  ELSE IF typeScore < 30 AND codeScore < 30:
    FLAG as "simple_code"
    SUGGEST: No action needed
```

### Type Propagation Analysis

```
function analyzeTypePropagation(typeName: string, sourceFile: string):
  directUsages = findDirectImports(typeName, sourceFile)
  indirectUsages = []

  FOR each directUsage:
    transitiveUsages = findTransitiveImports(directUsage.file)
    indirectUsages.extend(transitiveUsages)

  propagationDepth = calculateMaxDepth(directUsages, indirectUsages)
  propagationBreadth = directUsages.length + indirectUsages.length

  blastRadius = (
    typeComplexityScore × propagationBreadth × (1 + 0.2 × propagationDepth)
  )

  RETURN {
    directUsages,
    indirectUsages,
    depth: propagationDepth,
    breadth: propagationBreadth,
    blastRadius
  }
```

### Type Anti-Pattern Detection

```
function detectTypeAntiPatterns(file: SourceFile):
  anti-patterns = []

  // Generic Soup
  FOR each function/type:
    IF typeParameters.length > 5:
      anti-patterns.add({
        type: "generic_soup",
        severity: "warning",
        location: function.location,
        suggestion: "Extract to multiple simpler functions"
      })

  // Conditional Type Maze
  conditionalDepth = calculateConditionalNesting(file)
  IF conditionalDepth > 2:
    anti-patterns.add({
      type: "conditional_maze",
      severity: "warning",
      suggestion: "Extract conditional types to named helpers"
    })

  // Recursive Type Hell
  FOR each recursive type:
    IF genericDepth > 4:
      anti-patterns.add({
        type: "recursive_hell",
        severity: "critical",
        suggestion: "Limit recursion or use library utilities"
      })

  // Type Gymnastics
  IF typeComplexity > 60 AND codeComplexity < 30:
    anti-patterns.add({
      type: "type_gymnastics",
      severity: "info",
      suggestion: "Consider simpler types or runtime validation"
    })

  RETURN anti-patterns
```

### Alert Triggers

| Condition                            | Alert Type        | Notification              |
| ------------------------------------ | ----------------- | ------------------------- |
| Type complexity > 70                 | Critical          | Immediate notification    |
| Type propagation affects 50+ files   | High Impact       | Daily digest              |
| Type vs. code mismatch (gymnastics)  | Type Anti-Pattern | Weekly summary            |
| Type complexity increases 20+ points | Regression        | Snapshot comparison alert |
| Generic depth > 5                    | Deep Nesting      | Code review flag          |

---

## Integration with Existing Features

### Cognitive Load Heatmap (Phase 18)

**Addition: Type Complexity Layer**

The cognitive load heatmap should offer type complexity as a separate dimension alongside Halstead effort and cognitive complexity. Data comes from `metrics['ts-type-complexity']` on the TypeScript plugin result.

**Implementation:**

- Add "Type Complexity" to metric dropdown
- Type complexity uses same color scale (0-100)
- Comparison mode: Side-by-side with code complexity
- Hybrid mode: Blended view (average of type and code)

**Insight Generation:**

When both type and code complexity are high, generate insight:

```
"This file has high complexity in both types (65) and code (72).
Consider refactoring both or documenting the essential complexity."
```

When type is high but code is low:

```
"Type complexity (68) significantly exceeds code complexity (22).
Review types for over-engineering or unnecessary type-level logic."
```

### Blast Radius Hotspot View (Phase 01)

**Addition: Type Propagation Factor**

Blast radius should account for type propagation, not just import dependencies.

**Enhanced Formula:**

```
BlastRadius = (
  ComplexityScore × DependencyFactor × TypePropagationFactor
)

Where:
  TypePropagationFactor = 1 + (0.1 × typePropagationBreadth)
```

**Why:** A complex generic type used in 50 components creates massive blast radius even if the file itself has low code complexity.

**Example:**

- File: `api-types.ts`
- Code Complexity: 25 (low)
- Type Complexity: 72 (high, from `ts-type-complexity`)
- Type Propagation: 47 files
- Result: Blast radius elevated from 50 to 85 due to type propagation

**Visual Indicator:**

Add badge to blast radius view showing type propagation:

```
[api-types.ts]
Blast Radius: 85
├─ Code Complexity: 25
├─ Dependencies: 12 direct
└─ Type Propagation: 47 files [!]
```

### Complexity Velocity Dashboard (Phase 02)

**Addition: Type Complexity Velocity Track**

Track type complexity changes over time alongside code complexity. Source type velocity data from the `ts-type-complexity` analysis rows in the snapshots table.

**New Chart: Dual-Track Velocity**

```
Complexity Velocity (Dual Track)
================================================================================

Score
100 |
    |                     CODE COMPLEXITY
 75 |                ___---    \
    |          ___---           \___
 50 |    ___--                      ---___
    |
 25 |              TYPE COMPLEXITY
    |         ___----------___
  0 +-----|-----|-----|-----|-----|-----|-----|-----|-----|-->
      Jan 5  Jan 10 Jan 15 Jan 20 Jan 25 Jan 30 Feb 1  Feb 5

Code Velocity: -1.2 pts/week [Improving]
Type Velocity: +0.8 pts/week [Degrading]

ANALYSIS: Type complexity increasing while code complexity decreasing.
Possible over-engineering or excessive type safety efforts.
================================================================================
```

**Insights from Divergent Trends:**

1. **Type velocity up, code velocity down:** Possible over-engineering
2. **Both velocity up:** Technical debt accumulation
3. **Type velocity down, code velocity up:** Good refactoring (simplifying types while adding features)
4. **Both velocity down:** Successful refactoring campaign

### Architectural Anti-Patterns Detection (Phase 04)

**Addition: Type-Level Anti-Patterns**

Extend architectural anti-patterns to include type-specific anti-patterns. Detection signals come from the TypeScript plugin analyses listed in the table below.

**New Anti-Pattern Categories:**

| Anti-Pattern              | Description                                    | Detection Signal (TypeScript Plugin)                                                               |
| ------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Type Gymnastics**       | Overly complex types for simple problems       | `ts-type-complexity.score` > 60, cyclomatic < 30                                                   |
| **Generic Soup**          | Too many type parameters                       | `ts-generics.maxArity` > 5 in single declaration                                                   |
| **Conditional Maze**      | Nested conditional types                       | `ts-type-complexity.conditionalBranches` > 5, nesting > 2                                          |
| **Type Propagation Bomb** | Type used in 50+ files                         | typePropagationBreadth > 50                                                                        |
| **Recursive Type Hell**   | Deep recursive types                           | `ts-type-complexity.recursiveTypes` > 0, `ts-type-complexity.genericDepth` > 4                     |
| **Union Explosion**       | Massive union types                            | `ts-type-complexity.unionSize` > 15                                                                |
| **Safety Erosion**        | Heavy any / @ts-ignore usage                   | `ts-type-safety.anyCount` + `ts-type-safety.tsIgnoreCount` > 10                                    |
| **Structural Decay**      | Empty catches, global state, singleton overuse | `ts-structural-quality.emptyCatchCount` > 3 or `ts-structural-quality.globalStateAccessCount` > 10 |

**Example Anti-Pattern Report:**

```
Type Gymnastics Anti-Pattern
================================================================================

File: utils/types.ts
Severity: Warning

This file has high type complexity (68) but low code complexity (22).
The types are doing more work than the runtime code.

Complex Types:
1. `DeepPartial<T>` - Recursive mapped type with conditionals
2. `StrictOmit<T, K>` - Complex conditional type for omitting keys
3. `PathsToProps<T>` - Template literal type maze

RECOMMENDATION:
Consider using established library utilities (e.g., type-fest) rather than
implementing complex type transformations. For unique cases, ensure types
are well-documented and tested with type assertions.

[Generate Simplification Prompt]   [Mark as Justified]
================================================================================
```

---

## Psychological Principles

### Separating Type from Code Complexity

Developers mentally separate "type problems" from "code problems." Visualizing them separately:

- Validates their experience ("these types ARE hard")
- Enables targeted action (simplify types without touching code)
- Prevents conflation (high type complexity != bad code)

### Making Invisible Work Visible

Type complexity is invisible in traditional metrics. Showing it explicitly:

- Acknowledges the cognitive load developers experience
- Justifies time spent understanding types
- Validates requests for type simplification

### Framing Type Complexity as a Trade-off

Avoid framing complex types as "bad." Instead, frame as trade-off:

- High type complexity + high safety = worthwhile
- High type complexity + low bug prevention = over-engineering
- Context determines whether complexity is justified

### Type Propagation as Blast Radius

Using the same "blast radius" mental model for types:

- Leverages existing understanding
- Shows consequences of type changes
- Encourages consideration before modifying widely-used types

---

## Success Metrics

| Metric                         | Target       | Measurement                                        |
| ------------------------------ | ------------ | -------------------------------------------------- |
| Type hotspot identification    | < 10 seconds | Time to find highest type complexity file          |
| Type vs. code comparison usage | > 40%        | Users who view type/code comparison table          |
| Type anti-pattern recognition  | > 60%        | Users who correctly identify type gymnastics       |
| Type simplification prompts    | > 25%        | Type anti-patterns leading to AI prompt generation |
| Type propagation review        | > 50%        | Users who check propagation before changing type   |

---

## Example Scenarios

### Scenario 1: Type Gymnastics Detection

**File:** `utils/type-helpers.ts`
**Metrics:**

- Type Complexity: 78 (Very Complex) - from `ts-type-complexity.score`
- Code Complexity: 18 (Low)
- Type/Code Ratio: 4.3x

**Analysis:**
The file implements custom utility types with deep recursion and conditional logic:

```typescript
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;

type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

type PathsToProps<T> = T extends object
  ? {
      [K in keyof T]: K extends string
        ? T[K] extends object
          ? `${K}` | `${K}.${PathsToProps<T[K]>}`
          : `${K}`
        : never;
    }[keyof T]
  : never;
```

**Complexity Breakdown:**

- DeepPartial: Recursive (4 pts), Conditional (2.5 pts), Mapped (2 pts) = 8.5
- StrictOmit: Mapped (2 pts), Generic depth 2 (6 pts) = 8
- PathsToProps: Recursive (4 pts), Template literal (4.5 pts), Nested conditionals (7.5 pts) = 16

**Recommendation:**
Replace with established library utilities:

```typescript
// Before (custom, complex)
type DeepPartial<T> = ...

// After (library, simple)
import type { PartialDeep } from 'type-fest';
```

**Expected Outcome:**
Type complexity drops from 78 to 25. Maintenance burden reduced. IDE performance improves.

---

### Scenario 2: Type Propagation Bomb

**File:** `types/api.ts`
**Type:** `ApiResponse<T, E = Error>`
**Metrics:**

- Type Complexity: 45 (Moderate) - from `ts-type-complexity.score`
- Direct Usage: 23 files
- Indirect Usage: 68 files
- Propagation Depth: 5 levels
- Blast Radius: 92 (Critical)

**Analysis:**
The type itself is moderately complex, but it propagates through the entire application:

```
ApiResponse (api.ts)
  ├─ useApi (hooks/useApi.ts) - 12 usages
  │   ├─ useUser (hooks/useUser.ts) - 15 usages
  │   │   ├─ UserProfile (components/UserProfile.tsx)
  │   │   ├─ UserSettings (components/UserSettings.tsx)
  │   │   └─ ... (13 more)
  │   ├─ useProducts (hooks/useProducts.ts) - 18 usages
  │   └─ ... (10 more hooks)
  ├─ apiService (services/api.ts) - 8 usages
  └─ ... (11 more direct usages)
```

**Impact:**
Any change to `ApiResponse` requires:

1. Reviewing 23 direct consumers
2. Testing 68 indirect consumers
3. Potentially breaking 91 total files

**Recommendation:**
Before modifying `ApiResponse`:

1. Create compatibility wrapper for transition
2. Use feature flags for gradual rollout
3. Generate migration guide with AI prompt
4. Consider versioning strategy (ApiResponseV2)

---

### Scenario 3: Essential Type Complexity

**File:** `state-machine/types.ts`
**Metrics:**

- Type Complexity: 68 (Complex) - from `ts-type-complexity.score`
- Code Complexity: 62 (High)
- Both High: Essential Complexity

**Analysis:**
The file implements a type-safe state machine with strict transitions:

```typescript
type State = 'idle' | 'loading' | 'success' | 'error';

type Transition<From extends State, To extends State> = {
  from: From;
  to: To;
  action: string;
};

type ValidTransitions =
  | Transition<'idle', 'loading'>
  | Transition<'loading', 'success'>
  | Transition<'loading', 'error'>
  | Transition<'success', 'idle'>
  | Transition<'error', 'idle'>;

type Machine<S extends State> = {
  current: S;
  transition<T extends Extract<ValidTransitions, { from: S }>>(transition: T): Machine<T['to']>;
};
```

**Complexity Justification:**

- Type complexity ensures impossible states are unrepresentable
- Catches invalid transitions at compile time
- Complexity reflects problem domain complexity (state machines are inherently complex)

**Recommendation:**
Accept the complexity but:

1. Document thoroughly with examples
2. Provide helper functions for common patterns
3. Mark as "justified complexity" in anti-pattern detector
4. Ensure comprehensive type tests

**Outcome:**
Type complexity marked as "essential" rather than "anti-pattern." No refactoring needed.

---

### Scenario 4: Type Velocity Regression

**Timeline:** 6 weeks
**Starting Type Score:** 35 (Moderate)
**Ending Type Score:** 58 (Complex)
**Type Velocity:** +3.8 pts/week
**Code Velocity:** +1.2 pts/week

**Analysis:**
Type complexity growing faster than code complexity. Investigation reveals:

Week 2: Added complex generic utilities (+8 type pts)
Week 4: Implemented conditional type helpers (+10 type pts)
Week 6: Created recursive type transformations (+5 type pts)

**Root Cause:**
Team experimenting with advanced TypeScript features without considering maintenance burden.

**Recommendation:**

1. Pause new type-level features
2. Review recent type additions for simplification
3. Establish type complexity budget
4. Create guidelines for when to use advanced types
5. Target: Return to type score ~40 within 4 weeks

---

## Open Questions

1. **Type vs. Code Weight:** Should type complexity weight equally with code complexity in overall scores, or should it be a separate dimension?

2. **Type Anti-Pattern Severity:** What threshold makes type complexity an anti-pattern vs. acceptable? Should this be configurable per project?

3. **IDE Performance Correlation:** Can we measure IDE slowdown and correlate it with type complexity scores?

4. **Type Testing:** Should we encourage type tests (using TypeScript's type system to verify type behavior) for complex types?

5. **Library Types:** Should type complexity from node_modules be tracked separately? Some libraries have intentionally complex types.

6. **Type Inference Complexity:** Should we track type inference complexity (how hard for TypeScript to infer types) vs. readability complexity?

7. **Additional Analysis Weighting:** Should `ts-type-safety` signals (any count, @ts-ignore) factor into the composite type health score, or remain a separate reporting dimension?

---

## Implementation Recommendations

> **Dependency:** Phase 21A (TypeScript Analyzer Plugin) must be completed and merged before any work on Phase 21B begins. The `@vipr/typescript` plugin and its IPC surface are the sole data source for this visualization.

### Phase 1: Foundation (Weeks 1-2)

Requires: `ts-type-complexity` analysis available from Phase 21A.

1. Add type complexity to file detail view (reads `metrics['ts-type-complexity']`)
2. Show type complexity score alongside code complexity
3. Implement basic type heatmap
4. Wire up DB queries against `plugin_id = 'typescript'` (see Technical Considerations)

### Phase 2: Integration (Weeks 3-4)

Requires: All nine analyses available from Phase 21A.

1. Add type complexity layer to cognitive heatmap (Phase 18)
2. Integrate type propagation factor into blast radius (Phase 01)
3. Add type complexity velocity track to velocity dashboard (Phase 02)
4. Surface `ts-type-safety` and `ts-declaration-shape` panels in file detail

### Phase 3: Advanced (Weeks 5-6)

1. Surface `ts-generics`, `ts-import-discipline`, `ts-type-guards`, `ts-module-augmentation`, and `ts-structural-quality` panels in file detail
2. Add type-specific anti-patterns to architectural anti-patterns (Phase 04) using the extended detection signals from Phase 21A
3. Create type propagation visualization
4. Generate type-aware AI prompts using the full TypeScript plugin metrics shape

### Phase 4: Refinement (Weeks 7-8)

1. User testing and feedback incorporation
2. Threshold calibration based on real codebases
3. Documentation and onboarding materials
4. Performance optimization for large codebases

---

## Technical Considerations

### IPC Types

The renderer receives type metrics via the IPC layer. The TypeScript plugin exposes its metrics under the following shape:

```typescript
// Source of truth: analyzers/typescript/src/types/index.ts
interface TypeScriptPluginMetrics {
  'ts-type-complexity': TypeComplexity;
  'ts-type-safety': TypeSafety;
  'ts-declaration-shape': DeclarationShape;
  'ts-generics': GenericsMetrics;
  'ts-utility-types': UtilityTypeAnalysis;
  'ts-import-discipline': ImportDiscipline;
  'ts-type-guards': TypeGuardAnalysis;
  'ts-module-augmentation': ModuleAugmentation;
  'ts-structural-quality': StructuralQuality;
}

interface TypeComplexity {
  score: number; // Quality score 0-100. Higher is better (simpler type system).
  genericDepth: number;
  conditionalBranches: number;
  distributiveConditionalCount: number;
  unionSize: number;
  intersectionSize: number;
  mappedTypeCount: number;
  mappedTypeWithRemappingCount: number;
  recursiveTypes: number;
  maxRecursionDepth: number;
  templateLiteralComplexity: number;
  typeParameterCount: number;
  inferKeywordCount: number;
  variadicTupleCount: number;
  indexedAccessDepth: number;
  insights: string[];
  examples: ComplexTypeExample[];
}

interface ComplexTypeExample {
  type: string;
  complexity: number;
  location: string;
}

interface TypeSafety {
  score: number; // Quality score 0-100. Higher is better.
  anyCount: number;
  typeAssertionCount: number;
  doubleAssertionCount: number;
  tsIgnoreCount: number;
  tsExpectErrorCount: number;
  tsNoCheckCount: number;
  nonNullAssertionCount: number;
  implicitAnyCatchCount: number;
  indexSignatureAbuseCount: number;
  unknownCount: number;
  narrowingPatternCount: number;
  satisfiesCount: number;
  readonlyPropertyCount: number;
  brandedTypeCount: number;
  resultTypeCount: number;
  insights: string[];
}

interface DeclarationShape {
  score: number;
  interfaceCount: number;
  typeAliasCount: number;
  enumCount: number;
  constEnumCount: number;
  namespaceCount: number;
  classCount: number;
  abstractClassCount: number;
  emptyInterfaceCount: number;
  maxInterfacePropertyCount: number;
  avgInterfaceMemberCount: number;
  functionOverloadCount: number;
  declarationMergingCount: number;
  maxInheritanceDepth: number;
  avgTypeAliasComplexity: number;
  exportedFunctionCount: number;
  exportedFunctionWithExplicitReturnCount: number;
  readonlyPropertyRatio: number;
  maxValueParameterCount: number;
  insights: string[];
}

interface GenericsMetrics {
  score: number;
  unconstrainedParamCount: number;
  broadConstraintCount: number;
  unusedTypeParamCount: number;
  singleUseTypeParamCount: number;
  varianceAnnotationCount: number;
  constTypeParamCount: number;
  maxArity: number;
  defaultTypeParamCount: number;
  constraintComplexityScore: number;
  insights: string[];
}

interface UtilityTypeAnalysis {
  score: number;
  builtinUtilityCount: number;
  uniqueUtilitiesUsed: number;
  customMappedTypeCount: number;
  redundantCustomTypeCount: number;
  utilityUsageBreakdown: Record<string, number>;
  insights: string[];
}

interface ImportDiscipline {
  score: number;
  totalImports: number;
  typeOnlyImportCount: number;
  typeOnlyExportCount: number;
  namespaceImportCount: number;
  importTypeRatio: number;
  valueImportsCarryingTypes: number;
  reExportCount: number;
  reExportRatio: number;
  insights: string[];
}

interface TypeGuardAnalysis {
  score: number;
  typePredicateCount: number;
  assertionFunctionCount: number;
  discriminatedUnionCount: number;
  exhaustivenessCheckCount: number;
  typeofGuardCount: number;
  instanceofGuardCount: number;
  inOperatorGuardCount: number;
  nullishCheckCount: number;
  userDefinedGuardCount: number;
  insights: string[];
}

interface ModuleAugmentation {
  score: number;
  declareModuleCount: number;
  declareGlobalCount: number;
  ambientModuleCount: number;
  augmentationTargets: string[];
  globalPollutionScore: number;
  insights: string[];
}

interface StructuralQuality {
  score: number;
  singletonPatternCount: number;
  factoryFunctionCount: number;
  constructorInjectionCount: number;
  concreteClassImportCount: number;
  throwStatementCount: number;
  untypedThrowCount: number;
  emptyCatchCount: number;
  switchCaseCount: number;
  decoratorCount: number;
  legacyDecoratorCount: number;
  usingDeclarationCount: number;
  typeToRuntimeRatio: number;
  globalStateAccessCount: number;
  insights: string[];
}
```

### Data Storage

Phase 21B reads pre-computed metrics stored by the TypeScript analyzer plugin. The key change from Phase 21 is the plugin identifier and the nested metric key structure.

```sql
-- OLD (Phase 21): Query type metrics from React plugin
SELECT json_extract(a_react.metrics, '$.types.genericDepth') AS genericDepth,
       json_extract(a_react.metrics, '$.types.conditionalBranches') AS conditionalBranches
FROM analyses a_react
WHERE a_react.plugin_id = 'react'
  AND a_react.file_id = ?;

-- NEW (Phase 21B): Query type metrics from TypeScript plugin
SELECT json_extract(a_ts.metrics, '$."ts-type-complexity".genericDepth') AS genericDepth,
       json_extract(a_ts.metrics, '$."ts-type-complexity".conditionalBranches') AS conditionalBranches,
       json_extract(a_ts.metrics, '$."ts-type-safety".anyCount') AS anyCount,
       json_extract(a_ts.metrics, '$."ts-type-safety".tsIgnoreCount') AS tsIgnoreCount,
       json_extract(a_ts.metrics, '$."ts-generics".maxArity') AS maxArity,
       a_ts.score AS compositeScore
FROM analyses a_ts
WHERE a_ts.plugin_id = 'typescript'
  AND a_ts.file_id = ?;
```

**Analysis IDs stored as metric keys:**

| Key                      | Content                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| `ts-type-complexity`     | Core type complexity sub-metrics and score                                                        |
| `ts-type-safety`         | any, typeAssertionCount, @ts-ignore, nonNullAssertionCount, etc.                                  |
| `ts-declaration-shape`   | interfaceCount, typeAliasCount, enumCount, functionOverloadCount, maxInheritanceDepth, etc.       |
| `ts-generics`            | unconstrainedParamCount, broadConstraintCount, maxArity, etc.                                     |
| `ts-utility-types`       | builtinUtilityCount, uniqueUtilitiesUsed, redundantCustomTypeCount                                |
| `ts-import-discipline`   | importTypeRatio, typeOnlyExportCount, valueImportsCarryingTypes                                   |
| `ts-type-guards`         | typePredicateCount, discriminatedUnionCount, exhaustivenessCheckCount                             |
| `ts-module-augmentation` | declareModuleCount, declareGlobalCount, ambientModuleCount, globalPollutionScore                  |
| `ts-structural-quality`  | emptyCatchCount, untypedThrowCount, globalStateAccessCount, decoratorCount, usingDeclarationCount |

The composite TypeScript health score is stored at `analyses.score` for the plugin row and is computed by the plugin's `buildCompositeScore()` method. Phase 21B reads this value directly rather than recomputing it in the renderer or main process.

### Performance

Type analysis can be slow on large codebases. Optimize by:

- Caching type complexity calculations (handled in Phase 21A plugin)
- Computing propagation on-demand (lazy)
- Incremental analysis (only changed files)
- Background processing for visualization
- Aggregate metrics in main process before sending to renderer (avoid 1000+ item payloads)

### Accuracy

Type complexity scoring is heuristic-based. Improve accuracy by:

- Calibrating weights against real-world codebases
- Allowing user feedback on false positives
- Machine learning on "marked as justified" patterns
- Community-driven threshold refinement

---

## Conclusion

TypeScript type complexity is a distinct dimension of code maintainability that deserves separate visualization and analysis. Phase 21B makes the dedicated TypeScript analyzer plugin (Phase 21A) visible to developers by surfacing its nine analyses in a coherent UI.

By making type complexity visible, measurable, and actionable, Vipr can help teams:

1. Identify where type complexity adds cognitive load
2. Balance type safety with maintainability
3. Detect type-level anti-patterns using richer signals than Phase 21 provided
4. Make informed decisions about type system usage
5. Track type complexity trends over time

The key insight remains that type complexity is not inherently bad but must be considered as a trade-off. Complex types that catch bugs and enable safe refactoring are worthwhile. Complex types that exist for their own sake are technical debt. The additional analyses from the TypeScript plugin (type safety, generics health, import discipline, type guard coverage, structural quality) give developers the context to make that judgment accurately.
