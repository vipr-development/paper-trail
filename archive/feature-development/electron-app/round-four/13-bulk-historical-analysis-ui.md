---
id: 13-bulk-historical-analysis-ui
title: Bulk Historical Analysis UI
phase: 4
dependencies: [08, 04]
status: planned
---

# Bulk Historical Analysis UI

## Problem Statement

No UI exists to trigger analysis of N commits backward in time. Users who open Vipr on a new repository have no historical snapshots, making the Velocity Dashboard (Phase 11) show no data. The bulk historical analysis UI lets users explicitly populate their history by specifying a commit depth, starting the job, and monitoring progress — with a license gate that caps free users at 30 days of history.

## New Files

| File                                                                                 | Role                                                                      |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| `clients/desktop/src/shared/ipc/bulk-analysis-types.ts`                              | Shared request and progress types used by both renderer and main process  |
| `clients/desktop/src/renderer/hooks/useBulkHistoricalAnalysis.ts`                    | Lifecycle hook — status, progress, enqueue, pause, resume, cancel         |
| `clients/desktop/src/renderer/components/settings/BulkHistoricalAnalysisSection.tsx` | Settings page section with depth input, license gate, and inline progress |
| `clients/desktop/src/renderer/components/common/BulkAnalysisProgressBar.tsx`         | Global sticky bottom strip mounted in AppShell                            |

## Shared Types (`bulk-analysis-types.ts`)

```typescript
// clients/desktop/src/shared/ipc/bulk-analysis-types.ts

export interface BulkAnalysisRequest {
  repoPath: string;
  depth: number; // number of commits to analyze backward from HEAD
}

export interface BulkAnalysisProgress {
  jobId: string;
  processedCommits: number;
  totalCommits: number;
  currentSha: string;
  currentMessage: string; // commit subject line
  status: 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';
  startedAt: number; // epoch ms
  estimatedCompletionMs: number | null;
}
```

## New IPC Additions

Add to `clients/desktop/src/shared/ipc/api-types.ts` under the top-level `api` namespace:

```typescript
backfill: {
  getSuggestedDepth: () => Promise<number>;
  enqueue: (req: BulkAnalysisRequest) => Promise<{ jobId: string }>;
  cancel: (req: { jobId: string }) => Promise<void>;
  pause: () => Promise<void>;
  resume: () => Promise<void>;
  getStatus: () => Promise<BulkAnalysisProgress | null>;
  onProgress: (callback: (progress: BulkAnalysisProgress) => void) => () => void;
  onCompleted: (callback: (result: { jobId: string; processedCommits: number }) => void) => () => void;
  onFailed: (callback: (error: { jobId: string; error: string }) => void) => () => void;
};
```

