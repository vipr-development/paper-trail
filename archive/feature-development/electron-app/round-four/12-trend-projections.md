---
id: 12-trend-projections
title: Forward Trend Projections
phase: 4
dependencies: [11]
status: planned
---

# Forward Trend Projections

## Problem Statement

Trend charts stop at today. Users cannot see whether the current trajectory will breach a quality threshold — for example, "at this rate, our score will drop below 60 in 45 days." Forward projections help teams take preventive action before degradation compounds. This phase adds projection overlays and threshold-crossing alerts to both the Overview trend panel and the Velocity Dashboard introduced in Phase 11.

## New Files

| File                                                                            | Role                                                                                  |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/hooks/useTrendProjection.ts`                      | Computes projection points and threshold-crossing date from linear regression metrics |
| `clients/desktop/src/renderer/components/overview/ProjectionOverlay.tsx`        | Adds dashed projection line + confidence bands to an existing Chart.js LineChart      |
| `clients/desktop/src/renderer/components/overview/ProjectionThresholdAlert.tsx` | Renders a warning Alert when threshold crossing is detected                           |

## All Math is Client-Side

No new IPC channels are introduced. `velocity.getRepoTrend` already returns a `VelocityMetrics` object containing `{ slope, r2, intercept }`. This hook consumes those values directly.

## Hook: `useTrendProjection`

```typescript
// clients/desktop/src/renderer/hooks/useTrendProjection.ts

interface VelocityMetrics {
  slope: number; // points per day (can be negative)
  intercept: number; // y-intercept of the regression line
  r2: number; // goodness of fit; 0 (no fit) to 1 (perfect fit)
}

interface ProjectionPoint {
  date: Date;
  projected: number;
  upperBand: number;
  lowerBand: number;
}

interface UseTrendProjectionOptions {
  metrics: VelocityMetrics;
  lastKnownDate: Date;
  lastKnownScore: number;
  horizonDays: 30 | 60 | 90;
}

interface UseTrendProjectionReturn {
  projectionPoints: ProjectionPoint[];
  thresholdCrossing: {
    date: Date;
    projectedScore: number;
    threshold: number;
  } | null;
}

export function useTrendProjection(
  options: UseTrendProjectionOptions,
  threshold?: number // defaults to 60
): UseTrendProjectionReturn;
```

### Projection Formula

For each day `d` from `1` to `horizonDays`:

```
date             = lastKnownDate + d days
projected        = lastKnownScore + slope × d
confidenceFactor = (1 - r2) × 10 × (d / 30)
upperBand        = projected + confidenceFactor
lowerBand        = projected - confidenceFactor
```

The confidence band widens with time (more uncertainty at distance) and narrows with higher `r2` (better regression fit). Both `projected`, `upperBand`, and `lowerBand` are clamped to `[0, 100]` before returning — scores outside the valid range have no meaning.

### Threshold Crossing Detection

After building `projectionPoints`, scan forward and find the first point where `projected < threshold`:

```
thresholdCrossing = projectionPoints.find(p => p.projected < threshold) ?? null

If found:
  thresholdCrossing = {
    date: point.date,
    projectedScore: point.projected,
    threshold,
  }
```

### Memoization

Wrap the entire computation in `useMemo` — dependencies are `metrics.slope`, `metrics.intercept`, `metrics.r2`, `lastKnownDate`, `lastKnownScore`, `horizonDays`, and `threshold`.

## Component: `ProjectionOverlay`

```typescript
// clients/desktop/src/renderer/components/overview/ProjectionOverlay.tsx

interface ProjectionOverlayProps {
  historicalDataset: { x: Date; y: number }[];
  projectionPoints: ProjectionPoint[];
  threshold?: number;
}
```

This component does not render DOM elements. It imperatively mutates the `ChartJS` instance it receives via a ref forwarded from the parent chart component.

> **Implementation prerequisite:** Verify that the `LineChart` component in `@vipr/ui` forwards a `ChartJS` ref via `React.forwardRef` before implementing this pattern. If `LineChart` does not currently forward a ref, add ref-forwarding to that component first (a one-line change: wrap the component with `React.forwardRef` and pass the ref to the underlying Chart.js `canvas`). Attempting to use `chartRef.current` without this will always yield `null`.

Call pattern:

```typescript
// Parent (OverviewTrendPanel or VelocityTrendSection):
const chartRef = useRef<ChartJS | null>(null);

