---
id: 05-git-commit-graph
title: 'Git Commit Graph Visualization'
phase: 5
dependencies: [01]
status: planned
---

# Git Commit Graph Visualization

## Problem Statement

The existing commit browser introduced in Phase 14 (Round Four) presents commits as a flat, scrollable list. This representation loses the structural information developers depend on: which commits are on the main branch spine, where branches diverged, where merges happened, and how health degraded across that topology.

Developers think in directed acyclic graphs (DAGs). A visual commit graph overlaid with health scores makes quality regressions immediately spatial — a cluster of red nodes on a feature branch tells a story that a sorted list cannot. The graph also serves as the primary interaction surface for selecting two commits to compare (Phase 06), replacing the current linear range selector with direct node-to-node selection on a spatially meaningful canvas.

This feature is Pro-gated via the Phase 01 license system. The graph view renders only when a valid Pro license is active. Free-tier users see the existing list view and a `ProGate` upgrade prompt where the graph canvas would appear.

## New Files

| File                                                                      | Role                                                                                            |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/components/history/CommitGraph.tsx`         | Main ReactFlow canvas — reusable across full-page and modal contexts                            |
| `clients/desktop/src/renderer/components/history/CommitGraphNode.tsx`     | Custom ReactFlow node: compact pill with `shortSha`, `subject`, `authorName`, health badge      |
| `clients/desktop/src/renderer/components/history/CommitGraphEdge.tsx`     | Custom directional edge; solid for first-parent, dashed for merge sources                       |
| `clients/desktop/src/renderer/components/history/CommitGraphControls.tsx` | Zoom in/out, fit-to-view, and orientation toggle controls                                       |
| `clients/desktop/src/renderer/components/history/CommitGraphSidebar.tsx`  | Detail panel shown when a node is selected; full commit info + metric summary                   |
| `clients/desktop/src/renderer/hooks/useCommitGraph.ts`                    | Fetches commits via `browseCommits` IPC, builds dagre layout, returns ReactFlow `nodes`/`edges` |

## Modified Files

| File                                             | Changes                                                                                            |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/pages/History.tsx` | Add Graph View tab alongside existing List View; render `CommitGraph` in graph tab with Pro gating |

## Shared Types

```typescript
// Types are defined within component and hook files — no new shared IPC types required.
// CommitSummary from clients/desktop/src/shared/ipc/history-types.ts is the sole data contract.

// Internal to CommitGraph and useCommitGraph:
interface CommitGraphProps {
  commits: CommitSummary[];
  onCompare?: (shaA: string, shaB: string) => void;
  className?: string;
}

interface SelectedNodes {
  first: string | null; // SHA of first selected node
  second: string | null; // SHA of second selected node
}
```

`CommitSummary` already carries `parentShas: string[]` which is sufficient for DAG construction. No IPC changes are required for this phase.

## Hook: `useCommitGraph`

```typescript
// clients/desktop/src/renderer/hooks/useCommitGraph.ts

import type { Node, Edge } from '@xyflow/react';
import type { CommitSummary } from '../../shared/ipc/history-types';

interface UseCommitGraphOptions {
  limit?: number; // default 100; bounded by backfill depth
}

interface UseCommitGraphReturn {
  nodes: Node[];
  edges: Edge[];
  commits: CommitSummary[];
  isLoading: boolean;
  error: string | null;
  refetch: () => Promise<void>;
}

export function useCommitGraph(options?: UseCommitGraphOptions): UseCommitGraphReturn;
```

### Implementation Notes

**Data Fetching:**

Call `api.history.browseCommits({ limit: options?.limit ?? 100, offset: 0 })` on mount (where `api` is obtained via `useViprDesktopAPI()`). `browseCommits` is the existing IPC channel registered in the history handler — it returns `CommitSummary[]` with `parentShas` populated. See `clients/desktop/src/renderer/hooks/useCommitBrowser.ts` for the existing pattern.

**DAG Construction:**

Build a dagre graph from the fetched commits before converting to ReactFlow format:

```typescript
import dagre from '@dagrejs/dagre';

function buildLayout(commits: CommitSummary[]): { nodes: Node[]; edges: Edge[] } {
  const g = new dagre.graphlib.Graph();
  g.setGraph({ rankdir: 'LR', nodesep: 50, ranksep: 100 });
  g.setDefaultEdgeLabel(() => ({}));

  const NODE_WIDTH = 300;
  const NODE_HEIGHT = 56;

  for (const commit of commits) {
    g.setNode(commit.sha, { width: NODE_WIDTH, height: NODE_HEIGHT });
  }

  // Build a set of all known SHAs for boundary checking
  const knownShas = new Set(commits.map(c => c.sha));

  for (const commit of commits) {
    for (const parentSha of commit.parentShas) {
      // Only add edges where both nodes exist in the fetched set
      if (knownShas.has(parentSha)) {
        g.setEdge(parentSha, commit.sha);
      }
    }
  }

  dagre.layout(g);

  const nodes: Node[] = commits.map(commit => {
    const { x, y } = g.node(commit.sha);
    return {
      id: commit.sha,
      type: 'commitNode',
      position: { x: x - NODE_WIDTH / 2, y: y - NODE_HEIGHT / 2 },
      data: { commit },
    };
  });

  const edges: Edge[] = [];
  for (const commit of commits) {
    commit.parentShas.forEach((parentSha, index) => {
      if (knownShas.has(parentSha)) {
        edges.push({
          id: `${parentSha}-${commit.sha}`,
          source: parentSha,
          target: commit.sha,
          type: 'commitEdge',
          data: { isFirstParent: index === 0 },
        });
      }
    });
  }

  return { nodes, edges };
}
```

`parentShas[0]` is treated as the first-parent edge (main branch spine). All subsequent entries in `parentShas` are merge source edges (secondary parents).

## Component: `CommitGraph`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraph.tsx

import { ReactFlow, MiniMap, Background, useReactFlow } from '@xyflow/react';
import type { CommitSummary } from '../../../shared/ipc/history-types';

interface CommitGraphProps {
  commits: CommitSummary[];
  onCompare?: (shaA: string, shaB: string) => void;
  className?: string;
}

export function CommitGraph({ commits, onCompare, className }: CommitGraphProps): JSX.Element;
```

### Internal State

```typescript
const [selectedNodes, setSelectedNodes] = useState<SelectedNodes>({ first: null, second: null });
```

**Node selection logic:**

- First click on an unselected node: `setSelectedNodes({ first: sha, second: null })`
- Second click on a different node: `setSelectedNodes(prev => ({ ...prev, second: sha }))`
- Click on an already-selected node: deselect it
- Shift+click: fill whichever slot (`first` or `second`) is empty

When both `first` and `second` are non-null, the "Compare Selected (2)" action bar appears at the bottom of the canvas. Clicking it calls `onCompare?.(selectedNodes.first, selectedNodes.second)`.

### ReactFlow Configuration

```typescript
<ReactFlow
  nodes={nodes}
  edges={edges}
  nodeTypes={{ commitNode: CommitGraphNode }}
  edgeTypes={{ commitEdge: CommitGraphEdge }}
  fitView
  fitViewOptions={{ padding: 0.1 }}
  minZoom={0.1}
  maxZoom={2}
  onNodeClick={handleNodeClick}
  proOptions={{ hideAttribution: true }}
>
  <MiniMap
    nodeColor={(node) => healthScoreToColor(node.data.commit.healthScore)}
    maskColor="rgb(0,0,0,0.1)"
  />
  <Background gap={16} size={1} color="currentColor" className="text-gray-200 dark:text-gray-800" />
  <CommitGraphControls />
