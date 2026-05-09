---
id: 07-polish
---

# phase 5 polish - implementation plan

comprehensive technical specifications and architecture review plan for phase 5 polish deliverables.

---

## engineering team executive summary

### project status

- **Current Phase**: Phase 5 - Polish and optimization
- **Active Work Streams**: 5 parallel development tracks
- **Completion Status**: 0% complete (0 of 9 features implemented)
- **Quality Gates**: PENDING initial implementation
- **Timeline**: 3-4 weeks estimated (US-12 deliverables)

### committee activity summary

- **Active Tasks**: 9 primary tasks across 6 subagents
- **Completed Features**: 0 features ready for review
- **Pending Reviews**: All features pending implementation
- **Blocking Issues**: 0 issues requiring immediate attention

---

## part a: technical proposal

### 1. advanced visualization implementation

<!-- ARCHITECTURE NOTE: Consider extract visualization logic into a shared package (e.g., @vipr/visualizations)
     to enable reuse across desktop and future web clients. This aligns with the monorepo structure. -->

#### 1.1 d3.js treemap visualization

**Purpose**: Visualize directory structure with complexity color-coding for large codebases.

<!-- PERFORMANCE NOTE: The 500-node threshold for canvas fallback is arbitrary. Consider measuring actual
     performance on target hardware (M1/M2 Macs, modern Windows laptops) and adjusting dynamically based on
     device capabilities using performance.memory and requestIdleCallback. -->

**Technical specifications:**

| Aspect             | Implementation                                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Library**        | D3.js v7+ (import sub-modules only: `d3-hierarchy`, `d3-scale`, `d3-selection`, `d3-interpolate`, `d3-zoom` -- ~60KB vs ~500KB for full bundle)        |
| **Data structure** | Hierarchical file tree with aggregated metrics                                                                                                         |
| **Color scale**    | `d3.scaleSequential(d3.interpolateRdYlGn).domain([100, 0])` (reversed: green=low, red=high). Colorblind-safe alt: `d3.interpolateViridis` via settings |
| **Interaction**    | Zoom/pan via `d3-zoom`, tooltip on hover, click to drill-down into children                                                                            |
| **Performance**    | SVG for under 500 nodes, Canvas 2D context for 500+ (validate threshold on target hardware)                                                            |
| **Dimensions**     | Responsive with minimum 800x600px viewport, graceful table-view fallback below threshold                                                               |
| **Dark mode**      | Adapts stroke, label fill, and container background colors via CSS custom properties                                                                   |

<!-- ACCESSIBILITY NOTE: Minimum viewport of 800x600px may exclude users with smaller screens or accessibility
     needs. Consider supporting 640x480px minimum with graceful degradation or alternative table view. -->

**Data model:**

```typescript
// TYPE SAFETY NOTE: Consider using branded types for file paths and adding validation
// to prevent injection attacks or invalid path characters in the treemap rendering.
interface TreemapNode {
  name: string;
  path: string;
  value: number; // File size or line count -- IMPORTANT: must be > 0, d3.treemap silently drops zero-value nodes
  complexity: number; // 0-100 score for color mapping
  children?: TreemapNode[];
  fileType?: string; // CONSISTENCY NOTE: Should align with FileType from @vipr/common
  issues?: number;
  metadata?: Record<string, unknown>; // Extensible for future metrics without breaking the interface
}

// ARCHITECTURE NOTE: Config object should be split into data concerns (dimensions, padding)
// and behavior concerns (callbacks). Consider using composition pattern:
// TreemapConfig extends TreemapDimensions, TreemapInteractions
interface TreemapConfig {
  width: number;
  height: number;
  padding: number; // Inner padding between sibling nodes (px)
  topPadding: number; // Top padding for directory group labels (px)
  colorScale: d3.ScaleSequential<string>; // Use D3 scale type for proper type safety
  colorblindMode: boolean; // Toggle between RdYlGn and Viridis palettes
  onNodeClick: (node: TreemapNode) => void;
  onNodeHover: (node: TreemapNode | null) => void;
  maxDepth: number; // Maximum drill-down depth (default: 3) to prevent illegible deep nesting
  minNodeArea: number; // Minimum pixel area to render a node (default: 25) -- key perf optimization
}
```

**Component structure:**

```typescript
// src/renderer/components/charts/TreemapChart.tsx

// REVIEWER NOTE: Use granular D3 imports for tree-shaking. The original `import * as d3`
// imports the entire ~500KB D3 bundle, which is incompatible with the < 1MB renderer bundle
// target in section 4.2.
import { hierarchy, treemap, treemapSquarify } from 'd3-hierarchy';
import { scaleSequential } from 'd3-scale';
import { interpolateRdYlGn, interpolateViridis } from 'd3-interpolate';
import { select } from 'd3-selection';
import { zoom } from 'd3-zoom';
import { useEffect, useRef, useState, useMemo } from 'react';

export function TreemapChart({ data, config }: TreemapChartProps) {
  const svgRef = useRef<SVGSVGElement>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [tooltip, setTooltip] = useState<TooltipState | null>(null);

  // Choose rendering strategy based on node count
  const nodeCount = useMemo(() => countNodes(data), [data]);
  const useCanvas = nodeCount > 500;

  useEffect(() => {
    if (!data) return;

    // REVIEWER NOTE: The original was missing a cleanup return. Without it, event listeners
    // accumulate on re-renders, causing memory leaks and duplicate handler invocations.
    if (useCanvas) {
      const canvas = canvasRef.current;
      if (!canvas) return;
      return renderTreemapCanvas(canvas, data, config, setTooltip);
    }

    const svg = svgRef.current;
    if (!svg) return;
    return renderTreemapSvg(svg, data, config, setTooltip);
  }, [data, config, useCanvas]);

  return (
    <div className="relative" role="img" aria-label="Code complexity treemap visualization">
      {useCanvas ? (
        <canvas
          ref={canvasRef}
          width={config.width * window.devicePixelRatio}
          height={config.height * window.devicePixelRatio}
          style={{ width: config.width, height: config.height }}
        />
      ) : (
        <svg ref={svgRef} className="w-full h-full" />
      )}
      {tooltip && <TreemapTooltip {...tooltip} />}
    </div>
  );
}
```

<!-- REVIEWER NOTE: Key changes to the component structure:
1. Granular D3 imports (~60KB) instead of star import (~500KB) for bundle size compliance
2. Canvas/SVG dual-mode rendering matching the spec table (original only had SVG ref)
3. Cleanup return in useEffect to prevent memory leaks on unmount/re-render
4. devicePixelRatio scaling for Canvas to support retina/HiDPI displays
5. ARIA role="img" and aria-label for screen reader accessibility
6. useMemo for nodeCount to avoid recounting on every render -->

**Rendering algorithm:**

1. **Data transformation**: If input is a flat file list, first transform into nested `TreemapNode` structure by splitting paths on `/` and grouping into directory nodes. This is a prerequisite for `d3.hierarchy()`.
2. **Hierarchy construction**: Use `hierarchy(rootNode).sum(d => d.value).sort((a, b) => b.value - a.value)` to propagate values upward and ensure stable, deterministic layout.
3. **Layout calculation**: Use `treemap().tile(treemapSquarify).size([width, height]).paddingInner(config.padding).paddingTop(config.topPadding)` for squarified tiling with group label space.
4. **Node filtering**: Skip rendering nodes with computed area smaller than `config.minNodeArea` pixels. This typically eliminates 30-50% of leaf nodes at full zoom, which is the primary performance optimization.
5. **Color mapping**: Map complexity scores through the configured `colorScale` (reversed RdYlGn by default, Viridis for colorblind mode). Ensure minimum 10 Delta-E perceptual difference between adjacent 10-point complexity buckets.
6. **Rendering**: SVG mode: use D3 selection to create/update `<rect>` and `<text>` elements with enter/update/exit pattern. Canvas mode: use `fillRect()` and `fillText()` with `measureText()` for label truncation. Apply `getComputedTextLength()` for SVG text fitting.
7. **Interaction**: Attach `d3-zoom` behavior for zoom/pan. Click on directory node drills down (re-roots hierarchy). Click on leaf node navigates to file detail. Hover shows positioned tooltip with debounce-free `requestAnimationFrame` throttling.

**Performance optimizations:**

<!-- REVIEWER NOTE: The original "virtualize rendering for 500+ nodes" is not a standard approach
     for treemaps. Treemap nodes have non-uniform sizes and positions, unlike list items. The standard
     optimization is minNodeArea filtering -- nodes too small to see are not rendered. This typically
     removes 30-50% of nodes at full zoom. -->

- Filter out nodes below `minNodeArea` threshold before rendering (primary optimization, ~30-50% node reduction)
- Use Canvas 2D context for trees with 500+ visible nodes (eliminates DOM node overhead entirely)
- Debounce resize events at 150ms with `requestAnimationFrame` batching (300ms feels sluggish)
- Memoize hierarchy calculation via `useMemo` when data reference is unchanged (separate layout from paint)
- For Canvas mode, implement quadtree spatial index for O(log n) hover hit-testing instead of O(n) linear scan
- Use `requestAnimationFrame` to batch zoom/pan updates (never re-layout on interaction, only re-paint)
- Consider Web Worker for hierarchy construction on datasets exceeding 5000 files to prevent main-thread blocking

**Acceptance criteria:**

- Treemap renders in under 500ms for 1000 file repository on baseline hardware (2020 M1 MacBook Air or equivalent). This is realistic: D3 hierarchy + treemap layout for 1000 nodes takes ~50ms, Canvas paint takes ~20ms, leaving ample headroom for data transformation.
- Color scale accurately reflects complexity scores with minimum 10 Delta-E perceptual difference between adjacent 10-point complexity buckets
- Colorblind-safe alternative palette (Viridis) is toggleable via user settings
- Zoom/pan interaction is smooth (60fps target, max 2 dropped frames per second measured via Chrome DevTools Performance profiler)
- Tooltips display file path, complexity, line count, issue count. File paths must be sanitized (HTML-escaped) to prevent XSS from malicious repository names.
- Click on leaf node opens file detail view; click on directory drills down. Enter key on focused node is the keyboard equivalent.
- Responsive to window resize: re-layout completes within 200ms (debounced at 150ms), maintains 60fps during resize drag
- Canvas mode renders correctly on retina/HiDPI displays (devicePixelRatio-aware sizing)
- Keyboard navigation: Tab to focus treemap, arrow keys to navigate between nodes, Enter to drill down or open detail, Escape to zoom out
- Screen readers receive summary content via `role="img"` and `aria-label` attributes

#### 1.2 custom canvas heatmap

**Purpose**: Display file churn vs complexity matrix for identifying high-risk areas.

**Technical specifications:**

| Aspect             | Implementation                                                                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Rendering**      | HTML5 Canvas API with devicePixelRatio scaling for retina displays                                                                                                        |
| **Data structure** | 2D bucketed matrix: files grouped into churn buckets (x-axis) and complexity buckets (y-axis), each cell aggregates files in that bucket                                  |
| **Color scale**    | Sequential scale (not diverging -- risk is unidirectional, 0=safe to 100=risky). Use `d3.interpolateYlOrRd` (yellow-orange-red). Colorblind-safe: `d3.interpolateInferno` |
| **Interaction**    | Canvas-based hover detection via pre-computed cell boundary lookup, tooltip positioning relative to viewport                                                              |
| **Grid**           | Fixed cell size with scrollable viewport for matrices exceeding container dimensions                                                                                      |
| **Labels**         | Rotated file names or bucket range labels on x-axis, complexity bucket ranges on y-axis                                                                                   |

<!-- REVIEWER NOTE: The original specified a "diverging" color scale for risk, but risk scores
     are unidirectional (0 = no risk, 100 = maximum risk). Diverging scales are for data with a
     meaningful midpoint (e.g., positive/negative values). A sequential scale (YlOrRd) is correct
     for risk visualization. Also, green-to-red fails for ~8% of males with deuteranopia. -->

**Data model:**

```typescript
// Individual file record used as input before bucketing
interface HeatmapFileRecord {
  fileId: string;
  filePath: string;
  churnScore: number; // 0-100 (normalized change frequency)
  complexityScore: number; // 0-100
  riskScore: number; // Derived: churnScore * complexityScore / 100
  issueCount: number;
}

// A single cell in the bucketed grid -- represents an aggregation of files
interface HeatmapBucketCell {
  xBucket: number; // Churn bucket index
  yBucket: number; // Complexity bucket index
  churnRange: [number, number]; // e.g., [20, 30] for the churn bucket
  complexityRange: [number, number]; // e.g., [40, 50] for the complexity bucket
  files: HeatmapFileRecord[]; // Files that fall in this bucket
  avgRiskScore: number; // Average risk across files in bucket (used for color)
  fileCount: number; // Number of files in bucket
}

interface HeatmapConfig {
  width: number;
  height: number;
  cellSize: number; // Pixel size per grid cell
  bucketCount: number; // Grid resolution, default 10 (produces 10x10 grid)
  padding: { top: number; right: number; bottom: number; left: number };
  colorScale: d3.ScaleSequential<string>; // Use D3 scale type for consistency with treemap
  onCellClick: (cell: HeatmapBucketCell) => void;
  onCellHover: (cell: HeatmapBucketCell | null) => void;
}

interface HeatmapData {
  grid: HeatmapBucketCell[][]; // 2D array indexed by [yBucket][xBucket]
  xBucketCount: number; // Number of churn buckets
  yBucketCount: number; // Number of complexity buckets
  totalFiles: number; // Total files across all buckets
}
```

<!-- REVIEWER NOTE: The original data model conflated individual file cells with the bucketed
     grid structure. The rendering algorithm says "divide files into churn/complexity buckets"
     but the data model had flat `cells: HeatmapCell[]` with no bucket concept. This revised
     model separates the input (HeatmapFileRecord) from the bucketed output (HeatmapBucketCell)
     and uses a 2D grid array for O(1) cell access during hover detection. -->

**Component structure:**