<LineChart ref={chartRef} ... />
{projectionEnabled && (
  <ProjectionOverlay
    chartRef={chartRef}
    historicalDataset={scoreDataset}
    projectionPoints={projectionPoints}
    threshold={60}
  />
)}
```

When mounted, `ProjectionOverlay` uses `useEffect` to call `chart.data.datasets.push(...)` for each of the following datasets, then `chart.update('none')`:

| Dataset        | Style                                                                                                                                                                                                                               |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Projected line | `borderDash: [6, 3]`, `borderColor: rgba(59, 130, 246, 0.5)`, `pointRadius: 0`                                                                                                                                                      |
| Upper band     | `fill: '+1'`, `backgroundColor: rgba(59, 130, 246, 0.08)`, `pointRadius: 0`, `borderWidth: 0`                                                                                                                                       |
| Lower band     | `fill: false`, `borderWidth: 0`, `pointRadius: 0`                                                                                                                                                                                   |
| Threshold line | Horizontal annotation using the Chart.js annotation plugin: `type: 'line'`, `yMin: threshold`, `yMax: threshold`, `borderColor: rgba(239, 68, 68, 0.7)`, `borderDash: [4, 4]`, `label: { content: 'Threshold: 60', enabled: true }` |

On unmount, remove only the datasets added by this component (track by `dataset.label` prefix `'__projection'`) and call `chart.update('none')`.

## Component: `ProjectionThresholdAlert`

```typescript
// clients/desktop/src/renderer/components/overview/ProjectionThresholdAlert.tsx

interface ProjectionThresholdAlertProps {
  thresholdCrossing: {
    date: Date;
    projectedScore: number;
    threshold: number;
  } | null;
  className?: string;
}
```

When `thresholdCrossing` is `null`, the component renders `null`.

When non-null, renders an `Alert` component from `@vipr/ui` with `variant="notification"` and `type="warning"`:

```
Quality projection warning

At the current rate, your score is projected to drop below {threshold} on
{format(date, 'MMM d')} (in {differenceInDays(date, new Date())} days).
Consider addressing high-velocity debt files.
```

Use `date-fns` `format` and `differenceInDays` for date formatting. The `className` prop is forwarded to the `Alert` wrapper for layout positioning by the parent.

## Pages to Modify

### `OverviewTrendPanel.tsx`

File: `clients/desktop/src/renderer/pages/Overview/OverviewTrendPanel.tsx`

Add state and controls to the existing panel header:

```typescript
const [projectionEnabled, setProjectionEnabled] = useState(false);
const [horizonDays, setHorizonDays] = useState<30 | 60 | 90>(30);

const { projectionPoints, thresholdCrossing } = useTrendProjection(
  {
    metrics: velocityMetrics, // from existing velocity.getRepoTrend call
    lastKnownDate: lastHistoricalDate,
    lastKnownScore: lastHistoricalScore,
    horizonDays,
  },
  60
);
```

Add to panel header controls (right-aligned, alongside any existing controls):

```typescript
<Switch
  checked={projectionEnabled}
  onChange={setProjectionEnabled}
  label="Show projection"
/>
{projectionEnabled && (
  <Dropdown
    variant="select"
    value={horizonDays}
    onChange={setHorizonDays}
    options={[
      { label: '30 days', value: 30 },
      { label: '60 days', value: 60 },
      { label: '90 days', value: 90 },
    ]}
  />
)}
```

Render conditionally below the chart:

```typescript
{projectionEnabled && velocityMetrics && (
  <>
    <ProjectionOverlay
      chartRef={chartRef}
      historicalDataset={scoreDataset}
      projectionPoints={projectionPoints}
    />
    <ProjectionThresholdAlert
      thresholdCrossing={thresholdCrossing}
      className="mt-4"
    />
  </>
)}
```

### `Velocity.tsx` — `VelocityTrendSection` integration

File: `clients/desktop/src/renderer/pages/Velocity.tsx`

Extend `VelocityFilters` to include the projection controls (add `projectionEnabled` and `horizonDays` props to `VelocityFiltersProps`). Mount `ProjectionOverlay` and `ProjectionThresholdAlert` identically to the Overview panel, using the velocity page's own `velocityMetrics` state from the `velocity.getRepoTrend` response.

## Testing

### `useTrendProjection.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useTrendProjection.test.ts

