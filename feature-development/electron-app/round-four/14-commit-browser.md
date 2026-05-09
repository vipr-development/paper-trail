---
id: 14-commit-browser
title: Commit Browser & Time-Travel Navigator
phase: 4
dependencies: [08, 09, 13]
status: planned
---

# Commit Browser & Time-Travel Navigator

## Problem Statement

`History.tsx` is a flat page with no search, no filtering, no health score annotation, and no direct time-context navigation. Users cannot find the commit where quality degraded, select a specific commit to time-travel to, or trigger point-in-time analysis on demand. This phase adds a filterable `CommitBrowserDrawer`, per-row health scores, "Analyze now" actions for unanalyzed commits, and commit-range selection for the Compare panel.

## New Files

| File                                                                       | Role                                                         |
| -------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `clients/desktop/src/renderer/components/history/CommitBrowserDrawer.tsx`  | Right-side drawer — wraps the full commit browser experience |
| `clients/desktop/src/renderer/components/history/CommitRow.tsx`            | Single commit row with health badge, action buttons          |
| `clients/desktop/src/renderer/components/history/CommitBrowserFilters.tsx` | Search, author, analyzed-only, and date-range controls       |
| `clients/desktop/src/renderer/components/history/CommitRangeSelector.tsx`  | Two-slot base + target selector for compare mode             |
| `clients/desktop/src/renderer/hooks/useCommitBrowser.ts`                   | Commit list fetch, client-side filter state, pagination      |
| `clients/desktop/src/renderer/hooks/useCommitAnalysisStatus.ts`            | Batched SHA → snapshot-exists lookup with module-level cache |

## New IPC Additions

Add to `clients/desktop/src/shared/ipc/api-types.ts`:

```typescript
history: {
  // Returns a paginated list of commit summaries from git log
  getCommitSummaries: (options: { limit: number; offset: number }) => Promise<CommitSummary[]>;
  // Triggers a historical snapshot for a single commit; returns its new snapshot ID
  analyzeCommit: (options: { sha: string }) => Promise<{ snapshotId: number }>;
}
snapshots: {
  // Returns true if a snapshot exists for the given commit SHA
  hasForCommit: (sha: string) => Promise<boolean>;
}
```

> **Phase 08 dependency note:** `history:getCommitSummaries`, `history:analyzeCommit`, and `snapshots:hasForCommit` must be registered as IPC handlers in Phase 08 (`08-ipc-wiring.md`). These channels are listed in the Phase 08 channel inventory. Confirm they are wired before implementing this phase.

## Shared Types

```typescript
// Declare alongside other history types in shared/ipc/api-types.ts or a dedicated
// clients/desktop/src/shared/ipc/history-types.ts file

interface CommitSummary {
  sha: string;
  shortSha: string; // sha.slice(0, 7)
  subject: string; // first line of commit message
  authorName: string;
  timestamp: number; // epoch ms
  parentShas: string[];
}

interface CommitBrowserFilters {
  search: string; // matches subject and SHA; debounced 300ms
  author: string | null;
  analyzedOnly: boolean;
  dateFrom: Date | null;
  dateTo: Date | null;
}
```

## Hook: `useCommitBrowser`

```typescript
// clients/desktop/src/renderer/hooks/useCommitBrowser.ts

interface UseCommitBrowserReturn {
  commits: CommitSummary[]; // full fetched list (across pages)
  filteredCommits: CommitSummary[]; // after client-side filter application
  isLoading: boolean;
  filters: CommitBrowserFilters;
  setFilters: (filters: Partial<CommitBrowserFilters>) => void;
  authors: string[]; // unique author names for Dropdown options
  loadMore: () => Promise<void>;
  hasMore: boolean;
}

export function useCommitBrowser(): UseCommitBrowserReturn;
```

### Implementation Notes

- On mount: call `window.api.history.getCommitSummaries({ limit: 50, offset: 0 })`.
- `loadMore`: increments an internal `offset` state by 50 and calls `getCommitSummaries` again, appending results to `commits`. Sets `hasMore` to `false` when the response contains fewer than 50 entries.
- `authors`: derived from the full `commits` list using `[...new Set(commits.map(c => c.authorName))]`.
- Filtering is client-side — apply against the full fetched `commits` array:
  - `search`: case-insensitive match against `commit.subject` or `commit.sha` (debounced 300ms using a `useRef` timer).
  - `author`: exact match against `commit.authorName`; `null` means no filter.
  - `analyzedOnly`: include only commits where `useCommitAnalysisStatus` returns `statusMap.get(sha) === true`.
  - `dateFrom` / `dateTo`: include commits where `new Date(commit.timestamp) >= dateFrom` and `<= dateTo`.