</ReactFlow>
```

### Layout Structure

```
┌─ History ─────────────────────────────────────────────────────────────────┐
│ [List View] [Graph View]                            Filter     Search     │
├──────────────────────────────────────────────────────────┬────────────────┤
│                                                          │                │
│  CommitGraph (ReactFlow canvas)                          │ CommitGraph    │
│                                                          │ Sidebar        │
│  ● abc123 ─── ● def456 ─── ● ghi789                      │                │
│       └── ● feature1 ──┘                                 │ (commit        │
│                                                          │  details       │
│                [CommitGraphControls]  [MiniMap]          │  when node     │
│                                                          │  selected)     │
├──────────────────────────────────────────────────────────┴────────────────┤
│ [Compare Selected (2)]                            (hidden until 2 nodes)  │
└───────────────────────────────────────────────────────────────────────────┘
```

Grid classes:

- Canvas area: `col-span-12 lg:col-span-9`
- Sidebar: `col-span-12 lg:col-span-3`

The bottom action bar (`flex items-center justify-between p-3 border-t border-gray-200 dark:border-gray-700`) renders inside `CommitGraph` and is conditionally shown using `selectedNodes.first && selectedNodes.second`.

## Component: `CommitGraphNode`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraphNode.tsx

import { Handle, Position, type NodeProps } from '@xyflow/react';
import type { CommitSummary } from '../../../shared/ipc/history-types';

interface CommitNodeData {
  commit: CommitSummary;
  isSelected: boolean;
  selectionIndex: 1 | 2 | null; // which selection slot this node occupies
}

export function CommitGraphNode({ data }: NodeProps<CommitNodeData>): JSX.Element;
```

### Visual Layout

```
┌────────────────────────────────────────────────┐
│ ●  abc1234  Fix auth bug  — jsmith        [92] │
└────────────────────────────────────────────────┘
```

- Outer wrapper: `flex items-center gap-2 px-3 py-2 rounded-full border text-sm font-mono`
- The colored circle (`●`) is a `w-3 h-3 rounded-full` element colored by `healthScore`
- `shortSha`: monospace, `text-xs text-gray-500 dark:text-gray-400`
- `subject`: truncated to 35 characters, `text-gray-900 dark:text-gray-100`
- `authorName`: `text-xs text-gray-500 dark:text-gray-400` after an em-dash separator
- Health badge: `Badge` component from `@vipr/ui` displaying the numeric score

**Base node style:**

```
bg-white dark:bg-gray-800
border border-gray-200 dark:border-gray-700
shadow-xs
```

**Selected node style (replaces border):**

```
ring-2 ring-violet-500 border-violet-500
```

When `selectionIndex` is `1` or `2`, show a small superscript badge (`A` or `B`) in the top-left corner of the node to indicate selection order for comparison.

### Health Score Color Dot

```typescript
function healthDotClass(score: number | null): string {
  if (score === null) return 'bg-gray-400 dark:bg-gray-600';
  if (score >= 80) return 'bg-green-500';
  if (score >= 60) return 'bg-yellow-500';
  return 'bg-red-500';
}
```

### ReactFlow Handles

```tsx
<Handle type="target" position={Position.Left} className="opacity-0" />
<Handle type="source" position={Position.Right} className="opacity-0" />
```

Handles are invisible — edges are the sole visual indicator of connectivity.

## Component: `CommitGraphEdge`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraphEdge.tsx

import { BaseEdge, EdgeLabelRenderer, getSmoothStepPath, type EdgeProps } from '@xyflow/react';

interface CommitEdgeData {
  isFirstParent: boolean;
}