describe('useTrendProjection', () => {
  it('generates projectionPoints equal to horizonDays in length', () => {
    const { projectionPoints } = useTrendProjection(
      {
        metrics: { slope: -0.5, intercept: 80, r2: 0.8 },
        lastKnownDate: new Date('2024-01-01'),
        lastKnownScore: 75,
        horizonDays: 30,
      },
      60
    );
    expect(projectionPoints).toHaveLength(30);
  });

  it('confidence band widens with time', () => {
    const { projectionPoints } = useTrendProjection({
      metrics: { slope: 0, intercept: 75, r2: 0.5 },
      lastKnownDate: new Date(),
      lastKnownScore: 75,
      horizonDays: 30,
    });
    const day1Width = projectionPoints[0].upperBand - projectionPoints[0].lowerBand;
    const day30Width = projectionPoints[29].upperBand - projectionPoints[29].lowerBand;
    expect(day30Width).toBeGreaterThan(day1Width);
  });

  it('confidence band is narrower with r2=0.95 than r2=0.3', () => {
    const makeOptions = (r2: number) => ({
      metrics: { slope: 0, intercept: 75, r2 },
      lastKnownDate: new Date(),
      lastKnownScore: 75,
      horizonDays: 30,
    });
    const { projectionPoints: highR2 } = useTrendProjection(makeOptions(0.95));
    const { projectionPoints: lowR2 } = useTrendProjection(makeOptions(0.3));
    const highWidth = highR2[29].upperBand - highR2[29].lowerBand;
    const lowWidth = lowR2[29].upperBand - lowR2[29].lowerBand;
    expect(highWidth).toBeLessThan(lowWidth);
  });

  it('detects threshold crossing when slope is sufficiently negative', () => {
    const { thresholdCrossing } = useTrendProjection(
      {
        metrics: { slope: -1, intercept: 80, r2: 0.7 },
        lastKnownDate: new Date('2024-01-01'),
        lastKnownScore: 75,
        horizonDays: 90,
      },
      60
    );
    expect(thresholdCrossing).not.toBeNull();
    expect(thresholdCrossing?.projectedScore).toBeLessThanOrEqual(60);
  });

  it('returns null thresholdCrossing when score stays above threshold', () => {
    const { thresholdCrossing } = useTrendProjection(
      {
        metrics: { slope: 0.2, intercept: 75, r2: 0.9 },
        lastKnownDate: new Date('2024-01-01'),
        lastKnownScore: 75,
        horizonDays: 30,
      },
      60
    );
    expect(thresholdCrossing).toBeNull();
  });

  it('clamps projected scores to [0, 100]', () => {
    const { projectionPoints } = useTrendProjection({
      metrics: { slope: -5, intercept: 80, r2: 0.9 },
      lastKnownDate: new Date(),
      lastKnownScore: 10,
      horizonDays: 30,
    });
    projectionPoints.forEach(p => {
      expect(p.projected).toBeGreaterThanOrEqual(0);
      expect(p.projected).toBeLessThanOrEqual(100);
    });
  });
});
```

### `ProjectionThresholdAlert.test.tsx`

```typescript
// clients/desktop/src/renderer/components/overview/ProjectionThresholdAlert.test.tsx

describe('ProjectionThresholdAlert', () => {
  it('renders nothing when thresholdCrossing is null', () => {
    const { container } = render(<ProjectionThresholdAlert thresholdCrossing={null} />);
    expect(container).toBeEmptyDOMElement();
  });

  it('shows the formatted crossing date in the warning message', () => {
    const crossing = {
      date: new Date('2024-03-15'),
      projectedScore: 58.3,
      threshold: 60,
    };
    render(<ProjectionThresholdAlert thresholdCrossing={crossing} />);
    expect(screen.getByText(/Mar 15/)).toBeInTheDocument();
  });

  it('shows the threshold value in the warning message', () => {
    const crossing = {
      date: new Date('2024-03-15'),
      projectedScore: 58.3,
      threshold: 60,
    };
    render(<ProjectionThresholdAlert thresholdCrossing={crossing} />);
    expect(screen.getByText(/below 60/)).toBeInTheDocument();
  });

  it('shows days remaining in the warning message', () => {
    // Use a fixed crossing date 23 days in the future
    // Assert: "in 23 days" present in rendered output
  });
});
```
