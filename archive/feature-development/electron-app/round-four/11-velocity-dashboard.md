---
id: 11-velocity-dashboard
title: Velocity Dashboard Page
phase: 4
dependencies: [08]
status: planned
---

# Velocity Dashboard Page

## Problem Statement

ChurnAnalysis has a scatter plot but no page answers "how fast is quality changing over time, bucketed by week or sprint?" Users cannot see velocity trends, compare metric improvement rates, or identify which files are improving fastest. This phase adds a dedicated `/velocity` route with time-bucketed charts, per-metric breakdown, a file leaderboard, and inflection point detection — all powered by pre-existing IPC channels from the velocity feature.

## New Files

| File                                                                                | Role                                                          |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `clients/desktop/src/renderer/pages/Velocity.tsx`                                   | Top-level page, layout grid, data orchestration               |
| `clients/desktop/src/renderer/hooks/useVelocityBuckets.ts`                          | Pure client-side bucketing of raw trend points                |
| `clients/desktop/src/renderer/components/velocity/VelocityFilters.tsx`              | Time range, bucket size, metric, and directory scope controls |
| `clients/desktop/src/renderer/components/velocity/VelocityTrendSection.tsx`         | Dual-axis LineChart: quality score + velocity overlay         |
| `clients/desktop/src/renderer/components/velocity/VelocityMetricBreakdownTable.tsx` | Per-metric velocity rates and confidence table                |
| `clients/desktop/src/renderer/components/velocity/VelocityLeaderboardSection.tsx`   | Most/least improved file CardTables                           |
| `clients/desktop/src/renderer/components/velocity/InflectionPointsSection.tsx`      | Commits where velocity changed direction                      |

## Page Layout (`Velocity.tsx`)

```
┌─ Velocity Dashboard ──────────────────────────────────────────────────────┐
│ VelocityFilters (time range | bucket size | metric | directory scope)   │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  VelocityTrendSection                                                     │
│  [LineChart: quality score over time + velocity overlay (dashed)]         │
│                                                                           │
├──────────────────────────────┬────────────────────────────────────────────┤
│ VelocityMetricBreakdownTable │ VelocityLeaderboardSection                 │
│ (per-metric velocity rates)  │ (most / least improved files)              │
├──────────────────────────────┴────────────────────────────────────────────┤
│ InflectionPointsSection                                                   │
│ (commits where velocity changed direction, links to commit browser)       │
└───────────────────────────────────────────────────────────────────────────┘
```

Grid rules: `grid grid-cols-12 gap-6`.

| Region                         | Class                       |
| ------------------------------ | --------------------------- |
| `VelocityFilters`              | `col-span-full`             |
| `VelocityTrendSection`         | `col-span-full`             |
| `VelocityMetricBreakdownTable` | `col-span-12 lg:col-span-5` |
| `VelocityLeaderboardSection`   | `col-span-12 lg:col-span-7` |
| `InflectionPointsSection`      | `col-span-full`             |

### `Velocity.tsx` Skeleton

```typescript
// clients/desktop/src/renderer/pages/Velocity.tsx

import { useState } from 'react';
import { useVelocityBuckets } from '../hooks/useVelocityBuckets';
import { VelocityFilters } from '../components/velocity/VelocityFilters';
import { VelocityTrendSection } from '../components/velocity/VelocityTrendSection';
import { VelocityMetricBreakdownTable } from '../components/velocity/VelocityMetricBreakdownTable';
import { VelocityLeaderboardSection } from '../components/velocity/VelocityLeaderboardSection';
import { InflectionPointsSection } from '../components/velocity/InflectionPointsSection';

export function Velocity() {
  const [timeRange, setTimeRange] = useState<'30d' | '90d' | '6mo' | '1yr' | 'all'>('90d');
  const [bucketSize, setBucketSize] = useState<'week' | 'month'>('week');
  const [metric, setMetric] = useState<'overall' | 'cyclomatic' | 'halstead' | 'maintainability'>('overall');
  const [directoryScope, setDirectoryScope] = useState<string | null>(null);

  // IPC data fetched via existing channels — see "IPC Used" section
  // useEffect blocks call window.api.velocity.getRepoTrend, getLeaderboard,
  // getMetricBreakdown, getInflectionPoints when filters change

  const { buckets, scoreDataset, velocityDataset } = useVelocityBuckets({
    data: trendPoints,
    bucketSize,
    startDate,
    endDate,
  });

  return (
    <div className="grid grid-cols-12 gap-6 p-6">
      <div className="col-span-full">
        <VelocityFilters
          timeRange={timeRange}
          bucketSize={bucketSize}
          metric={metric}
          directoryScope={directoryScope}
          onTimeRangeChange={setTimeRange}
          onBucketSizeChange={setBucketSize}
          onMetricChange={setMetric}
          onDirectoryScopeChange={setDirectoryScope}
        />
      </div>
      <div className="col-span-full">
        <VelocityTrendSection
          buckets={buckets}
          scoreDataset={scoreDataset}
          velocityDataset={velocityDataset}
          isLoading={trendLoading}
        />
      </div>
      <div className="col-span-12 lg:col-span-5">
        <VelocityMetricBreakdownTable data={metricBreakdown} isLoading={breakdownLoading} />
      </div>
      <div className="col-span-12 lg:col-span-7">
        <VelocityLeaderboardSection
          mostImproved={leaderboard.mostImproved}
          leastImproved={leaderboard.leastImproved}
          isLoading={leaderboardLoading}
        />
      </div>
      <div className="col-span-full">
        <InflectionPointsSection
          inflectionPoints={inflectionPoints}
          onNavigateToCommit={(sha) => setCommitDrawerSha(sha)}
          isLoading={inflectionLoading}
        />
      </div>
    </div>
  );
}
```