- `setFilters` merges the partial update into the existing filter state (spread merge).

## Hook: `useCommitAnalysisStatus`

```typescript
// clients/desktop/src/renderer/hooks/useCommitAnalysisStatus.ts

interface UseCommitAnalysisStatusReturn {
  statusMap: Map<string, boolean>; // sha → true if snapshot exists
  checkSha: (sha: string) => boolean;
}

export function useCommitAnalysisStatus(shas: string[]): UseCommitAnalysisStatusReturn;
```

### Implementation Notes

- Module-level cache: `const cache = new Map<string, boolean>()` — persists for the lifetime of the renderer process, not just the component.
- **Export `clearCommitAnalysisCache()` for test isolation:** Tests that render multiple independent hook instances need the ability to reset the module-level cache between tests. Export a top-level function: `export function clearCommitAnalysisCache(): void { cache.clear(); }`. Call this in `beforeEach` of `useCommitAnalysisStatus.test.ts`.
- On each render, compute `uncachedShas = shas.filter(sha => !cache.has(sha))`.
- If `uncachedShas.length > 0`, batch into groups of 20 and run `Promise.all` on each group:
  ```typescript
  const batches = chunk(uncachedShas, 20);
  for (const batch of batches) {
    const results = await Promise.all(batch.map(sha => window.api.snapshots.hasForCommit(sha)));
    batch.forEach((sha, i) => cache.set(sha, results[i]));
  }
  ```
- After all batches resolve, trigger a re-render by setting a `tick` state counter.
- Re-queries (runs the above logic again) when `shas.length` changes between renders.
- `checkSha(sha)`: returns `cache.get(sha) ?? false`.
- `statusMap`: a stable reference — return `cache` directly (the ref is stable; contents mutate).

## Component: `CommitBrowserDrawer`

```typescript
// clients/desktop/src/renderer/components/history/CommitBrowserDrawer.tsx

interface CommitBrowserDrawerProps {
  isOpen: boolean;
  mode: 'single' | 'range';
  onClose: () => void;
  onSelectCommit?: (sha: string) => void; // used when mode='single'
  onSelectRange?: (baseSha: string, targetSha: string) => void; // used when mode='range'
}
```

Renders as a right-anchored overlay using the `Modal` component from `@vipr/ui`. `@vipr/ui` does not have a `SideDrawer` component — do not build a custom one. Use `Modal` with a `maxWidth` override (`className="max-w-lg"`) and position it at the right edge via CSS (`ml-auto`). If the Modal component does not support right-anchoring, render a fixed-position `div` with `right-0 top-0 h-full w-[480px] bg-white dark:bg-gray-900 shadow-xl` and `aria-dialog` role as a fallback — but prefer the existing Modal component.

### Layout

```
┌─ Browse Commits ────────────────── [mode badge] ──── [✕ close] ─┐
│ CommitBrowserFilters                                             │
│ ─────────────────────────────────────────────────────────────── │
│ [Scrollable CommitRow list]                                      │
│   CommitRow                                                      │
│   CommitRow                                                      │
│   ...                                                            │
│   [Load more] button (if hasMore)                                │
│ ─────────────────────────────────────────────────────────────── │
│ CommitRangeSelector (only when mode='range')                     │
└──────────────────────────────────────────────────────────────────┘
```

The scrollable commit list area has `overflow-y-auto flex-1`. The `CommitRangeSelector` is sticky at the bottom inside the drawer.

In `mode='single'`: clicking "View →" on any `CommitRow` calls `onSelectCommit(sha)` and closes the drawer.
In `mode='range'`: clicking a row selects it into `CommitRangeSelector`'s base or target slot.

## Component: `CommitRow`

```typescript
// clients/desktop/src/renderer/components/history/CommitRow.tsx

interface CommitRowProps {
  commit: CommitSummary;
  hasSnapshot: boolean;
  healthScore: number | null; // null when hasSnapshot=false
  mode: 'single' | 'range';
  isSelected?: boolean; // for range selection highlight
  onNavigateTo?: () => void; // set time context to this commit (mode='single')
  onAnalyzeNow?: () => void; // trigger historical snapshot (hasSnapshot=false)
  onView?: () => void; // open snapshot detail (hasSnapshot=true)
}
```

### Row Layout

```
[SHA 7-char mono]  [authorName]  [subject — 60-char truncated]        [date]
                   [health Badge]  [status text]  [action button]
```

