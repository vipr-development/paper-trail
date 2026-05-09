---
id: 07-backfill-ux-simplification
title: 'Backfill UX Simplification'
phase: 5
dependencies: [01]
status: planned
---

# Backfill UX Simplification

## Problem Statement

The existing bulk historical analysis UI (Round Four, Phase 13) exposes the full complexity of adaptive depth calculation: a numeric commit count input pre-populated by `BackfillScheduler`'s heuristics, scheduling options, and a queue management section. For most users this creates decision fatigue at the moment they most need to get started — right after first analysis when the Velocity dashboard is empty.

The goal of this phase is to reduce the backfill trigger to its simplest possible form: three preset buttons representing 30, 60, and 90 days of history. The algorithmic complexity (adaptive depth, queue prioritisation) stays in the `BackfillScheduler` but stops being exposed in the UI. Pro users additionally receive automatic background backfill triggered silently after their first successful analysis. Free users see a capped preview of the last five commits with an upgrade CTA.

## New Files

| File                                                                         | Role                                                                                    |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/components/backfill/SimpleBackfillTrigger.tsx` | Three-button trigger component rendering "Last 30 days", "Last 60 days", "Last 90 days" |
| `clients/desktop/src/renderer/components/backfill/BackfillProgressBar.tsx`   | Compact header-area progress bar for silent background backfill                         |
| `clients/desktop/src/renderer/components/backfill/BackfillPreview.tsx`       | Free-tier component: last 5 commits with health scores and upgrade CTA                  |
| `clients/desktop/src/main/analysis/auto-backfill.ts`                         | Main-process module that triggers background backfill after first analysis completes    |

## Modified Files

| File                                                      | Changes                                                                                                                                                                                        |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `clients/desktop/src/main/analysis/backfill-scheduler.ts` | Add `estimateDepthForDays(repoPath: string, days: number): Promise<number>` method that imports `gatherBackfillInputs` and `calculateAdaptiveBackfillDepth` from `../git/adaptive-backfill.ts` |
| `clients/desktop/src/shared/ipc/settings-types.ts`        | Add `'backfill.autoEnabled': boolean` and `'backfill.autoDepth': '30d' \| '60d' \| '90d'` to the `Settings` interface                                                                          |
| `clients/desktop/src/renderer/pages/Velocity.tsx`         | Replace the complex backfill prompt with `SimpleBackfillTrigger` when no historical data exists                                                                                                |
| `clients/desktop/src/renderer/pages/History.tsx`          | Add `SimpleBackfillTrigger` in the empty state when fewer than five snapshots exist                                                                                                            |
| `clients/desktop/src/renderer/pages/Settings.tsx`         | Add auto-backfill toggle section using the `SettingCard` pattern                                                                                                                               |

## Settings Interface Extension

Add to `clients/desktop/src/shared/ipc/settings-types.ts`:

```typescript
export interface Settings {
  // ... existing fields ...
  'backfill.autoEnabled': boolean;
  'backfill.autoDepth': '30d' | '60d' | '90d';
}
```

Default values (add to the `defaults` constant in `clients/desktop/src/main/ipc/handlers/settings.ts`):

```typescript
'backfill.autoEnabled': true,
'backfill.autoDepth': '30d',
```

## Shared Types

Add to `clients/desktop/src/shared/ipc/backfill-types.ts`:

```typescript
/** Preset depth options exposed in the simplified trigger UI. */
export type BackfillDepthPreset = '30d' | '60d' | '90d';

export interface BackfillPreset {
  label: string; // "Last 30 days"
  depth: BackfillDepthPreset;
  estimatedCommits: number | null; // from browseCommits count, null while loading
}
```

## Component: `SimpleBackfillTrigger`

```typescript
// clients/desktop/src/renderer/components/backfill/SimpleBackfillTrigger.tsx

interface SimpleBackfillTriggerProps {
  /** Called when the user selects a preset. Parent initiates the backfill job. */
  onSelectDepth: (depth: BackfillDepthPreset) => void;
  /** If provided, each button shows an estimated commit count below the label. */
  commitEstimates?: Partial<Record<BackfillDepthPreset, number>>;
  isLoading?: boolean;
}
```

### Layout

```
┌─ Analyze History ──────────────────────────────────┐
│                                                    │
│  How far back should we analyze?                  │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ 30 days  │  │ 60 days  │  │ 90 days  │        │
│  │ ~15 cmts │  │ ~30 cmts │  │ ~45 cmts │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                    │
│  Estimated time: ~2–5 minutes                     │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Implementation Notes