## Hook: `useVelocityBuckets`

Pure client-side bucketing — no IPC. Consumes raw `VelocityTrendPoint[]` from the trend response and groups points into discrete time buckets.

> **Dependency note:** `date-fns` is not currently installed in `clients/desktop`. Before implementing `useVelocityBuckets`, either add it to `clients/desktop/package.json` (`pnpm add date-fns`) or replace the `date-fns` import with native `Date` arithmetic. Both approaches are acceptable; prefer `date-fns` for correctness on DST boundaries, but native is fine for week/month approximations.

```typescript
// clients/desktop/src/renderer/hooks/useVelocityBuckets.ts

// If date-fns is added as a dependency:
import {
  startOfWeek,
  startOfMonth,
  eachWeekOfInterval,
  eachMonthOfInterval,
  addWeeks,
  addMonths,
  isWithinInterval,
} from 'date-fns';
// Otherwise, implement equivalent helpers using native Date operations.

interface VelocityTrendPoint {
  date: number; // epoch ms — from velocity.getRepoTrend IPC
  score: number;
}

interface VelocityBucket {
  bucketStart: Date;
  bucketEnd: Date;
  avgScore: number;
  scoreChange: number; // change from previous bucket; 0 for first bucket
  snapshotCount: number; // snapshots included in this bucket
}

interface UseVelocityBucketsOptions {
  data: VelocityTrendPoint[];
  bucketSize: 'week' | 'month';
  startDate: Date;
  endDate: Date;
}

interface UseVelocityBucketsReturn {
  buckets: VelocityBucket[];
  velocityDataset: { x: Date; y: number }[]; // scoreChange per bucket midpoint
  scoreDataset: { x: Date; y: number }[]; // avgScore per bucket start
}

export function useVelocityBuckets(options: UseVelocityBucketsOptions): UseVelocityBucketsReturn;
```

### Implementation Notes

- Generate bucket boundaries using `eachWeekOfInterval` or `eachMonthOfInterval` from `date-fns`.
- For each bucket, collect all `VelocityTrendPoint` entries where `new Date(point.date)` falls within `[bucketStart, bucketEnd)`.
- `avgScore`: mean of `point.score` values in the bucket; `0` if no points exist (gap bucket).
- `scoreChange`: `currentBucket.avgScore - previousBucket.avgScore`; `0` for the first bucket.
- `snapshotCount`: number of points in the bucket.
- `scoreDataset`: `buckets.map(b => ({ x: b.bucketStart, y: b.avgScore }))`.
- `velocityDataset`: `buckets.map(b => ({ x: b.bucketStart, y: b.scoreChange }))`.
- Memoize using `useMemo` — re-compute only when `data`, `bucketSize`, `startDate`, or `endDate` change.

## Component: `VelocityFilters`

```typescript
// clients/desktop/src/renderer/components/velocity/VelocityFilters.tsx

interface VelocityFiltersProps {
  timeRange: '30d' | '90d' | '6mo' | '1yr' | 'all';
  bucketSize: 'week' | 'month';
  metric: 'overall' | 'cyclomatic' | 'halstead' | 'maintainability';
  directoryScope: string | null;
  onTimeRangeChange: (range: VelocityFiltersProps['timeRange']) => void;
  onBucketSizeChange: (size: 'week' | 'month') => void;
  onMetricChange: (metric: VelocityFiltersProps['metric']) => void;
  onDirectoryScopeChange: (dir: string | null) => void;
}
```

Renders a horizontal strip using `flex items-center gap-4`. Each filter is a `Dropdown` (select variant) from `@vipr/ui`:

| Control         | Options                                                                    |
| --------------- | -------------------------------------------------------------------------- |
| Time range      | Last 30 days, Last 90 days, Last 6 months, Last year, All time             |
| Bucket size     | Week, Month                                                                |
| Metric          | Overall, Cyclomatic complexity, Halstead, Maintainability                  |
| Directory scope | `Input` with placeholder `/src/components` + clear `Button` (icon variant) |

Directory scope clears by calling `onDirectoryScopeChange(null)` when the Input value is empty or the clear button is clicked.

## Component: `VelocityTrendSection`

```typescript
// clients/desktop/src/renderer/components/velocity/VelocityTrendSection.tsx

interface VelocityTrendSectionProps {
  buckets: VelocityBucket[];
  scoreDataset: { x: Date; y: number }[];
  velocityDataset: { x: Date; y: number }[];
  isLoading: boolean;
  bucketSize: 'week' | 'month'; // drives x-axis label format (e.g., "Jan 6" vs "January")
}
```

Renders a Chart.js `LineChart` from `@vipr/ui/charts` with two datasets and two Y-axes:

| Dataset                              | Style                                                                | Y-axis                        |
| ------------------------------------ | -------------------------------------------------------------------- | ----------------------------- |
| Quality score (`scoreDataset`)       | Solid line, `borderColor: '#3b82f6'`                                 | Left (`y`) — range 0–100      |
| Velocity overlay (`velocityDataset`) | Dashed `borderDash: [5, 5]`, `borderColor: '#f59e0b'`, lower opacity | Right (`y1`) — range -5 to +5 |

Axis configuration:

- Left Y-axis (`y`): `min: 0`, `max: 100`, label "Quality Score"
- Right Y-axis (`y1`): `position: 'right'`, `grid: { drawOnChartArea: false }`, label "Velocity (pts/wk)"
- X-axis: time scale, `unit: bucketSize` from parent

Tooltip callback:

```
Week of Mar 3: Score 72.4 (↑1.2 pts from last week)
```

When `isLoading`, render a skeleton placeholder of the same height (`h-64`) using `bg-gray-100 dark:bg-gray-800 animate-pulse rounded`.

## Component: `VelocityMetricBreakdownTable`

```typescript
// clients/desktop/src/renderer/components/velocity/VelocityMetricBreakdownTable.tsx

interface VelocityMetricBreakdown {
  metric: string;
  currentScore: number;
  weeklyVelocity: number; // points per week; positive = improving
  trend: 'improving' | 'stable' | 'degrading';
  r2: number; // goodness of fit 0–1
}

interface VelocityMetricBreakdownTableProps {
  data: VelocityMetricBreakdown[];
  isLoading: boolean;
}
```

Renders a `CardTable` from `@vipr/ui` with these columns:

| Column     | Value                              | Notes                  |
| ---------- | ---------------------------------- | ---------------------- |
| Metric     | `metric` string                    | Display name           |
| Current    | `currentScore.toFixed(1)`          | —                      |
| Velocity   | `weeklyVelocity.toFixed(2)` pts/wk | Sign-prefixed          |
| Trend      | `Badge`                            | See color tokens below |
| Confidence | `(r2 * 100).toFixed(0)%`           | —                      |

Trend `Badge` color tokens:

- `improving`: `bg-green-500/20 text-green-700 dark:bg-green-500/10 dark:text-green-400`
- `stable`: `bg-yellow-500/20 text-yellow-700 dark:bg-yellow-500/10 dark:text-yellow-400`
- `degrading`: `bg-red-500/20 text-red-700 dark:bg-red-500/10 dark:text-red-400`

## Component: `VelocityLeaderboardSection`

```typescript
// clients/desktop/src/renderer/components/velocity/VelocityLeaderboardSection.tsx

interface FileVelocityEntry {
  filePath: string;
  weeklyVelocity: number;
  snapshotCount: number;
  currentScore: number;
}

interface VelocityLeaderboardSectionProps {
  mostImproved: FileVelocityEntry[];
  leastImproved: FileVelocityEntry[];
  isLoading: boolean;
}
```

Renders two `CardTable` components side by side in a `grid grid-cols-2 gap-4` wrapper:

- Left table: "Most Improved" — sorted descending by `weeklyVelocity`
- Right table: "Least Improved" — sorted ascending by `weeklyVelocity`

Columns for each table: File | Velocity | Score | Snapshots.

Display rules:

- `filePath`: truncate to 40 characters from the right, prefix `…` if truncated
- `weeklyVelocity`: `text-green-600 dark:text-green-400` when positive; `text-red-600 dark:text-red-400` when negative; include sign prefix (`+1.2`, `-0.8`)

## Component: `InflectionPointsSection`

