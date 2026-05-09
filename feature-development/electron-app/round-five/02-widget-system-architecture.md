---
id: 02-widget-system-architecture
title: 'Widget System Architecture'
phase: 5
dependencies: [01]
status: planned
---

# Widget System Architecture

## Problem Statement

The current `WorkspaceDashboard` is a static layout hard-coded in JSX. Every user sees the same tiles in the same order regardless of which metrics matter to their workflow. There is no mechanism to add, remove, reorder, or resize individual visualizations. As the widget library grows, a static layout becomes unmaintainable and fails to serve teams with divergent priorities (e.g., a security team vs. a performance team).

A widget system replaces the static dashboard with a composable grid where each widget is an independent, self-contained unit responsible for its own data fetching, configuration, and rendering. The system stores the user's layout in SQLite and restores it on every launch.

**Scope for v1:** One dashboard per workspace. Multiple named dashboards are deferred to v2.

---

## New Files

| File                                                                       | Role                                                                        |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/types/widget-types.ts`                       | Widget interfaces and registry types                                        |
| `clients/desktop/src/renderer/components/dashboard/WidgetShell.tsx`        | Wrapper providing drag handle, title bar, three-dot menu, and resize handle |
| `clients/desktop/src/renderer/components/dashboard/WidgetGrid.tsx`         | Grid container with drag-and-drop layout engine                             |
| `clients/desktop/src/renderer/components/dashboard/WidgetLibraryModal.tsx` | Categorized widget picker modal                                             |
| `clients/desktop/src/renderer/components/dashboard/WidgetConfigMenu.tsx`   | Per-widget three-dot dropdown config                                        |
| `clients/desktop/src/renderer/hooks/useWidgetRegistry.ts`                  | Hook providing widget catalog and instantiation                             |
| `clients/desktop/src/renderer/hooks/useDashboardLayout.ts`                 | Hook managing layout state, persistence, and drag events                    |
| `clients/desktop/src/main/ipc/handlers/dashboard.ts`                       | IPC handlers for dashboard layout CRUD                                      |
| `clients/desktop/src/shared/ipc/dashboard-types.ts`                        | Shared types for dashboard layout persistence                               |

---

## Modified Files

| File                                                        | Change                                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/pages/WorkspaceDashboard.tsx` | Replace static grid with `<WidgetGrid>` and integrate `useDashboardLayout` |
| `clients/desktop/src/main/db/migrations/index.ts`           | Add migration 19: `dashboard_layouts` table                                |
| `clients/desktop/src/shared/ipc/settings-types.ts`          | No changes required; layout stored in DB, not settings                     |

---

## Shared Types and Interfaces

### `clients/desktop/src/shared/ipc/dashboard-types.ts`

```typescript
export interface WidgetPosition {
  col: number; // 0-indexed, 0..11
  row: number; // 0-indexed
}

export interface WidgetSize {
  cols: number; // number of grid columns spanned
  rows: number; // number of row units spanned
}

export interface PersistedWidgetInstance {
  instanceId: string;
  definitionId: string;
  position: WidgetPosition;
  size: WidgetSize;
  config: Record<string, unknown>;
}

export interface DashboardLayout {
  id: string; // 'default' for v1 single-dashboard
  workspaceId: string;
  widgets: PersistedWidgetInstance[];
  updatedAt: number; // unix epoch ms
}

// IPC channel payloads
export interface GetLayoutArgs {
  workspaceId: string;
}
export interface SaveLayoutArgs {
  workspaceId: string;
  layout: DashboardLayout;
}
export interface ResetLayoutArgs {
  workspaceId: string;
}
```

### `clients/desktop/src/renderer/types/widget-types.ts`

```typescript
import type { ComponentType } from 'react';
import type { PersistedWidgetInstance } from '../../shared/ipc/dashboard-types';

export type WidgetCategory = 'overview' | 'trends' | 'issues' | 'metrics';
export type WidgetTier = 'free' | 'pro';

export interface WidgetConfigSchema {
  fields: WidgetConfigField[];
}

export interface WidgetConfigField {
  key: string;
  label: string;
  type: 'select' | 'toggle' | 'number';
  options?: { label: string; value: string | number | boolean }[];
  defaultValue: unknown;
}

export interface WidgetProps {
  instance: PersistedWidgetInstance;
  onConfigChange: (config: Record<string, unknown>) => void;
}

export interface WidgetDefinition {
  id: string;
  label: string;
  description: string;
  category: WidgetCategory;
  tier: WidgetTier;
  icon: string;
  defaultSize: WidgetSize;
  minSize: WidgetSize;
  maxSize: WidgetSize;
  configSchema: WidgetConfigSchema;
  component: ComponentType<WidgetProps>;
}

// WidgetSize is imported from dashboard-types.ts — do not redefine here.
// import type { WidgetSize } from '../../shared/ipc/dashboard-types';

// Runtime instance — extends persisted shape with resolved definition
export interface WidgetInstance extends PersistedWidgetInstance {
  definition: WidgetDefinition;
}

// Registry map keyed by definitionId
export type WidgetRegistry = Map<string, WidgetDefinition>;
```