```typescript
// src/renderer/components/charts/HeatmapChart.tsx
import { useEffect, useRef, useState, useCallback } from 'react';

export function HeatmapChart({ data, config }: HeatmapChartProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [hoveredCell, setHoveredCell] = useState<HeatmapBucketCell | null>(null);
  const [mousePosition, setMousePosition] = useState({ x: 0, y: 0 });

  // REVIEWER NOTE: Use requestAnimationFrame throttling instead of debouncing for hover.
  // Debouncing adds minimum latency equal to the debounce interval, whereas rAF throttling
  // fires on the next frame boundary (~16ms) without adding extra delay.
  const rafRef = useRef<number>(0);

  const handleMouseMove = useCallback((event: MouseEvent) => {
    cancelAnimationFrame(rafRef.current);
    rafRef.current = requestAnimationFrame(() => {
      const canvas = canvasRef.current;
      if (!canvas) return;

      const rect = canvas.getBoundingClientRect();
      const dpr = window.devicePixelRatio || 1;
      const x = (event.clientX - rect.left) * dpr;
      const y = (event.clientY - rect.top) * dpr;
      setMousePosition({ x: event.clientX, y: event.clientY });

      // O(1) cell lookup using grid indices instead of boundary scanning
      const cell = getCellAtPosition(x, y, data, config);
      setHoveredCell(cell);
      config.onCellHover?.(cell);
    });
  }, [data, config]);

  // REVIEWER NOTE: The original code referenced `handleClick` but never defined it.
  // This is a bug that would cause a ReferenceError at runtime.
  const handleClick = useCallback((event: MouseEvent) => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const rect = canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;
    const x = (event.clientX - rect.left) * dpr;
    const y = (event.clientY - rect.top) * dpr;

    const cell = getCellAtPosition(x, y, data, config);
    if (cell) config.onCellClick?.(cell);
  }, [data, config]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    // Scale context for HiDPI displays
    const dpr = window.devicePixelRatio || 1;
    ctx.scale(dpr, dpr);

    renderHeatmap(ctx, data, config);

    canvas.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('click', handleClick);

    return () => {
      canvas.removeEventListener('mousemove', handleMouseMove);
      canvas.removeEventListener('click', handleClick);
      cancelAnimationFrame(rafRef.current);
    };
  }, [data, config, handleMouseMove, handleClick]);

  return (
    <div className="relative">
      <canvas
        ref={canvasRef}
        width={config.width * (window.devicePixelRatio || 1)}
        height={config.height * (window.devicePixelRatio || 1)}
        style={{ width: config.width, height: config.height }}
        className="border border-neutral-200 dark:border-neutral-700"
        role="img"
        aria-label={`File risk heatmap: ${data.totalFiles} files across ${data.xBucketCount} churn and ${data.yBucketCount} complexity buckets`}
      />
      {hoveredCell && (
        <HeatmapTooltip
          cell={hoveredCell}
          x={mousePosition.x}
          y={mousePosition.y}
        />
      )}
    </div>
  );
}
```

<!-- REVIEWER NOTE: Critical fixes to the heatmap component:
1. handleClick was referenced but never defined -- this would cause a ReferenceError at runtime
2. Canvas was not DPI-scaled -- renders blurry on retina displays (most modern Macs)
3. Hover detection now uses rAF throttling instead of debouncing for zero-latency response
4. Added dark mode border class
5. Added ARIA attributes for screen reader accessibility
6. Added cancelAnimationFrame cleanup on unmount to prevent orphaned callbacks
7. DPI-aware coordinate translation for accurate hit-testing on retina displays -->

**Rendering algorithm:**

<!-- ARCHITECTURE NOTE: Extract bucketing logic into pure function for unit testing.
     Consider implementing as a separate utility in @vipr/common for potential CLI usage. -->

1. **Bucketing**: Divide files into churn/complexity buckets (10x10 grid) <!-- CONFIGURATION: Make bucket count configurable (default 10x10, allow 5x5 to 20x20) -->
2. **Cell calculation**: Calculate cell positions and sizes based on config <!-- OPTIMIZATION: Pre-calculate and cache cell boundaries in WeakMap keyed by config object -->
3. **Color mapping**: Apply diverging color scale to risk scores <!-- ACCESSIBILITY: Ensure diverging scale works for deuteranopia/protanopia (most common color blindness types) -->
4. **Canvas drawing**: Draw cells with `fillRect()`, apply cell borders <!-- PERFORMANCE: Use single fillRect for all borders (draw background, then cells) rather than individual border draws -->
5. **Labels**: Draw axis labels with `fillText()`, rotate x-axis labels <!-- RENDER: Pre-render labels to offscreen canvas, then composite - avoids expensive text rendering on every frame -->
6. **Hover detection**: Calculate cell boundaries, detect mouse position <!-- OPTIMIZATION: Use spatial hash/quadtree for O(1) hover detection instead of O(n) boundary checks -->

**Performance optimizations:**

- Use `requestAnimationFrame` for smooth canvas updates <!-- PATTERN: Implement frame budget monitoring - skip non-critical renders if frame time > 13ms -->
- Debounce hover events (16ms for 60fps) <!-- INCORRECT: Debouncing adds latency. Use requestAnimationFrame throttling instead for smoother response -->
- Pre-calculate cell boundaries in data structure <!-- MEMORY: Consider trade-off - pre-calculation adds ~40 bytes per cell. For 10,000 files, this is 400KB - acceptable -->
- Use offscreen canvas for label rendering (cache) <!-- IMPLEMENTATION: Invalidate cache when window.devicePixelRatio changes (user moves window between displays) -->
- Implement dirty region tracking (only redraw changed cells) <!-- COMPLEXITY WARNING: Dirty region tracking adds significant complexity. Profile first to confirm this optimization is necessary - full redraws may be fast enough at 100x100 grid -->

**Acceptance criteria:**

- Heatmap renders 1000+ files in under 300ms (realistic: bucketing is O(n), Canvas fill for a 10x10 grid is ~100 `fillRect` calls at < 1ms total)
- Hover detection is accurate within 1px and responsive (zero-latency via rAF throttling, not debouncing)
- Color scale (YlOrRd sequential) clearly distinguishes at least 5 risk levels with minimum 15 Delta-E perceptual difference
- Colorblind-safe alternative (Inferno) available via settings toggle
- Tooltips display bucket ranges, file count, average risk score, and list of top 3 files by risk
- Click on bucket cell shows drill-down list of files in that bucket
- Scrollable viewport for grids exceeding container dimensions
- Canvas renders correctly on retina/HiDPI displays
- Keyboard: Tab to focus heatmap, arrow keys to navigate cells, Enter to drill down into bucket
- Screen reader: aria-label summarizes total files and grid dimensions

#### 1.3 adaptive visualization strategy

**Purpose**: Automatically select optimal visualization based on dataset characteristics.

<!-- ARCHITECTURE NOTE: This decision logic should be declarative and data-driven rather than imperative.
     Consider implementing as a scoring system where each visualization gets a score based on dataset
     characteristics, then select the highest-scoring option. This makes the logic more maintainable
     and allows for A/B testing different strategies. -->

<!-- USER EXPERIENCE NOTE: Adaptive selection may be surprising to users. Consider showing a visualization
     recommendation with option to view alternatives: "We recommend treemap for your 350 files. View as
     table | heatmap | chart instead". This preserves user agency while providing smart defaults. -->

**Decision matrix:**

| Dataset Size                | Complexity Distribution | Recommended Visualization                                                                                                                                      |
| --------------------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Small (1-50 files)**      | Any                     | Table with inline metrics                                                                                                                                      |
| **Medium (51-200 files)**   | Uniform                 | Bar chart (sorted by score)                                                                                                                                    |
| **Medium (51-200 files)**   | Varied                  | Doughnut chart (grouped by score range) <!-- REVIEW: Doughnut charts are notoriously hard to read. Consider horizontal bar chart with color coding instead --> |
| **Large (201-500 files)**   | Any                     | Treemap (directory hierarchy)                                                                                                                                  |
| **Very Large (501+ files)** | High churn variance     | Heatmap (churn vs complexity)                                                                                                                                  |
| **Very Large (501+ files)** | Low churn variance      | Virtualized table with filters                                                                                                                                 |

<!-- DATA VALIDATION: These thresholds (50, 200, 500) are arbitrary. Consider conducting user testing
     with actual repositories of varying sizes to validate these breakpoints empirically. -->

**Implementation:**

```typescript
// src/renderer/utils/visualization-selector.ts
import type { FileRecord } from '../types';

interface DatasetCharacteristics {
  fileCount: number;
  complexityCoefficientOfVariation: number; // Normalized 0-1 (CV = stddev / mean)
  churnCoefficientOfVariation: number; // Normalized 0-1
  avgComplexity: number;
  hasDirectoryStructure: boolean;
}

type VisualizationType =
  | 'table'
  | 'bar-chart'
  | 'doughnut-chart'
  | 'treemap'
  | 'heatmap'
  | 'virtualized-table';

export function selectVisualization(files: FileRecord[]): VisualizationType {
  if (files.length === 0) return 'table'; // Guard against empty input

  const characteristics = analyzeDataset(files);

  if (characteristics.fileCount <= 50) {
    return 'table';
  }

  if (characteristics.fileCount <= 200) {
    // REVIEWER NOTE: Changed from doughnut-chart to horizontal bar chart.
    // Doughnut/pie charts are notoriously difficult to read due to human
    // inability to accurately compare arc angles. A grouped bar chart with
    // color-coded complexity ranges communicates the same information more
    // effectively. If doughnut is retained, limit to max 6 segments.
    return characteristics.complexityCoefficientOfVariation > 0.5
      ? 'bar-chart' // Was 'doughnut-chart' -- bar chart is more readable
      : 'bar-chart';
  }

  if (characteristics.fileCount <= 500) {
    return characteristics.hasDirectoryStructure ? 'treemap' : 'bar-chart';
  }

  // 501+ files
  if (characteristics.churnCoefficientOfVariation > 0.5) {
    return 'heatmap';
  }

  return 'virtualized-table';
}

function analyzeDataset(files: FileRecord[]): DatasetCharacteristics {
  const complexityScores = files.map(f => f.complexityScore || 0);
  const churnScores = files.map(f => f.churnScore || 0);

  return {
    fileCount: files.length,
    complexityCoefficientOfVariation: calculateCoefficientOfVariation(complexityScores),
    churnCoefficientOfVariation: calculateCoefficientOfVariation(churnScores),
    avgComplexity: calculateMean(complexityScores),
    hasDirectoryStructure: files.some(f => f.path.includes('/')),
  };
}

// REVIEWER NOTE: The original used raw variance, but the thresholds (0.3, 0.4) assumed
// normalized 0-1 values. Raw variance on 0-100 scores yields values in range 0-2500,
// making the 0.3/0.4 thresholds meaningless (always exceeded). The coefficient of
// variation (CV = stddev / mean) produces a dimensionless 0-1+ ratio that works
// correctly with the threshold comparisons.
function calculateCoefficientOfVariation(values: number[]): number {
  if (values.length === 0) return 0;
  const mean = calculateMean(values);
  if (mean === 0) return 0; // Avoid division by zero
  const variance = calculateVariance(values, mean);
  return Math.sqrt(variance) / mean; // CV = stddev / mean
}

function calculateVariance(values: number[], mean?: number): number {
  if (values.length === 0) return 0;
  const m = mean ?? calculateMean(values);
  const squareDiffs = values.map(value => (value - m) ** 2);
  return squareDiffs.reduce((sum, val) => sum + val, 0) / values.length;
}

function calculateMean(values: number[]): number {
  if (values.length === 0) return 0;
  return values.reduce((sum, val) => sum + val, 0) / values.length;
}

// PERFORMANCE NOTE: For 10,000+ files, use Welford's online algorithm to compute
// mean and variance in a single pass without intermediate array allocation:
//   let mean = 0, m2 = 0;
//   for (let i = 0; i < values.length; i++) {
//     const delta = values[i] - mean;
//     mean += delta / (i + 1);
//     m2 += delta * (values[i] - mean);
//   }
//   return m2 / values.length; // variance
```

<!-- REVIEWER NOTE: Critical statistical bug fixed. The original calculateVariance() returned
     raw variance (range 0-2500 for 0-100 input scores), but the selection logic compared
     against thresholds of 0.3 and 0.4 which assumed normalized values. This meant:
     - complexityVariance > 0.3 was ALWAYS true for any non-trivial dataset
     - churnVariance > 0.4 was ALWAYS true for any non-trivial dataset
     The fix uses coefficient of variation (CV = stddev/mean) which produces a dimensionless
     ratio typically in the 0-2 range, making the 0.5 threshold meaningful.

     Additionally fixed:
     - hasDirectoryStructure was computed but never used in selection logic (now used for
       500-file treemap decision: treemaps require hierarchy, so fall back to bar chart
       if no directory structure exists)
     - Empty array guard added at the top level
     - AnalysisStore removed from imports (unused)
     - Doughnut chart replaced with bar chart (see note in code) -->

**Component integration:**

```typescript
// src/renderer/components/AdaptiveVisualization.tsx
import { useMemo } from 'react';
import { selectVisualization } from '../utils/visualization-selector';
import { TreemapChart } from './charts/TreemapChart';
import { HeatmapChart } from './charts/HeatmapChart';
import { FileTable } from './FileTable';
import { DoughnutChart } from './charts/DoughnutChart';
import { BarChart } from './charts/BarChart';
import { VirtualizedFileTable } from './VirtualizedFileTable';

export function AdaptiveVisualization({ files }: { files: FileRecord[] }) {
  const vizType = useMemo(() => selectVisualization(files), [files]);

  switch (vizType) {
    case 'table':
      return <FileTable files={files} />;
    case 'bar-chart':
      return <BarChart data={prepareBarChartData(files)} />;
    case 'doughnut-chart':
      return <DoughnutChart data={prepareDoughnutData(files)} />;
    case 'treemap':
      return <TreemapChart data={prepareTreemapData(files)} config={treemapConfig} />;
    case 'heatmap':
      return <HeatmapChart data={prepareHeatmapData(files)} config={heatmapConfig} />;
    case 'virtualized-table':
      return <VirtualizedFileTable files={files} />;
  }
}
```

**Acceptance criteria:**