export function CommitGraphEdge(props: EdgeProps<CommitEdgeData>): JSX.Element;
```

### Rendering Rules

| Edge type                            | `strokeWidth` | `strokeDasharray` | Color                                  |
| ------------------------------------ | ------------- | ----------------- | -------------------------------------- |
| First-parent (`isFirstParent: true`) | `2`           | none (solid)      | `stroke-gray-400 dark:stroke-gray-500` |
| Secondary parent (merge source)      | `1.5`         | `4 2`             | `stroke-gray-300 dark:stroke-gray-600` |

Uses `getSmoothStepPath` from `@xyflow/react` for routing. Directional arrowhead is specified via `markerEnd`:

```typescript
markerEnd={{ type: MarkerType.ArrowClosed, width: 12, height: 12 }
```

## Component: `CommitGraphControls`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraphControls.tsx

export function CommitGraphControls(): JSX.Element;
```

Renders as a `Panel` (from `@xyflow/react`) positioned at `bottom-left`. Contains three `Button` (sm, outline) elements:

| Button       | Action                                        |
| ------------ | --------------------------------------------- |
| Zoom In (+)  | `reactFlowInstance.zoomIn()`                  |
| Zoom Out (−) | `reactFlowInstance.zoomOut()`                 |
| Fit View     | `reactFlowInstance.fitView({ padding: 0.1 })` |

Uses `useReactFlow()` hook to access the instance. Renders inside the ReactFlow canvas so it inherits the canvas coordinate system.

## Component: `CommitGraphSidebar`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraphSidebar.tsx

import type { CommitSummary } from '../../../shared/ipc/history-types';

interface CommitGraphSidebarProps {
  commit: CommitSummary | null;
  onClose: () => void;
}

export function CommitGraphSidebar({ commit, onClose }: CommitGraphSidebarProps): JSX.Element;
```

### Layout

```
┌─ Commit Details ─────────────── [✕] ─┐
│                                       │
│ abc1234def5678  (monospace, full SHA) │
│ Fix authentication bug                │
│                                       │
│ Author    jsmith                      │
│ Date      2024-03-15 14:32 (from timestamp) │
│ Parents   def5678  ghi9012            │
│                                       │
│ ── Health ────────────────────────── │
│ [92]  Health Score                    │
│                                       │
│ ── Parents ──────────────────────── │
│ def5678  (clickable — focuses node)   │
│ ghi9012  (clickable — focuses node)   │
│                                       │
└───────────────────────────────────────┘
```

When `commit === null`, render a muted placeholder:

```
┌───────────────────────────────────────┐
│                                       │
│  Select a commit to view details      │
│                                       │
└───────────────────────────────────────┘
```

Clicking a parent SHA in the sidebar calls `onNodeFocus(sha)` — a callback injected from `CommitGraph` that centers the canvas on the target node using `reactFlowInstance.fitBounds(nodeBounds)`.

`CommitGraphSidebar` uses `DataList` from `@vipr/ui` for the author/date/parents rows. The health score uses `Badge` with color-matched class based on the score thresholds.

**Note:** `CommitSummary` has a `timestamp: number` field (epoch ms), not a `date: string` field. Format `timestamp` to a display date string using `new Date(commit.timestamp).toLocaleString()` or similar.

## Health Score Color Gradient

| Score               | Node dot                       | Badge classes                                                                 |
| ------------------- | ------------------------------ | ----------------------------------------------------------------------------- |
| 80–100              | `bg-green-500`                 | `bg-green-500/20 text-green-700 dark:bg-green-500/10 dark:text-green-400`     |
| 60–79               | `bg-yellow-500`                | `bg-yellow-500/20 text-yellow-700 dark:bg-yellow-500/10 dark:text-yellow-400` |
| 0–59                | `bg-red-500`                   | `bg-red-500/20 text-red-700 dark:bg-red-500/10 dark:text-red-400`             |
| `null` (unanalyzed) | `bg-gray-400 dark:bg-gray-600` | `bg-gray-500/20 text-gray-500 dark:text-gray-400`                             |

The same `healthScoreToColor` helper is used by `CommitGraphNode` and the `MiniMap` `nodeColor` callback.

## Modified Files: History Page

`History.tsx` currently renders a single list-based view. After this phase it renders two tabs:

```typescript
// clients/desktop/src/renderer/pages/History.tsx — tab addition

type HistoryView = 'list' | 'graph';

const [activeView, setActiveView] = useState<HistoryView>('list');
const [selectedCommitSha, setSelectedCommitSha] = useState<string | null>(null);

// In the JSX:
<Tabs value={activeView} onChange={setActiveView}>
  <Tab value="list" label="List View" />
  <Tab value="graph" label="Graph View" />
</Tabs>

{activeView === 'list' && <ExistingListView ... />}

{activeView === 'graph' && (
  <ProGate featureId="commit-graph" featureLabel="Commit Graph">
    <div className="grid grid-cols-12 gap-6 h-[calc(100vh-220px)]">
      <div className="col-span-12 lg:col-span-9 relative">
        <CommitGraph
          commits={graphCommits}
          onCompare={(shaA, shaB) => { /* Phase 06 handler */ }}
        />
      </div>
      <div className="col-span-12 lg:col-span-3">
        <CommitGraphSidebar
          commit={selectedCommit}
          onClose={() => setSelectedCommitSha(null)}
        />
      </div>
    </div>
  </ProGate>
)}
```

`ProGate` is the wrapper component from Phase 01. It renders `UpgradeCTA` inline when the user is on the free tier. When the license is valid, it renders its children normally.

`graphCommits` is fetched via `useCommitGraph` when the graph tab is active. The existing list-view commits fetched by `useHistory` are unaffected.

## Data Source

`browseCommits` is the IPC channel registered in `clients/desktop/src/main/ipc/handlers/history.ts`. It returns `CommitSummary[]` which already includes `parentShas`. No new IPC channels are needed.

Performance is bounded by the `limit` passed to `browseCommits`. At the default of 100 commits, dagre layout is computed in under 50ms on a modern machine. ReactFlow handles up to approximately 1000 nodes before requiring virtualization — this phase's scope is well within that boundary.

If the repository has fewer than 2 commits in the fetched set, `CommitGraph` renders an `EmptyState` with the message "Not enough commit history to display a graph."

## Reusable Architecture

`CommitGraph` accepts `commits: CommitSummary[]` as a prop and is self-contained. It does not know where the commits came from. This makes it embeddable:

- **Full-page** (this phase): rendered in `History.tsx` graph tab
- **Modal** (future): drop `<CommitGraph commits={commits} />` inside a `Modal` from `@vipr/ui`
- **Widget** (Phase 08): a small graph widget in the dashboard showing the last 20 commits

The `onCompare` callback is optional. When omitted, the two-node selection still works but no action bar appears. This allows the component to be used in read-only contexts.

## Existing Components and Dependencies to Reuse

| Component / Package | Source                          | How Used                                                   |
| ------------------- | ------------------------------- | ---------------------------------------------------------- |
| `@xyflow/react`     | Already in project dependencies | ReactFlow canvas, nodes, edges, MiniMap, Background, Panel |
| `@dagrejs/dagre`    | Already in project dependencies | Graph layout computation                                   |
| `Tabs`, `Tab`       | `@vipr/ui/tabs`                 | List View / Graph View tab switch in `History.tsx`         |
| `Badge`             | `@vipr/ui/badge`                | Health score display in nodes and sidebar                  |
| `Button`            | `@vipr/ui/button`               | Graph controls and Compare action bar                      |
| `DataList`          | `@vipr/ui/data-list`            | Author/date/SHA rows in `CommitGraphSidebar`               |
| `EmptyState`        | `@vipr/ui/empty-state`          | Empty graph state and empty sidebar state                  |
| `ProGate`           | Phase 01 component              | License check wrapper around the graph canvas              |

## Color and Theme Tokens

```
Node base:           bg-white dark:bg-gray-800
                     border border-gray-200 dark:border-gray-700
                     shadow-xs

Node selected:       ring-2 ring-violet-500 border-violet-500

Canvas background:   text-gray-200 dark:text-gray-800 (for Background dots)

Sidebar:             bg-white dark:bg-gray-800
                     border-l border-gray-200 dark:border-gray-700

Action bar:          border-t border-gray-200 dark:border-gray-700
                     bg-white dark:bg-gray-800/50
```

All health score colors follow the standard paired tokens (see Health Score Color Gradient table above).

## Testing

### `useCommitGraph.test.ts`

```typescript
// clients/desktop/src/renderer/hooks/useCommitGraph.test.ts

describe('useCommitGraph', () => {
  it('fetches commits on mount with default limit of 100', async () => {
    const { result } = renderHook(() => useCommitGraph());
    await waitFor(() =>
      expect(mockApi.history.browseCommits).toHaveBeenCalledWith({ limit: 100, offset: 0 })
    );
  });

  it('builds ReactFlow nodes from CommitSummary array', async () => {
    const commits = generateCommits(3); // sha1 → sha2 → sha3 linear
    mockApi.history.browseCommits.mockResolvedValue(commits);
    const { result } = renderHook(() => useCommitGraph());
    await waitFor(() => expect(result.current.nodes).toHaveLength(3));
    expect(result.current.nodes[0].type).toBe('commitNode');
  });

  it('creates edges from parentShas relationships', async () => {
    const commits: CommitSummary[] = [
      {
        sha: 'aaa',
        shortSha: 'aaa',
        subject: 'Root',
        authorName: 'dev',
        timestamp: 1,
        parentShas: [],
        healthScore: null,
      },
      {
        sha: 'bbb',
        shortSha: 'bbb',
        subject: 'Child',
        authorName: 'dev',
        timestamp: 2,
        parentShas: ['aaa'],
        healthScore: 85,
      },
    ];
    mockApi.history.browseCommits.mockResolvedValue(commits);
    const { result } = renderHook(() => useCommitGraph());
    await waitFor(() => expect(result.current.edges).toHaveLength(1));
    expect(result.current.edges[0].id).toBe('aaa-bbb');
  });

  it('marks parentShas[0] edge as first-parent', async () => {
    const commits: CommitSummary[] = [
      {
        sha: 'aaa',
        shortSha: 'aaa',
        subject: 'Root',
        authorName: 'dev',
        timestamp: 1,
        parentShas: [],
        healthScore: null,
      },
      {
        sha: 'bbb',
        shortSha: 'bbb',
        subject: 'Merge',
        authorName: 'dev',
        timestamp: 3,
        parentShas: ['aaa', 'ccc'],
        healthScore: 70,
      },
    ];
    // Note: ccc is not in the fetched set — edge should not be created for it
    mockApi.history.browseCommits.mockResolvedValue(commits);
    const { result } = renderHook(() => useCommitGraph());
    await waitFor(() => expect(result.current.edges).toHaveLength(1));
    expect(result.current.edges[0].data.isFirstParent).toBe(true);
  });

  it('does not create edges for parentShas not present in the fetched set', async () => {
    const commits: CommitSummary[] = [
      {
        sha: 'bbb',
        shortSha: 'bbb',
        subject: 'Child',
        authorName: 'dev',
        timestamp: 2,
        parentShas: ['aaa'],
        healthScore: 90,
      },
    ];
    // 'aaa' is not in the fetched set (boundary commit)
    mockApi.history.browseCommits.mockResolvedValue(commits);
    const { result } = renderHook(() => useCommitGraph());
    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.edges).toHaveLength(0);
  });
});
```

### `CommitGraphNode.test.tsx`

```typescript
// clients/desktop/src/renderer/components/history/CommitGraphNode.test.tsx

