---
id: 03-default-dashboard
title: 'Default Dashboard and Widget Library'
phase: 5
dependencies: [01, 02]
status: planned
---

# Default Dashboard and Widget Library

## Problem Statement

The widget system from Phase 02 provides the infrastructure for a composable dashboard, but an empty grid on first launch creates a poor onboarding experience. Users need an immediately useful starting state that surfaces the most actionable metrics without requiring any configuration.

This phase defines:

1. The curated default widget layout that loads on first analysis completion.
2. The full widget library — implementations of each widget component.
3. The widget library modal experience for adding and removing widgets.
4. Pro-tier gates on premium widgets with upgrade CTAs for free-tier users.

---

## New Files

| File                                                                                    | Role                                                              |
| --------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `clients/desktop/src/renderer/components/dashboard/widgets/OverallScoreWidget.tsx`      | `StatCard` showing overall codebase health score with trend arrow |
| `clients/desktop/src/renderer/components/dashboard/widgets/FileCountWidget.tsx`         | `StatCard` showing total analyzed file count                      |
| `clients/desktop/src/renderer/components/dashboard/widgets/IssueCountWidget.tsx`        | `StatCard` showing total issue count with severity breakdown      |
| `clients/desktop/src/renderer/components/dashboard/widgets/ScoreDistributionWidget.tsx` | `DoughnutChart` of score band distribution (A–F grades)           |
| `clients/desktop/src/renderer/components/dashboard/widgets/HealthTrendWidget.tsx`       | `LineChart` of overall health score over time — Pro               |
| `clients/desktop/src/renderer/components/dashboard/widgets/VelocityWidget.tsx`          | `LineChart` of commit velocity over time — Pro                    |
| `clients/desktop/src/renderer/components/dashboard/widgets/TopIssuesWidget.tsx`         | `CardTable` of highest-severity issues across the codebase        |
| `clients/desktop/src/renderer/components/dashboard/widgets/HotspotFilesWidget.tsx`      | `CardTable` of files with highest churn × complexity score        |
| `clients/desktop/src/renderer/components/dashboard/widgets/AntiPatternWidget.tsx`       | `CardTable` of anti-pattern categories with occurrence counts     |
| `clients/desktop/src/renderer/components/dashboard/widgets/RecentCommitsWidget.tsx`     | Compact list of recent commits with health delta indicators       |
| `clients/desktop/src/renderer/config/default-dashboard-layout.ts`                       | Default `PersistedWidgetInstance[]` with positions and sizes      |

---

## Modified Files

| File                                                                       | Change                                                   |
| -------------------------------------------------------------------------- | -------------------------------------------------------- |
| `clients/desktop/src/renderer/components/dashboard/WidgetLibraryModal.tsx` | Populate with widget catalog from `useWidgetRegistry`    |
| `clients/desktop/src/renderer/hooks/useWidgetRegistry.ts`                  | Register all widget definitions from this phase          |
| `clients/desktop/src/main/ipc/handlers/dashboard.ts`                       | `dashboard:reset-layout` returns compiled default layout |

---

## Default Dashboard Layout

### ASCII Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Overall Score (3col)  │  File Count (3col)  │  Issue Count (3col)  │  Score Dist (3col)  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Health Trend (8col, Pro)                          │  Top Issues (4col)    │
├─────────────────────────────────────────────────────────────────────────────┤
│  Hotspot Files (6col)                 │  Anti-Pattern Summary (6col)        │
├─────────────────────────────────────────────────────────────────────────────┤
│  Recent Commits (12col)                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Grid Positions

| Widget               | `col` | `row` | `cols` | `rows` |
| -------------------- | ----- | ----- | ------ | ------ |
| Overall Score        | 0     | 0     | 3      | 1      |
| File Count           | 3     | 0     | 3      | 1      |
| Issue Count          | 6     | 0     | 3      | 1      |
| Score Distribution   | 9     | 0     | 3      | 1      |
| Health Trend (Pro)   | 0     | 1     | 8      | 2      |
| Top Issues           | 8     | 1     | 4      | 2      |
| Hotspot Files        | 0     | 3     | 6      | 2      |
| Anti-Pattern Summary | 6     | 3     | 6      | 2      |
| Recent Commits       | 0     | 5     | 12     | 1      |

