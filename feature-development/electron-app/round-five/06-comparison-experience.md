---
id: 06-comparison-experience
title: 'A/B Snapshot Comparison Experience'
phase: 5
dependencies: [01, 05]
status: planned
---

# A/B Snapshot Comparison Experience

## Problem Statement

Users can browse historical snapshots and see individual commit health scores, but there is no dedicated experience for comparing two points in time side by side. The existing `history:compareCommits` IPC handler (`clients/desktop/src/main/ipc/handlers/history.ts`) returns base commit, target commit, file-level changes, and a summary — but the data layer has no polished UI counterpart.

Without a comparison view, developers cannot answer questions like: "Did the refactor I merged last week actually improve maintainability?" or "Which files regressed between my feature branch and main?" They must mentally diff two separate snapshot views, losing context between navigation steps.

This phase introduces a dedicated `/comparison` page with an A/B layout: two commit panels side by side, a summary row of top-line deltas, a file changes table with per-file score deltas, and a per-metric breakdown table. Every delta value is color-coded to distinguish improvements from regressions, accounting for whether higher or lower raw values indicate quality.

## New Files

| File                                                                       | Role                                                                                                            |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/pages/Comparison.tsx`                        | Top-level page; reads `shaA` and `shaB` from query params, owns data fetch lifecycle                            |
| `clients/desktop/src/renderer/components/comparison/ComparisonHeader.tsx`  | Side-by-side snapshot A vs B panels showing SHA, date, author, and overall health score                         |
| `clients/desktop/src/renderer/components/comparison/ComparisonSummary.tsx` | Row of four StatCards: score delta, file count delta, issue count delta, critical issue delta                   |
| `clients/desktop/src/renderer/components/comparison/FileChangesTable.tsx`  | CardTable of files with status column (improved / degraded / added / removed) and score delta                   |
| `clients/desktop/src/renderer/components/comparison/MetricDeltaTable.tsx`  | Per-metric breakdown: cyclomatic, halstead, maintainability values for A and B plus computed delta              |
| ~~`DeltaBadge.tsx`~~                                                       | **Not needed** — use existing `DeltaBadge` from `@vipr/ui` (`packages/ui/src/components/common/DeltaBadge.tsx`) |
| `clients/desktop/src/renderer/hooks/useComparison.ts`                      | Hook that calls `history:compareCommits` IPC and manages loading / error state                                  |

## Modified Files

| File                                              | Changes                                                                |
| ------------------------------------------------- | ---------------------------------------------------------------------- |
| `clients/desktop/src/renderer/App.tsx`            | Add route `/comparison` pointing to `Comparison` page                  |
| `clients/desktop/src/renderer/pages/Velocity.tsx` | Add "Compare snapshots" action when two trend data points are selected |
| `clients/desktop/src/renderer/pages/History.tsx`  | Wire `CommitRangeSelector` confirmation to navigate to `/comparison`   |

## Shared Types

**Important:** `ComparisonResult` and related types already exist in `clients/desktop/src/shared/ipc/comparison-types.ts`. Use and extend the existing types rather than creating parallel ones in `history-types.ts`.

Existing types (reference — do not duplicate):

```typescript
// clients/desktop/src/shared/ipc/comparison-types.ts (EXISTING)

export type ComparisonStatus = 'improved' | 'degraded' | 'unchanged' | 'added' | 'removed';

export type ComparisonSummary = {
  filesChanged: number;
  filesAdded: number;
  filesRemoved: number;
  filesImproved: number;
  filesDegraded: number;
  scoreChange: number | null; // NOTE: field is 'scoreChange', not 'scoreDelta'
  scoreBefore: number | null;
  scoreAfter: number | null;
};

export type ComparisonFile = {
  filePath: string; // NOTE: field is 'filePath', not 'path'
  fileId: number;
  scoreBefore: number | null;
  scoreAfter: number | null;
  delta: number; // NOTE: field is 'delta', not 'scoreDelta'
  status: ComparisonStatus;
};

export type MetricChange = {
  pluginId: string;
  avgScoreBefore: number | null;
  avgScoreAfter: number | null;
  fileCountBefore: number;
  fileCountAfter: number;
  delta: number;
};