- Render a heading `"How far back should we analyze?"` in `text-sm font-medium text-gray-700 dark:text-gray-300`.
- Render three `Button` components (variant `outline`) in a `flex gap-3` row. Each button label: "Last 30 days", "Last 60 days", "Last 90 days".
- Below each button label, render the estimated commit count in `text-xs text-gray-500 dark:text-gray-400` if `commitEstimates[depth]` is defined. Format as `~{n} commits`.
- Disable all three buttons when `isLoading` is true; show a spinner inside the active button using the `Button`'s loading prop if available.
- Below the button row, render a helper line `"Estimated time: ~2–5 minutes"` in `text-xs text-gray-500 dark:text-gray-400`.
- When a button is clicked, call `onSelectDepth(depth)` immediately. The parent component owns the IPC call.

### Commit Estimate Derivation

The parent page fetches commit estimates via `useViprDesktopAPI()` — call `api.backfill.getSuggestedDepth()` once on mount. Alternatively, the existing `useBulkHistoricalAnalysis` hook already exposes a `suggestedDepth` field. The returned integer represents the repository's commit density. Approximate presets from that baseline:

```typescript
function estimateCommitsForPreset(suggestedDepth: number, preset: BackfillDepthPreset): number {
  // suggestedDepth represents ~30 days; scale linearly for 60d and 90d
  const multiplier = preset === '30d' ? 1 : preset === '60d' ? 2 : 3;
  return Math.round(suggestedDepth * multiplier);
}
```

Pass the derived map as `commitEstimates` to `SimpleBackfillTrigger`.

## Component: `BackfillProgressBar`

```typescript
// clients/desktop/src/renderer/components/backfill/BackfillProgressBar.tsx

interface BackfillProgressBarProps {
  processedCommits: number;
  totalCommits: number;
  onCancel: () => void;
}
```

### Layout

```
┌───────────────────────────────────────────────────────────────┐
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░  Analyzing history: 14/30  ✕      │
└───────────────────────────────────────────────────────────────┘
```

This component is a compact strip rendered in the app header area — not the sticky bottom bar of the Round Four `BulkAnalysisProgressBar`. It occupies minimal vertical space and does not block page content.

### Implementation Notes

- Outer wrapper: `flex items-center gap-3 px-4 py-1.5 bg-gray-100 dark:bg-gray-800 border-b border-gray-200 dark:border-gray-700 text-sm`.
- Progress bar: an inline `<div>` with `h-1.5 rounded-full bg-gray-200 dark:bg-gray-700 w-32 flex-shrink-0` containing a filled child `<div>` whose width is `${(processedCommits / totalCommits) * 100}%` with `bg-blue-500 rounded-full h-full transition-all`.
- Label: `"Analyzing history: {processedCommits}/{totalCommits}"` in `text-gray-700 dark:text-gray-300`.
- Cancel button: `Button` (sm, ghost, icon-only) showing "✕" — calls `onCancel`. Use `aria-label="Cancel backfill"`.
- Do not auto-hide. The parent (AppShell or the header component) conditionally mounts this component and unmounts it when the backfill job status reaches `'completed'` or `'cancelled'`.

### Mounting Location

Mount `BackfillProgressBar` in the existing app header bar (the component that renders the workspace name and navigation controls). Add to the header:

```typescript
const { status, progress, cancel } = useBulkHistoricalAnalysis();

{(status === 'running' || status === 'paused') && progress && (
  <BackfillProgressBar
    processedCommits={progress.processedCommits}
    totalCommits={progress.totalCommits}
    onCancel={cancel}
  />
)}
```

`useBulkHistoricalAnalysis` is the existing hook from Round Four Phase 13 — no duplication.

**Important:** `useBulkHistoricalAnalysis` is a context-based hook that requires `BulkAnalysisProvider` to be mounted as an ancestor in the component tree. Ensure `BulkAnalysisProvider` wraps the app header or the relevant layout component before mounting `BackfillProgressBar`.