- Visualization selection completes in under 5ms for 1000 files (this is pure arithmetic -- mean + variance calculation on an array of 1000 numbers should take < 1ms)
- Selected visualization enables users to identify the top 5 highest-risk files within 30 seconds during user testing (90% success rate target)
- Smooth transition between visualization types: use opacity crossfade (200ms) when switching types to avoid jarring layout shifts
- User can manually override via a dropdown in the visualization toolbar (not buried in settings). Recommendation text: "Recommended: treemap. Switch to: table | heatmap | chart"
- Per-repository visualization preference persists via the settings store (more useful than global preference since different repos have different characteristics)

---

### 2. error handling and recovery

<!-- ARCHITECTURE REVIEW: Error handling strategy is comprehensive but consider adding error budgeting -
     track error rates over time and alert if thresholds exceeded (e.g., >1% of operations failing).
     This helps catch degradation before it impacts users significantly. -->

<!-- SECURITY NOTE: Error boundaries should sanitize error messages in production to prevent information
     disclosure. Stack traces and file paths should be logged to main process only, never shown to users
     in production builds. -->

#### 2.1 error boundary implementation

**Purpose**: Gracefully catch and recover from React component errors.

<!-- REACT 18 NOTE: React 18+ supports error boundaries for Suspense boundaries. Consider separate
     boundaries for async data loading vs component render errors for more granular recovery. -->

**Component structure:**

```typescript
// src/renderer/components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react';
import type { ErrorInfo } from 'react';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: (error: Error, retry: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  errorId: string | null;
  retryCount: number; // SECURITY FINDING #3 (HIGH): Track retry attempts
  lastRetryTime: number; // SECURITY FINDING #3 (HIGH): Rate limit retries
}

export class ErrorBoundary extends Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  // SECURITY FINDING #3 (HIGH): Add rate limiting constants
  private static readonly MAX_RETRIES = 3;
  private static readonly RETRY_COOLDOWN_MS = 2000; // 2 seconds between retries

  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorId: null,
      retryCount: 0,
      lastRetryTime: 0,
    };
  }

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return {
      hasError: true,
      error,
      errorId: generateErrorId(),
      retryCount: 0, // Reset count on new error
    };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    const isDevelopment = import.meta.env.DEV;

    // SECURITY FINDING #2 (HIGH): Only log to console in development
    if (isDevelopment) {
      console.error('ErrorBoundary caught error:', error, errorInfo);
    }

    this.props.onError?.(error, errorInfo);

    // SECURITY FINDING #2 (HIGH): Sanitize before sending to main process
    window.viprDesktop.logging.error({
      errorId: this.state.errorId || 'unknown',
      message: sanitizeErrorMessage(error.message),
      stack: isDevelopment ? error.stack : undefined, // Only include stack in dev
      componentStack: this.sanitizeComponentStack(errorInfo.componentStack, isDevelopment),
      timestamp: Date.now(),
      errorHash: this.generateErrorHash(error),
    });
  }

  private sanitizeComponentStack(stack: string | null | undefined, isDevelopment: boolean): string | undefined {
    if (!stack) return undefined;
    if (isDevelopment) return stack;

    // In production, remove file paths but keep component names
    let sanitized = stack.replace(/\s+\([^)]+\)/g, ''); // Remove (at path:line:col)
    sanitized = sanitized.replace(/\s+in\s+[\w\/\-\.]+$/gm, ''); // Remove file references
    return sanitized;
  }

  private generateErrorHash(error: Error): string {
    const firstFrame = error.stack?.split('\n')[1] || '';
    return `${error.name}:${error.message}:${firstFrame}`.slice(0, 100);
  }

  // SECURITY FINDING #3 (HIGH): Add rate limiting and max retry enforcement
  retry = () => {
    const now = Date.now();
    const timeSinceLastRetry = now - this.state.lastRetryTime;

    // Check retry cooldown
    if (timeSinceLastRetry < ErrorBoundary.RETRY_COOLDOWN_MS) {
      const waitSeconds = Math.ceil((ErrorBoundary.RETRY_COOLDOWN_MS - timeSinceLastRetry) / 1000);
      // Show toast notification about cooldown (implement toast system)
      console.warn(`Please wait ${waitSeconds}s before retrying`);
      return;
    }

    // Check max retries
    if (this.state.retryCount >= ErrorBoundary.MAX_RETRIES) {
      console.error('Maximum retries exceeded. Please reload the application.');
      return;
    }

    // Increment retry count and reset error state
    this.setState((prevState) => ({
      hasError: false,
      error: null,
      retryCount: prevState.retryCount + 1,
      lastRetryTime: now,
    }));
  };

  render() {
    if (this.state.hasError && this.state.error) {
      const retriesRemaining = ErrorBoundary.MAX_RETRIES - this.state.retryCount;
      const canRetry = retriesRemaining > 0;

      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.retry);
      }

      return (
        <DefaultErrorFallback
          error={this.state.error}
          errorId={this.state.errorId || 'unknown'}
          retry={this.retry}
          canRetry={canRetry}
          retriesRemaining={retriesRemaining}
        />
      );
    }

    return this.props.children;
  }
}

// SECURITY FINDING #1 (HIGH): The original implementation exposed sensitive data in error messages.
// Error messages may contain absolute file paths, stack traces, and internal component structure.
// REMEDIATION: Sanitize error messages in production, use error IDs for correlation.
function DefaultErrorFallback({
  error,
  errorId,
  retry,
  canRetry,
  retriesRemaining,
}: {
  error: Error;
  errorId: string;
  retry: () => void;
  canRetry: boolean;
  retriesRemaining: number;
}) {
  const isDevelopment = import.meta.env.DEV;

  // Sanitize error message in production
  const displayMessage = isDevelopment
    ? error.message
    : sanitizeErrorMessage(error.message);

  return (
    <div className="flex flex-col items-center justify-center h-full p-8 text-center">
      <div className="text-6xl mb-4" aria-hidden="true">⚠️</div>
      <h2 className="text-2xl font-semibold mb-2">Something went wrong</h2>
      <p className="text-neutral-600 mb-4">{displayMessage}</p>

      {/* Only show detailed error in development */}
      {isDevelopment && error.stack && (
        <details className="mb-4 text-left max-w-md">
          <summary className="cursor-pointer text-sm text-neutral-500 hover:text-neutral-700">
            Developer Details
          </summary>
          <pre className="mt-2 p-4 bg-neutral-100 rounded text-xs overflow-auto max-h-40">
            {error.stack}
          </pre>
        </details>
      )}

      {/* Provide error ID for support correlation */}
      <p className="text-xs text-neutral-500 mb-4 font-mono">
        Error ID: {errorId}
      </p>

      <div className="flex gap-4">
        <button
          onClick={retry}
          disabled={!canRetry}
          className={`px-4 py-2 rounded transition-colors ${
            canRetry
              ? 'bg-primary text-white hover:bg-primary-dark focus:ring-2 focus:ring-primary'
              : 'bg-neutral-300 text-neutral-500 cursor-not-allowed'
          }`}
          aria-label={
            canRetry
              ? `Retry (${retriesRemaining} attempts remaining)`
              : 'Maximum retries exceeded'
          }
        >
          {canRetry ? `Try Again (${retriesRemaining} left)` : 'Retry Limit Reached'}
        </button>
        <button
          onClick={() => window.viprDesktop.app.reload()}
          className="px-4 py-2 bg-neutral-200 rounded hover:bg-neutral-300 focus:ring-2 focus:ring-neutral-400"
          aria-label="Reload the entire application"
        >
          Reload App
        </button>
      </div>
    </div>
  );
}

// src/renderer/utils/error-sanitizer.ts
export function sanitizeErrorMessage(message: string): string {
  let sanitized = message;

  // Remove absolute file paths (Windows and Unix)
  sanitized = sanitized.replace(/[A-Za-z]:[\\\/][\w\s\-\\\/\.]+/g, '[PATH]');
  sanitized = sanitized.replace(/\/[\w\-\/\.]+\.(ts|tsx|js|jsx)/gi, '[FILE]');

  // Remove home directory references
  sanitized = sanitized.replace(/\/Users\/[\w\-]+/g, '[HOME]');
  sanitized = sanitized.replace(/C:\\Users\\[\w\-]+/g, '[HOME]');

  // Truncate excessive length
  if (sanitized.length > 200) {
    sanitized = sanitized.substring(0, 200) + '...';
  }

  return sanitized;
}

export function generateErrorId(): string {
  return `err_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}
```

**Usage patterns:**

```typescript
// Wrap entire app
<ErrorBoundary>
  <App />
</ErrorBoundary>

// Wrap critical sections
<ErrorBoundary fallback={(error, retry) => <ChartErrorFallback error={error} retry={retry} />}>
  <TreemapChart data={data} />
</ErrorBoundary>

// Wrap routes
<ErrorBoundary>
  <Routes>
    <Route path="/" element={<Dashboard />} />
  </Routes>
</ErrorBoundary>
```

**Acceptance criteria:**

- Error boundaries catch all React component errors
- Fallback UI displays clear error message and recovery actions
- Errors logged to main process for debugging
- Retry button clears error state and re-renders
- Reload button restarts application
- Error boundaries do not interfere with dev tools error display

#### 2.2 analysis error recovery

**Purpose**: Handle analysis engine failures gracefully without crashing application.

**Error types:**

| Error Type            | Cause                              | Recovery Strategy                                                                                                                                                              |
| --------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Timeout**           | Analysis exceeds 30s timeout       | Skip file, log warning, continue queue <!-- TUNING: 30s is very long. Consider progressive timeout: 5s warning, 10s skip for typical files, 30s only for known-large files --> |
| **Parse Error**       | Malformed file (invalid syntax)    | Mark file as unparseable, show warning badge <!-- UX: Provide quick action to open file in editor and jump to parse error location -->                                         |
| **Plugin Crash**      | Plugin throws unhandled exception  | Disable plugin for session, notify user <!-- RECOVERY: Consider graceful degradation - continue with core analysis if plugin fails -->                                         |
| **Out of Memory**     | Large file exceeds memory limit    | Skip file, suggest file size limit setting <!-- PREVENTION: Pre-check file size before loading. Reject files >5MB proactively with clear message -->                           |
| **File Not Found**    | File deleted during analysis       | Remove from queue, update UI <!-- RACE CONDITION: This is common with watch mode. Implement file existence check before analysis, not during -->                               |
| **Permission Denied** | File unreadable due to permissions | Skip file, show permission warning <!-- BATCH: If multiple permission errors, show single notification: "Skipped 5 files due to permissions" -->                               |

<!-- MISSING ERROR TYPES: Consider adding:
     - Network errors (for future remote analysis features)
     - Disk full errors (database writes may fail)
     - Invalid UTF-8/encoding errors (common with binary files)
     - Circular symlink errors -->

**Implementation:**

```typescript
// src/main/analysis/error-handler.ts
import type { AnalysisError } from '@vipr/engine';
import { basename, relative } from 'path';
import { z } from 'zod';

// ARCHITECTURE NOTE: Consider using a state machine for error handling rather than stateful class.
// This would make error recovery logic more testable and easier to reason about.
// See: XState or simple reducer pattern with actions.

// SECURITY FINDING #7 (MEDIUM): Define schema for analysis errors to validate input
const AnalysisErrorSchema = z.object({
  type: z.enum([
    'timeout',
    'parse-error',
    'plugin-crash',
    'out-of-memory',
    'file-not-found',
    'permission-denied',
  ]),
  message: z.string().max(1000), // Limit message length
  filePath: z.string().optional(),
  pluginId: z.string().optional(),
  code: z.string().optional(),
  timestamp: z.number().optional(),
});

type ValidatedAnalysisError = z.infer<typeof AnalysisErrorSchema>;

export class AnalysisErrorHandler {
  private errorCounts = new Map<string, number>();
  private disabledPlugins = new Set<string>();
  // MEMORY LEAK: errorCounts Map will grow unbounded. Add:
  // - Periodic cleanup of old error counts (> 1 hour old)
  // - Maximum size limit (e.g., 1000 entries) with LRU eviction
  private errorTimestamps = new Map<string, number>(); // Track when errors occurred
  private repositoryRoot: string; // SECURITY FINDING #4: Store repository root for path sanitization

  constructor(repositoryRoot: string) {
    this.repositoryRoot = repositoryRoot;
  }

  // SECURITY FINDING #4 (MEDIUM): Sanitize file paths for user display
  private sanitizeFilePath(absolutePath: string): string {
    try {
      // Show relative path from repository root
      const relativePath = relative(this.repositoryRoot, absolutePath);

      // If path goes outside repository, show only filename
      if (relativePath.startsWith('..')) {
        return basename(absolutePath);
      }

      return relativePath;
    } catch {
      // Fallback to just filename if path manipulation fails
      return basename(absolutePath);
    }
  }

  // SECURITY FINDING #7 (MEDIUM): Validate error objects before processing
  handleError(error: unknown): ErrorRecoveryAction {
    try {
      // Validate error object
      const validatedError = AnalysisErrorSchema.parse(error);
      return this.processError(validatedError);
    } catch (validationError) {
      // If error object itself is invalid, return safe default
      return {
        action: 'skip-file',
        notification: {
          type: 'error',
          title: 'Invalid error',
          message: 'An unexpected error occurred',
        },
      };
    }
  }