---

## `clients/desktop/src/renderer/config/default-dashboard-layout.ts`

```typescript
import type { PersistedWidgetInstance } from '../../shared/ipc/dashboard-types';

export const DEFAULT_DASHBOARD_LAYOUT: PersistedWidgetInstance[] = [
  {
    instanceId: 'default-overall-score',
    definitionId: 'overall-score',
    position: { col: 0, row: 0 },
    size: { cols: 3, rows: 1 },
    config: {},
  },
  {
    instanceId: 'default-file-count',
    definitionId: 'file-count',
    position: { col: 3, row: 0 },
    size: { cols: 3, rows: 1 },
    config: {},
  },
  {
    instanceId: 'default-issue-count',
    definitionId: 'issue-count',
    position: { col: 6, row: 0 },
    size: { cols: 3, rows: 1 },
    config: {},
  },
  {
    instanceId: 'default-score-distribution',
    definitionId: 'score-distribution',
    position: { col: 9, row: 0 },
    size: { cols: 3, rows: 1 },
    config: {},
  },
  {
    instanceId: 'default-health-trend',
    definitionId: 'health-trend',
    position: { col: 0, row: 1 },
    size: { cols: 8, rows: 2 },
    config: { timeRange: '30d' },
  },
  {
    instanceId: 'default-top-issues',
    definitionId: 'top-issues',
    position: { col: 8, row: 1 },
    size: { cols: 4, rows: 2 },
    config: { limit: 10, minSeverity: 'warning' },
  },
  {
    instanceId: 'default-hotspot-files',
    definitionId: 'hotspot-files',
    position: { col: 0, row: 3 },
    size: { cols: 6, rows: 2 },
    config: { limit: 10 },
  },
  {
    instanceId: 'default-anti-patterns',
    definitionId: 'anti-patterns',
    position: { col: 6, row: 3 },
    size: { cols: 6, rows: 2 },
    config: {},
  },
  {
    instanceId: 'default-recent-commits',
    definitionId: 'recent-commits',
    position: { col: 0, row: 5 },
    size: { cols: 12, rows: 1 },
    config: { limit: 8 },
  },
];
```

Stable `instanceId` values allow the reset handler to return a deterministic layout without calling `uuidv4` at runtime.

---

## Widget Catalog

### Full Specification Table

| id                   | label                | category | tier | default size | config options                        | data source                                                                |
| -------------------- | -------------------- | -------- | ---- | ------------ | ------------------------------------- | -------------------------------------------------------------------------- |
| `overall-score`      | Overall Score        | overview | free | 3×1          | none                                  | `api.snapshots.list()` or `api.database.getDashboardSummary()`             |
| `file-count`         | File Count           | overview | free | 3×1          | none                                  | `api.snapshots.list()` or `api.database.getDashboardSummary()`             |
| `issue-count`        | Issue Count          | overview | free | 3×1          | `minSeverity`: warning/error/critical | `api.database.getIssueList()`                                              |
| `score-distribution` | Score Distribution   | overview | free | 3×1          | none                                  | `api.snapshots.list()` or `api.database.getDashboardSummary()`             |
| `health-trend`       | Health Trend         | trends   | pro  | 8×2          | `timeRange`: 7d/30d/90d/1y            | `api.snapshots.getHealthTrend()`                                           |
| `velocity`           | Commit Velocity      | trends   | pro  | 6×2          | `timeRange`: 30d/90d                  | `api.velocity.getRepoTrend()`                                              |
| `top-issues`         | Top Issues           | issues   | free | 4×2          | `limit`: 5/10/25, `minSeverity`       | `api.database.getIssueList()`                                              |
| `hotspot-files`      | Hotspot Files        | issues   | free | 6×2          | `limit`: 10/25                        | `api.database.getHotspotData()` or `api.database.getAdaptiveHotspotData()` |
| `anti-patterns`      | Anti-Pattern Summary | issues   | free | 6×2          | none                                  | `api['anti-patterns'].getCategories()`                                     |
| `recent-commits`     | Recent Commits       | metrics  | free | 12×1         | `limit`: 5/8/15                       | `api.history.browseCommits()`                                              |