export type ComparisonResult = {
  summary: ComparisonSummary;
  changedFiles: ComparisonFile[]; // NOTE: field is 'changedFiles', not 'fileChanges'
  metricChanges: MetricChange[]; // NOTE: field is 'metricChanges', not 'metricDeltas'
  snapshotA: Snapshot; // NOTE: full Snapshot objects, not CommitInfo
  snapshotB: Snapshot;
};
```

Components in this phase must use these existing field names. The `Snapshot` type carries `sha`, `authorName`, `timestamp`, and `healthScore`, so a separate `ComparisonCommitInfo` type is not needed.

If the `ComparisonSummary` needs `issueCountDelta` or `criticalCountDelta`, extend the existing type. To add `lowerIsBetter` semantics for per-metric display, derive it at render time from a constant lookup (cyclomatic and halstead are `lowerIsBetter: true`; maintainability is `lowerIsBetter: false`).

## IPC Channel

The existing `history:compareCommits` handler in `clients/desktop/src/main/ipc/handlers/history.ts` already returns a `ComparisonResult` shape (from `comparison-types.ts`). Extend that handler to fully populate `changedFiles` and `metricChanges`. No new IPC channel or alias is needed — use the existing `api.history.compareCommits(shaA, shaB)` via `useViprDesktopAPI()`.

Do not create a `comparison:get` alias. The existing `history.compareCommits` channel is sufficient and avoids the overhead of wiring a new preload context for a duplicate surface.

## Hook: `useComparison`

```typescript
// clients/desktop/src/renderer/hooks/useComparison.ts

import type { ComparisonResult } from '../../shared/ipc/comparison-types';

interface UseComparisonReturn {
  data: ComparisonResult | null;
  isLoading: boolean;
  error: string | null;
}

export function useComparison(shaA: string | null, shaB: string | null): UseComparisonReturn;
```

### Implementation Notes

- Only fetch when both `shaA` and `shaB` are non-null strings.
- Use `useViprDesktopAPI()` to get the API object, then call `api.history.compareCommits(shaA, shaB)` on mount and whenever either SHA changes.
- If either SHA is null or empty, return `{ data: null, isLoading: false, error: null }`.
- Errors from the IPC call set `error` to the message string; do not re-throw.

## Page: `Comparison.tsx`

```typescript
// clients/desktop/src/renderer/pages/Comparison.tsx

export default function Comparison(): JSX.Element;
```

### Implementation Notes

- Read `shaA` and `shaB` from route parameters using `useParams` (the app uses `MemoryRouter` — `useSearchParams` works but the codebase convention is `useParams` with path segments). Define the route as `/comparison/:shaA/:shaB` in `App.tsx`.
- Pass both to `useComparison`. Render a loading skeleton while `isLoading` is true. Render an `Alert` with `variant="notification"` and `type="error"` if `error` is set.
- If both SHAs are missing from the URL, render an `EmptyState` with label "No comparison selected" and a back button.
- When data is available, render in order: `ComparisonHeader`, `ComparisonSummary`, then the two-column grid with `FileChangesTable` and `MetricDeltaTable`.

## Component: `ComparisonHeader`

```typescript
// clients/desktop/src/renderer/components/comparison/ComparisonHeader.tsx

interface ComparisonHeaderProps {
  baseCommit: ComparisonCommitInfo;
  targetCommit: ComparisonCommitInfo;
}
```

### Layout

```
┌─ Comparison ───────────────────────────────────────────────────────────┐
│ ComparisonHeader                                                       │
│ ┌──────────────────────┐         ┌──────────────────────┐             │
│ │ Snapshot A           │   vs    │ Snapshot B           │             │
│ │ abc1234 · Jan 15     │  ───►   │ def5678 · Feb 28    │             │
│ │ Alice Smith          │         │ Alice Smith          │             │
│ │ Score: 72            │         │ Score: 81            │             │
│ └──────────────────────┘         └──────────────────────┘             │
└────────────────────────────────────────────────────────────────────────┘
```

Use `grid grid-cols-[1fr_auto_1fr] items-center gap-6`. Each commit panel is a `div` with `rounded-lg border border-gray-200 dark:border-gray-700 p-4`. The center column renders a right-pointing arrow `→` in `text-gray-400`.

Display `shortSha` in monospace, `date` formatted as "MMM D, YYYY", `authorName`, and `healthScore`. Apply the health score color tokens:

| Score   | Class                                  |
| ------- | -------------------------------------- |
| `>= 80` | `text-green-700 dark:text-green-400`   |
| `60–79` | `text-yellow-700 dark:text-yellow-400` |
| `< 60`  | `text-red-700 dark:text-red-400`       |
| `null`  | `text-gray-500 dark:text-gray-400`     |

## Component: `ComparisonSummary`

```typescript
// clients/desktop/src/renderer/components/comparison/ComparisonSummary.tsx