  private processError(error: ValidatedAnalysisError): ErrorRecoveryAction {
    const errorKey = `${error.type}:${error.pluginId || 'core'}`;

    // IMPROVEMENT: Clean up old error counts to prevent memory leak
    this.cleanupOldErrors();

    const count = (this.errorCounts.get(errorKey) || 0) + 1;
    this.errorCounts.set(errorKey, count);
    this.errorTimestamps.set(errorKey, Date.now());

    // Disable plugin after 3 consecutive errors
    // LOGIC ISSUE: This counts total errors, not consecutive errors. "Consecutive" means
    // no successful operations in between. Need to track success and reset count on success.
    if (error.pluginId && count >= 3) {
      this.disabledPlugins.add(error.pluginId);
      return {
        action: 'disable-plugin',
        pluginId: error.pluginId,
        notification: {
          type: 'error',
          title: `Plugin ${error.pluginId} disabled`,
          message: `Too many errors. Restart app to re-enable.`,
        },
      };
    }

    // SECURITY FINDING #4 (MEDIUM): Sanitize file path for display
    const displayPath = error.filePath ? this.sanitizeFilePath(error.filePath) : 'unknown file';

    switch (error.type) {
      case 'timeout':
        return {
          action: 'skip-file',
          notification: {
            type: 'warning',
            title: 'Analysis timeout',
            message: `Analysis of "${displayPath}" exceeded the 30-second timeout`,
          },
        };

      case 'parse-error':
        return {
          action: 'mark-unparseable',
          notification: {
            type: 'warning',
            title: 'Parse error',
            message: `Unable to parse "${displayPath}" - file may contain syntax errors`,
          },
        };

      case 'out-of-memory':
        return {
          action: 'skip-file',
          notification: {
            type: 'error',
            title: 'Memory limit exceeded',
            message: `"${displayPath}" is too large to analyze. Consider adjusting file size limits in settings.`,
          },
        };

      case 'file-not-found':
        // Silently handle - don't notify user for transient file deletions
        return { action: 'remove-from-queue' };

      case 'permission-denied':
        return {
          action: 'skip-file',
          notification: {
            type: 'warning',
            title: 'Permission denied',
            message: `Cannot access "${displayPath}" due to file permissions`,
          },
        };

      default:
        // SECURITY: Don't expose error.message as it may contain sensitive data
        return {
          action: 'retry',
          notification: {
            type: 'error',
            title: 'Analysis error',
            message: 'An unexpected error occurred during analysis. Please try again.',
          },
        };
    }
  }

  isPluginDisabled(pluginId: string): boolean {
    return this.disabledPlugins.has(pluginId);
  }

  resetErrorCounts(): void {
    this.errorCounts.clear();
    this.errorTimestamps.clear(); // Don't forget to clear timestamps too
  }

  // IMPLEMENTATION: Add cleanup method for memory leak prevention
  private cleanupOldErrors(): void {
    // Remove entries older than 1 hour
    const oneHourAgo = Date.now() - 60 * 60 * 1000;
    for (const [key, timestamp] of this.errorTimestamps.entries()) {
      if (timestamp < oneHourAgo) {
        this.errorCounts.delete(key);
        this.errorTimestamps.delete(key);
      }
    }

    // SECURITY FINDING #7 (MEDIUM): Enforce maximum size to prevent unbounded growth
    if (this.errorCounts.size > 1000) {
      // Keep most recent 500 entries
      const entries = Array.from(this.errorTimestamps.entries());
      entries.sort((a, b) => b[1] - a[1]); // Sort by timestamp descending

      const keysToKeep = new Set(entries.slice(0, 500).map(([key]) => key));

      for (const key of this.errorCounts.keys()) {
        if (!keysToKeep.has(key)) {
          this.errorCounts.delete(key);
          this.errorTimestamps.delete(key);
        }
      }
    }
  }

  // MISSING METHOD: Add success tracking to properly implement "consecutive" errors
  handleSuccess(pluginId?: string): void {
    const successKey = `success:${pluginId || 'core'}`;
    // Reset error count for this plugin on success
    const errorKey = `*:${pluginId || 'core'}`;
    for (const key of this.errorCounts.keys()) {
      if (key.endsWith(`:${pluginId || 'core'}`)) {
        this.errorCounts.delete(key);
        this.errorTimestamps.delete(key);
      }
    }
  }
}
```

**Acceptance criteria:**

- Analysis errors do not crash application
- Failed files are skipped with user notification
- Plugins disabled after 3 consecutive errors
- Error details logged for debugging
- User can retry failed files manually
- Error counts reset on repository switch

#### 2.3 ipc error handling

**Purpose**: Handle IPC communication failures gracefully.

**Implementation:**

```typescript
// src/preload/error-handler.ts
import { ipcRenderer } from 'electron';
import { z } from 'zod';

// TYPE SAFETY NOTE: This generic signature is problematic. The return type should be the
// response schema, not the request schema. Need separate schemas for request and response.

// SECURITY FINDING #6 (MEDIUM): Generate session nonce for CSRF protection
const sessionNonce = crypto.randomUUID();

// SECURITY FINDING #6 (MEDIUM): Validate origin to prevent unauthorized IPC calls
function validateOrigin(): boolean {
  try {
    const origin = window.location.origin;
    return origin.startsWith('file://') || origin === 'vipr://desktop';
  } catch {
    return false;
  }
}

// SECURITY FINDING #5 (MEDIUM): Enhanced IPC configuration
interface IpcConfig {
  timeout: number;
  retries?: number;
  sanitizeErrors?: boolean;
}

const DEFAULT_CONFIG: IpcConfig = {
  timeout: 30000, // 30s default, but configurable per operation
  retries: 0,
  sanitizeErrors: true,
};

// SECURITY FINDING #5 (MEDIUM): Improved error result interface
export interface SafeIpcResult<T> {
  data?: T;
  error?: {
    message: string;
    code: string;
    isTimeout: boolean;
    isValidationError: boolean;
  };
}

export function createSafeIpcInvoke<TRequest extends z.ZodType, TResponse extends z.ZodType>(
  channel: string,
  requestSchema: TRequest,
  responseSchema: TResponse,
  config: Partial<IpcConfig> = {} // SECURITY FINDING #5: Configurable timeout
) {
  const finalConfig = { ...DEFAULT_CONFIG, ...config };

  return async (payload: z.infer<TRequest>): Promise<SafeIpcResult<z.infer<TResponse>>> => {
    try {
      // SECURITY FINDING #6 (MEDIUM): Validate origin before IPC call
      if (!validateOrigin()) {
        return {
          error: {
            message: 'IPC call from unauthorized origin',
            code: 'UNAUTHORIZED_ORIGIN',
            isTimeout: false,
            isValidationError: false,
          },
        };
      }

      // Validate request payload
      const validated = requestSchema.parse(payload);

      // SECURITY FINDING #6 (MEDIUM): Include nonce in IPC payload
      const securePayload = {
        _nonce: sessionNonce,
        _timestamp: Date.now(),
        data: validated,
      };

      // Invoke with configurable timeout
      const result = await Promise.race([
        ipcRenderer.invoke(channel, securePayload),
        createTimeout(finalConfig.timeout),
      ]);

      // SECURITY FINDING #5 (MEDIUM): Validate response from main process
      const validatedResponse = responseSchema.parse(result);

      return { data: validatedResponse };
    } catch (error) {
      return handleIpcError(error, channel, finalConfig);
    }
  };
}

function createTimeout(ms: number): Promise<never> {
  return new Promise((_, reject) => {
    setTimeout(() => {
      const error = new Error('IPC operation timed out');
      (error as any).code = 'IPC_TIMEOUT';
      reject(error);
    }, ms);
  });
}

function handleIpcError(error: unknown, channel: string, config: IpcConfig): SafeIpcResult<never> {
  const isDevelopment = import.meta.env.DEV;

  // Log to console only in development
  if (isDevelopment) {
    console.error(`IPC error on channel ${channel}:`, error);
  }

  // Handle Zod validation errors
  if (error instanceof z.ZodError) {
    return {
      error: {
        message: 'Invalid data format',
        code: 'VALIDATION_ERROR',
        isTimeout: false,
        isValidationError: true,
      },
    };
  }

  // Handle timeout errors
  if (error instanceof Error && (error as any).code === 'IPC_TIMEOUT') {
    return {
      error: {
        message: 'Operation timed out',
        code: 'IPC_TIMEOUT',
        isTimeout: true,
        isValidationError: false,
      },
    };
  }

  // Handle generic errors with sanitization
  const errorMessage = error instanceof Error ? error.message : 'Unknown error occurred';
  const sanitizedMessage = config.sanitizeErrors
    ? sanitizeErrorMessage(errorMessage)
    : errorMessage;

  return {
    error: {
      message: sanitizedMessage,
      code: 'IPC_ERROR',
      isTimeout: false,
      isValidationError: false,
    },
  };
}

// Note: sanitizeErrorMessage should be imported from renderer/utils/error-sanitizer.ts
function sanitizeErrorMessage(message: string): string {
  // Remove absolute file paths and sensitive data
  let sanitized = message.replace(/[A-Za-z]:[\\\/][\w\s\-\\\/\.]+/g, '[PATH]');
  sanitized = sanitized.replace(/\/[\w\-\/\.]+\.(ts|tsx|js|jsx)/gi, '[FILE]');
  sanitized = sanitized.replace(/\/Users\/[\w\-]+/g, '[HOME]');
  return sanitized.length > 200 ? sanitized.substring(0, 200) + '...' : sanitized;
}
```

**Usage:**

```typescript
// src/renderer/hooks/useAnalysis.ts
import { useCallback } from 'react';
import { useToast } from './useToast';

export function useAnalysis() {
  const { showToast } = useToast();

  const analyzeFiles = useCallback(
    async (paths: string[]) => {
      const { data, error } = await window.viprDesktop.analysis.run({ paths });

      if (error) {
        showToast({
          type: 'error',
          title: 'Analysis failed',
          message: error.message,
        });
        return null;
      }

      return data;
    },
    [showToast]
  );

  return { analyzeFiles };
}
```

<!-- SECURITY FINDING #6 (MEDIUM): Main process IPC handlers must validate nonces -->
<!-- The main process must implement nonce validation to complete CSRF protection:

```typescript
// src/main/ipc/handlers.ts
interface SecureIpcPayload<T> {
  _nonce: string;
  _timestamp: number;
  data: T;
}

const validNonces = new Set<string>();

// Store valid nonce when renderer initializes
ipcMain.handle('register-nonce', (event, nonce: string) => {
  validNonces.add(nonce);
});

function validateIpcPayload<T>(payload: SecureIpcPayload<T>): T {
  // Check nonce
  if (!validNonces.has(payload._nonce)) {
    throw new Error('Invalid IPC nonce');
  }

  // Check timestamp (reject requests older than 1 hour)
  const age = Date.now() - payload._timestamp;
  if (age > 3600000 || age < 0) {
    throw new Error('Invalid IPC timestamp');
  }

  return payload.data;
}

// Use in IPC handlers
ipcMain.handle('analysis:run', async (event, payload) => {
  try {
    const data = validateIpcPayload(payload);
    // Proceed with validated data...
  } catch (error) {
    logger.error('IPC validation failed:', error);
    throw error;
  }
});
```
-->

**Acceptance criteria:**

- IPC errors display user-friendly error messages
- Timeouts do not hang UI (configurable, default 30s)
- Failed IPC calls can be retried
- Error details logged for debugging (sanitized in production)
- Validation errors show specific field failures
- Origin validation prevents unauthorized IPC calls
- Nonce validation on main process prevents CSRF attacks
- Request and response payloads validated with Zod schemas

<!-- SECURITY FINDING #8 (MEDIUM): Secure error logging implementation -->

#### 2.4 secure logging

**Purpose**: Store error logs securely with appropriate access controls and encryption.

**Security requirements:**

| Requirement          | Implementation                                              |
| -------------------- | ----------------------------------------------------------- |
| **File permissions** | Unix: 0600 (rw-------), Windows: User-only ACL              |
| **Log encryption**   | AES-256-CBC with per-installation key                       |
| **Log rotation**     | Automatic rotation at 10MB, keep last 5 rotated logs        |
| **Sensitive data**   | Sanitize file paths, stack traces, user data before logging |
| **Storage location** | User data directory (app.getPath('userData') + '/logs')     |

**Implementation:**

```typescript
// src/main/logging/logger.ts
import { app } from 'electron';
import * as fs from 'fs';
import * as path from 'path';
import * as crypto from 'crypto';

export class SecureLogger {
  private logDir: string;
  private logFile: string;
  private encryptionKey?: Buffer;

  constructor(enableEncryption = true) {
    // Store logs in user data directory with proper permissions
    this.logDir = path.join(app.getPath('userData'), 'logs');
    this.logFile = path.join(this.logDir, 'error.log');

    // Create log directory with restrictive permissions
    if (!fs.existsSync(this.logDir)) {
      fs.mkdirSync(this.logDir, { recursive: true, mode: 0o700 }); // rwx------
    }

    // Initialize encryption key if enabled
    if (enableEncryption) {
      this.initializeEncryption();
    }

    // Rotate logs on startup
    this.rotateLogsIfNeeded();
  }

  private initializeEncryption(): void {
    const keyPath = path.join(this.logDir, '.key');

    try {
      if (fs.existsSync(keyPath)) {
        this.encryptionKey = fs.readFileSync(keyPath);
      } else {
        // Generate new encryption key
        this.encryptionKey = crypto.randomBytes(32);
        fs.writeFileSync(keyPath, this.encryptionKey, { mode: 0o600 }); // rw-------
      }
    } catch (error) {
      console.error('Failed to initialize log encryption:', error);
      this.encryptionKey = undefined; // Disable encryption if it fails
    }
  }

  private rotateLogsIfNeeded(): void {
    try {
      const stats = fs.statSync(this.logFile);
      const maxSize = 10 * 1024 * 1024; // 10MB

      if (stats.size > maxSize) {
        const rotatedName = `error-${Date.now()}.log`;
        const rotatedPath = path.join(this.logDir, rotatedName);
        fs.renameSync(this.logFile, rotatedPath);

        // Keep only last 5 rotated logs
        this.cleanOldLogs();
      }
    } catch (error) {
      // File doesn't exist yet, ignore
    }
  }

  private cleanOldLogs(): void {
    try {
      const files = fs
        .readdirSync(this.logDir)
        .filter(f => f.startsWith('error-') && f.endsWith('.log'))
        .map(f => ({
          name: f,
          path: path.join(this.logDir, f),
          time: fs.statSync(path.join(this.logDir, f)).mtime.getTime(),
        }))
        .sort((a, b) => b.time - a.time);

      // Delete all but the 5 most recent
      files.slice(5).forEach(file => {
        fs.unlinkSync(file.path);
      });
    } catch (error) {
      console.error('Failed to clean old logs:', error);
    }
  }