describe('CommitGraphNode', () => {
  const baseCommit: CommitSummary = {
    sha: 'abc1234def567',
    shortSha: 'abc1234',
    subject: 'Fix authentication bug',
    authorName: 'jsmith',
    timestamp: Date.now(),
    parentShas: [],
    healthScore: null,
  };

  it('renders shortSha, subject, and authorName', () => {
    render(<CommitGraphNode data={{ commit: baseCommit, isSelected: false, selectionIndex: null }} />);
    expect(screen.getByText('abc1234')).toBeInTheDocument();
    expect(screen.getByText(/Fix authentication bug/)).toBeInTheDocument();
    expect(screen.getByText(/jsmith/)).toBeInTheDocument();
  });

  it('shows green health dot for score >= 80', () => {
    const commit = { ...baseCommit, healthScore: 92 };
    render(<CommitGraphNode data={{ commit, isSelected: false, selectionIndex: null }} />);
    const dot = document.querySelector('.bg-green-500');
    expect(dot).toBeTruthy();
  });

  it('shows yellow health dot for score 60–79', () => {
    const commit = { ...baseCommit, healthScore: 65 };
    render(<CommitGraphNode data={{ commit, isSelected: false, selectionIndex: null }} />);
    const dot = document.querySelector('.bg-yellow-500');
    expect(dot).toBeTruthy();
  });

  it('shows red health dot for score < 60', () => {
    const commit = { ...baseCommit, healthScore: 40 };
    render(<CommitGraphNode data={{ commit, isSelected: false, selectionIndex: null }} />);
    const dot = document.querySelector('.bg-red-500');
    expect(dot).toBeTruthy();
  });

  it('shows gray health dot for unanalyzed commits (healthScore: null)', () => {
    render(<CommitGraphNode data={{ commit: baseCommit, isSelected: false, selectionIndex: null }} />);
    const dot = document.querySelector('.bg-gray-400');
    expect(dot).toBeTruthy();
  });

  it('applies ring classes when isSelected=true', () => {
    render(<CommitGraphNode data={{ commit: baseCommit, isSelected: true, selectionIndex: 1 }} />);
    const node = document.querySelector('[class*="ring-violet-500"]');
    expect(node).toBeTruthy();
  });

  it('shows selection index badge A for selectionIndex=1', () => {
    render(<CommitGraphNode data={{ commit: baseCommit, isSelected: true, selectionIndex: 1 }} />);
    expect(screen.getByText('A')).toBeInTheDocument();
  });

  it('shows selection index badge B for selectionIndex=2', () => {
    render(<CommitGraphNode data={{ commit: baseCommit, isSelected: true, selectionIndex: 2 }} />);
    expect(screen.getByText('B')).toBeInTheDocument();
  });

  it('truncates subject longer than 35 characters', () => {
    const commit = { ...baseCommit, subject: 'This is a very long commit message that exceeds the display limit' };
    render(<CommitGraphNode data={{ commit, isSelected: false, selectionIndex: null }} />);
    expect(screen.queryByText(commit.subject)).not.toBeInTheDocument();
    // Truncated version should be present
    expect(screen.getByText(/This is a very long commit message …/)).toBeInTheDocument();
  });
});
```

### Integration Test: Two-Node Selection and Compare Trigger

```typescript
// clients/desktop/src/renderer/components/history/CommitGraph.integration.test.tsx