`getSuggestedDepth` is a lightweight IPC that runs `gatherBackfillInputs` and `calculateAdaptiveBackfillDepth` (from Phase 04's `BackfillScheduler`) against the current `repoPath` and returns the computed commit count. The main process handler for this channel must not enqueue any work — it only reads and returns the suggested integer.

`onProgress`, `onCompleted`, and `onFailed` register push-event listeners. Each returns an unsubscribe function that the renderer calls on unmount.

## Hook: `useBulkHistoricalAnalysis`

```typescript
// clients/desktop/src/renderer/hooks/useBulkHistoricalAnalysis.ts

import { BulkAnalysisProgress } from '@shared/ipc/bulk-analysis-types';

interface UseBulkHistoricalAnalysisReturn {
  status: 'idle' | 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';
  progress: BulkAnalysisProgress | null;
  suggestedDepth: number | null;
  enqueue: (depth: number) => Promise<void>;
  pause: () => Promise<void>;
  resume: () => Promise<void>;
  cancel: () => Promise<void>;
}

export function useBulkHistoricalAnalysis(): UseBulkHistoricalAnalysisReturn;
```

### Implementation Notes

On mount:

1. Call `window.api.backfill.getSuggestedDepth()` and store result in `suggestedDepth`.
2. Call `window.api.backfill.getStatus()` to hydrate `progress` — a job may already be running from a previous session.
3. Subscribe to `window.api.backfill.onProgress`, `onCompleted`, and `onFailed`. Store the three unsubscribe functions.

On unmount: call all three unsubscribe functions.

`enqueue(depth)`:

- Reads `repoPath` from the workspace store.
- Calls `window.api.backfill.enqueue({ repoPath, depth })`.
- On resolve: sets `status` to `'running'`.

`pause()`: calls `window.api.backfill.pause()`, sets `status` to `'paused'` optimistically.

`resume()`: calls `window.api.backfill.resume()`, sets `status` to `'running'` optimistically.

`cancel()`: calls `window.api.backfill.cancel({ jobId: progress.jobId })`, sets `status` to `'cancelled'`.

Progress event handler: `(p) => setProgress(p)` — also derives `status` from `p.status`.

## Component: `BulkHistoricalAnalysisSection`

```typescript
// clients/desktop/src/renderer/components/settings/BulkHistoricalAnalysisSection.tsx

interface BulkHistoricalAnalysisSectionProps {
  isPro: boolean;
}
```

Rendered as a group of `SettingCard` elements within the Settings page's "Historical Analysis" section (see Settings page modification below).

### Layout

```
┌─ Historical Analysis ─────────────────────────────────────────────────────┐
│                                                                           │
│ ┌─ Analyze Git History ─────────────────────────────────────────────────┐ │
│ │ description: "Analyze past commits to populate the Velocity           │ │
│ │ Dashboard and trend projections."                                     │ │
│ │                                                                       │ │
│ │ Commits to analyze: [  27  ]                                         │ │
│ │                      (suggested based on your repository)            │ │
│ │                                                                       │ │
│ │  [  Start Analysis  ]   Pro users: no limit  /  Free: 30 days max   │ │
│ └───────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│ ┌─ Analysis Progress ───────────────────────────────────────────────────┐ │
│ │ (visible only when status is 'running' or 'paused')                  │ │
│ │ 12 / 27 commits  ████████░░░░░░░░  44%                               │ │
│ │ Current: a1b2c3 — "Fix authentication bug"                           │ │
│ │  [  Pause  ]   [  Cancel  ]                                          │ │
│ └───────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

### License Gate

Compute the free-tier depth limit from the suggested depth metadata:

```typescript
// Approximation: avgCommitsPerWeek from BackfillScheduler inputs × 4.3 weeks
const freeTierMaxDepth = Math.round(avgCommitsPerWeek * 4.3);
const exceedsFreeTier = !isPro && depth > freeTierMaxDepth;
```

Behavior when `exceedsFreeTier`:

- Reset the depth `Input` value to `freeTierMaxDepth`.
- Show an `Alert` with `variant="notification"` and `type="info"`: "Upgrade to Pro to analyze unlimited history."
- Disable the Start button and show a `Badge variant="info"` label "Pro required" adjacent to it.

When `isPro` is `true`, no limit is enforced.

### Depth Input

Use an HTML `<input type="number" min={1}>` styled with the `Input` component from `@vipr/ui`. Pre-populate with `suggestedDepth` once resolved. Show a helper text line below: "(suggested based on your repository)" using `text-sm text-gray-500 dark:text-gray-400`.

### Progress Display

Only mount the "Analysis Progress" `SettingCard` when `status === 'running' || status === 'paused'`.

Progress bar: HTML `<progress>` element or a simple `div` with width derived from `(processedCommits / totalCommits) * 100` — do not use a heavy chart component.

Current commit line: `{progress.currentSha.slice(0, 7)} — "{progress.currentMessage}"` in monospace.

Pause/Resume toggle: show "Pause" when `status === 'running'`; show "Resume" when `status === 'paused'`.

## Component: `BulkAnalysisProgressBar`

```typescript
// clients/desktop/src/renderer/components/common/BulkAnalysisProgressBar.tsx

interface BulkAnalysisProgressBarProps {
  progress: BulkAnalysisProgress;
  onPause: () => void;
  onCancel: () => void;
}
```

A sticky bottom strip mounted globally in `AppShell` (see AppShell modification below). Always visible when mounted.

### Layout

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Analyzing history  ████████████░░░░░  12/27 commits  a1b2c3 – Fix auth bug  [Pause] [✕] │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

Styling: `fixed bottom-0 left-0 right-0 z-50 bg-gray-900 dark:bg-gray-950 border-t border-gray-700 px-4 py-2 flex items-center gap-4 text-sm text-white`.

| Element       | Details                                                                      |
| ------------- | ---------------------------------------------------------------------------- |
| Label         | "Analyzing history" — `font-medium`                                          |
| Progress bar  | Inline `<progress>` or styled `div`, `w-32`                                  |
| Count         | `{processedCommits}/{totalCommits} commits`                                  |
| Current SHA   | `{currentSha.slice(0, 7)} – {currentMessage}` — truncate message at 40 chars |
| Pause button  | `Button` (sm, outline) — "Pause" or "Resume" based on `progress.status`      |
| Cancel button | `Button` (sm, ghost, icon-only) — "✕" — calls `onCancel`                     |

Does not auto-hide. The parent (`AppShell`) controls visibility by not mounting the component once `status` reaches `'completed'` or `'cancelled'`.

## Settings Page Modification

File: `clients/desktop/src/renderer/pages/Settings.tsx`

Add a new section "Historical Analysis" between existing settings sections. Render inside it:

```typescript
<BulkHistoricalAnalysisSection isPro={licenseState.isPro} />
```

`licenseState.isPro` comes from the existing license state hook/store used elsewhere in Settings.

## AppShell Modification

File: `clients/desktop/src/renderer/components/layout/AppShell.tsx`

Mount `BulkAnalysisProgressBar` globally at the bottom of the layout, above any existing status bar. Use the hook at the AppShell level:

```typescript
const { status, progress, pause, cancel } = useBulkHistoricalAnalysis();

// Render:
{(status === 'running' || status === 'paused') && progress && (
  <BulkAnalysisProgressBar
    progress={progress}
    onPause={pause}
    onCancel={cancel}
  />
)}
```

Because `useBulkHistoricalAnalysis` is called both in AppShell and in `BulkHistoricalAnalysisSection`, it must be safe to mount multiple times. **Use a React Context provider** rather than a module-level singleton — context is easier to reset between tests, avoids stale-closure bugs on workspace switch, and follows the existing pattern in the codebase. Create a `BulkAnalysisProvider` at the AppShell level and expose a `useBulkHistoricalAnalysis` hook that reads from that context. Both AppShell and `BulkHistoricalAnalysisSection` call the hook; state is shared through the provider.

## Testing

### `useBulkHistoricalAnalysis.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useBulkHistoricalAnalysis.test.ts

describe('useBulkHistoricalAnalysis', () => {
  it('fetches suggestedDepth on mount', async () => {
    mockApi.backfill.getSuggestedDepth.mockResolvedValue(27);
    const { result } = renderHook(() => useBulkHistoricalAnalysis());
    await waitFor(() => expect(result.current.suggestedDepth).toBe(27));
  });

  it('subscribes to backfill progress events on mount', async () => {
    renderHook(() => useBulkHistoricalAnalysis());
    expect(mockApi.backfill.onProgress).toHaveBeenCalledTimes(1);
  });

  it('unsubscribes from progress events on unmount', async () => {
    const unsubscribe = vi.fn();
    mockApi.backfill.onProgress.mockReturnValue(unsubscribe);
    const { unmount } = renderHook(() => useBulkHistoricalAnalysis());
    unmount();
    expect(unsubscribe).toHaveBeenCalled();
  });

  it('transitions status from idle to running when enqueue is called', async () => {
    mockApi.backfill.enqueue.mockResolvedValue({ jobId: 'job-1' });
    const { result } = renderHook(() => useBulkHistoricalAnalysis());
    await act(() => result.current.enqueue(30));
    expect(result.current.status).toBe('running');
  });

  it('transitions status to paused when pause is called', async () => {
    // Set initial status to running, then call pause
    // Assert: status === 'paused'
  });
});
```

### `BulkHistoricalAnalysisSection.test.tsx`

```typescript
// clients/desktop/src/renderer/components/settings/BulkHistoricalAnalysisSection.test.tsx

describe('BulkHistoricalAnalysisSection', () => {
  it('shows suggested depth as default value in the depth input', async () => {
    mockUseBulkHistoricalAnalysis.mockReturnValue({ suggestedDepth: 27, status: 'idle', ... });
    render(<BulkHistoricalAnalysisSection isPro={true} />);
    await waitFor(() => expect(screen.getByRole('spinbutton')).toHaveValue(27));
  });

  it('disables start button and shows Pro badge when depth exceeds free limit and isPro=false', () => {
    render(<BulkHistoricalAnalysisSection isPro={false} />);
    // Set depth input to value above freeTierMaxDepth
    fireEvent.change(screen.getByRole('spinbutton'), { target: { value: '200' } });
    expect(screen.getByRole('button', { name: /Start Analysis/ })).toBeDisabled();
    expect(screen.getByText('Pro required')).toBeInTheDocument();
  });

  it('shows progress section when status is running', () => {
    mockUseBulkHistoricalAnalysis.mockReturnValue({ status: 'running', progress: mockProgress, ... });
    render(<BulkHistoricalAnalysisSection isPro={true} />);
    expect(screen.getByText(/commits/)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /Pause/ })).toBeInTheDocument();
  });

  it('calls pause when Pause button is clicked', () => {
    const pause = vi.fn();
    mockUseBulkHistoricalAnalysis.mockReturnValue({ status: 'running', progress: mockProgress, pause, ... });
    render(<BulkHistoricalAnalysisSection isPro={true} />);
    fireEvent.click(screen.getByRole('button', { name: /Pause/ }));
    expect(pause).toHaveBeenCalledTimes(1);
  });
});
```

### `BulkAnalysisProgressBar.test.tsx`

```typescript
// clients/desktop/src/renderer/components/common/BulkAnalysisProgressBar.test.tsx

describe('BulkAnalysisProgressBar', () => {
  const mockProgress: BulkAnalysisProgress = {
    jobId: 'job-1',
    processedCommits: 12,
    totalCommits: 27,
    currentSha: 'a1b2c3d4e5f',
    currentMessage: 'Fix authentication bug',
    status: 'running',
    startedAt: Date.now(),
    estimatedCompletionMs: null,
  };

  it('renders the progress percentage derived from processedCommits / totalCommits', () => {
    render(<BulkAnalysisProgressBar progress={mockProgress} onPause={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.getByText('12/27 commits')).toBeInTheDocument();
  });

  it('shows current commit SHA truncated to 7 characters', () => {
    render(<BulkAnalysisProgressBar progress={mockProgress} onPause={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.getByText(/a1b2c3d/)).toBeInTheDocument();
  });

  it('calls onCancel when the cancel button is clicked', () => {
    const onCancel = vi.fn();
    render(<BulkAnalysisProgressBar progress={mockProgress} onPause={vi.fn()} onCancel={onCancel} />);
    fireEvent.click(screen.getByRole('button', { name: /✕/ }));
    expect(onCancel).toHaveBeenCalledTimes(1);
  });
});
```