---

## Drag-and-Drop Library Evaluation

### Candidate: `@dnd-kit`

`@dnd-kit/core` + `@dnd-kit/sortable` is the recommended choice for the following reasons:

- Accessibility-first: keyboard navigation and ARIA live regions built-in.
- Headless: no bundled CSS conflicts with Tailwind.
- React-native: no legacy class component wrappers.
- Active maintenance: consistent releases aligned with React 18+.
- Grid reordering: `@dnd-kit/sortable` supports two-dimensional grid layouts with collision strategies.

### Fallback

If `@dnd-kit` introduces bundle size concerns or conflicts with Electron's CSP policy, fall back to native HTML5 Drag-and-Drop API with a custom `dragstart`/`dragover`/`drop` implementation. Ghost element rendering is handled by cloning the `WidgetShell` node and positioning it absolutely during drag. This path requires manual collision detection (see Grid Layout Engine below).

### Recommended Addition

```
pnpm --filter @vipr/desktop add @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

---

## Grid Layout Engine

### Column Model

The grid uses the existing 12-column system (`grid-cols-12`) from `WorkspaceDashboard`. Widgets snap to column boundaries on drop. Row height is determined by the tallest widget in a row unit; content inside each widget scrolls if it overflows.

```
Columns:  0   1   2   3   4   5   6   7   8   9  10  11
          |---|---|---|---|---|---|---|---|---|---|---|---|
