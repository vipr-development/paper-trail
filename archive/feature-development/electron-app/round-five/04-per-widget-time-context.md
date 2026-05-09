---
id: 04-per-widget-time-context
title: 'Per-Widget Time Context System'
phase: 5
dependencies: [02]
status: planned
---

# Per-Widget Time Context System

## Problem Statement

The existing global time context system couples every view to a single time range, forcing all charts and tables to share one mode selection. Users who want to compare a 30-day velocity trend against a 90-day health overview must navigate away, change the global setting, and navigate back—destroying the multi-perspective view they were building.

Decomposing time context to the widget level makes each visualization independently configurable. A user can place a 30-day trend section next to a 1-year health overview on the same screen without any cognitive overhead. This also simplifies the mental model: instead of a nine-mode global control, each chart carries a compact inline selector that speaks directly to the question it answers.

The global time context in `stores/time-context.ts` is not removed in this phase. It remains available for power-user scenarios and bulk operations. New dashboard widgets and the Velocity page migrate to per-widget time while existing code that reads global time context continues to work unchanged.

## New Files

| File                                                                   | Role                                                                                   |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/hooks/useTimeRange.ts`                   | Hook providing time range state and derived date values for a single widget or section |
| `clients/desktop/src/renderer/components/common/TimeRangeSelector.tsx` | Compact inline preset selector (30d / 60d / 90d / 6mo / 1yr / Custom)                  |
| `clients/desktop/src/renderer/types/time-range-types.ts`               | `TimePreset` and `TimeRange` types shared across hooks and components                  |

## Modified Files

| File                                                                   | Changes                                                                                  |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/stores/time-context.ts`                  | Add deprecation comment on global time context slice; keep all existing selectors intact |
| `clients/desktop/src/renderer/pages/Velocity.tsx`                      | Replace page-level `timeRange` state with `useTimeRange` instances per section           |
| `clients/desktop/src/renderer/components/velocity/VelocityFilters.tsx` | Accept `TimeRangeSelector` output instead of maintaining internal time range state       |

## Shared Types

```typescript
// clients/desktop/src/renderer/types/time-range-types.ts

export type TimePreset = '30d' | '60d' | '90d' | '6mo' | '1yr' | 'custom';

export interface TimeRange {
  preset: TimePreset;
  startDate: Date;
  endDate: Date;
  /** True when the user has selected explicit start/end dates via the DatePicker. */
  isCustom: boolean;
}
```

The `TimePreset` values map to human-readable labels used inside `TimeRangeSelector`. The `startDate`/`endDate` fields are what data-fetching hooks consume—they never see the preset token directly.

**IPC `TimeRange` mapping:** The existing IPC layer uses `TimeRange = '7d' | '30d' | '90d' | '1y' | 'all'` (from `packages/common/src/types/analysis/churn.ts`). The `TimePreset` values `'60d'`, `'6mo'`, and `'1yr'` have no direct IPC equivalent. The `Velocity.tsx` page already has a `toIpcTimeRange()` helper function that maps local tokens to IPC tokens. Data-fetching hooks that accept `TimeRange` string tokens (like `useVelocityTrend` and `useVelocityLeaderboard`) must either:

1. Be updated to accept `{ startDate: Date; endDate: Date }` instead of a string token, OR
2. Use the existing `toIpcTimeRange()` mapping at the call site to convert `TimePreset` to the nearest IPC `TimeRange`

Option 2 is the simpler migration path.

## Hook: `useTimeRange`

```typescript
// clients/desktop/src/renderer/hooks/useTimeRange.ts

import { useState, useMemo } from 'react';
import type { TimePreset, TimeRange } from '../types/time-range-types';

interface UseTimeRangeReturn {
  timeRange: TimeRange;
  setPreset: (preset: TimePreset) => void;
  setCustomRange: (startDate: Date, endDate: Date) => void;
}

export function useTimeRange(defaultPreset: TimePreset = '90d'): UseTimeRangeReturn;
```

### Implementation Notes

- On mount, compute `startDate`/`endDate` from `defaultPreset` using the preset-to-date mapping below.
- `setPreset(preset)`: updates the active preset and recalculates `startDate`/`endDate` from the current wall clock. Sets `isCustom: false`.
- `setCustomRange(startDate, endDate)`: sets `preset: 'custom'` and `isCustom: true`. Stores the caller-supplied dates directly without recalculation.
- Derive `startDate` from preset using `useMemo` so the same preset reference does not trigger unnecessary re-renders in consumers.

**Preset-to-date mapping:**