All data sources are accessed via `useViprDesktopAPI()` — never call `window.api` directly.

---

## Widget Component Specifications

### OverallScoreWidget

Renders a `StatCard` with the overall health score (0–100) from the latest snapshot. Includes a trend arrow comparing to the previous snapshot (`+3.2` / `-1.8` / `=`).

**Empty state:** "No analysis data yet. Run an analysis to populate this widget."

**Data shape:**

```typescript
interface OverallScoreData {
  score: number;
  previousScore: number | null;
  analyzedAt: number;
}
```

### FileCountWidget

Renders a `StatCard` with the total file count from the latest snapshot. No config options.

### IssueCountWidget

Renders a `StatCard` with the total issue count. Config option `minSeverity` filters what counts. Subtitle shows breakdown: `12 critical · 34 warning`.

### ScoreDistributionWidget

Renders a `DoughnutChart` with five segments corresponding to score bands:

| Band | Score Range | Color                  |
| ---- | ----------- | ---------------------- |
| A    | 80–100      | `#22c55e` (green-500)  |
| B    | 60–79       | `#84cc16` (lime-500)   |
| C    | 40–59       | `#eab308` (yellow-500) |
| D    | 20–39       | `#f97316` (orange-500) |
| F    | 0–19        | `#ef4444` (red-500)    |

Legend renders below chart using `DataList` component.

### HealthTrendWidget (Pro)

Renders a `LineChart` with health score on the Y axis and date on the X axis. Config option `timeRange` determines the data window. Supports 7d / 30d / 90d / 1y.

On free tier: shows `EmptyState` with upgrade CTA (see Pro Gate section below).

### VelocityWidget (Pro)

Renders a `LineChart` with commit count per day on the Y axis. Config option `timeRange` determines the data window. Supports 30d / 90d.

On free tier: shows `EmptyState` with upgrade CTA.

### TopIssuesWidget

Renders a `CardTable` with columns: Severity, File, Rule, Line. Rows are sorted by severity descending. Config option `limit` controls how many rows appear. No pagination — users navigate to the Issues page for full list.

**Column specs:**

| Column   | Width | Content                   |
| -------- | ----- | ------------------------- |
| Severity | 80px  | `Badge` with color token  |
| File     | flex  | truncated path, monospace |
| Rule     | 160px | rule identifier           |
| Line     | 60px  | line number               |

### HotspotFilesWidget

Renders a `CardTable` of files sorted by composite hotspot score (churn × complexity). Columns: File, Score, Changes (30d), Complexity.

### AntiPatternWidget

Renders a `CardTable` of anti-pattern categories. Columns: Category, Occurrences, Severity. Clicking a row navigates to the Issues page filtered to that category.

### RecentCommitsWidget

Renders a compact list (not `CardTable` — too heavy for single-row height). Each row: commit SHA (7 chars), author avatar initial, message (truncated to 60 chars), date, health delta badge (`+2.1` in green / `-3.4` in red).