```typescript
// clients/desktop/src/renderer/components/velocity/InflectionPointsSection.tsx

interface InflectionPoint {
  commitSha: string;
  date: number; // epoch ms
  scoreBefore: number;
  scoreAfter: number;
  velocityChange: number; // e.g. from -0.5/wk to +1.2/wk
  direction: 'acceleration' | 'deceleration' | 'reversal';
}

interface InflectionPointsSectionProps {
  inflectionPoints: InflectionPoint[];
  onNavigateToCommit: (sha: string) => void;
  isLoading: boolean;
}
```

Renders a `CardTable` with columns:

| Column          | Value                                                                 |
| --------------- | --------------------------------------------------------------------- |
| Date            | Formatted date from `point.date`                                      |
| Commit          | `point.commitSha.slice(0, 7)` in monospace                            |
| Score Change    | `scoreBefore → scoreAfter` with direction arrow                       |
| Velocity Change | Formatted `velocityChange` with sign                                  |
| Direction       | `Badge` — acceleration (green), deceleration (yellow), reversal (red) |
| Action          | "View commit" `Button` (outline/sm)                                   |

The "View commit" button calls `onNavigateToCommit(point.commitSha)`, which the parent page uses to open `CommitBrowserDrawer` (Phase 14).

## Route and Navigation

### `App.tsx`

Add the route inside the existing router:

```typescript
// clients/desktop/src/renderer/App.tsx
import { Velocity } from './pages/Velocity';

// Inside <Routes>:
<Route path="/velocity" element={<Velocity />} />
```

### `Sidebar.tsx`

Add nav item after the existing Churn or History entry:

```typescript
// clients/desktop/src/renderer/components/layout/Sidebar.tsx
{
  icon: ChartLineIcon,   // use an existing icon from the icon set
  label: 'Velocity',
  path: '/velocity',
}
```

## IPC Used (all pre-existing)

All channels were wired in Phase 08. No new IPC is introduced in this phase.

| Channel                                 | Return type                                | Usage                          |
| --------------------------------------- | ------------------------------------------ | ------------------------------ |
| `velocity.getRepoTrend(options)`        | `VelocityTrendPoint[]` + `VelocityMetrics` | Feeds `useVelocityBuckets`     |
| `velocity.getLeaderboard(options)`      | `{ mostImproved, leastImproved }`          | `VelocityLeaderboardSection`   |
| `velocity.getMetricBreakdown(options)`  | `VelocityMetricBreakdown[]`                | `VelocityMetricBreakdownTable` |
| `velocity.getInflectionPoints(options)` | `InflectionPoint[]`                        | `InflectionPointsSection`      |

The `options` object passed to each channel includes: `{ repoPath, timeRange, metric, directoryScope }` derived from the current filter state in `Velocity.tsx`.

## Testing

### `useVelocityBuckets.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useVelocityBuckets.test.ts

describe('useVelocityBuckets', () => {
  it('correctly groups weekly snapshots into buckets', () => {
    // Arrange: 14 trend points spread over 2 weeks
    // Assert: buckets.length === 2, each bucket has snapshotCount === 7
  });

  it('correctly groups monthly snapshots into buckets', () => {
    // Arrange: trend points spanning 3 calendar months
    // Assert: buckets.length === 3
  });

  it('calculates scoreChange as difference from previous bucket', () => {
    // Arrange: bucket 1 avgScore=70, bucket 2 avgScore=72
    // Assert: buckets[1].scoreChange === 2
  });

  it('returns scoreChange=0 for the first bucket', () => {
    // Assert: buckets[0].scoreChange === 0
  });

  it('handles gaps in data (no snapshots in a bucket period)', () => {
    // Arrange: data has points in week 1 and week 3, none in week 2
    // Assert: buckets[1].snapshotCount === 0, buckets[1].avgScore === 0
  });

  it('produces scoreDataset with x=bucketStart and y=avgScore', () => { ... });
  it('produces velocityDataset with x=bucketStart and y=scoreChange', () => { ... });
});
```

### `Velocity.test.tsx`

```typescript
// clients/desktop/src/renderer/pages/Velocity.test.tsx

describe('Velocity page', () => {
  it('renders VelocityFilters with all four controls', () => {
    // Assert: time range, bucket size, metric, and directory scope controls are present
  });

  it('renders VelocityTrendSection with loading skeleton when data is loading', () => {
    // Mock IPC to delay response
    // Assert: loading skeleton visible
  });

  it('re-fetches trend data when time range filter changes', async () => {
    // Change time range dropdown
    // Assert: velocity.getRepoTrend called with updated timeRange
  });

  it('re-fetches all data when metric filter changes', async () => { ... });

  it('inflection point "View commit" calls onNavigateToCommit with correct SHA', () => {
    // Click "View commit" button in InflectionPointsSection
    // Assert: handler called with commitSha
  });
});
```