## Component: `BackfillPreview`

```typescript
// clients/desktop/src/renderer/components/backfill/BackfillPreview.tsx

interface BackfillPreviewProps {
  commits: CommitSummary[]; // last 5 commits with healthScore where available
}
```

### Layout

```
┌─ Recent History Preview ───────────────────────────────────────┐
│ abc1234 · Jan 15 · Score: 81                                   │
│ def5678 · Jan 12 · Score: 74                                   │
│ ghi9012 · Jan 10 · Score: 78                                   │
│ jkl3456 · Jan 8  · Score: —                                    │
│ mno7890 · Jan 5  · Score: —                                    │
│                                                                │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Unlock full history analysis with Vipr Pro                 │ │
│ │ [Upgrade to Pro]                                          │ │
│ └────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

### Implementation Notes

- Accept up to five `CommitSummary` items. Render each as a compact row: `shortSha` in monospace, date formatted as "MMM D", health score badge (or `"—"` when `healthScore` is null).
- Health score badge colors follow the same thresholds as `CommitRow` (Round Four Phase 14): green `>= 80`, yellow `60–79`, red `< 60`.
- Below the list, render the `UpgradeCTA` component from Phase 01 with the message `"Unlock full history analysis with Vipr Pro"`.
- Use `EmptyState` from `@vipr/ui` as the wrapper if the `commits` array is empty: label `"No recent commits found"`.
- This component renders only for free-tier users. The parent checks `isPro` and renders either `SimpleBackfillTrigger` (Pro) or `BackfillPreview` (free).

## Main Process Module: `auto-backfill.ts`

```typescript
// clients/desktop/src/main/analysis/auto-backfill.ts

import type { BackfillScheduler } from './backfill-scheduler';
import { createLogger } from '@vipr/logging';
import { getSettingsDb } from '../ipc/handlers/settings';

const logger = createLogger({ tag: 'auto-backfill' });

/**
 * Called once after a successful first analysis for a given workspace.
 * Reads backfill settings and triggers a background job if autoEnabled is true.
 * Safe to call multiple times — guards against redundant triggers by checking
 * whether a backfill job is already running via the public getStatus() method.
 *
 * Settings are read via getSettingsDb() — there is no SettingsStore class
 * in the main process. Use the same settings read pattern as other handlers.
 */
export async function triggerAutoBackfillIfEnabled(
  repoPath: string,
  scheduler: BackfillScheduler
): Promise<void> {
  const db = getSettingsDb();
  const autoEnabled = db.get('backfill.autoEnabled') ?? true;
  if (!autoEnabled) {
    logger.debug('Auto-backfill disabled in settings');
    return;
  }

  // currentStatus is private — use the public getStatus() method
  if (scheduler.getStatus() !== 'idle') {
    logger.debug('Backfill already running, skipping auto-trigger');
    return;
  }

  const depthPreset = db.get('backfill.autoDepth') ?? '30d';
  const depthDays = depthPreset === '30d' ? 30 : depthPreset === '60d' ? 60 : 90;

  logger.info('Auto-backfill triggered', { repoPath, depthPreset, depthDays });

  try {
    // Convert days to an approximate commit count using the scheduler's
    // estimateDepthForDays method (which internally imports gatherBackfillInputs
    // and calculateAdaptiveBackfillDepth from ../git/adaptive-backfill.ts).
    const depth = await scheduler.estimateDepthForDays(repoPath, depthDays);
    await scheduler.enqueueBackfill(repoPath, depth);
  } catch (error) {
    // Auto-backfill failures are non-fatal — log and continue.
    logger.error('Auto-backfill enqueue failed', { error });
  }
}
```

Add `estimateDepthForDays(repoPath: string, days: number): Promise<number>` to `BackfillScheduler`. This method imports and calls `gatherBackfillInputs` and `calculateAdaptiveBackfillDepth` from `clients/desktop/src/main/git/adaptive-backfill.ts` (where they are standalone exported functions, not methods on BackfillScheduler). The existing `backfill:getSuggestedDepth` IPC handler at `clients/desktop/src/main/ipc/handlers/backfill.ts:234` shows the correct import pattern.

### Trigger Point

Call `triggerAutoBackfillIfEnabled` from the analysis coordinator or repository-open handler immediately after the first successful analysis result is emitted. Check that no historical snapshots exist yet before calling (avoids triggering on repositories with pre-existing backfill history):

```typescript
// In the analysis completion handler (main process):
const snapshotCount = db.prepare('SELECT COUNT(*) as n FROM snapshots').get() as { n: number };
if (snapshotCount.n === 1) {
  // This is the first snapshot — trigger auto-backfill
  await triggerAutoBackfillIfEnabled(repoPath, backfillScheduler);
}
```

## BackfillScheduler Changes

Add to `clients/desktop/src/main/analysis/backfill-scheduler.ts`:

```typescript
import { gatherBackfillInputs } from '../git/adaptive-backfill';