```
┌──────────────────────────────────────────────────────────────────────┐
│  a3f8c12  J  "Refactor authentication module"   2 days ago   +2.1   │
│  89d1e44  M  "Add user profile endpoint"        3 days ago   -0.4   │
│  22cc901  J  "Fix null pointer in parser"       4 days ago   +0.8   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Widget Library Modal

The `WidgetLibraryModal` (shell defined in Phase 02) is populated by this phase with the full widget catalog.

### Modal Layout Detail

```
┌─ Widget Library ──────────────────────────────────────────────────────┐
│  [Search widgets...]                                             [✕]  │
│  ────────────────────────────────────────────────────────────────────  │
│  [Overview]  [Trends]  [Issues]  [Metrics]                            │
│  ────────────────────────────────────────────────────────────────────  │
│                                                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │
│  │   [icon]        │  │   [icon]        │  │   [icon]  [Pro] │       │
│  │ Overall Score   │  │ File Count      │  │ Health Trend    │       │
│  │ Current health  │  │ Files analyzed  │  │ Score over time │       │
│  │ score for your  │  │ in the latest   │  │ for the past    │       │
│  │ codebase.       │  │ snapshot.       │  │ 30/90/365 days. │       │
│  │ [Free]          │  │ [Free]          │  │ [Pro]           │       │
│  │ [+ Add]         │  │ [+ Add]         │  │ [+ Add]         │       │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘       │
│                                                                        │
│  Already on dashboard: [Overall Score ✓] [File Count ✓]              │
│                                                          [Close]       │
└────────────────────────────────────────────────────────────────────────┘
```

**Behavior notes:**

- Widgets already present on the dashboard show a checkmark and a disabled "Added" button.
- Searching filters by `label` and `description` (case-insensitive substring match).
- Pro badge uses purple token: `bg-purple-500/20 text-purple-700 dark:bg-purple-500/10 dark:text-purple-400`.
- Free badge uses green token: `bg-green-500/20 text-green-700 dark:bg-green-500/10 dark:text-green-400`.
- "Add" places the widget at the next available row in the layout (col 0, first row with no collision).

---

## Pro Gate: Empty States for Gated Widgets

When `instance.definition.tier === 'pro'` and the current user is on the free tier, the widget content area renders `EmptyState` instead of the visualization.

### EmptyState Template for Pro Widgets

```tsx
<EmptyState
  icon="lock"
  title="Pro Feature"
  description={`${definition.label} is available on the Pro plan. Upgrade to unlock trend data, velocity insights, and advanced metrics.`}
  action={
    <Button variant="primary" onClick={onUpgradeClick}>
      Upgrade to Pro
    </Button>
  }