Use two rows within a `div` block — not a `<tr>`. Outer wrapper: `flex flex-col gap-0.5 p-3 border-b border-gray-200 dark:border-gray-700 hover:bg-gray-50 dark:hover:bg-gray-800/50`.

When `isSelected`: add `ring-2 ring-blue-500` to the outer wrapper.

### Health Badge Colors

| Condition           | Badge class                                                                   |
| ------------------- | ----------------------------------------------------------------------------- |
| `score >= 80`       | `bg-green-500/20 text-green-700 dark:bg-green-500/10 dark:text-green-400`     |
| `score 60–79`       | `bg-yellow-500/20 text-yellow-700 dark:bg-yellow-500/10 dark:text-yellow-400` |
| `score < 60`        | `bg-red-500/20 text-red-700 dark:bg-red-500/10 dark:text-red-400`             |
| `hasSnapshot=false` | `bg-gray-500/20 text-gray-500 dark:text-gray-400` — label "not analyzed"      |

### Action Buttons

- `hasSnapshot=false`: "Analyze now" `Button` (sm, outline) — calls `onAnalyzeNow?.()`. After the IPC resolves, the parent re-renders via `useCommitAnalysisStatus` cache update.
- `hasSnapshot=true, mode='single'`: "View →" `Button` (sm, outline) — calls `onView?.()`.
- `hasSnapshot=true, mode='range'`: "Select" `Button` (sm, outline) — calls `onNavigateTo?.()` which feeds `CommitRangeSelector`.

## Component: `CommitBrowserFilters`

```typescript
// clients/desktop/src/renderer/components/history/CommitBrowserFilters.tsx

interface CommitBrowserFiltersProps {
  filters: CommitBrowserFilters;
  authors: string[];
  onChange: (filters: Partial<CommitBrowserFilters>) => void;
}
```

Renders a compact filter strip:

| Control       | Component                                           | Detail                                                                      |
| ------------- | --------------------------------------------------- | --------------------------------------------------------------------------- |
| Search        | `Input`                                             | Placeholder "Search commits…"; debounce 300ms internally                    |
| Author        | `Dropdown` (select)                                 | Options built from `['All authors', ...authors]`; `null` when "All authors" |
| Analyzed only | `Switch`                                            | Label "Analyzed only"                                                       |
| Date from     | `<input type="date">` styled with `Input` component | `onChange` parses to `Date` or `null`                                       |
| Date to       | `<input type="date">` styled with `Input` component | Same                                                                        |

Debounce the search `Input` using a `useRef` timeout — call `onChange({ search: value })` after 300ms idle, not on every keystroke.

## Component: `CommitRangeSelector`

```typescript
// clients/desktop/src/renderer/components/history/CommitRangeSelector.tsx

interface CommitRangeSelectorProps {
  baseCommit: CommitSummary | null;
  targetCommit: CommitSummary | null;
  onConfirmRange: (baseSha: string, targetSha: string) => void;
}
```

### Layout

```
┌─ Base commit ──────────────────┐  ┌─ Target commit ────────────────┐
│ a1b2c3 — "Fix auth bug"        │  │ d4e5f6 — "Add feature"         │
│ [Click row above to change]    │  │ [Click row above to change]    │
└────────────────────────────────┘  └────────────────────────────────┘
                      [  Compare these commits  ]
```

Use `grid grid-cols-2 gap-3`. Each slot is a styled `div` with `border border-dashed rounded p-3`. When a commit is set, display `shortSha — subject` truncated to 50 chars. When empty, display placeholder text in `text-gray-400`.

"Compare these commits" `Button` (primary): disabled until both `baseCommit` and `targetCommit` are non-null. On click: calls `onConfirmRange(baseCommit.sha, targetCommit.sha)`.

Clicking "Click to change" (or clicking within an empty slot while a row in the drawer list is hovered) does not require extra interaction — the drawer's row selection flow drives both slots in sequence: first click fills `base`, second click fills `target`.

## History.tsx Modifications

File: `clients/desktop/src/renderer/pages/History.tsx`

Changes:

1. Replace the existing inline table rows with `CommitRow` components. Pass `hasSnapshot` from `useCommitAnalysisStatus` and `healthScore` from the snapshot data (if available in the existing commit list data shape).
2. Mount `CommitBrowserFilters` at the top of the page above the commit list, wired to `useCommitBrowser`'s `filters` and `setFilters`.
3. Add a "Browse & compare" `Button` (outline) in the page header — opens `CommitBrowserDrawer mode="range"`.
4. Per-row "Analyze now" button: calls `window.api.history.analyzeCommit({ sha })` directly, then invalidates the cache entry for that SHA via the exported `clearCommitAnalysisCache()` (clears all) or a more targeted `invalidateCommitAnalysisCache(sha: string)` export if fine-grained invalidation is preferable.