| Preset   | `startDate` calculation                 |
| -------- | --------------------------------------- |
| `30d`    | `now - 30 days`                         |
| `60d`    | `now - 60 days`                         |
| `90d`    | `now - 90 days`                         |
| `6mo`    | `now - 182 days`                        |
| `1yr`    | `now - 365 days`                        |
| `custom` | Supplied by caller via `setCustomRange` |

### Usage Pattern

Each widget or page section instantiates its own `useTimeRange` call. Data-fetching hooks accept `startDate` and `endDate` directly:

```typescript
// In a widget component
const { timeRange, setPreset, setCustomRange } = useTimeRange('90d');

const { data, isLoading } = useVelocityBuckets({
  startDate: timeRange.startDate,
  endDate: timeRange.endDate,
  metric: selectedMetric,
});
```

Two widgets on the same page maintain completely independent state. There is no shared store slice between them.

## Component: `TimeRangeSelector`

```typescript
// clients/desktop/src/renderer/components/common/TimeRangeSelector.tsx

import type { TimePreset } from '../../types/time-range-types';

interface TimeRangeSelectorProps {
  value: TimePreset;
  onPresetChange: (preset: TimePreset) => void;
  onCustomRange?: (startDate: Date, endDate: Date) => void;
  /** When true, renders without the Custom option. Useful for widgets where
   *  date picker placement would be awkward. */
  disableCustom?: boolean;
  size?: 'xs' | 'sm' | 'default';
  className?: string;
}

export function TimeRangeSelector(props: TimeRangeSelectorProps): JSX.Element;
```

### Visual Layout

```
┌─────────────────────────────────────────────┐
│ 30d │ 60d │ 90d │ 6mo │ 1yr │ Custom ▾     │
└─────────────────────────────────────────────┘
```

The preset pills use `ButtonGroup` from `@vipr/ui` with `variant="outline"` and `size="sm"`. The active preset receives the standard violet active state from `ButtonGroup`.

The `Custom ▾` button is a separate trigger (not a `ButtonGroup` item) that opens a two-field date picker inline below the selector. When `disableCustom={true}`, the Custom button is omitted entirely.

### Custom Date Picker Expansion

When `Custom ▾` is clicked, a small popover renders below the selector:

```
┌─────────────────────────────────────────────┐
│ From  [         ] (date input)              │
│ To    [         ] (date input)              │
│                        [Apply]              │
└─────────────────────────────────────────────┘
```

Date inputs are standard `<input type="date">` elements. "Apply" calls `onCustomRange(startDate, endDate)` and closes the popover. No third-party date picker library is required.

### `ButtonGroup` Items

```typescript
const PRESET_ITEMS: ButtonGroupItem[] = [
  { value: '30d', label: '30d' },
  { value: '60d', label: '60d' },
  { value: '90d', label: '90d' },
  { value: '6mo', label: '6mo' },
  { value: '1yr', label: '1yr' },
];
```

When `value === 'custom'`, no `ButtonGroup` item is active (pass `value=""` to `ButtonGroup`). The `Custom ▾` button instead shows a `bg-violet-500/10 text-violet-600 dark:text-violet-400` state to indicate it is the active mode.

## Modified Files: Velocity Page

`Velocity.tsx` currently holds a single `const [timeRange, setTimeRange] = useState<string>('90d')` that feeds all sections. After this phase:

1. Replace the single state with two independent `useTimeRange` instances—one for the trend section, one for the leaderboard section.
2. Pass `startDate`/`endDate` to `useVelocityTrend`, `useVelocityLeaderboard`, and related hooks instead of the string token.
3. Render a `TimeRangeSelector` adjacent to each section header rather than relying on `VelocityFilters` for time range selection.

```typescript
// Velocity.tsx — after migration
const trendTimeRange = useTimeRange('90d');
const leaderboardTimeRange = useTimeRange('30d');

// Option A: Update hooks to accept startDate/endDate (requires hook refactor)
// const { velocity, trendData, isLoading } = useVelocityTrend(
//   ipcMetric,
//   trendTimeRange.timeRange.startDate,
//   trendTimeRange.timeRange.endDate
// );

// Option B (preferred): Map TimePreset to IPC TimeRange token at the call site.
// Current hook signatures: useVelocityTrend(metric: string, timeRange: TimeRange)
// where TimeRange = '7d' | '30d' | '90d' | '1y' | 'all'
const { velocity, trendData, isLoading } = useVelocityTrend(
  ipcMetric,
  toIpcTimeRange(trendTimeRange.timeRange.preset)
);

const { data: leaderboardData } = useVelocityLeaderboard(
  ipcMetric,
  toIpcTimeRange(leaderboardTimeRange.timeRange.preset),
  20
);
```