/**
 * Estimates the number of commits that represent the given number of calendar days
 * of history without enqueueing a job.
 *
 * Note: gatherBackfillInputs and calculateAdaptiveBackfillDepth are standalone
 * exported functions in clients/desktop/src/main/git/adaptive-backfill.ts —
 * they are NOT methods on BackfillScheduler. Import them directly.
 */
async estimateDepthForDays(repoPath: string, days: number): Promise<number> {
  const inputs = await gatherBackfillInputs(repoPath);
  const avgPerDay = inputs.avgCommitsPerWeek / 7;
  return Math.max(1, Math.round(avgPerDay * days));
}
```

The existing `enqueueBackfill` signature is `async enqueueBackfill(repoPath: string, depth: number): Promise<string>`. For the auto-backfill "silent" mode (suppressing completion toasts), either:

1. Add `options?: { silent?: boolean }` as a third argument to `enqueueBackfill` (requires updating the existing backfill IPC handler that also calls it), or
2. Have `auto-backfill.ts` set a flag on the scheduler before enqueueing that downstream toast logic checks

Option 1 is cleaner but note `enqueueBackfill` is also called from `clients/desktop/src/main/ipc/handlers/backfill.ts` — that call site must not break.

## Settings Page

Add to `clients/desktop/src/renderer/pages/Settings.tsx` under a new "Historical Analysis" section:

```tsx
<SettingCard
  label="Automatic History Analysis"
  description="After your first analysis, automatically analyze the last 30 days of commits in the background."
>
  <Switch
    checked={settings['backfill.autoEnabled']}
    onChange={(enabled) => updateSetting('backfill.autoEnabled', enabled)}
  />
</SettingCard>

<SettingCard
  label="Auto-Analysis Depth"
  description="How far back to analyze automatically."
>
  <Dropdown
    variant="select"
    value={settings['backfill.autoDepth']}
    options={[
      { label: 'Last 30 days', value: '30d' },
      { label: 'Last 60 days', value: '60d' },
      { label: 'Last 90 days', value: '90d' },
    ]}
    onChange={(value) => updateSetting('backfill.autoDepth', value)}
    disabled={!settings['backfill.autoEnabled']}
  />
</SettingCard>
```

This follows the `SettingCard` pattern established in Phase 12 (MCP Server settings).

## Velocity Page Integration

In `clients/desktop/src/renderer/pages/Velocity.tsx`, detect the empty history state (no trend data points available) and replace the existing complex backfill prompt with:

```tsx
{
  !hasHistoricalData && isPro && (
    <Alert variant="notification" type="info">
      <SimpleBackfillTrigger
        onSelectDepth={handleDepthSelect}
        commitEstimates={commitEstimates}
        isLoading={backfillStatus === 'running'}
      />
    </Alert>
  );
}