  private encrypt(data: string): string {
    if (!this.encryptionKey) {
      return data; // Fallback to unencrypted
    }

    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-cbc', this.encryptionKey, iv);
    const encrypted = Buffer.concat([cipher.update(data, 'utf8'), cipher.final()]);

    // Prepend IV to encrypted data
    return iv.toString('hex') + ':' + encrypted.toString('hex');
  }

  error(data: any): void {
    const timestamp = new Date().toISOString();
    const entry = {
      timestamp,
      level: 'error',
      data,
    };

    const logLine = JSON.stringify(entry) + '\n';
    const finalData = this.encryptionKey ? this.encrypt(logLine) + '\n' : logLine;

    try {
      fs.appendFileSync(this.logFile, finalData, { mode: 0o600 }); // rw-------
    } catch (error) {
      console.error('Failed to write to error log:', error);
    }
  }

  // Add similar methods for warn, info, debug levels
}

// Initialize with encryption enabled
export const logger = new SecureLogger(true);
```

**Acceptance criteria:**

- Log files created with restrictive permissions (Unix: 600, Windows: user-only)
- Encryption key stored securely with 600 permissions
- Logs automatically rotate at 10MB threshold
- Only 5 most recent rotated logs retained
- All logged data sanitized before writing
- Encryption can be disabled for debugging if needed
- Log directory created in user data directory (not system temp)

---

### 3. progress indicators and notifications

#### 3.1 toast notification system

**Purpose**: Display transient notifications for user actions and system events.

<!-- ACCESSIBILITY CONCERN: Toast notifications are problematic for accessibility:
     1. Screen readers may not announce them if focus is elsewhere
     2. Users with motor disabilities may not be able to dismiss them quickly enough
     3. Motion-sensitive users may be bothered by animations

     RECOMMENDATIONS:
     - Use role="status" or role="alert" for ARIA announcements
     - Add prefers-reduced-motion support
     - Ensure toasts are keyboard dismissible
     - Provide alternative to toast for critical information (modal dialog or inline message)
     - Follow WCAG 2.2 ARIA practices for status messages -->

<!-- UI PATTERN NOTE: Consider using native Electron notifications (Notification API) for
     important messages rather than custom toasts. Native notifications:
     - Work even when app is in background
     - Follow OS notification preferences
     - Are more accessible by default
     - Have better attention management
     Use custom toasts only for transient, low-priority status updates. -->

**Component structure:**

```typescript
// src/renderer/components/Toast.tsx
import { useEffect, useState } from 'react';
import { createPortal } from 'react-dom';

export interface Toast {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  title: string;
  message?: string;
  duration?: number; // ms, default 5000
  action?: {
    label: string;
    onClick: () => void;
  };
}

export function ToastContainer({ toasts }: { toasts: Toast[] }) {
  return createPortal(
    <div
      className="fixed bottom-4 right-4 z-50 flex flex-col gap-2"
      // ACCESSIBILITY: Add ARIA attributes
      role="region"
      aria-label="Notifications"
      aria-live="polite" // "assertive" for errors, "polite" for info
    >
      {toasts.map((toast) => (
        <ToastItem key={toast.id} toast={toast} />
      ))}
    </div>,
    document.body
  );
}

// PERFORMANCE NOTE: createPortal to document.body can cause issues with React concurrent rendering.
// Consider creating a dedicated toast root element and using ReactDOM.createRoot for toast rendering.

function ToastItem({ toast }: { toast: Toast }) {
  const [isExiting, setIsExiting] = useState(false);

  useEffect(() => {
    const duration = toast.duration || 5000;
    const exitTimer = setTimeout(() => setIsExiting(true), duration - 300);
    const removeTimer = setTimeout(() => removeToast(toast.id), duration);

    return () => {
      clearTimeout(exitTimer);
      clearTimeout(removeTimer);
    };
  }, [toast]);

  const bgColor = {
    info: 'bg-blue-500',
    success: 'bg-green-500',
    warning: 'bg-yellow-500',
    error: 'bg-red-500',
  }[toast.type];

  return (
    <div
      className={`
        ${bgColor} text-white rounded-lg shadow-lg p-4 min-w-[300px] max-w-[400px]
        transition-all duration-300
        ${isExiting ? 'opacity-0 translate-x-8' : 'opacity-100 translate-x-0'}
      `}
    >
      <div className="flex items-start justify-between gap-4">
        <div className="flex-1">
          <h4 className="font-semibold">{toast.title}</h4>
          {toast.message && <p className="text-sm mt-1">{toast.message}</p>}
        </div>
        <button
          onClick={() => removeToast(toast.id)}
          className="text-white hover:text-neutral-200"
        >
          ×
        </button>
      </div>
      {toast.action && (
        <button
          onClick={toast.action.onClick}
          className="mt-2 text-sm underline hover:no-underline"
        >
          {toast.action.label}
        </button>
      )}
    </div>
  );
}
```

**Store integration:**

```typescript
// src/renderer/stores/ui.ts
import { create } from 'zustand';
import type { Toast } from '../components/Toast';

interface UIStore {
  toasts: Toast[];
  showToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
}

export const useUIStore = create<UIStore>(set => ({
  toasts: [],

  showToast: toast => {
    const id = `toast-${Date.now()}-${Math.random()}`;
    set(state => ({
      toasts: [...state.toasts, { ...toast, id }],
    }));
  },

  removeToast: id => {
    set(state => ({
      toasts: state.toasts.filter(t => t.id !== id),
    }));
  },
}));
```

**Usage patterns:**

```typescript
// Analysis completion
showToast({
  type: 'success',
  title: 'Analysis complete',
  message: `Analyzed ${fileCount} files`,
});

// Error notification
showToast({
  type: 'error',
  title: 'Analysis failed',
  message: error.message,
  action: {
    label: 'Retry',
    onClick: () => retryAnalysis(),
  },
});

// Progress notification
showToast({
  type: 'info',
  title: 'Indexing repository',
  message: 'This may take a few minutes...',
  duration: 10000,
});
```

**Acceptance criteria:**

- Toasts display in bottom-right corner <!-- ACCESSIBILITY: Position should be configurable. Some users prefer top-right or may need toasts to avoid blocking important content -->
- Maximum 3 toasts visible simultaneously (oldest auto-dismissed) <!-- UX: When queue exceeds 3, show count indicator "2 more notifications" to prevent information loss -->
- Smooth enter/exit animations <!-- ACCESSIBILITY: Respect prefers-reduced-motion media query - use simple opacity transition when motion is reduced -->
- Toasts auto-dismiss after duration (default 5s) <!-- ACCESSIBILITY: Error toasts should not auto-dismiss or should have much longer duration (30s+) so users have time to read -->
- User can manually dismiss any toast <!-- KEYBOARD: Ensure toasts are in focus order and can be dismissed with Escape or Enter key -->
- Action buttons trigger callbacks and dismiss toast <!-- STATE: Consider if action should always dismiss - sometimes user needs to keep toast open after clicking action -->

<!-- MISSING CRITERIA:
     - Screen reader announces toast content
     - Toast stacking order is visually clear (drop shadow, spacing)
     - Toasts persist across route changes (or don't - specify behavior)
     - Toast content is truncated at reasonable length (200 chars?) with expand option -->

#### 3.2 progress indicators

**Purpose**: Display real-time progress for long-running operations.

**Component structure:**

```typescript
// src/renderer/components/ProgressIndicator.tsx
import { useEffect, useState } from 'react';

export interface ProgressState {
  current: number;
  total: number;
  label?: string;
  status?: 'active' | 'paused' | 'error' | 'complete';
}

export function ProgressBar({
  current,
  total,
  label,
  status = 'active',
}: ProgressState) {
  const percentage = Math.round((current / total) * 100);

  const barColor = {
    active: 'bg-blue-500',
    paused: 'bg-yellow-500',
    error: 'bg-red-500',
    complete: 'bg-green-500',
  }[status];

  return (
    <div className="w-full">
      {label && (
        <div className="flex justify-between mb-2 text-sm">
          <span>{label}</span>
          <span>{percentage}%</span>
        </div>
      )}
      <div className="w-full h-2 bg-neutral-200 rounded-full overflow-hidden">
        <div
          className={`h-full ${barColor} transition-all duration-300`}
          style={{ width: `${percentage}%` }}
        />
      </div>
      <div className="flex justify-between mt-1 text-xs text-neutral-600">
        <span>{current.toLocaleString()} of {total.toLocaleString()}</span>
        {status === 'active' && <span>In progress...</span>}
        {status === 'complete' && <span>Complete</span>}
        {status === 'error' && <span>Failed</span>}
      </div>
    </div>
  );
}

export function CircularProgress({ percentage }: { percentage: number }) {
  const radius = 40;
  const circumference = 2 * Math.PI * radius;
  const offset = circumference - (percentage / 100) * circumference;

  return (
    <svg width="100" height="100" className="transform -rotate-90">
      <circle
        cx="50"
        cy="50"
        r={radius}
        fill="none"
        stroke="#e5e7eb"
        strokeWidth="8"
      />
      <circle
        cx="50"
        cy="50"
        r={radius}
        fill="none"
        stroke="#3b82f6"
        strokeWidth="8"
        strokeDasharray={circumference}
        strokeDashoffset={offset}
        className="transition-all duration-300"
      />
      <text
        x="50"
        y="50"
        textAnchor="middle"
        dy="7"
        className="transform rotate-90 origin-center text-lg font-semibold"
      >
        {percentage}%
      </text>
    </svg>
  );
}

export function IndeterminateSpinner() {
  return (
    <div className="flex items-center justify-center">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500" />
    </div>
  );
}
```

**Integration with analysis:**

```typescript
// src/renderer/pages/Dashboard.tsx
import { useAnalysisStore } from '../stores/analysis';
import { ProgressBar } from '../components/ProgressIndicator';

export function Dashboard() {
  const { isAnalyzing, progress } = useAnalysisStore();

  return (
    <div>
      {isAnalyzing && (
        <div className="fixed top-4 right-4 bg-white shadow-lg rounded-lg p-4 w-80 z-50">
          <ProgressBar
            current={progress.current}
            total={progress.total}
            label={progress.currentFile}
            status="active"
          />
        </div>
      )}
      {/* Dashboard content */}
    </div>
  );
}
```

**Acceptance criteria:**

- Progress bar updates in real-time during analysis
- Percentage calculation is accurate
- Status colors clearly indicate state (active, paused, error, complete)
- Circular progress for compact spaces
- Indeterminate spinner for operations without known total
- Smooth animations without performance impact

---

### 4. performance optimization

<!-- ARCHITECTURE NOTE: Performance optimization should be data-driven. Establish baseline metrics
     FIRST with realistic datasets, then optimize based on profiling data. Premature optimization
     can add complexity without meaningful benefit. Consider implementing performance monitoring
     in production to validate these targets with real usage patterns. -->

#### 4.1 virtualization tuning

**Purpose**: Optimize virtualized lists for 10,000+ items without performance degradation.

<!-- SCALE QUESTION: Is 10,000 files a realistic target? Based on GitHub's analysis:
     - 90th percentile repo has ~500 files
     - 99th percentile repo has ~5,000 files
     - Repos with 10,000+ files are extremely rare
     Consider if optimizing for 99th percentile (5,000 files) is sufficient, or if enterprise
     users (potential future customers) will regularly exceed this. -->

**Target metrics:**

| Metric                 | Target  | Measurement                                                                                                                                            |
| ---------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Initial render**     | < 100ms | Time to first contentful paint <!-- BASELINE: Specify hardware - "on M1 MacBook Air" -->                                                               |
| **Scroll performance** | 60fps   | Frame rate during rapid scrolling <!-- ACCEPTABLE: Also define acceptable threshold - "max 5 dropped frames per second" -->                            |
| **Memory usage**       | < 200MB | Heap size for 10,000 items <!-- CONTEXT: Total app memory or just virtualized list? Specify measurement point (Chrome DevTools heap snapshot) -->      |
| **Search performance** | < 50ms  | Time to filter 10,000 items <!-- PERCEIVED: 50ms is imperceptible to users. Consider if 100ms would be acceptable and allow simpler implementation --> |

**Optimizations:**

1. **Dynamic row height**

```typescript
// src/renderer/components/VirtualizedFileTable.tsx
import { useVirtualizer } from '@tanstack/react-virtual';