**Note:** `Velocity.tsx` currently has four sections consuming time range: `useVelocityTrend`, `useVelocityLeaderboard`, `useMetricBreakdown`, and `useInflectionPoints`. All four must be migrated to per-section `useTimeRange` instances.

## Modified Files: VelocityFilters

`VelocityFilters.tsx` currently owns time range as a radio group inside its dropdown. After this phase, the time range radio group is removed from the filter dropdown. `VelocityFilters` retains control of:

- Bucket size (week / month)
- Metric selector (overall / cyclomatic / halstead / maintainability)
- Projection toggle and horizon days

The `timeRange` and `onTimeRangeChange` props are removed from `VelocityFiltersProps`. Callers that previously controlled time range via `VelocityFilters` switch to rendering `TimeRangeSelector` inline.

## Which Pages Adopt Per-Widget Time

| Page / Section                   | Adopts Per-Widget Time | Default Preset                             |
| -------------------------------- | ---------------------- | ------------------------------------------ |
| Dashboard — all temporal widgets | Yes                    | Widget-specific (defined in widget config) |
| Velocity — trend section         | Yes                    | `90d`                                      |
| Velocity — leaderboard section   | Yes                    | `30d`                                      |
| Velocity — metric breakdown      | Yes                    | `90d`                                      |
| File Detail — historical panel   | Yes                    | `90d`                                      |

**Global time context remains for:**

- Bulk operations (analyze all files for a time period)
- Power users who want a single control to shift all views simultaneously (future enhancement)

## Migration Path

The global time context slice in `stores/time-context.ts` receives a deprecation comment but no code removal in this phase. Existing pages that read from the global context continue to work. Migration is opt-in per-page. New dashboard widgets built in Phase 02 and Phase 03 use `useTimeRange` exclusively.

```typescript
// stores/time-context.ts — deprecation comment added above global time context state
// @todo: Global time context is superseded by per-widget useTimeRange. Remove after all pages migrate.
```

## Existing Components to Reuse

| Component     | Source                  | How Used                                                                     |
| ------------- | ----------------------- | ---------------------------------------------------------------------------- |
| `ButtonGroup` | `@vipr/ui/button-group` | Preset pills in `TimeRangeSelector`                                          |
| `Button`      | `@vipr/ui/button`       | Custom date range trigger and Apply button                                   |
| `Badge`       | `@vipr/ui/badge`        | Optional active-preset label in compact contexts                             |
| `Dropdown`    | `@vipr/ui/dropdown`     | Alternative to inline popover for custom date picker if space is constrained |

`DatePicker` exists in `@vipr/ui` at `packages/ui/src/components/forms/DatePicker.tsx`. Evaluate whether it fits the custom date range popover use case. If it does, prefer it over raw `<input type="date">` elements. If it does not fit (e.g., the popover needs a dual-date inline picker), fall back to `<input type="date">` styled with the app's input tokens.

## Color and Theme Tokens

Follow the standard dark/light paired convention for state indication:

- Active preset selection: inherited from `ButtonGroup` violet active state
- Custom active indicator: `bg-violet-500/10 text-violet-600 dark:text-violet-400`
- Popover background: `bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700/60 rounded-lg shadow-lg`
- Date inputs: `bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-700 rounded text-sm`

## Grid Layout Classes

`TimeRangeSelector` is an inline component — it has no grid responsibility of its own. It sits inside section headers:

```
┌─ Section Header ──────────────────────────────────────────────┐
│ [Section Title]                    [TimeRangeSelector]        │
└───────────────────────────────────────────────────────────────┘
```

Pattern: `flex items-center justify-between` on the section header `div`.

## IPC Considerations

No new IPC channels are required. Existing data-fetching hooks (`useVelocityTrend`, `useVelocityLeaderboard`, `useVelocityBuckets`) already accept start/end parameters. The change is at the call site: instead of deriving dates from a page-level string token, they receive `timeRange.startDate` and `timeRange.endDate` from the hook.

If any data-fetching hook currently only accepts a `TimeRange` string token (not start/end dates), it must be updated to accept `{ startDate: Date; endDate: Date }` before this phase is complete.

## Testing

### `useTimeRange.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useTimeRange.test.ts