{
  !hasHistoricalData && !isPro && <BackfillPreview commits={recentCommits} />;
}
```

`handleDepthSelect` converts the `BackfillDepthPreset` to a commit count using `estimateCommitsForPreset` and calls `api.backfill.enqueue({ repoPath, depth: commitCount })` via `useViprDesktopAPI()`.

## History Page Integration

In `clients/desktop/src/renderer/pages/History.tsx`, when fewer than five snapshots exist, render `SimpleBackfillTrigger` (Pro) or `BackfillPreview` (free) above the commit list:

```tsx
{
  snapshotCount < 5 && (
    <div className="mb-6">
      {isPro ? (
        <SimpleBackfillTrigger
          onSelectDepth={handleDepthSelect}
          commitEstimates={commitEstimates}
        />
      ) : (
        <BackfillPreview commits={recentCommits.slice(0, 5)} />
      )}
    </div>
  );
}
```

## What Is Hidden (Still Runs Internally)

| Feature                                | Fate                                                                                                                                                                                                                      |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Adaptive depth calculation             | Still runs via `gatherBackfillInputs` + `calculateAdaptiveBackfillDepth` from `git/adaptive-backfill.ts`, called by `BackfillScheduler.estimateDepthForDays`; result surfaces as estimated commit counts in button labels |
| Queue priority management              | Still active inside `BackfillScheduler`; not exposed in any UI surface                                                                                                                                                    |
| Numeric commit range input             | Removed from all UI surfaces; replaced by 30/60/90-day presets                                                                                                                                                            |
| Pause / resume                         | Available through the `BackfillProgressBar` cancel action; pause is hidden (auto-backfill runs silently or not at all)                                                                                                    |
| Job status persistence across restarts | Still handled by `BackfillScheduler`; `useBulkHistoricalAnalysis` hydrates from `getStatus()` on mount                                                                                                                    |

## Existing Components to Reuse

| Component                                         | Usage                                                |
| ------------------------------------------------- | ---------------------------------------------------- |
| `Button`                                          | Three preset buttons in `SimpleBackfillTrigger`      |
| `Alert`                                           | Wraps `SimpleBackfillTrigger` on Velocity page       |
| `EmptyState`                                      | Empty state in `BackfillPreview`                     |
| `Badge`                                           | Health score badges in `BackfillPreview` commit list |
| `SettingCard`                                     | Auto-backfill settings section                       |
| `Switch`                                          | Auto-backfill enabled toggle                         |
| `Dropdown`                                        | Auto-backfill depth selector                         |
| `UpgradeCTA` (Phase 01)                           | CTA inside `BackfillPreview`                         |
| `useBulkHistoricalAnalysis` (Round Four Phase 13) | Hook powering `BackfillProgressBar` progress data    |

## Testing

### `SimpleBackfillTrigger.test.tsx`

```typescript
// clients/desktop/src/renderer/components/backfill/SimpleBackfillTrigger.test.tsx

describe('SimpleBackfillTrigger', () => {
  it('renders three preset buttons', () => {
    render(<SimpleBackfillTrigger onSelectDepth={vi.fn()} />);
    expect(screen.getByRole('button', { name: /30 days/ })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /60 days/ })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /90 days/ })).toBeInTheDocument();
  });

  it('calls onSelectDepth with "30d" when first button is clicked', () => {
    const onSelectDepth = vi.fn();
    render(<SimpleBackfillTrigger onSelectDepth={onSelectDepth} />);
    fireEvent.click(screen.getByRole('button', { name: /30 days/ }));
    expect(onSelectDepth).toHaveBeenCalledWith('30d');
  });

  it('calls onSelectDepth with "60d" when second button is clicked', () => {
    const onSelectDepth = vi.fn();
    render(<SimpleBackfillTrigger onSelectDepth={onSelectDepth} />);
    fireEvent.click(screen.getByRole('button', { name: /60 days/ }));
    expect(onSelectDepth).toHaveBeenCalledWith('60d');
  });

  it('calls onSelectDepth with "90d" when third button is clicked', () => {
    const onSelectDepth = vi.fn();
    render(<SimpleBackfillTrigger onSelectDepth={onSelectDepth} />);
    fireEvent.click(screen.getByRole('button', { name: /90 days/ }));
    expect(onSelectDepth).toHaveBeenCalledWith('90d');
  });

  it('disables all buttons when isLoading is true', () => {
    render(<SimpleBackfillTrigger onSelectDepth={vi.fn()} isLoading={true} />);
    expect(screen.getByRole('button', { name: /30 days/ })).toBeDisabled();
    expect(screen.getByRole('button', { name: /60 days/ })).toBeDisabled();
    expect(screen.getByRole('button', { name: /90 days/ })).toBeDisabled();
  });

  it('shows estimated commit counts when commitEstimates is provided', () => {
    render(
      <SimpleBackfillTrigger
        onSelectDepth={vi.fn()}
        commitEstimates={{ '30d': 15, '60d': 30, '90d': 45 }}
      />
    );
    expect(screen.getByText('~15 commits')).toBeInTheDocument();
    expect(screen.getByText('~30 commits')).toBeInTheDocument();
    expect(screen.getByText('~45 commits')).toBeInTheDocument();
  });
});
```

### `auto-backfill.test.ts`

```typescript
// clients/desktop/src/main/analysis/auto-backfill.test.ts