## Overview Panel Modifications

### `OverviewHistoricalPanel.tsx`

File: `clients/desktop/src/renderer/pages/Overview/OverviewHistoricalPanel.tsx`

Add:

- State: `const [drawerOpen, setDrawerOpen] = useState(false)`.
- "Browse commits" `Button` (outline, sm) in the panel header — sets `drawerOpen` to `true`.
- Mount `<CommitBrowserDrawer isOpen={drawerOpen} mode="single" onClose={() => setDrawerOpen(false)} onSelectCommit={(sha) => { setTimeContextSha(sha); setDrawerOpen(false); }} />`.

`setTimeContextSha` should update the time-context store introduced in Phase 09.

### `OverviewComparePanel.tsx`

File: `clients/desktop/src/renderer/pages/Overview/OverviewComparePanel.tsx`

Replace the existing plain date pickers with commit-based selection:

```typescript
const [drawerOpen, setDrawerOpen] = useState(false);
const [drawerSlot, setDrawerSlot] = useState<'base' | 'target'>('base');
const [baseSha, setBaseSha] = useState<string | null>(null);
const [targetSha, setTargetSha] = useState<string | null>(null);

// Render:
<Button onClick={() => { setDrawerSlot('base'); setDrawerOpen(true); }}>
  {baseSha ? <CommitChip sha={baseSha} onClear={() => setBaseSha(null)} /> : 'Choose base commit'}
</Button>
<Button onClick={() => { setDrawerSlot('target'); setDrawerOpen(true); }}>
  {targetSha ? <CommitChip sha={targetSha} onClear={() => setTargetSha(null)} /> : 'Choose target commit'}
</Button>

<CommitBrowserDrawer
  isOpen={drawerOpen}
  mode="single"
  onClose={() => setDrawerOpen(false)}
  onSelectCommit={(sha) => {
    if (drawerSlot === 'base') setBaseSha(sha);
    else setTargetSha(sha);
    setDrawerOpen(false);
  }}
/>
```

`CommitChip`: a small inline component — `shortSha` in monospace + "×" clear button. Inline it in this file rather than creating a new file (single-use).

## Testing

### `useCommitBrowser.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useCommitBrowser.test.ts

describe('useCommitBrowser', () => {
  it('fetches commits on mount with limit=50 and offset=0', async () => {
    const { result } = renderHook(() => useCommitBrowser());
    await waitFor(() =>
      expect(mockApi.history.getCommitSummaries).toHaveBeenCalledWith({ limit: 50, offset: 0 })
    );
  });

  it('filters commits by search term after 300ms debounce', async () => {
    const { result } = renderHook(() => useCommitBrowser());
    act(() => result.current.setFilters({ search: 'auth' }));
    await waitFor(
      () => {
        expect(
          result.current.filteredCommits.every(
            c => c.subject.includes('auth') || c.sha.includes('auth')
          )
        ).toBe(true);
      },
      { timeout: 500 }
    );
  });

  it('filters commits by author', async () => {
    const { result } = renderHook(() => useCommitBrowser());
    await waitFor(() => expect(result.current.commits.length).toBeGreaterThan(0));
    act(() => result.current.setFilters({ author: 'Alice' }));
    expect(result.current.filteredCommits.every(c => c.authorName === 'Alice')).toBe(true);
  });

  it('filters to analyzedOnly using statusMap', async () => {
    // Mock useCommitAnalysisStatus to return known SHAs as analyzed
    // Assert: filteredCommits contains only those SHAs
  });

  it('loadMore fetches the next page with incremented offset', async () => {
    const { result } = renderHook(() => useCommitBrowser());
    await waitFor(() => expect(result.current.commits.length).toBe(50));
    await act(() => result.current.loadMore());
    expect(mockApi.history.getCommitSummaries).toHaveBeenCalledWith({ limit: 50, offset: 50 });
  });

  it('sets hasMore=false when response has fewer than 50 commits', async () => {
    mockApi.history.getCommitSummaries.mockResolvedValueOnce(generateCommits(30));
    const { result } = renderHook(() => useCommitBrowser());
    await waitFor(() => expect(result.current.hasMore).toBe(false));
  });
});
```

### `useCommitAnalysisStatus.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useCommitAnalysisStatus.test.ts
import { clearCommitAnalysisCache } from './useCommitAnalysisStatus';