describe('useTimeRange', () => {
  it('initializes with the default preset and correct date range', () => {
    const { result } = renderHook(() => useTimeRange('30d'));
    expect(result.current.timeRange.preset).toBe('30d');
    expect(result.current.timeRange.isCustom).toBe(false);
    // startDate should be approximately 30 days ago
    const expectedStart = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    expect(result.current.timeRange.startDate.getDate()).toBe(expectedStart.getDate());
  });

  it('updates startDate when setPreset is called', () => {
    const { result } = renderHook(() => useTimeRange('30d'));
    act(() => result.current.setPreset('1yr'));
    expect(result.current.timeRange.preset).toBe('1yr');
    expect(result.current.timeRange.isCustom).toBe(false);
    const expectedStart = new Date(Date.now() - 365 * 24 * 60 * 60 * 1000);
    expect(result.current.timeRange.startDate.getFullYear()).toBe(expectedStart.getFullYear());
  });

  it('sets isCustom=true and stores supplied dates when setCustomRange is called', () => {
    const { result } = renderHook(() => useTimeRange('90d'));
    const start = new Date('2024-01-01');
    const end = new Date('2024-06-30');
    act(() => result.current.setCustomRange(start, end));
    expect(result.current.timeRange.isCustom).toBe(true);
    expect(result.current.timeRange.preset).toBe('custom');
    expect(result.current.timeRange.startDate).toEqual(start);
    expect(result.current.timeRange.endDate).toEqual(end);
  });

  it('two independent hook instances do not share state', () => {
    const { result: a } = renderHook(() => useTimeRange('30d'));
    const { result: b } = renderHook(() => useTimeRange('90d'));
    act(() => a.current.setPreset('1yr'));
    expect(a.current.timeRange.preset).toBe('1yr');
    expect(b.current.timeRange.preset).toBe('90d');
  });
});
```

### `TimeRangeSelector.test.tsx`

```typescript
// clients/desktop/src/renderer/components/common/TimeRangeSelector.test.tsx

describe('TimeRangeSelector', () => {
  it('renders all preset buttons', () => {
    render(<TimeRangeSelector value="90d" onPresetChange={vi.fn()} />);
    expect(screen.getByRole('button', { name: '30d' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '60d' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '90d' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '6mo' })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: '1yr' })).toBeInTheDocument();
  });

  it('calls onPresetChange when a preset button is clicked', () => {
    const onPresetChange = vi.fn();
    render(<TimeRangeSelector value="90d" onPresetChange={onPresetChange} />);
    fireEvent.click(screen.getByRole('button', { name: '30d' }));
    expect(onPresetChange).toHaveBeenCalledWith('30d');
  });

  it('does not render Custom button when disableCustom=true', () => {
    render(<TimeRangeSelector value="90d" onPresetChange={vi.fn()} disableCustom />);
    expect(screen.queryByText(/Custom/)).not.toBeInTheDocument();
  });

  it('opens date picker popover when Custom is clicked', () => {
    render(<TimeRangeSelector value="90d" onPresetChange={vi.fn()} onCustomRange={vi.fn()} />);
    fireEvent.click(screen.getByText(/Custom/));
    expect(screen.getByLabelText(/From/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/To/i)).toBeInTheDocument();
  });

  it('calls onCustomRange with parsed dates when Apply is clicked', () => {
    const onCustomRange = vi.fn();
    render(<TimeRangeSelector value="90d" onPresetChange={vi.fn()} onCustomRange={onCustomRange} />);
    fireEvent.click(screen.getByText(/Custom/));
    fireEvent.change(screen.getByLabelText(/From/i), { target: { value: '2024-01-01' } });
    fireEvent.change(screen.getByLabelText(/To/i), { target: { value: '2024-06-30' } });
    fireEvent.click(screen.getByRole('button', { name: /Apply/ }));
    expect(onCustomRange).toHaveBeenCalledWith(new Date('2024-01-01'), new Date('2024-06-30'));
  });
});
```

### Integration Test

```typescript
// Verify two widgets on the same page hold independent time ranges.
// clients/desktop/src/renderer/components/common/TimeRangeSelector.integration.test.tsx

it('two TimeRangeSelector instances on the same page operate independently', () => {
  const onChangeA = vi.fn();
  const onChangeB = vi.fn();

  render(
    <>
      <TimeRangeSelector value="90d" onPresetChange={onChangeA} />
      <TimeRangeSelector value="30d" onPresetChange={onChangeB} />
    </>
  );

  // Click 30d on the first selector
  const [first, second] = screen.getAllByRole('button', { name: '30d' });
  fireEvent.click(first);

  expect(onChangeA).toHaveBeenCalledWith('30d');
  expect(onChangeB).not.toHaveBeenCalled();
});
```

## Dependencies on Other Phases

| Phase                                 | Dependency                                                                                                                               |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Phase 02 (Widget System Architecture) | Widget configuration types must include a `defaultPreset: TimePreset` field that `useTimeRange` is initialized with when a widget mounts |
| Phase 03 (Default Dashboard)          | Each temporal widget in the default layout specifies its `defaultPreset` in the widget descriptor                                        |

The global time context in `stores/time-context.ts` has no phase dependency — it is unchanged by this phase.