/>
```

The `onUpgradeClick` handler navigates to the upgrade flow defined in Phase 01.

The `WidgetShell` still renders normally (drag handle, title, three-dot menu) so the user understands where the widget sits in their layout and can remove it if desired.

**No blur preview** is implemented in v1. A blurred screenshot overlay is a potential v2 enhancement once the widget visuals are stable.

---

## Dashboard Header Controls

The dashboard page header adds two controls alongside the existing workspace selector:

```
[Workspace Name ▾]                    [+ Add Widget]  [⋮]
```

The `⋮` menu contains:

| Item              | Action                                                         |
| ----------------- | -------------------------------------------------------------- |
| Reset to Defaults | Opens a `ConfirmModal` then calls `dashboard:reset-layout` IPC |
| Export Layout     | Downloads layout JSON (v2 deferred)                            |

"Add Widget" opens `WidgetLibraryModal`.

---

## Existing Components to Reuse

| Component       | Package    | Widget Usage                                                 |
| --------------- | ---------- | ------------------------------------------------------------ |
| `StatCard`      | `@vipr/ui` | `OverallScoreWidget`, `FileCountWidget`, `IssueCountWidget`  |
| `DoughnutChart` | `@vipr/ui` | `ScoreDistributionWidget`                                    |
| `LineChart`     | `@vipr/ui` | `HealthTrendWidget`, `VelocityWidget`                        |
| `CardTable`     | `@vipr/ui` | `TopIssuesWidget`, `HotspotFilesWidget`, `AntiPatternWidget` |
| `Modal`         | `@vipr/ui` | `WidgetLibraryModal` shell                                   |
| `Tabs`          | `@vipr/ui` | Category filter in `WidgetLibraryModal`                      |
| `Badge`         | `@vipr/ui` | Tier labels, severity indicators                             |
| `EmptyState`    | `@vipr/ui` | Pro gate empty state, no-analysis empty state                |
| `Button`        | `@vipr/ui` | "Add Widget", "Upgrade to Pro", "Reset to Defaults"          |
| `Dropdown`      | `@vipr/ui` | Dashboard header `⋮` menu                                    |
| `Input`         | `@vipr/ui` | Search field in `WidgetLibraryModal`                         |
| `DataList`      | `@vipr/ui` | Score distribution legend                                    |
| `ConfirmModal`  | `@vipr/ui` | "Reset to Defaults" confirmation                             |

---

## Color and Theme Tokens

| Semantic            | Light                              | Dark                                         |
| ------------------- | ---------------------------------- | -------------------------------------------- |
| Score A (excellent) | `text-green-700 bg-green-500/20`   | `dark:text-green-400 dark:bg-green-500/10`   |
| Score B (good)      | `text-lime-700 bg-lime-500/20`     | `dark:text-lime-400 dark:bg-lime-500/10`     |
| Score C (fair)      | `text-yellow-700 bg-yellow-500/20` | `dark:text-yellow-400 dark:bg-yellow-500/10` |
| Score D (poor)      | `text-orange-700 bg-orange-500/20` | `dark:text-orange-400 dark:bg-orange-500/10` |
| Score F (critical)  | `text-red-700 bg-red-500/20`       | `dark:text-red-400 dark:bg-red-500/10`       |
| Trend positive      | `text-green-700`                   | `dark:text-green-400`                        |
| Trend negative      | `text-red-700`                     | `dark:text-red-400`                          |
| Trend neutral       | `text-neutral-500`                 | `dark:text-neutral-400`                      |
| Pro badge           | `text-purple-700 bg-purple-500/20` | `dark:text-purple-400 dark:bg-purple-500/10` |
| Free badge          | `text-green-700 bg-green-500/20`   | `dark:text-green-400 dark:bg-green-500/10`   |

---

## Grid Layout Classes

```
Dashboard page:     px-4 sm:px-6 lg:px-8 py-8
Dashboard grid:     grid grid-cols-12 gap-4
Widget span:        col-span-{cols}  (static Tailwind classes at compile time)
Widget responsive:  max-lg:col-span-12
Row span:           style={{ gridRow: 'span N' }}  (dynamic, not JIT-safe)
```

Tailwind classes for widget col-spans must be included as complete strings in component files so JIT can detect them:

```
col-span-3  col-span-4  col-span-6  col-span-8  col-span-12
```

---

## Testing Approach

### Component Tests (Vitest + Testing Library)

- **Each widget in isolation:** mock IPC data, verify correct component renders (StatCard title value, chart data props, table row count).
- **OverallScoreWidget:** positive trend shows green arrow; negative trend shows red arrow; null previous score hides trend.
- **HealthTrendWidget (Pro, free tier):** renders `EmptyState` with "Upgrade to Pro" button, not `LineChart`.
- **VelocityWidget (Pro, free tier):** same pattern as above.
- **TopIssuesWidget:** respects `limit` config; severity badge colors match token map.
- **RecentCommitsWidget:** health delta renders with correct color; SHA truncated to 7 chars.
- **ScoreDistributionWidget:** five doughnut segments rendered with correct labels.

### Layout Tests

- Default layout: render all nine default widget instances using `DEFAULT_DASHBOARD_LAYOUT`; verify no collision exists between any two widget rectangles.
- Verify each widget's `definitionId` resolves to a registered `WidgetDefinition` (registry completeness check).

### Integration Tests

- `WidgetLibraryModal`: open modal, switch to "Trends" tab, verify only `health-trend` and `velocity` widgets appear; add `health-trend` to dashboard; verify it appears in layout state.
- Dashboard header "Reset to Defaults": confirm dialog → IPC call → layout restored to `DEFAULT_DASHBOARD_LAYOUT` positions.

---

## Dependencies on Other Phases

| Phase                           | Dependency                                                                                                                                           |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01 — Pro Tier Gating            | `tier` check for free/pro gate; `onUpgradeClick` navigation target; `UpgradeCTA` integration in `EmptyState`                                         |
| 02 — Widget System Architecture | `WidgetShell`, `WidgetGrid`, `useDashboardLayout`, `useWidgetRegistry`, `WidgetLibraryModal` shell, `dashboard-types.ts`, IPC handlers, migration 19 |