describe('CommitGraph — two-node selection', () => {
  const commits: CommitSummary[] = [
    { sha: 'aaa', shortSha: 'aaa', subject: 'Root', authorName: 'dev', timestamp: 1, parentShas: [], healthScore: 90 },
    { sha: 'bbb', shortSha: 'bbb', subject: 'Child', authorName: 'dev', timestamp: 2, parentShas: ['aaa'], healthScore: 75 },
    { sha: 'ccc', shortSha: 'ccc', subject: 'Latest', authorName: 'dev', timestamp: 3, parentShas: ['bbb'], healthScore: 60 },
  ];

  it('does not show Compare action bar with zero nodes selected', () => {
    render(<CommitGraph commits={commits} onCompare={vi.fn()} />);
    expect(screen.queryByRole('button', { name: /Compare Selected/ })).not.toBeInTheDocument();
  });

  it('shows Compare action bar after two nodes are selected', async () => {
    render(<CommitGraph commits={commits} onCompare={vi.fn()} />);
    // Select first node (click on node text content)
    fireEvent.click(screen.getByText('Root'));
    // Select second node
    fireEvent.click(screen.getByText('Latest'));
    await waitFor(() =>
      expect(screen.getByRole('button', { name: /Compare Selected \(2\)/ })).toBeInTheDocument()
    );
  });

  it('calls onCompare with both SHAs when Compare button is clicked', async () => {
    const onCompare = vi.fn();
    render(<CommitGraph commits={commits} onCompare={onCompare} />);
    fireEvent.click(screen.getByText('Root'));
    fireEvent.click(screen.getByText('Latest'));
    await waitFor(() =>
      expect(screen.getByRole('button', { name: /Compare Selected/ })).toBeInTheDocument()
    );
    fireEvent.click(screen.getByRole('button', { name: /Compare Selected/ }));
    expect(onCompare).toHaveBeenCalledWith('aaa', 'ccc');
  });
});
```

## Dependencies on Other Phases

| Phase                            | Dependency                                                                                                                                                                                    |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Phase 01 (Pro Tier Gating)       | `ProGate` component and `useLicense` hook must be available before this phase. The graph canvas is wrapped in `ProGate` — without Phase 01, the feature flag cannot be evaluated.             |
| Phase 06 (Comparison Experience) | The `onCompare` callback receives two SHAs. Phase 06 defines what happens next: the A/B comparison panel. `CommitGraph` passes the SHAs up but does not implement the comparison view itself. |