Widget A  [=======3col=======]
Widget B                      [=======3col=======]
Widget C  [===============8col===============]
Widget D                                      [==4col==]
```

### Collision Detection

Before committing a drop, the layout engine checks whether the proposed `(col, row, cols, rows)` rectangle intersects any existing widget rectangle. Two rectangles `A` and `B` overlap when:

```
A.col < B.col + B.cols  &&
B.col < A.col + A.cols  &&
A.row < B.row + B.rows  &&
B.row < A.row + A.rows
```

If a collision is detected, the engine attempts to push the displaced widget down by incrementing its `row` until no collision exists. If no valid position is found within a reasonable depth limit, the drag is rejected and the widget returns to its original position.

### Responsive Collapse

Below the `lg` breakpoint (`< 1024px`), all widgets collapse to full width (`cols: 12`) and stack vertically in their current row order. `WidgetGrid` reads `window.innerWidth` and applies the override at render time; no layout mutation is persisted.

---

## WidgetShell Component

### ASCII Layout

```
┌─ WidgetShell ──────────────────────────────────┐
│ ⠿  Widget Title                         ⋮  ▾  │
│ ─────────────────────────────────────────────── │
│                                                 │
│  [Widget Content Area — rendered by component]  │
│                                                 │
│                                            ◢    │
└─────────────────────────────────────────────────┘
```

| Element      | Details                                                                    |
| ------------ | -------------------------------------------------------------------------- |
| `⠿`          | Drag handle — `cursor-grab`, triggers `@dnd-kit` drag start on `mousedown` |
| Widget Title | `definition.label`, truncated with `truncate` at 200px max-width           |
| `⋮ ▾`        | Three-dot config menu trigger — opens `WidgetConfigMenu`                   |
| Content Area | `flex-1 min-h-0 overflow-auto` — scrolls independently                     |
| `◢`          | Resize handle — bottom-right corner, `cursor-se-resize`                    |

### Props

```typescript
interface WidgetShellProps {
  instance: WidgetInstance;
  isDragging: boolean;
  onRemove: (instanceId: string) => void;
  onConfigure: (instanceId: string) => void;
  onRefresh: (instanceId: string) => void;
  children: React.ReactNode;
}
```

### Styling

```
bg-white dark:bg-neutral-900
border border-neutral-200 dark:border-neutral-800
rounded-xl shadow-sm
flex flex-col
transition-shadow duration-150
```

During drag: `opacity-50 shadow-lg ring-2 ring-blue-500/40`

---

## WidgetGrid Component

### Responsibilities

1. Render the 12-column grid container.
2. Wrap each `WidgetInstance` in a `WidgetShell`.
3. Provide `@dnd-kit` `DndContext` and `SortableContext`.
4. Emit layout change events to `useDashboardLayout` on drag end.

### Props

```typescript
interface WidgetGridProps {
  instances: WidgetInstance[];
  onLayoutChange: (updated: PersistedWidgetInstance[]) => void;
  onRemove: (instanceId: string) => void;
  onConfigure: (instanceId: string) => void;
}
```

---

## WidgetConfigMenu Component

Uses the existing `Dropdown` component from `@vipr/ui`. The menu is anchored to the three-dot button in `WidgetShell`.

### Menu Items

| Item                  | Behavior                                      | Availability                                                            |
| --------------------- | --------------------------------------------- | ----------------------------------------------------------------------- |
| Configure             | Opens widget-specific config panel in a Modal | All tiers; Pro-gated widgets on free tier show "Upgrade to Pro" tooltip |
| Refresh               | Re-fetches widget data from main process      | Always                                                                  |
| Remove from Dashboard | Removes the instance from the layout          | Always                                                                  |

### Pro Gate on Free Tier

When `instance.definition.tier === 'pro'` and the user is on the free tier, the "Configure" item is replaced with an "Upgrade to Pro" item that navigates to the upgrade flow from Phase 01. The widget content itself shows the `EmptyState` upgrade CTA defined in `03-default-dashboard.md`.

---

## WidgetLibraryModal Component

Uses the existing `Modal` component from `@vipr/ui`.

### ASCII Layout

```
┌─ Widget Library ─────────────────────────────────────────────┐
│  [Search widgets...]                                    [✕]  │
│  ───────────────────────────────────────────────────────────  │
│  [Overview] [Trends] [Issues] [Metrics]                       │
│  ───────────────────────────────────────────────────────────  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ [icon]       │  │ [icon]       │  │ [icon]       │       │
│  │ Widget Name  │  │ Widget Name  │  │ Widget Name  │       │
│  │ Description  │  │ Description  │  │ Description  │       │
│  │  [Free]      │  │  [Pro]       │  │  [Free]      │       │
│  │ [+ Add]      │  │ [+ Add]      │  │ [+ Add]      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                               │
│                                        [Close]                │
└───────────────────────────────────────────────────────────────┘
```

### Component Reuse

| UI Need       | Component                |
| ------------- | ------------------------ |
| Modal shell   | `Modal` from `@vipr/ui`  |
| Category tabs | `Tabs` from `@vipr/ui`   |
| Tier label    | `Badge` from `@vipr/ui`  |
| Search input  | `Input` from `@vipr/ui`  |
| Add button    | `Button` from `@vipr/ui` |

---

## Hooks

### `useWidgetRegistry`

Returns the static widget catalog (all `WidgetDefinition` objects). The registry is a module-level constant imported from widget index files; no async loading is needed because widgets are bundled with the renderer.

```typescript
function useWidgetRegistry(): {
  registry: WidgetRegistry;
  getDefinition: (id: string) => WidgetDefinition | undefined;
  getByCategory: (category: WidgetCategory) => WidgetDefinition[];
};
```

### `useDashboardLayout`

Manages the mutable layout state, communicates with the main process via IPC for persistence.

```typescript
function useDashboardLayout(workspaceId: string): {
  instances: WidgetInstance[];
  isLoading: boolean;
  addWidget: (definitionId: string) => void;
  removeWidget: (instanceId: string) => void;
  updatePosition: (instanceId: string, position: WidgetPosition, size: WidgetSize) => void;
  updateConfig: (instanceId: string, config: Record<string, unknown>) => void;
  resetToDefaults: () => Promise<void>;
  saveLayout: () => Promise<void>;
};
```

Layout is auto-saved with a 500 ms debounce after any mutation. Manual save is exposed for "Reset to Defaults" confirmation flows.

---

## IPC Additions

All channels follow the existing `ipcMain.handle` pattern from `clients/desktop/src/main/ipc/handlers/`.

### `clients/desktop/src/main/ipc/handlers/dashboard.ts`

| Channel                  | Direction       | Payload           | Return                             |
| ------------------------ | --------------- | ----------------- | ---------------------------------- |
| `dashboard:get-layout`   | renderer → main | `GetLayoutArgs`   | `DashboardLayout \| null`          |
| `dashboard:save-layout`  | renderer → main | `SaveLayoutArgs`  | `void`                             |
| `dashboard:reset-layout` | renderer → main | `ResetLayoutArgs` | `DashboardLayout` (default layout) |

The `reset-layout` handler deletes the stored row for `workspaceId` and returns the compiled default layout from `clients/desktop/src/renderer/config/default-dashboard-layout.ts` (defined in Phase 03).

**IPC registration wiring:** The new `dashboard.ts` handler file must be:

1. Imported and called from `clients/desktop/src/main/index.ts` (where all other handlers are registered)
2. Added to the `ViprDesktopAPI` type in `clients/desktop/src/shared/ipc/api-types.ts` as a `dashboard` namespace with `getLayout`, `saveLayout`, and `resetLayout` methods
3. Wired through a new preload context file in `clients/desktop/src/preload/contexts/`

This follows the same pattern as all existing IPC handlers (settings, history, backfill, etc.).

---

## Database Migration

### Migration 19: `dashboard_layouts` Table

Added to `clients/desktop/src/main/db/migrations/index.ts` as `version: 19`. Continues from the existing version 18 schema.

```sql
-- Migration 19: Dashboard layout persistence
CREATE TABLE IF NOT EXISTS dashboard_layouts (
  id TEXT NOT NULL,                  -- 'default' for v1
  workspace_id TEXT NOT NULL,
  layout_json TEXT NOT NULL,         -- JSON blob of PersistedWidgetInstance[]
  updated_at INTEGER NOT NULL DEFAULT (unixepoch('now', 'subsec') * 1000),
  PRIMARY KEY (id, workspace_id)
);
```

Storing layout as a JSON blob avoids a complex relational schema for widget positions and defers normalization until multi-dashboard support (v2) justifies it.

---

## Existing Components to Reuse

| Component    | Package    | Usage                                   |
| ------------ | ---------- | --------------------------------------- |
| `Dropdown`   | `@vipr/ui` | `WidgetConfigMenu` three-dot options    |
| `Modal`      | `@vipr/ui` | `WidgetLibraryModal` shell              |
| `Tabs`       | `@vipr/ui` | Category filter in `WidgetLibraryModal` |
| `Badge`      | `@vipr/ui` | Free/Pro tier label on widget cards     |
| `Input`      | `@vipr/ui` | Search field in `WidgetLibraryModal`    |
| `Button`     | `@vipr/ui` | "Add to Dashboard", "Reset to Defaults" |
| `EmptyState` | `@vipr/ui` | Empty dashboard (no widgets added)      |

---

## Color and Theme Tokens

All tokens use dark mode pairings as specified in `CLAUDE.md`.

| Semantic          | Light                              | Dark                                         |
| ----------------- | ---------------------------------- | -------------------------------------------- |
| Widget border     | `border-neutral-200`               | `dark:border-neutral-800`                    |
| Widget background | `bg-white`                         | `dark:bg-neutral-900`                        |
| Drag active ring  | `ring-blue-500/40`                 | same (ring opacity handles both)             |
| Pro badge         | `bg-purple-500/20 text-purple-700` | `dark:bg-purple-500/10 dark:text-purple-400` |
| Free badge        | `bg-green-500/20 text-green-700`   | `dark:bg-green-500/10 dark:text-green-400`   |

---

## Grid Layout Classes

```
WidgetGrid container:   grid grid-cols-12 gap-4 px-4 sm:px-6 lg:px-8 py-8
WidgetShell base:       col-span-{cols} row-span-{rows}
WidgetShell responsive: max-lg:col-span-12
```

Row span is implemented via inline `style={{ gridRow: 'span N' }}` since Tailwind's JIT cannot safely generate arbitrary `row-span-*` classes from dynamic values.

---

## Testing Approach

### Unit Tests

- `useDashboardLayout` — layout mutation, debounced save trigger, IPC round-trip (mock IPC).
- Collision detection algorithm — property-based tests covering overlap, adjacency, push-down logic.
- Drag reorder logic — verify widget array reordering does not corrupt positions.

### Component Tests (Vitest + Testing Library)

- `WidgetShell` — renders title, drag handle, three-dot menu; fires `onRemove` and `onConfigure` callbacks.
- `WidgetLibraryModal` — renders widget cards per category; search filters by label; "Add to Dashboard" fires callback; Pro badge visible on gated widgets.
- `WidgetConfigMenu` — shows "Upgrade to Pro" item when tier is `pro` and user is on free plan.

### Integration Tests

- Layout persistence round-trip: add widget → save → reload → verify same instances and positions.
- Reset to defaults: mutate layout → reset → verify default layout restored.

---

## Dependencies on Other Phases

| Phase                  | Dependency                                                                                                                  |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 01 — Pro Tier Gating   | `tier` field on `WidgetDefinition`; upgrade CTA in Pro-gated config menu items                                              |
| 03 — Default Dashboard | Provides the `default-dashboard-layout.ts` config used by `dashboard:reset-layout` IPC handler and first-run initialization |