interface ComparisonSummaryProps {
  summary: ComparisonResult['summary'];
}
```

### Layout

```
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Score  +9   │ │ Files  +3   │ │ Issues  -12 │ │ Critical -2 │
│    ↑ 12.5%  │ │    ↑ 5.0%   │ │    ↓ 15.0%  │ │    ↓ 40.0%  │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

Render four `StatCard` components in a `grid grid-cols-2 lg:grid-cols-4 gap-4`. Each card shows:

- Label: "Score", "Files", "Issues", "Critical Issues"
- Value: the signed delta (e.g. `+9`, `-12`)
- Trend indicator: percentage change relative to the base value

For "Score" and "Critical Issues" deltas: positive = good (green), negative = bad (red). For "Issues": negative = good (fewer issues is better), so invert the color.

## Component: `FileChangesTable`

```typescript
// clients/desktop/src/renderer/components/comparison/FileChangesTable.tsx

interface FileChangesTableProps {
  changedFiles: ComparisonFile[]; // uses existing ComparisonFile type from comparison-types.ts
}
```

Use `CardTable` from `@vipr/ui`. Columns:

| Column  | Source               | Notes                                                           |
| ------- | -------------------- | --------------------------------------------------------------- |
| File    | `change.filePath`    | Monospace, truncate long paths with `…` prefix showing filename |
| Status  | `change.status`      | `Badge` colored per status (see below)                          |
| Score A | `change.scoreBefore` | `—` when null                                                   |
| Score B | `change.scoreAfter`  | `—` when null                                                   |
| Delta   | `change.delta`       | `DeltaBadge` from `@vipr/ui`; `—` for added/removed             |

Status badge colors:

| Status      | Badge classes                                                             |
| ----------- | ------------------------------------------------------------------------- |
| `improved`  | `bg-green-500/20 text-green-700 dark:bg-green-500/10 dark:text-green-400` |
| `degraded`  | `bg-red-500/20 text-red-700 dark:bg-red-500/10 dark:text-red-400`         |
| `added`     | `bg-blue-500/20 text-blue-700 dark:bg-blue-500/10 dark:text-blue-400`     |
| `removed`   | `bg-gray-500/20 text-gray-700 dark:bg-gray-500/10 dark:text-gray-400`     |
| `unchanged` | `bg-gray-500/20 text-gray-500 dark:text-gray-400`                         |

Default sort: degraded files first, then improved, then added, then removed, then unchanged.

Add a filter row above the table: five `Button` toggle pills — "All", "Improved", "Degraded", "Added", "Removed" — that filter the visible rows client-side.

## Component: `MetricDeltaTable`

```typescript
// clients/desktop/src/renderer/components/comparison/MetricDeltaTable.tsx

interface MetricDeltaTableProps {
  metricChanges: MetricChange[]; // uses existing MetricChange type from comparison-types.ts
}
```

Use `CardTable` from `@vipr/ui`. Columns:

| Column     | Source                        | Notes                                                                    |
| ---------- | ----------------------------- | ------------------------------------------------------------------------ |
| Metric     | `metricChange.pluginId`       | Map pluginId to a human-readable label                                   |
| Snapshot A | `metricChange.avgScoreBefore` | `—` when null                                                            |
| Snapshot B | `metricChange.avgScoreAfter`  | `—` when null                                                            |
| Delta      | `metricChange.delta`          | `DeltaBadge` from `@vipr/ui` with `higherIsBetter` derived from pluginId |

The three standard metrics: cyclomatic complexity (`higherIsBetter: false`), halstead volume (`higherIsBetter: false`), and maintainability index (`higherIsBetter: true`).

## Using `DeltaBadge` from `@vipr/ui`

**Do not create a new `DeltaBadge` component.** Use the existing `DeltaBadge` from `@vipr/ui` (`packages/ui/src/components/common/DeltaBadge.tsx`).

Existing interface:

```typescript
interface DeltaBadgeProps {
  value: number; // NOTE: named 'value', not 'delta'; does not accept null
  threshold?: number; // values within threshold render as neutral
  format?: 'absolute' | 'percent';
  size?: 'sm' | 'md';
  higherIsBetter?: boolean; // NOTE: named 'higherIsBetter', not 'lowerIsBetter'
  className?: string;
}
```

Key differences from the originally proposed local component:

- Prop is `value`, not `delta` — pass the numeric delta value directly
- Prop is `higherIsBetter` (default `true`), not `lowerIsBetter` — for cyclomatic/halstead, pass `higherIsBetter={false}`
- Does not accept `null` — handle null at the call site by rendering `"—"` in a `Badge variant="gray"` instead
- Already handles sign formatting and color coding internally

## Page Layout

```
┌─ Comparison ───────────────────────────────────────────────────────────┐
│ ComparisonHeader                               [col-span-full]        │
│ ┌──────────────────────┐  vs  ┌──────────────────────┐               │
│ │ Snapshot A           │      │ Snapshot B           │               │
│ │ abc1234 · Jan 15     │      │ def5678 · Feb 28    │               │
│ │ Score: 72            │      │ Score: 81            │               │
│ └──────────────────────┘      └──────────────────────┘               │
├────────────────────────────────────────────────────────────────────────┤
│ ComparisonSummary                              [col-span-full]        │
│ [Score +9] [Files +3] [Issues -12] [Critical -2]                     │
├───────────────────────────────────┬────────────────────────────────────┤
│ FileChangesTable  [col-span-8]    │ MetricDeltaTable  [col-span-4]    │
│ Filter: All Improved Degraded ... │ Metric          A    B      Δ    │
│ File         Status    A   B  Δ  │ Cyclomatic     12   8.0   -4.0  │
│ src/auth.ts  Improved  65  80 +15│ Halstead      450  380   -70.0  │
│ src/db.ts    Degraded  78  70  -8│ Maintainability 65  78.0  +13.0 │
│ src/new.ts   Added     —   82   —│                                    │
│ src/old.ts   Removed   71  —    —│                                    │
└───────────────────────────────────┴────────────────────────────────────┘
```

Grid classes: `grid grid-cols-12 gap-6 px-4 sm:px-6 lg:px-8 py-8`. Header and summary use `col-span-full`. File changes table: `col-span-12 lg:col-span-8`. Metric delta table: `col-span-12 lg:col-span-4`.

## Entry Points

### From the commit graph (Phase 05)

When the user selects two nodes in the DAG, a "Compare" button appears in the graph toolbar. Clicking it navigates to `/comparison/{olderSha}/{newerSha}`. The commit graph component calls `navigate(`/comparison/${shaA}/${shaB}`)` using the router's programmatic navigation.

### From Velocity (trend chart)

Add a selection mode to `VelocityTrendSection`: when the user clicks a data point, it enters "compare mode" — a second click selects the target and a "Compare selected" button appears above the chart. Clicking it navigates to `/comparison` with the two SHAs derived from the selected trend points.

### From History (CommitRangeSelector)

The `CommitRangeSelector` component (Phase 14 of Round Four) already has a "Compare these commits" button. Wire its `onConfirmRange` callback to `navigate(`/comparison/${baseSha}/${targetSha}`)`.

### Programmatic navigation

The app uses `MemoryRouter` — there is no browser URL bar. All entry points reach the comparison page via `navigate(`/comparison/${shaA}/${shaB}`)` using React Router's programmatic navigation. The page reads SHAs from `useParams()`. Missing or invalid SHAs render an `EmptyState`.

## Existing Components to Reuse

| Component    | Usage                                               |
| ------------ | --------------------------------------------------- |
| `CardTable`  | `FileChangesTable` and `MetricDeltaTable`           |
| `StatCard`   | Four delta cards in `ComparisonSummary`             |
| `Badge`      | Status column in `FileChangesTable`                 |
| `Button`     | Filter pills, back navigation, entry-point triggers |
| `EmptyState` | No-SHAs state and zero-results state in file table  |
| `Alert`      | IPC error display                                   |

## Testing

### `DeltaBadge` usage (existing `@vipr/ui` component)

`DeltaBadge` is already tested in `packages/ui/src/components/common/DeltaBadge.test.tsx`. No new test file is needed. Usage in comparison components:

```tsx
// For metrics where lower is better (cyclomatic, halstead):
<DeltaBadge value={delta} higherIsBetter={false} />

// For metrics where higher is better (maintainability, health score):
<DeltaBadge value={delta} higherIsBetter={true} />

// For null deltas (added/removed files), render inline:
{delta !== null ? <DeltaBadge value={delta} /> : <Badge variant="gray">—</Badge>}
```

### `Comparison.test.tsx`

```typescript
// clients/desktop/src/renderer/pages/Comparison.test.tsx

describe('Comparison page', () => {
  it('renders EmptyState when shaA or shaB is missing from URL', () => {
    renderWithRouter(<Comparison />, { route: '/comparison' });
    expect(screen.getByText(/No comparison selected/)).toBeInTheDocument();
  });

  it('renders loading state while data is fetching', () => {
    mockUseComparison.mockReturnValue({ data: null, isLoading: true, error: null });
    renderWithRouter(<Comparison />, { route: '/comparison/abc1234/def5678' });
    // Expect skeleton or loading indicator
    expect(screen.queryByText(/Snapshot A/)).not.toBeInTheDocument();
  });

  it('renders ComparisonHeader, ComparisonSummary, and both tables when data is available', () => {
    mockUseComparison.mockReturnValue({ data: mockComparisonResult, isLoading: false, error: null });
    renderWithRouter(<Comparison />, { route: '/comparison/abc1234/def5678' });
    expect(screen.getByText('abc1234')).toBeInTheDocument();
    expect(screen.getByText('def5678')).toBeInTheDocument();
  });

  it('renders Alert when IPC returns an error', () => {
    mockUseComparison.mockReturnValue({ data: null, isLoading: false, error: 'Commit not found' });
    renderWithRouter(<Comparison />, { route: '/comparison/abc1234/def5678' });
    expect(screen.getByText(/Commit not found/)).toBeInTheDocument();
  });
});
```

### `useComparison.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useComparison.test.ts

describe('useComparison', () => {
  it('returns loading=true while fetch is in flight', async () => {
    mockApi.history.compareCommits.mockReturnValue(new Promise(() => {}));
    const { result } = renderHook(() => useComparison('abc1234', 'def5678'));
    expect(result.current.isLoading).toBe(true);
  });

  it('resolves data when IPC call succeeds', async () => {
    mockApi.history.compareCommits.mockResolvedValue(mockComparisonResult);
    const { result } = renderHook(() => useComparison('abc1234', 'def5678'));
    await waitFor(() => expect(result.current.data).toEqual(mockComparisonResult));
  });

  it('sets error string when IPC call rejects', async () => {
    mockApi.history.compareCommits.mockRejectedValue(new Error('Commit not found'));
    const { result } = renderHook(() => useComparison('abc1234', 'def5678'));
    await waitFor(() => expect(result.current.error).toBe('Commit not found'));
  });

  it('does not fetch when shaA is null', () => {
    renderHook(() => useComparison(null, 'def5678'));
    expect(mockApi.history.compareCommits).not.toHaveBeenCalled();
  });
});
```

### Integration: entry point navigation

```typescript
// clients/desktop/src/renderer/components/history/CommitRangeSelector.test.tsx
// (additions to existing test file)

it('navigates to /comparison with correct SHAs when Compare button is clicked', async () => {
  const navigate = vi.fn();
  mockUseNavigate.mockReturnValue(navigate);
  renderWithRouter(
    <CommitRangeSelector
      baseCommit={mockBase}
      targetCommit={mockTarget}
      onConfirmRange={(a, b) => navigate(`/comparison?shaA=${a}&shaB=${b}`)}
    />
  );
  fireEvent.click(screen.getByRole('button', { name: /Compare these commits/ }));
  expect(navigate).toHaveBeenCalledWith(
    expect.stringContaining('/comparison?shaA=abc1234&shaB=def5678')
  );
});
```

## Dependencies on Other Phases

| Phase               | Dependency                                                                                                                                                                                                                                                                              |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01 Pro Tier Gating  | Comparison is a Pro-only feature; `useComparison` must only be callable when `isPro` is true. Wrap the `Comparison` page in `ProGate` — free-tier users navigating to `/comparison` see the `UpgradeCTA` instead of the comparison layout.                                              |
| 05 Commit Graph DAG | The primary entry point for comparison is the commit graph's dual-node selection mode. Phase 05 must expose a "Compare" button that fires the navigation. If Phase 05 is not complete, the History page `CommitRangeSelector` entry point (Round Four Phase 14) serves as the fallback. |