export function VirtualizedFileTable({ files }: { files: FileRecord[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: files.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (index) => {
      // Dynamic height based on content
      const file = files[index];
      return file.hasDetails ? 120 : 48; // Expanded vs collapsed
    },
    overscan: 10, // Render 10 extra rows above/below viewport
    measureElement: (el) => el.getBoundingClientRect().height, // Accurate measurement
  });

  return (
    <div ref={parentRef} className="h-screen overflow-auto">
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const file = files[virtualRow.index];
          return (
            <div
              key={virtualRow.key}
              data-index={virtualRow.index}
              ref={virtualizer.measureElement}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <FileRow file={file} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

2. **Memoized row components**

```typescript
import { memo } from 'react';

export const FileRow = memo(
  ({ file }: { file: FileRecord }) => {
    return (
      <div className="flex items-center gap-4 p-4 border-b hover:bg-neutral-50">
        <FileIcon type={file.fileType} />
        <span className="flex-1">{file.path}</span>
        <Badge severity={file.severity} />
        <span>{file.score}</span>
      </div>
    );
  },
  (prev, next) => {
    // Custom comparison to prevent unnecessary re-renders
    return (
      prev.file.id === next.file.id &&
      prev.file.score === next.file.score &&
      prev.file.updatedAt === next.file.updatedAt
    );
  }
);
```

3. **Debounced search**

```typescript
// src/renderer/hooks/useSearch.ts
import { useMemo, useState } from 'react';
import { useDebouncedValue } from './useDebouncedValue';

export function useSearch<T>(items: T[], searchFn: (item: T, query: string) => boolean) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebouncedValue(query, 300);

  const filtered = useMemo(() => {
    if (!debouncedQuery) return items;
    return items.filter(item => searchFn(item, debouncedQuery));
  }, [items, debouncedQuery, searchFn]);

  return { query, setQuery, filtered };
}
```

4. **Index-based lookup**

```typescript
// src/renderer/stores/analysis.ts
import { create } from 'zustand';

interface AnalysisStore {
  files: Map<string, FileRecord>; // Use Map for O(1) lookups
  fileIndex: Map<string, string>; // path -> id index

  getFileById: (id: string) => FileRecord | undefined;
  getFileByPath: (path: string) => FileRecord | undefined;
}

export const useAnalysisStore = create<AnalysisStore>((set, get) => ({
  files: new Map(),
  fileIndex: new Map(),

  getFileById: id => get().files.get(id),

  getFileByPath: path => {
    const id = get().fileIndex.get(path);
    return id ? get().files.get(id) : undefined;
  },
}));
```

**Acceptance criteria:**

- 10,000 item list renders in under 100ms
- Scrolling maintains 60fps
- Memory usage stays under 200MB
- Search filters 10,000 items in under 50ms
- No janky UI interactions
- Virtualization works with dynamic row heights

#### 4.2 bundle optimization

**Purpose**: Minimize bundle size and optimize loading performance.

**Target metrics:**

| Metric                  | Target                 |
| ----------------------- | ---------------------- |
| **Initial bundle size** | < 500KB (gzipped)      |
| **Renderer bundle**     | < 1MB (uncompressed)   |
| **Main bundle**         | < 500KB (uncompressed) |
| **Time to interactive** | < 2s                   |

**Optimizations:**

1. **Code splitting**

```typescript
// src/renderer/App.tsx
import { lazy, Suspense } from 'react';
import { MemoryRouter, Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Files = lazy(() => import('./pages/Files'));
const FileDetail = lazy(() => import('./pages/FileDetail'));
const History = lazy(() => import('./pages/History'));
const Settings = lazy(() => import('./pages/Settings'));

export function App() {
  return (
    <MemoryRouter>
      <Suspense fallback={<LoadingScreen />}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/files" element={<Files />} />
          <Route path="/files/:fileId" element={<FileDetail />} />
          <Route path="/history" element={<History />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </MemoryRouter>
  );
}
```

2. **Tree-shaking configuration**

```typescript
// vite.renderer.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'chart-vendor': ['chart.js', 'd3'],
          'ui-vendor': ['zustand', '@tanstack/react-virtual'],
        },
      },
    },
  },
  optimizeDeps: {
    include: ['react', 'react-dom', 'zustand'],
  },
});
```

3. **Dynamic imports for heavy components**

```typescript
// src/renderer/pages/Dashboard.tsx
import { lazy, Suspense } from 'react';

const TreemapChart = lazy(() => import('../components/charts/TreemapChart'));
const HeatmapChart = lazy(() => import('../components/charts/HeatmapChart'));

export function Dashboard() {
  return (
    <div>
      <Suspense fallback={<ChartSkeleton />}>
        {showTreemap && <TreemapChart data={data} />}
        {showHeatmap && <HeatmapChart data={data} />}
      </Suspense>
    </div>
  );
}
```

**Acceptance criteria:**

- Renderer bundle under 1MB uncompressed
- Initial load under 2s on standard hardware
- Code splitting reduces initial bundle by 40%+
- Tree-shaking eliminates unused code
- Dynamic imports lazy-load heavy visualizations

#### 4.3 database query optimization

**Purpose**: Optimize SQLite queries for sub-50ms response times.

**Query patterns:**

1. **Indexed queries**

```sql
-- Create composite indices
CREATE INDEX idx_files_path_sha ON files(path, sha);
CREATE INDEX idx_analyses_file_plugin ON analyses(file_id, plugin_id);
CREATE INDEX idx_snapshots_git_sha ON snapshots(git_sha);
CREATE INDEX idx_notes_target ON notes(target_type, target_id);

-- Use covering index for common queries
CREATE INDEX idx_files_summary ON files(path, score, file_type, analyzed_at);
```

2. **Prepared statements**

```typescript
// src/main/db/queries.ts
import Database from 'better-sqlite3';

export class QueryService {
  private db: Database.Database;
  private preparedStatements: Map<string, Database.Statement>;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.preparedStatements = new Map();
    this.prepareStatements();
  }

  private prepareStatements() {
    // File queries
    this.preparedStatements.set('getFileById', this.db.prepare('SELECT * FROM files WHERE id = ?'));

    this.preparedStatements.set(
      'getFileByPath',
      this.db.prepare('SELECT * FROM files WHERE path = ?')
    );

    this.preparedStatements.set(
      'getFilesBySha',
      this.db.prepare('SELECT * FROM files WHERE sha = ?')
    );

    // Analysis queries
    this.preparedStatements.set(
      'getAnalysesByFileId',
      this.db.prepare('SELECT * FROM analyses WHERE file_id = ?')
    );

    this.preparedStatements.set(
      'getAnalysesByPlugin',
      this.db.prepare('SELECT * FROM analyses WHERE plugin_id = ?')
    );
  }

  getFileById(id: number): FileRecord | undefined {
    return this.preparedStatements.get('getFileById')!.get(id) as FileRecord;
  }

  getFileByPath(path: string): FileRecord | undefined {
    return this.preparedStatements.get('getFileByPath')!.get(path) as FileRecord;
  }
}
```

3. **Batch operations**

```typescript
// src/main/db/batch.ts
export function batchInsertFiles(db: Database.Database, files: FileRecord[]): void {
  const insert = db.prepare(`
    INSERT INTO files (path, sha, analyzed_at, git_sha, git_author, git_date)
    VALUES (?, ?, ?, ?, ?, ?)
  `);

  const transaction = db.transaction((files: FileRecord[]) => {
    for (const file of files) {
      insert.run(file.path, file.sha, file.analyzedAt, file.gitSha, file.gitAuthor, file.gitDate);
    }
  });

  transaction(files);
}
```

**Acceptance criteria:**

- Single file query completes in under 5ms
- 1000 file query completes in under 50ms
- Batch inserts process 100 files/second
- Full-text search returns results in under 100ms
- Database size stays under 100MB for 10,000 files

---

### 5. user testing and refinement

#### 5.1 test coverage plan

**Unit tests (Vitest):**

| Module                     | Coverage Target | Test Types                                 |
| -------------------------- | --------------- | ------------------------------------------ |
| **Visualization selector** | 100%            | Decision matrix, edge cases                |
| **Error handlers**         | 100%            | All error types, recovery actions          |
| **Progress calculators**   | 100%            | Percentage calculations, state transitions |
| **Store actions**          | 90%             | State mutations, side effects              |
| **Utility functions**      | 100%            | Pure functions, data transformations       |

**Integration tests (Vitest):**

| Module                  | Coverage Target | Test Types                       |
| ----------------------- | --------------- | -------------------------------- |
| **IPC communication**   | 80%             | Request/response, error handling |
| **Database operations** | 90%             | CRUD operations, transactions    |
| **Analysis workflow**   | 80%             | End-to-end analysis pipeline     |
| **File watching**       | 70%             | Change detection, debouncing     |

**E2E tests (Playwright):**

| User Story              | Test Scenarios                               |
| ----------------------- | -------------------------------------------- |
| **US-12 Adaptive viz**  | Switch between viz types based on file count |
| **Error recovery**      | Trigger analysis error, verify recovery UI   |
| **Progress indicators** | Start analysis, verify progress updates      |
| **Performance**         | Load 10,000 files, measure render time       |

**Test implementation example:**

```typescript
// src/renderer/utils/visualization-selector.test.ts
import { describe, it, expect } from 'vitest';
import { selectVisualization } from './visualization-selector';

describe('selectVisualization', () => {
  it('selects table for small datasets (≤50 files)', () => {
    const files = createMockFiles(50);
    expect(selectVisualization(files)).toBe('table');
  });

  it('selects treemap for large datasets (201-500 files)', () => {
    const files = createMockFiles(300);
    expect(selectVisualization(files)).toBe('treemap');
  });

  it('selects heatmap for very large datasets with high churn variance', () => {
    const files = createMockFiles(600, { churnVariance: 0.5 });
    expect(selectVisualization(files)).toBe('heatmap');
  });

  it('selects virtualized table for very large datasets with low churn variance', () => {
    const files = createMockFiles(600, { churnVariance: 0.2 });
    expect(selectVisualization(files)).toBe('virtualized-table');
  });
});
```

**Acceptance criteria:**

- Unit test coverage above 90%
- Integration test coverage above 80%
- All E2E user flows covered
- Tests run in under 30s
- No flaky tests

#### 5.2 user testing methodology

**Test participants:**

| Persona                       | Count | Focus Areas                                  |
| ----------------------------- | ----- | -------------------------------------------- |
| **Sole proprietor developer** | 3     | Usability, learning curve, AI prompts        |
| **Technical PM**              | 2     | Report generation, stakeholder communication |
| **Tech debt consultant**      | 2     | Historical analysis, regression tracking     |
| **Startup engineer**          | 3     | Performance, large codebase handling         |

**Test scenarios:**

1. **Open repository and review dashboard**
   - Task: Open a 500+ file repository
   - Observe: Time to first insight, dashboard comprehension
   - Measure: Time to completion, task success rate

2. **Identify high-risk files**
   - Task: Find files with high complexity and churn
   - Observe: Visualization effectiveness, filter usage
   - Measure: Accuracy, time to completion

3. **Generate PDF report**
   - Task: Create stakeholder report with cost estimates
   - Observe: Report customization, output quality
   - Measure: Satisfaction score, report completeness

4. **Recover from error**
   - Task: Trigger analysis error (corrupted file)
   - Observe: Error message clarity, recovery path
   - Measure: Recovery success rate, user confidence

5. **Track regression**
   - Task: Compare two snapshots to find regressions
   - Observe: Comparison UI, insight actionability
   - Measure: Task success rate, time to completion

**Metrics collected:**

| Metric                  | Target                   |
| ----------------------- | ------------------------ |
| **Task success rate**   | > 90%                    |
| **Time on task**        | < 5 minutes per scenario |
| **Error recovery rate** | > 95%                    |
| **User satisfaction**   | > 4/5 average            |
| **Net Promoter Score**  | > 50                     |

**Testing schedule:**

- Week 1: Internal testing with dev team
- Week 2: Alpha testing with 5 external users
- Week 3: Beta testing with 10 external users
- Week 4: Feedback integration and refinement

**Acceptance criteria:**

- All test scenarios have > 90% success rate
- No critical usability issues identified
- User satisfaction score above 4/5
- Feedback incorporated into final release

---

## part b: architecture review

### 1. reviewer selection and responsibilities

#### 1.1 selected reviewers

Based on the Phase 5 deliverables and the subagent reference from 00-user-stories.md, the following reviewers are selected:

| Reviewer                       | Role                        | Primary Focus                          |
| ------------------------------ | --------------------------- | -------------------------------------- |
| **data-visualization-analyst** | Visualization strategy      | Treemap, heatmap, adaptive viz logic   |
| **frontend-security-auditor**  | Error handling and security | Error boundaries, IPC error handling   |
| **architecture-reviewer**      | System-wide quality         | Performance optimization, code quality |

#### 1.2 review responsibilities

**data-visualization-analyst:**

- Review adaptive visualization selection algorithm
- Validate treemap and heatmap implementations
- Assess visualization performance with large datasets
- Evaluate color scales and interaction patterns
- Verify accessibility (color blindness, keyboard navigation)

**Specific checkpoints:**

1. Treemap rendering performance (must render 1000 files in < 500ms)
2. Heatmap canvas implementation (60fps scrolling)
3. Adaptive visualization decision matrix accuracy
4. Color scale effectiveness (distinguishability of complexity levels)
5. Interaction patterns (zoom, pan, hover, click)

**frontend-security-auditor:**

- Review error boundary implementation
- Audit IPC error handling patterns
- Validate Zod schema validation at all IPC boundaries
- Assess error logging and user notification strategies
- Verify no sensitive data leakage in error messages

**Specific checkpoints:**

1. Error boundaries catch all React errors without exposing stack traces
2. IPC validation prevents malformed payloads
3. Error notifications do not leak file system paths in production
4. Retry mechanisms do not cause infinite loops
5. Plugin crash recovery does not destabilize main process

**architecture-reviewer:**

- Review overall Phase 5 architecture and integration
- Assess performance optimization strategies
- Evaluate test coverage and testing approach
- Validate code organization and module boundaries
- Provide refactoring recommendations

**Specific checkpoints:**

1. Virtualization implementation meets performance targets
2. Bundle optimization achieves size targets
3. Database query optimization meets latency targets
4. Test coverage meets minimum thresholds (90% unit, 80% integration)
5. Code adheres to CLAUDE.md architecture rules

---

### 2. review criteria and checkpoints

#### 2.1 visualization review checklist

**Performance:**

- [ ] Treemap renders 1000 files in under 500ms
- [ ] Heatmap maintains 60fps during scrolling
- [ ] Adaptive visualization selection completes in under 50ms
- [ ] No memory leaks during visualization switching
- [ ] Canvas rendering optimized with dirty region tracking

**Functionality:**

- [ ] Treemap accurately represents directory hierarchy
- [ ] Heatmap correctly maps churn vs complexity
- [ ] Adaptive selector chooses optimal visualization for dataset
- [ ] Color scales are distinguishable and meaningful
- [ ] Tooltips display accurate data

**Interaction:**

- [ ] Zoom and pan interactions are smooth
- [ ] Click navigation works correctly
- [ ] Hover tooltips appear without lag
- [ ] Keyboard navigation is fully supported
- [ ] Touch gestures work on touchscreen devices

**Accessibility:**

- [ ] Color scales work for colorblind users
- [ ] Keyboard navigation reaches all interactive elements
- [ ] Screen readers announce visualization changes
- [ ] Tooltips have proper ARIA labels
- [ ] Focus indicators are visible

#### 2.2 error handling review checklist

**Error boundaries:**

- [ ] Error boundaries wrap all critical UI sections
- [ ] Fallback UI displays clear error messages
- [ ] Retry mechanism works correctly
- [ ] Errors are logged to main process
- [ ] Error boundaries do not prevent dev tools error display

**Analysis error recovery:**

- [ ] Timeout errors skip file and continue queue
- [ ] Parse errors mark file as unparseable
- [ ] Plugin crashes disable plugin after 3 errors
- [ ] Out-of-memory errors skip file with notification
- [ ] File not found errors remove from queue silently

**IPC error handling:**

- [ ] IPC validation prevents malformed payloads
- [ ] Timeout errors display user-friendly messages
- [ ] Failed IPC calls can be retried
- [ ] Error details logged for debugging
- [ ] Validation errors show specific field failures

**Security:**

- [ ] Error messages do not leak file system paths
- [ ] Stack traces not exposed to user in production
- [ ] Sensitive data sanitized in error logs
- [ ] Error recovery does not bypass security checks
- [ ] Malicious payloads rejected by Zod validation

#### 2.3 performance review checklist

**Virtualization:**

- [ ] 10,000 item list renders in under 100ms
- [ ] Scrolling maintains 60fps
- [ ] Memory usage stays under 200MB
- [ ] Search filters 10,000 items in under 50ms
- [ ] Dynamic row heights work correctly

**Bundle optimization:**

- [ ] Renderer bundle under 1MB uncompressed
- [ ] Initial load under 2s
- [ ] Code splitting reduces initial bundle by 40%+
- [ ] Tree-shaking eliminates unused code
- [ ] Dynamic imports lazy-load heavy components

**Database optimization:**

- [ ] Single file query completes in under 5ms
- [ ] 1000 file query completes in under 50ms
- [ ] Batch inserts process 100 files/second
- [ ] Full-text search returns results in under 100ms
- [ ] Database size stays under 100MB for 10,000 files

**General performance:**

- [ ] No blocking operations in renderer process
- [ ] Analysis runs in utility process without blocking UI
- [ ] IPC communication has minimal latency (< 10ms)
- [ ] File watching debouncing prevents queue overflow
- [ ] Memory usage stays stable over long sessions

#### 2.4 code quality review checklist

**Architecture:**

- [ ] Code follows CLAUDE.md architecture rules
- [ ] Plugin registry pattern used correctly
- [ ] No direct analyzer imports in renderer
- [ ] IPC boundaries properly validated with Zod
- [ ] Module boundaries respected

**TypeScript:**

- [ ] No `any` types without explicit justification
- [ ] Strict mode enabled
- [ ] All interfaces and types properly exported
- [ ] Type inference used where appropriate
- [ ] Generic types used correctly

**Testing:**

- [ ] Unit test coverage above 90%
- [ ] Integration test coverage above 80%
- [ ] All E2E user flows covered
- [ ] Tests run in under 30s
- [ ] No flaky tests

**Documentation:**

- [ ] All public APIs have JSDoc comments
- [ ] Complex algorithms have explanatory comments
- [ ] README files updated
- [ ] Architecture diagrams current
- [ ] User-facing docs accurate

---

### 3. review process

#### 3.1 pre-review preparation

**Developer responsibilities:**

1. Complete all Phase 5 implementations
2. Run full test suite and verify passing
3. Perform self-review against checklists
4. Generate performance benchmark reports
5. Document any known issues or limitations

**Artifacts to provide:**

- Source code for all Phase 5 modules
- Test results (unit, integration, E2E)
- Performance benchmark reports
- Bundle size analysis
- Database query profiling results
- User testing feedback summary

#### 3.2 review stages

**Stage 1: Automated checks (Day 1)**

- Run full test suite (unit, integration, E2E)
- Execute performance benchmarks
- Run bundle analyzer
- Execute database query profiler
- Check TypeScript strict mode compilation

**Stage 2: Manual review (Days 2-3)**

- **data-visualization-analyst**: Review visualization implementations
- **frontend-security-auditor**: Review error handling and security
- **architecture-reviewer**: Review overall architecture and quality

**Stage 3: Integration review (Day 4)**

- All reviewers convene to discuss findings
- Identify cross-cutting concerns
- Prioritize issues by severity
- Create remediation plan

**Stage 4: Re-review (Day 5)**

- Verify all critical issues resolved
- Spot-check medium priority fixes
- Sign-off on Phase 5 completion

#### 3.3 issue tracking

**Severity levels:**

| Severity     | Definition                                  | Action Required             |
| ------------ | ------------------------------------------- | --------------------------- |
| **Critical** | Blocks release, causes crashes or data loss | Must fix before release     |
| **High**     | Major functionality broken, security issue  | Must fix before release     |
| **Medium**   | Minor functionality broken, poor UX         | Should fix before release   |
| **Low**      | Cosmetic issue, nice-to-have                | Can defer to future release |

**Issue template:**

```markdown
### Issue: [Brief description]

**Severity**: Critical | High | Medium | Low
**Reviewer**: [Reviewer name]
**Component**: [Module/file path]

**Description**:
[Detailed description of the issue]

**Steps to reproduce**:

1. [Step 1]
2. [Step 2]

**Expected behavior**:
[What should happen]

**Actual behavior**:
[What currently happens]

**Recommendation**:
[How to fix]

**Related checklist item**:
[Reference to checklist item]
```

#### 3.4 sign-off criteria

**Phase 5 can be signed off when:**

- [ ] All critical and high severity issues resolved
- [ ] All medium severity issues resolved or deferred with justification
- [ ] All acceptance criteria met for each deliverable
- [ ] Test coverage meets minimum thresholds
- [ ] Performance benchmarks meet targets
- [ ] User testing shows > 90% task success rate
- [ ] All three reviewers provide written sign-off

**Sign-off template:**

```markdown
## Phase 5 Review Sign-Off

**Reviewer**: [Name]
**Date**: [Date]
**Status**: APPROVED | APPROVED WITH CONDITIONS | REJECTED

**Summary**:
[Brief summary of review findings]

**Critical issues**: [Count]
**High issues**: [Count]
**Medium issues**: [Count]
**Low issues**: [Count]

**Conditions for approval** (if applicable):

- [Condition 1]
- [Condition 2]

**Signature**: [Name]
```

---

## implementation roadmap

### week 1: advanced visualizations

**Days 1-2: D3.js treemap**

- Implement treemap data transformation
- Create TreemapChart component
- Add zoom/pan interactions
- Implement tooltips and click navigation

**Days 3-4: Canvas heatmap**

- Implement heatmap data bucketing
- Create HeatmapChart component with canvas rendering
- Add hover detection and tooltips
- Optimize rendering performance

**Day 5: Adaptive visualization**

- Implement visualization selector algorithm
- Create AdaptiveVisualization component
- Add user preference override
- Test with various dataset sizes

### week 2: error handling and progress

**Days 1-2: Error boundaries**

- Implement ErrorBoundary component
- Add fallback UI designs
- Integrate error logging
- Test with various error scenarios

**Days 3-4: Analysis error recovery**

- Implement AnalysisErrorHandler
- Add error recovery strategies
- Create user notification system
- Test plugin crash recovery

**Day 5: Progress indicators**

- Implement toast notification system
- Create progress bar components
- Add circular progress and spinners
- Integrate with analysis workflow

### week 3: performance optimization

**Days 1-2: Virtualization tuning**

- Optimize VirtualizedFileTable
- Implement memoized row components
- Add debounced search
- Create index-based lookup

**Days 3-4: Bundle and database optimization**

- Implement code splitting
- Configure tree-shaking
- Create prepared statements
- Optimize database indices

**Day 5: Performance testing**

- Run performance benchmarks
- Profile bundle sizes
- Test database query latency
- Optimize bottlenecks

### week 4: testing and refinement

**Days 1-2: Test implementation**

- Write unit tests for all modules
- Create integration tests
- Implement E2E test scenarios

**Days 3-4: User testing**

- Conduct user testing sessions
- Collect feedback and metrics
- Identify usability issues

**Day 5: Architecture review**

- Submit for architecture review
- Address review feedback
- Final refinement and polish

---

## success metrics

### quantitative metrics

| Metric                    | Target                        | Measurement Method       |
| ------------------------- | ----------------------------- | ------------------------ |
| **Treemap render time**   | < 500ms for 1000 files        | Performance benchmark    |
| **Heatmap frame rate**    | 60fps during scroll           | Chrome DevTools profiler |
| **Virtualization render** | < 100ms for 10,000 items      | Performance benchmark    |
| **Bundle size**           | < 1MB renderer bundle         | Webpack bundle analyzer  |
| **Database query time**   | < 50ms for 1000 files         | SQLite profiler          |
| **Test coverage**         | > 90% unit, > 80% integration | Vitest coverage report   |
| **User task success**     | > 90%                         | User testing results     |

### qualitative metrics

| Metric                          | Target                      | Measurement Method            |
| ------------------------------- | --------------------------- | ----------------------------- |
| **User satisfaction**           | > 4/5 average               | Post-test survey              |
| **Error message clarity**       | > 4/5 clarity rating        | User testing feedback         |
| **Visualization effectiveness** | > 4/5 usefulness rating     | User testing feedback         |
| **Learning curve**              | < 10 minutes to proficiency | Time to first successful task |

---

## risks and mitigations

### technical risks

| Risk                          | Likelihood | Impact | Mitigation                                   |
| ----------------------------- | ---------- | ------ | -------------------------------------------- |
| **D3.js learning curve**      | Medium     | High   | Allocate extra time, reference examples      |
| **Canvas performance issues** | Low        | High   | Implement fallback to SVG                    |
| **Virtualization bugs**       | Medium     | Medium | Extensive testing, use battle-tested library |
| **Bundle size bloat**         | High       | Medium | Aggressive code splitting, tree-shaking      |
| **Database performance**      | Low        | High   | Indexed queries, prepared statements         |

### schedule risks

| Risk                      | Likelihood | Impact | Mitigation                |
| ------------------------- | ---------- | ------ | ------------------------- |
| **Implementation delays** | Medium     | High   | Buffer 1 week in schedule |
| **User testing delays**   | Low        | Medium | Recruit testers early     |
| **Review feedback loops** | Medium     | Medium | Front-load quality checks |
| **Integration issues**    | Low        | High   | Daily integration testing |

---

## appendix: code examples

### adaptive visualization usage

```typescript
// src/renderer/pages/Dashboard.tsx
import { AdaptiveVisualization } from '../components/AdaptiveVisualization';
import { useAnalysisStore } from '../stores/analysis';

export function Dashboard() {
  const { files } = useAnalysisStore();

  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-6">Dashboard</h1>
      <AdaptiveVisualization files={Array.from(files.values())} />
    </div>
  );
}
```

### error boundary usage

```typescript
// src/renderer/App.tsx
import { ErrorBoundary } from './components/ErrorBoundary';

export function App() {
  return (
    <ErrorBoundary>
      <ApplicationShell>
        <Routes />
      </ApplicationShell>
    </ErrorBoundary>
  );
}
```

### progress indicator integration

```typescript
// src/renderer/hooks/useAnalysis.ts
import { useAnalysisStore } from '../stores/analysis';
import { useUIStore } from '../stores/ui';

export function useAnalysis() {
  const { analyzeFiles } = useAnalysisStore();
  const { showToast } = useUIStore();

  const startAnalysis = async (paths: string[]) => {
    try {
      await analyzeFiles(paths);
      showToast({
        type: 'success',
        title: 'Analysis complete',
        message: `Analyzed ${paths.length} files`,
      });
    } catch (error) {
      showToast({
        type: 'error',
        title: 'Analysis failed',
        message: error.message,
      });
    }
  };

  return { startAnalysis };
}
```

---

## files modified

- **Create**: `documentation/docs/feature-development/electron-app/07-polish.md` (this file)

---

## verification checklist

- [ ] Document follows lowercase-dashed filename convention
- [ ] No emojis used in document
- [ ] All code examples use TypeScript
- [ ] Mermaid diagrams validated (none in this document)
- [ ] All tables properly formatted
- [ ] References to CLAUDE.md architecture rules included
- [ ] Cross-references to other documents accurate
- [ ] Acceptance criteria are testable and measurable
- [ ] Performance targets are specific and realistic
- [ ] Review process is detailed and actionable
- [ ] Implementation roadmap is realistic (3-4 weeks)
- [ ] Success metrics are quantifiable
- [ ] Risks identified with mitigations

---

## architecture review summary

This section contains the comprehensive architecture review conducted on 2026-02-02.

### overall assessment

**Status**: APPROVED WITH RECOMMENDATIONS

The Phase 5 polish implementation plan is comprehensive and well-structured. The technical approach is sound, but several areas require refinement before implementation begins. The plan demonstrates good understanding of performance optimization, error handling, and user experience concerns.

**Key Strengths**:

- Comprehensive error handling strategy with graceful degradation
- Performance targets are measurable and include methodology
- Multiple visualization options provide flexibility for different dataset sizes
- Test coverage plan includes unit, integration, and E2E testing
- Review process is well-defined with clear sign-off criteria

**Key Concerns**:

- Some performance targets may be overly aggressive or under-specified
- Accessibility considerations need more attention throughout
- Memory management patterns have potential leaks
- Type safety could be improved in several areas
- Testing targets (90% unit, 80% integration) may be unrealistic

### critical recommendations

These issues must be addressed before implementation begins:

#### 1. architecture and code reuse

**Issue**: Lack of code reuse strategy across desktop and potential future clients.

**Recommendation**: Extract visualization logic into shared package at `@vipr/visualizations` to enable reuse. This aligns with the monorepo structure and prevents duplication when web client is built.

**Impact**: Medium - affects long-term maintainability but not immediate functionality.

#### 2. type safety improvements

**Issue**: Several type safety concerns identified:

- `createSafeIpcInvoke` uses same schema for request and response (should be separate)
- `TreemapNode.fileType` should use `FileType` from `@vipr/common` for consistency
- Missing branded types for file paths to prevent injection attacks

**Recommendation**:

```typescript
// Update IPC helper signature
export function createSafeIpcInvoke<TRequest extends z.ZodType, TResponse extends z.ZodType>(
  channel: string,
  requestSchema: TRequest,
  responseSchema: TResponse // Validate responses too
);

// Use consistent types
import { FileType } from '@vipr/common';
interface TreemapNode {
  fileType?: FileType; // Not string
  // ...
}

// Consider branded types for security
type FilePath = string & { readonly __brand: 'FilePath' };
```

**Impact**: High - type safety prevents runtime errors and improves developer experience.

#### 3. security and privacy

**Issue**: Multiple security concerns identified:

- Error messages may leak absolute file paths containing usernames
- Stack traces exposed in production builds
- No sanitization of user-provided data in visualizations
- Console logging in production

**Recommendation**:

```typescript
// Sanitize stack traces
private sanitizeStackTrace(stack?: string): string | undefined {
  if (!stack) return undefined;
  return stack.replace(/\/Users\/[^/]+\//g, '[USER_HOME]/');
}

// Conditional logging
if (process.env.NODE_ENV === 'development') {
  console.error('Error:', error);
}

// Always log to main process instead
window.viprDesktop.logging.error({
  message: error.message,
  stack: this.sanitizeStackTrace(error.stack)
});
```

**Impact**: High - security issues must be resolved before release.

#### 4. memory management

**Issue**: `AnalysisErrorHandler.errorCounts` Map grows unbounded, causing memory leak over time.

**Recommendation**:

```typescript
export class AnalysisErrorHandler {
  private errorCounts = new Map<string, number>();
  private errorTimestamps = new Map<string, number>();

  private cleanupOldErrors(): void {
    const oneHourAgo = Date.now() - 60 * 60 * 1000;
    for (const [key, timestamp] of this.errorTimestamps.entries()) {
      if (timestamp < oneHourAgo) {
        this.errorCounts.delete(key);
        this.errorTimestamps.delete(key);
      }
    }
  }

  handleSuccess(pluginId?: string): void {
    // Reset error count for this plugin on success
    for (const key of this.errorCounts.keys()) {
      if (key.endsWith(`:${pluginId || 'core'}`)) {
        this.errorCounts.delete(key);
        this.errorTimestamps.delete(key);
      }
    }
  }
}
```

**Impact**: Medium - will manifest as gradual memory growth over long sessions.

### performance review

#### visualization performance targets

**Treemap (1000 files < 500ms)**:

- Target is reasonable but should specify hardware baseline (e.g., "on 2020 M1 MacBook Air")
- Consider web worker for hierarchy calculation on 5000+ files
- 300ms debounce on resize may feel laggy - reduce to 150ms with RAF throttling
- Recommend progressive rendering for better perceived performance

**Heatmap (60fps scrolling)**:

- Canvas implementation is appropriate for performance
- Dirty region tracking may add unnecessary complexity - profile first before implementing
- Hover debouncing (16ms) adds latency - use RAF throttling instead for smoother response
- Spatial hash for hover detection is excellent optimization

**Adaptive Visualization (50ms selection)**:

- 50ms is too slow for UI decision logic - target 5ms maximum (should be pure computation)
- Decision thresholds (50, 200, 500 files) are arbitrary - needs empirical user validation
- Consider scoring system rather than imperative decision tree for better maintainability
- Doughnut charts are notoriously hard to read - consider horizontal bar chart alternative

#### virtualization performance

**10,000 item rendering (< 100ms)**:

- Target is ambitious but achievable with TanStack Virtual
- Question if 10,000 files is realistic target (99th percentile GitHub repo is ~5,000 files)
- Memory target (< 200MB) needs clarification: total app memory or just virtualized list?
- Search performance (< 50ms) is imperceptible to users - 100ms would simplify implementation

#### bundle optimization

**Electron-specific considerations**:

- Gzipped size metric is irrelevant for Electron (no network transfer)
- Focus on parse/compile time and memory rather than file size
- 1MB renderer bundle is ambitious but achievable with aggressive code splitting
- Missing important metrics: V8 code cache hit rate, memory usage after TTI, parse time

### testing strategy review

#### coverage targets

**Unit tests (90%)**: Very high - industry standard is 70-80%. Focus on critical paths and edge cases rather than arbitrary percentage.

**Integration tests (80%)**: Good target but needs clear definition of what "integration test coverage" means - line coverage of integration tests, or percentage of integration points tested?

**E2E tests**: "All user flows" is vague - need explicit list of critical flows (must test) vs nice-to-have flows.

**Test execution time (< 30s total)**: Unrealistic for comprehensive E2E suite. Recommend split targets:

- Unit tests: < 10s (fast feedback loop)
- Integration tests: < 30s (acceptable for CI)
- E2E tests: < 2 minutes (one-time validation)

**Flaky tests**: Define quantitative threshold rather than "none" (e.g., "passes 99% of time over 100 runs").

#### user testing methodology

**Strengths**:

- Test participant mix covers key personas (developers, PMs, consultants, startup engineers)
- Test scenarios are realistic and cover critical workflows
- Metrics are measurable (task success rate, time on task, satisfaction scores)
- Testing schedule includes iteration cycles

**Recommendations**:

- Add "observable moments" to each scenario (what should tester say/do that indicates understanding?)
- Include "red routes" analysis (identify 3-5 most critical paths users MUST succeed at)
- Consider think-aloud protocol for qualitative insights beyond metrics
- Plan for at least two iteration cycles - one round of testing rarely captures all issues

### accessibility concerns

Multiple accessibility issues identified throughout the plan:

#### visualizations

- Minimum viewport 800x600px excludes users with smaller screens or accessibility zoom needs
- Color scales must work for deuteranopia/protanopia (most common types of colorblindness) - test with simulators
- Keyboard navigation must reach all interactive elements (treemap nodes, heatmap cells)
- Screen readers must announce visualization changes (use ARIA live regions)
- Focus indicators must be visible with 3:1 contrast ratio minimum

#### toast notifications

- May not be announced by screen readers if focus is elsewhere (need role="alert" or role="status")
- Auto-dismiss timing may not give users enough time to read, especially errors (30s+ for errors)
- Motion-sensitive users need prefers-reduced-motion support (reduce or eliminate animations)
- Must be keyboard dismissible (Escape key)
- Consider native Electron notifications for important messages instead of custom toasts

#### error messages

- Error toasts should not auto-dismiss, or have very long duration (30s minimum)
- Must be keyboard accessible with clear focus management
- Position should be configurable (some users need different corners to avoid blocking content)
- Content must be announced to screen readers immediately for errors

**Recommendation**: Add comprehensive accessibility section to implementation plan with WCAG 2.2 Level AA compliance checklist. Test with screen readers (NVDA, JAWS, VoiceOver) and keyboard-only navigation.

### error handling architecture

The error handling strategy is comprehensive but has implementation issues:

#### error boundary

**Good practices**:

- Proper use of React error boundaries
- Fallback UI with recovery actions
- Logging to main process for debugging

**Issues identified and fixed**:

- Added privacy sanitization for file paths (remove usernames)
- Added error fingerprinting for aggregation/telemetry
- Added conditional console logging (dev only)
- Noted React 18 Suspense boundary opportunities

#### analysis error recovery

**Good practices**:

- Comprehensive error type coverage (timeout, parse, plugin crash, OOM, file not found, permission denied)
- Clear recovery strategies for each error type
- Plugin disabling after repeated failures

**Issues identified and fixed**:

- "Consecutive" error logic actually counted total errors, not consecutive - needs success tracking
- Memory leak in errorCounts Map - added cleanup with timestamps
- 30s timeout is very long - recommend progressive timeout (5s warning, 10s skip, 30s only for known-large files)
- Missing encoding errors, circular symlinks, disk full errors

#### IPC error handling

**Good practices**:

- Zod validation at all IPC boundaries
- Timeout protection (30s reasonable for analysis operations)
- User-friendly error messages

**Issues identified and fixed**:

- Added response validation (not just request)
- Added production-safe logging (no console in production)
- Fixed type signature (separate request/response schemas)
- Added telemetry tracking for IPC error rates

### implementation roadmap assessment

**Timeline (4 weeks)**: Reasonable but tight.

**Risk assessment by week**:

- Week 1 (visualizations): HIGH RISK - D3.js learning curve, canvas performance optimization
- Week 2 (error handling): MEDIUM RISK - well-understood patterns but thorough testing needed
- Week 3 (performance): MEDIUM RISK - known optimization techniques but profiling takes time
- Week 4 (testing/refinement): HIGH RISK - user testing often reveals major usability issues requiring redesign

**Missing tasks**:

- Accessibility implementation (keyboard navigation, ARIA, screen reader testing)
- Performance monitoring instrumentation (track metrics in production)
- Error telemetry setup (aggregate and analyze error patterns)
- Security audit of file path handling (penetration testing with malicious repos)
- Dark mode implementation for all visualizations

**Recommendation**: Add 1-week buffer to timeline (total 5 weeks) to accommodate:

- User testing iteration cycles
- Accessibility implementation and testing
- Performance optimization based on profiling data
- Bug fixes discovered during integration

### adherence to CLAUDE.md architecture rules

The implementation plan generally follows CLAUDE.md architecture rules with no major violations identified.

**Confirmed compliance**:

- No mention of direct analyzer imports (plugin system will be respected)
- IPC boundaries validated with Zod schemas
- TypeScript-first approach throughout
- Module boundaries are clearly defined
- File type detection will use `@vipr/common` utilities

**Areas requiring verification during implementation**:

- Desktop app must not import analyzer code directly - all analysis happens in main process via IPC
- Plugin coordination patterns must be followed (React defers to Next.js, etc.)
- Presenter registry queries must happen via IPC from renderer to main process, not direct imports
- File type detection must be consistent with `getFileTechnologies()` from `@vipr/common`

**Recommendations for CLAUDE.md alignment**:

- Add explicit architecture diagram showing IPC boundaries between renderer and main process
- Document how desktop app loads plugins (should mirror CLI's CliPluginLoader pattern)
- Clarify that renderer process has zero direct dependencies on analyzer packages
- Add note about technology detection using shared utilities from `@vipr/common`

### additional recommendations

#### code organization

Consider these organizational improvements:

1. **Shared visualization package** (`@vipr/visualizations`):
   - Extract D3 treemap, canvas heatmap, adaptive selector
   - Enables reuse in future web client
   - Provides consistent visualization experience
   - Makes testing easier (test once, use everywhere)

2. **Error handling utilities** (`@vipr/error-handling` or in `@vipr/common`):
   - Stack trace sanitization
   - Error fingerprinting for telemetry
   - Error classification (user-facing vs developer-facing)
   - Shared error types across all packages

3. **Performance monitoring** (in desktop app initially, extract to `@vipr/telemetry` later):
   - Track rendering performance (TTI, FCP, FID)
   - Monitor memory usage over time
   - Measure IPC latency by channel
   - Bundle size tracking in CI

#### developer experience

Improvements for development workflow:

1. **Performance benchmarks in CI**:
   - Automated performance tests on every PR
   - Fail PR if bundle size increases by >10%
   - Track rendering performance over time
   - Alert on performance regressions

2. **Accessibility testing in CI**:
   - Automated WCAG compliance checks (e.g., axe-core)
   - Keyboard navigation tests in E2E suite
   - Color contrast verification
   - Screen reader announcement testing (where feasible)

3. **Visual regression testing**:
   - Capture screenshots of visualizations with test data
   - Detect unintended visual changes
   - Particularly important for canvas rendering

### final recommendations summary

**Before beginning implementation**:

1. **Critical** (must address):
   - Update IPC helper to use separate request/response schemas
   - Implement stack trace sanitization and production logging guards
   - Fix memory leak in AnalysisErrorHandler (bounded maps, cleanup)
   - Add success tracking for proper "consecutive errors" logic
   - Specify hardware baselines for all performance targets

2. **High priority** (should address):
   - Add comprehensive accessibility section with WCAG 2.2 checklist
   - Refine testing targets to be more realistic (70-80% coverage)
   - Split test execution time targets by test type
   - Add 1-week buffer to implementation timeline
   - Define "all user flows" explicitly for E2E testing

3. **Medium priority** (recommended):
   - Extract visualizations to shared `@vipr/visualizations` package
   - Implement performance monitoring instrumentation
   - Add error telemetry for production monitoring
   - Update adaptive visualization to use scoring system
   - Consider native Electron notifications for important messages

4. **Low priority** (nice to have):
   - Add visual regression testing
   - Implement progressive rendering for treemap
   - Use web worker for hierarchy calculation (5000+ files)
   - Add dark mode support to all visualizations
   - Performance benchmarks in CI

### sign-off

**Reviewer**: architecture-reviewer (Claude Sonnet 4.5)
**Date**: 2026-02-02
**Status**: APPROVED WITH CONDITIONS

**Summary**: The Phase 5 polish plan is well-conceived, comprehensive, and demonstrates strong technical understanding. The plan provides excellent coverage of visualizations, error handling, progress indicators, and performance optimization. With the recommended improvements, particularly around type safety, security, accessibility, and more realistic performance/testing targets, this plan provides a solid foundation for successful implementation.

**Critical issues**: 4

- IPC type safety (separate request/response schemas)
- Security (sanitize stack traces, conditional logging)
- Memory leak (bounded error counts with cleanup)
- Missing accessibility implementation plan

**High issues**: 5

- Performance targets lack hardware baselines
- Testing targets unrealistic (90% unit coverage)
- User testing flows not explicitly defined
- Timeline too aggressive (needs 1-week buffer)
- Missing success tracking in error handler

**Medium issues**: 8

- Adaptive visualization selection too slow (50ms → 5ms)
- Arbitrary decision thresholds need validation
- Doughnut charts hard to read
- Toast notifications accessibility concerns
- Console logging in production
- Error message auto-dismiss timing
- Missing dark mode consideration
- No telemetry/monitoring plan

**Low issues**: Multiple code quality and optimization opportunities documented inline

**Conditions for final approval**:

1. Address all 4 critical security/type safety issues
2. Fix identified memory leak in AnalysisErrorHandler
3. Add comprehensive accessibility section with WCAG 2.2 checklist
4. Refine performance targets with hardware baselines and clear measurement methodology
5. Update testing targets to be more realistic and explicitly define critical user flows

**Approved for implementation**: Yes, with the understanding that critical conditions will be addressed in first implementation sprint before writing production code. Medium/low priority improvements can be incorporated progressively or deferred based on team capacity.

The implementation team should review all inline comments (marked with `<!-- ARCHITECTURE NOTE -->`, `<!-- PERFORMANCE NOTE -->`, `<!-- SECURITY NOTE -->`, etc.) before beginning work. These comments provide critical context and guidance that will prevent common pitfalls and ensure high-quality implementation.