describe('triggerAutoBackfillIfEnabled', () => {
  it('does not enqueue when autoEnabled is false', async () => {
    mockSettingsDb.get.mockImplementation((key: string) =>
      key === 'backfill.autoEnabled' ? false : '30d'
    );
    await triggerAutoBackfillIfEnabled('/repo', mockScheduler);
    expect(mockScheduler.enqueueBackfill).not.toHaveBeenCalled();
  });

  it('does not enqueue when a backfill job is already running', async () => {
    mockSettingsDb.get.mockReturnValue(true);
    // Use public getStatus() — currentStatus is private
    mockScheduler.getStatus.mockReturnValue('running');
    await triggerAutoBackfillIfEnabled('/repo', mockScheduler);
    expect(mockScheduler.enqueueBackfill).not.toHaveBeenCalled();
  });

  it('enqueues a backfill job when autoEnabled is true and scheduler is idle', async () => {
    mockSettingsDb.get.mockImplementation((key: string) =>
      key === 'backfill.autoEnabled' ? true : '30d'
    );
    mockScheduler.getStatus.mockReturnValue('idle');
    mockScheduler.estimateDepthForDays.mockResolvedValue(15);
    await triggerAutoBackfillIfEnabled('/repo', mockScheduler);
    expect(mockScheduler.enqueueBackfill).toHaveBeenCalledWith('/repo', 15);
  });

  it('does not throw when enqueueBackfill rejects', async () => {
    mockSettingsDb.get.mockReturnValue(true);
    mockScheduler.getStatus.mockReturnValue('idle');
    mockScheduler.estimateDepthForDays.mockResolvedValue(15);
    mockScheduler.enqueueBackfill.mockRejectedValue(new Error('Queue full'));
    await expect(triggerAutoBackfillIfEnabled('/repo', mockScheduler)).resolves.toBeUndefined();
  });
});
```

### `BackfillPreview.test.tsx`

```typescript
// clients/desktop/src/renderer/components/backfill/BackfillPreview.test.tsx

describe('BackfillPreview', () => {
  it('renders up to 5 commit rows', () => {
    render(<BackfillPreview commits={mockCommits.slice(0, 5)} />);
    expect(screen.getAllByRole('listitem').length).toBeLessThanOrEqual(5);
  });

  it('renders UpgradeCTA below the commit list', () => {
    render(<BackfillPreview commits={mockCommits.slice(0, 3)} />);
    expect(screen.getByText(/Unlock full history/)).toBeInTheDocument();
  });

  it('renders EmptyState when commits array is empty', () => {
    render(<BackfillPreview commits={[]} />);
    expect(screen.getByText(/No recent commits found/)).toBeInTheDocument();
  });

  it('shows health score badge when healthScore is non-null', () => {
    const commits = [{ ...mockCommit, healthScore: 82 }];
    render(<BackfillPreview commits={commits} />);
    expect(screen.getByText('82')).toBeInTheDocument();
  });

  it('shows "—" when healthScore is null', () => {
    const commits = [{ ...mockCommit, healthScore: null }];
    render(<BackfillPreview commits={commits} />);
    expect(screen.getByText('—')).toBeInTheDocument();
  });
});
```

## Dependencies on Other Phases

| Phase              | Dependency                                                                                                                                                                                                                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01 Pro Tier Gating | `SimpleBackfillTrigger` renders only for Pro users. `BackfillPreview` with `UpgradeCTA` renders for free users. The `isPro` boolean comes from the `useLicense` hook introduced in Phase 01. All three new components require Phase 01 to be complete before integration into Velocity and History pages. |