describe('useCommitAnalysisStatus', () => {
  beforeEach(() => {
    // Reset the module-level cache so each test starts from a clean state
    clearCommitAnalysisCache();
  });

  it('batches SHA checks in groups of 20', async () => {
    const shas = Array.from({ length: 45 }, (_, i) => `sha${i}`);
    renderHook(() => useCommitAnalysisStatus(shas));
    await waitFor(() => {
      // 45 SHAs → 3 batches: 20 + 20 + 5
      expect(mockApi.snapshots.hasForCommit).toHaveBeenCalledTimes(45);
    });
  });

  it('caches results between re-renders', async () => {
    const shas = ['sha1', 'sha2'];
    const { rerender } = renderHook(() => useCommitAnalysisStatus(shas));
    await waitFor(() => expect(mockApi.snapshots.hasForCommit).toHaveBeenCalledTimes(2));
    rerender();
    // No additional calls — cache hit
    expect(mockApi.snapshots.hasForCommit).toHaveBeenCalledTimes(2);
  });

  it('re-queries when shas array length increases', async () => {
    const { rerender } = renderHook(({ shas }) => useCommitAnalysisStatus(shas), {
      initialProps: { shas: ['sha1'] },
    });
    await waitFor(() => expect(mockApi.snapshots.hasForCommit).toHaveBeenCalledTimes(1));
    rerender({ shas: ['sha1', 'sha2'] });
    await waitFor(() => expect(mockApi.snapshots.hasForCommit).toHaveBeenCalledTimes(2));
  });
});
```

### `CommitRow.test.tsx`

```typescript
// clients/desktop/src/renderer/components/history/CommitRow.test.tsx

describe('CommitRow', () => {
  const baseCommit: CommitSummary = {
    sha: 'abc1234def567',
    shortSha: 'abc1234',
    subject: 'Fix authentication bug',
    authorName: 'Alice',
    timestamp: Date.now(),
    parentShas: [],
  };

  it('shows green Badge for score >= 80', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={true} healthScore={85} mode="single" />);
    const badge = screen.getByText('85');
    expect(badge.className).toMatch(/green/);
  });

  it('shows yellow Badge for score 60–79', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={true} healthScore={72} mode="single" />);
    const badge = screen.getByText('72');
    expect(badge.className).toMatch(/yellow/);
  });

  it('shows red Badge for score < 60', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={true} healthScore={45} mode="single" />);
    const badge = screen.getByText('45');
    expect(badge.className).toMatch(/red/);
  });

  it('shows gray "not analyzed" Badge when hasSnapshot=false', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={false} healthScore={null} mode="single" />);
    expect(screen.getByText('not analyzed')).toBeInTheDocument();
  });

  it('shows "Analyze now" button for unanalyzed commits', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={false} healthScore={null} mode="single" onAnalyzeNow={vi.fn()} />);
    expect(screen.getByRole('button', { name: /Analyze now/ })).toBeInTheDocument();
  });

  it('shows "View →" button for analyzed commits in single mode', () => {
    render(<CommitRow commit={baseCommit} hasSnapshot={true} healthScore={80} mode="single" onView={vi.fn()} />);
    expect(screen.getByRole('button', { name: /View/ })).toBeInTheDocument();
  });
});
```

### `CommitBrowserFilters.test.tsx`

```typescript
// clients/desktop/src/renderer/components/history/CommitBrowserFilters.test.tsx

describe('CommitBrowserFilters', () => {
  const defaultFilters: CommitBrowserFilters = {
    search: '',
    author: null,
    analyzedOnly: false,
    dateFrom: null,
    dateTo: null,
  };

  it('debounces search input by 300ms before calling onChange', async () => {
    const onChange = vi.fn();
    render(
      <CommitBrowserFilters filters={defaultFilters} authors={['Alice', 'Bob']} onChange={onChange} />,
    );
    const searchInput = screen.getByPlaceholderText(/Search commits/);
    fireEvent.change(searchInput, { target: { value: 'auth' } });
    expect(onChange).not.toHaveBeenCalled();
    await waitFor(() => expect(onChange).toHaveBeenCalledWith({ search: 'auth' }), { timeout: 400 });
  });

  it('populates author Dropdown with provided authors', () => {
    render(
      <CommitBrowserFilters filters={defaultFilters} authors={['Alice', 'Bob']} onChange={vi.fn()} />,
    );
    // Open dropdown and verify options
    expect(screen.getByText('All authors')).toBeInTheDocument();
  });
});
```
