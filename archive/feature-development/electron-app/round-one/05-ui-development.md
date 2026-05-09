---
id: 05-ui-development
---

# vipr desktop - phase 3 ui development plan

comprehensive implementation plan for phase 3 ui development including technical specifications, component mapping, security implementation, and architecture review criteria.

---

## part a: technical proposal

### 1. overview

phase 3 delivers the complete react-based user interface for vipr desktop, implementing us-05 (dashboard), us-16 (advanced filtering), and foundational ui infrastructure including custom titlebar, routing, state management, virtualization, and comprehensive security.

**deliverables:**

- react application shell with platform-specific custom titlebar
- dashboard with summary cards and charts
- files view with virtualization
- issues view with advanced filtering
- settings interface
- presenter registry integration for dynamic report discovery
- comprehensive security implementation (IPC validation, CSP, XSS prevention)

**technology stack:**

| layer            | technology                       | rationale                                               |
| ---------------- | -------------------------------- | ------------------------------------------------------- |
| ui framework     | react 19 + typescript            | server components support, improved hooks, type safety  |
| styling          | tailwind css v4                  | consistency with styleguide                             |
| state management | zustand (6 stores)               | lightweight, excellent typescript support, ipc-friendly |
| routing          | react-router-dom v6 memoryrouter | required for electron file:// protocol                  |
| charts           | chart.js 4.x + react-chartjs-2   | token-efficient, performant, styleguide aligned         |
| virtualization   | @tanstack/react-virtual          | efficient rendering of 1000+ item lists                 |
| validation       | zod                              | type-safe IPC validation                                |

**note on react 19:** While React 19 introduces concurrent features (useTransition, useDeferredValue, Server Components), this Phase 3 implementation focuses on foundational UI with traditional client-side rendering. Concurrent features and React Server Components are **not applicable** in Electron's renderer process context, as:

- Electron renderer runs in a Chromium sandbox (client-side only)
- No server-side rendering capability in desktop context
- IPC communication replaces traditional data fetching patterns

Future optimization opportunities (Phase 4+):

- Use `useTransition` for non-blocking state updates during analysis
- Use `useDeferredValue` to defer expensive chart re-renders
- Leverage automatic batching for multiple state updates

### 2. component hierarchy and organization

```
src/renderer/
├── App.tsx                          # Root component with routing
├── index.tsx                        # React DOM entry point
├── pages/                           # Route-level pages
│   ├── Welcome.tsx                  # First-run experience (no repo open)
│   ├── Dashboard.tsx                # US-05: Dashboard overview
│   ├── Files.tsx                    # US-16: Files list with filters
│   ├── FileDetail.tsx               # File detail with tabs
│   ├── Issues.tsx                   # US-16: Issues list with filters
│   └── Settings.tsx                 # Settings interface
├── components/                      # Shared components
│   ├── layout/
│   │   ├── Titlebar.tsx            # Custom titlebar (platform-specific)
│   │   ├── Sidebar.tsx             # Navigation sidebar
│   │   └── Breadcrumbs.tsx         # Navigation breadcrumbs
│   ├── cards/
│   │   ├── MetricCard.tsx          # Dashboard summary cards
│   │   ├── TrendCard.tsx           # Cards with trend indicators
│   │   └── IssueCard.tsx           # Issue display cards
│   ├── charts/
│   │   ├── LineChart.tsx           # Line chart wrapper
│   │   ├── DoughnutChart.tsx       # Doughnut chart wrapper
│   │   └── BarChart.tsx            # Bar chart wrapper
│   ├── tables/
│   │   ├── VirtualizedTable.tsx    # Virtualized table for large datasets
│   │   └── FileRow.tsx             # File list row component
│   └── common/
│       ├── Badge.tsx               # Severity, file type badges
│       ├── Toast.tsx               # Toast notifications
│       ├── Modal.tsx               # Generic modal
│       ├── SafeText.tsx            # Safe text rendering (XSS prevention)
│       ├── FilePath.tsx            # Safe file path display
│       └── ErrorBoundary.tsx       # Error boundary with safe error display
├── hooks/                           # Custom hooks
│   ├── useRepository.ts            # Repository store hook
│   ├── useAnalysis.ts              # Analysis store hook
│   ├── useSettings.ts              # Settings store hook
│   ├── useUI.ts                    # UI store hook
│   └── useFilter.ts                # Filter store hook
├── stores/                          # Zustand stores
│   ├── repository.ts               # Repository state
│   ├── analysis.ts                 # Analysis state
│   ├── filter.ts                   # Filter state
│   ├── ui.ts                       # UI state (sidebar, modals, toasts)
│   └── settings.ts                 # Settings state
├── lib/                             # Shared utilities
│   ├── ipc-validator.ts            # Zod schemas and validation
│   ├── security-utils.ts           # Path sanitization, XSS prevention
│   └── chart-config.ts             # Chart.js configuration
└── utils/
    ├── csp-test.ts                 # CSP testing utilities (development)
    └── security-test.ts            # Security test suite
```

### 3. detailed technical specifications

#### 3.1 custom titlebar implementation

**platform-specific strategy:**

| platform | approach                       | native controls      | css regions                    |
| -------- | ------------------------------ | -------------------- | ------------------------------ |
| macos    | `titleBarStyle: 'hiddenInset'` | yes (traffic lights) | drag spacer for traffic lights |
| windows  | `frame: false`                 | no (custom react)    | custom minimize/maximize/close |
| linux    | `frame: false`                 | no (custom react)    | custom window controls         |

**implementation:**

```typescript
// src/main/window-manager.ts
import { BrowserWindow } from 'electron';

export function createMainWindow(): BrowserWindow {
  const isMacOS = process.platform === 'darwin';

  const mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    titleBarStyle: isMacOS ? 'hiddenInset' : undefined,
    frame: !isMacOS ? false : undefined,
    trafficLightPosition: isMacOS ? { x: 16, y: 16 } : undefined,
    webPreferences: {
      sandbox: true,
      contextIsolation: true,
      nodeIntegration: false,
      preload: path.join(__dirname, 'preload.js'),
    },
  });

  return mainWindow;
}
```

**titlebar component structure:**

```typescript
// src/renderer/components/layout/Titlebar.tsx
import React from 'react';
import { ThemeToggle } from '../common/ThemeToggle';
import { useSettings } from '../../hooks/useSettings';

interface TitlebarProps {
  title?: string;
}

export function Titlebar({ title = 'Vipr Desktop' }: TitlebarProps) {
  const platform = window.electron?.platform || 'unknown';
  const isMacOS = platform === 'darwin';

  return (
    <div className="h-12 bg-slate-50 dark:bg-slate-900 border-b border-slate-200 dark:border-slate-700 flex items-center px-4">
      {/* macOS traffic light spacer */}
      {isMacOS && <div className="w-20 shrink-0" style={{ WebkitAppRegion: 'drag' }} />}

      {/* Windows/Linux app icon */}
      {!isMacOS && (
        <div className="w-8 h-8 mr-3">
          <img src="/icon.svg" alt="Vipr" className="w-full h-full" />
        </div>
      )}

      {/* Draggable region */}
      <div className="flex-1 flex items-center" style={{ WebkitAppRegion: 'drag' }}>
        <span className="text-sm font-semibold text-slate-700 dark:text-slate-300">
          {title}
        </span>
      </div>

      {/* Theme toggle and search (no drag) */}
      <div style={{ WebkitAppRegion: 'no-drag' }} className="flex items-center gap-2">
        <ThemeToggle />
        <button
          className="px-3 py-1.5 text-sm text-slate-600 dark:text-slate-400 hover:bg-slate-100 dark:hover:bg-slate-800 rounded-md"
          onClick={() => window.viprDesktop.search.open()}
        >
          <kbd className="px-2 py-0.5 text-xs bg-slate-200 dark:bg-slate-700 rounded">⌘K</kbd>
        </button>
      </div>

      {/* Windows/Linux window controls */}
      {!isMacOS && (
        <div className="flex ml-3" style={{ WebkitAppRegion: 'no-drag' }}>
          <button
            className="w-10 h-10 flex items-center justify-center hover:bg-slate-100 dark:hover:bg-slate-800"
            onClick={() => window.viprDesktop.window.minimize()}
            aria-label="Minimize"
          >
            <svg width="12" height="2" viewBox="0 0 12 2" className="fill-current">
              <rect width="12" height="2" />
            </svg>
          </button>
          <button
            className="w-10 h-10 flex items-center justify-center hover:bg-slate-100 dark:hover:bg-slate-800"
            onClick={() => window.viprDesktop.window.maximize()}
            aria-label="Maximize"
          >
            <svg width="12" height="12" viewBox="0 0 12 12" className="fill-current">
              <rect width="12" height="12" strokeWidth="1.5" stroke="currentColor" fill="none" />
            </svg>
          </button>
          <button
            className="w-10 h-10 flex items-center justify-center hover:bg-red-500 hover:text-white"
            onClick={() => window.viprDesktop.window.close()}
            aria-label="Close"
          >
            <svg width="12" height="12" viewBox="0 0 12 12" className="fill-current">
              <path d="M1 1l10 10M11 1L1 11" strokeWidth="1.5" stroke="currentColor" />
            </svg>
          </button>
        </div>
      )}
    </div>
  );
}
```

**css for drag regions:**

```css
/* src/renderer/index.css */
[style*="WebkitAppRegion: 'drag'"] {
  -webkit-app-region: drag;
  user-select: none;
}

[style*="WebkitAppRegion: 'no-drag'"] {
  -webkit-app-region: no-drag;
}
```

**acceptance criteria:**

- macos shows native traffic lights in top-left with custom content
- windows/linux show custom window controls in top-right
- draggable region allows window movement
- buttons and interactive elements have no-drag region
- titlebar height is 48px (3rem) consistently
- dark mode support via tailwind dark: variants

##### 3.1.1 dark mode implementation strategy

**theme detection and toggle:**

```typescript
// src/renderer/hooks/useTheme.ts
import { useEffect } from 'react';
import { useSettingsStore } from '../stores/settings';

export function useTheme() {
  const { theme, setSetting } = useSettingsStore();

  useEffect(() => {
    const root = document.documentElement;

    if (theme === 'dark') {
      root.classList.add('dark');
    } else if (theme === 'light') {
      root.classList.remove('dark');
    } else if (theme === 'system') {
      // Follow system preference
      const isDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      if (isDark) {
        root.classList.add('dark');
      } else {
        root.classList.remove('dark');
      }

      // Listen for system theme changes
      const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
      const handleChange = (e: MediaQueryListEvent) => {
        if (e.matches) {
          root.classList.add('dark');
        } else {
          root.classList.remove('dark');
        }
      };

      mediaQuery.addEventListener('change', handleChange);
      return () => mediaQuery.removeEventListener('change', handleChange);
    }
  }, [theme]);

  const toggleTheme = () => {
    const nextTheme = theme === 'light' ? 'dark' : theme === 'dark' ? 'system' : 'light';
    setSetting('theme', nextTheme);
  };

  return { theme, toggleTheme };
}
```

**theme toggle button component:**

```typescript
// src/renderer/components/common/ThemeToggle.tsx
import React from 'react';
import { useTheme } from '../../hooks/useTheme';

export function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();

  const icons = {
    light: '☀️',
    dark: '🌙',
    system: '💻',
  };

  return (
    <button
      onClick={toggleTheme}
      className="w-10 h-10 flex items-center justify-center rounded-md hover:bg-slate-100 dark:hover:bg-slate-800 transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
      aria-label={`Theme: ${theme}`}
      title={`Current theme: ${theme} (click to cycle)`}
    >
      <span className="text-lg" role="img" aria-label={theme}>
        {icons[theme]}
      </span>
    </button>
  );
}
```

**chart.js dark mode theme:**

```typescript
// src/renderer/lib/chart-config.ts
import type { ChartOptions } from 'chart.js';

export function getChartOptions(isDark: boolean): ChartOptions {
  const textColor = isDark ? '#f1f5f9' : '#0f172a'; // slate-100 : slate-900
  const gridColor = isDark ? '#334155' : '#e2e8f0'; // slate-700 : slate-200

  return {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: {
        display: false,
      },
      tooltip: {
        backgroundColor: isDark ? '#1e293b' : '#ffffff',
        titleColor: textColor,
        bodyColor: textColor,
        borderColor: gridColor,
        borderWidth: 1,
        padding: 12,
        titleFont: {
          size: 14,
          weight: 'bold',
        },
        bodyFont: {
          size: 12,
        },
      },
    },
    scales: {
      x: {
        grid: {
          color: gridColor,
        },
        ticks: {
          color: textColor,
        },
      },
      y: {
        grid: {
          color: gridColor,
        },
        ticks: {
          color: textColor,
        },
      },
    },
  };
}
```

**dark mode acceptance criteria:**

- theme toggle cycles through light → dark → system
- system theme respects OS preference
- system theme updates when OS preference changes
- all text colors meet wcag aa contrast in both modes
- chart.js adapts colors, grids, tooltips to theme
- theme preference persists via settings store to sqlite
- smooth transitions between themes (transition-colors duration-200)

#### 3.2 dashboard implementation

##### 3.2.1 tailwind design token reference

**color palette:**

| semantic purpose        | tailwind token | hex value | css custom property      | usage                                      |
| ----------------------- | -------------- | --------- | ------------------------ | ------------------------------------------ |
| primary actions         | `blue-500`     | `#3b82f6` | `--color-primary`        | buttons, links, active nav states          |
| success / high health   | `green-500`    | `#22c55e` | `--color-success`        | health scores >80, success toasts          |
| warning / medium health | `yellow-500`   | `#eab308` | `--color-warning`        | scores 50-79, warning toasts               |
| error / low health      | `red-500`      | `#ef4444` | `--color-error`          | critical issues, scores < 50, error toasts |
| neutral bg light        | `slate-50`     | `#f8fafc` | `--bg-neutral-light`     | page backgrounds, sidebar light mode       |
| neutral bg dark         | `slate-900`    | `#0f172a` | `--bg-neutral-dark`      | page backgrounds dark mode                 |
| card bg light           | `white`        | `#ffffff` | `--bg-card-light`        | card surfaces, modal backgrounds           |
| card bg dark            | `slate-800`    | `#1e293b` | `--bg-card-dark`         | card surfaces dark mode                    |
| border light            | `slate-200`    | `#e2e8f0` | `--border-light`         | dividers, card borders                     |
| border dark             | `slate-700`    | `#334155` | `--border-dark`          | dividers dark mode                         |
| text primary light      | `slate-900`    | `#0f172a` | `--text-primary-light`   | headings, body text                        |
| text primary dark       | `slate-100`    | `#f1f5f9` | `--text-primary-dark`    | headings dark mode                         |
| text secondary light    | `slate-600`    | `#475569` | `--text-secondary-light` | captions, metadata                         |
| text secondary dark     | `slate-400`    | `#94a3b8` | `--text-secondary-dark`  | captions dark mode                         |

**spacing scale (adheres to 8pt grid):**

| size name | tailwind utility       | pixels | rem     | usage examples                          |
| --------- | ---------------------- | ------ | ------- | --------------------------------------- |
| xs        | `gap-1` `p-1` `m-1`    | 4px    | 0.25rem | tight icon gaps, badge internal padding |
| sm        | `gap-2` `p-2` `m-2`    | 8px    | 0.5rem  | compact layouts, small button padding   |
| base      | `gap-3` `p-3` `m-3`    | 12px   | 0.75rem | default element spacing                 |
| md        | `gap-4` `p-4` `m-4`    | 16px   | 1rem    | card padding, form input padding        |
| lg        | `gap-6` `p-6` `m-6`    | 24px   | 1.5rem  | generous card padding, section gaps     |
| xl        | `gap-8` `p-8` `m-8`    | 32px   | 2rem    | page padding, major section separation  |
| 2xl       | `gap-12` `p-12` `m-12` | 48px   | 3rem    | large section spacing                   |

**typography scale:**

| element type         | tailwind class      | font-size | line-height | weight          | letter-spacing | usage                              |
| -------------------- | ------------------- | --------- | ----------- | --------------- | -------------- | ---------------------------------- |
| page heading (h1)    | `text-3xl`          | 30px      | 36px        | `font-bold`     | `-0.01em`      | dashboard title, page titles       |
| section heading (h2) | `text-2xl`          | 24px      | 32px        | `font-bold`     | `-0.01em`      | "health over time", major sections |
| subsection (h3)      | `text-lg`           | 18px      | 28px        | `font-semibold` | `normal`       | card titles, modal titles          |
| body text            | `text-sm`           | 14px      | 20px        | `font-normal`   | `normal`       | paragraph text, table rows         |
| caption / metadata   | `text-xs`           | 12px      | 16px        | `font-medium`   | `0.05em`       | badges, timestamps, file counts    |
| code / monospace     | `text-sm font-mono` | 14px      | 20px        | `font-normal`   | `normal`       | file paths, code snippets          |

**responsive breakpoint system:**

| breakpoint | min-width | tailwind prefix | target device       | layout columns             |
| ---------- | --------- | --------------- | ------------------- | -------------------------- |
| mobile     | default   | (none)          | 375px+              | 1 column (col-span-12)     |
| tablet     | 640px     | `sm:`           | iPad, small tablets | 2 columns (sm:col-span-6)  |
| laptop     | 1024px    | `lg:`           | standard laptops    | 4 columns (lg:col-span-3)  |
| desktop    | 1280px    | `xl:`           | large desktops      | 4 columns (xl:col-span-3)  |
| wide       | 1536px    | `2xl:`          | ultra-wide displays | 4 columns (2xl:col-span-3) |

**example responsive card grid:**

```typescript
// Dashboard metric cards: 1 column mobile, 2 columns tablet, 4 columns desktop
<div className="grid grid-cols-12 gap-6">
  <MetricCard className="col-span-12 sm:col-span-6 lg:col-span-3" />
  <MetricCard className="col-span-12 sm:col-span-6 lg:col-span-3" />
  <MetricCard className="col-span-12 sm:col-span-6 lg:col-span-3" />
  <MetricCard className="col-span-12 sm:col-span-6 lg:col-span-3" />
</div>

// Chart layout: 1 column mobile, 8/4 split desktop
<div className="grid grid-cols-12 gap-6">
  <div className="col-span-12 lg:col-span-8"> {/* Line chart */} </div>
  <div className="col-span-12 lg:col-span-4"> {/* Doughnut chart */} </div>
</div>
```

**layout strategy:**

12-column css grid with responsive breakpoints following styleguide patterns.

**component mapping:**

| ui element      | styleguide component | source file                              | purpose                     |
| --------------- | -------------------- | ---------------------------------------- | --------------------------- |
| metric card     | dashboardcard01-03   | `partials/dashboard/DashboardCard01.jsx` | summary metrics with charts |
| stat card       | dashboardcard11      | `partials/dashboard/DashboardCard11.jsx` | simple stats with trend     |
| table card      | dashboardcard07      | `partials/dashboard/DashboardCard07.jsx` | top issues list             |
| line chart      | lineChart02          | `charts/LineChart02.jsx`                 | health over time            |
| doughnut chart  | doughnutChart        | `charts/DoughnutChart.jsx`               | complexity distribution     |
| filter dropdown | dropdownfilter       | `components/DropdownFilter.jsx`          | filter controls             |

**dashboard structure with loading states:**

```typescript
// src/renderer/pages/Dashboard.tsx
import React from 'react';
import { useAnalysis } from '../hooks/useAnalysis';
import { useFilter } from '../hooks/useFilter';
import { MetricCard } from '../components/cards/MetricCard';
import { MetricCardSkeleton } from '../components/cards/MetricCardSkeleton';
import { LineChart } from '../components/charts/LineChart';
import { DoughnutChart } from '../components/charts/DoughnutChart';
import { EmptyState } from '../components/common/EmptyState';
import { ErrorState } from '../components/common/ErrorState';
import { Skeleton } from '../components/common/Skeleton';

export function Dashboard() {
  const { files, summary, isAnalyzing, error } = useAnalysis();
  const { fileTypes, setFileTypes } = useFilter();

  // Loading state with 100ms delay to avoid flash
  const [showSkeleton, setShowSkeleton] = React.useState(false);

  React.useEffect(() => {
    if (isAnalyzing) {
      const timer = setTimeout(() => setShowSkeleton(true), 100);
      return () => clearTimeout(timer);
    } else {
      setShowSkeleton(false);
    }
  }, [isAnalyzing]);

  // Error state
  if (error) {
    return (
      <ErrorState
        title="Failed to load dashboard"
        message={error.message}
        action={{
          label: 'Retry',
          onClick: () => window.location.reload(),
        }}
      />
    );
  }

  // Empty state (no files analyzed yet)
  if (!isAnalyzing && files.size === 0) {
    return (
      <EmptyState
        icon="📊"
        title="No analysis data yet"
        message="Run an analysis to see your dashboard metrics"
        action={{
          label: 'Analyze Repository',
          onClick: () => window.viprDesktop.analysis.run({}),
        }}
      />
    );
  }

  return (
    <div className="px-4 sm:px-6 lg:px-8 py-8 w-full max-w-[96rem] mx-auto">
      {/* Header */}
      <div className="sm:flex sm:justify-between sm:items-center mb-8">
        <h1 className="text-2xl md:text-3xl text-slate-800 dark:text-slate-100 font-bold">
          Dashboard
        </h1>
      </div>

      {/* Summary cards */}
      <div className="grid grid-cols-12 gap-6 mb-6">
        {showSkeleton ? (
          <>
            <MetricCardSkeleton className="col-span-12 sm:col-span-6 lg:col-span-3" />
            <MetricCardSkeleton className="col-span-12 sm:col-span-6 lg:col-span-3" />
            <MetricCardSkeleton className="col-span-12 sm:col-span-6 lg:col-span-3" />
            <MetricCardSkeleton className="col-span-12 sm:col-span-6 lg:col-span-3" />
          </>
        ) : (
          <>
            <MetricCard
              title="Health Score"
              value={summary.healthScore}
              trend={summary.healthTrend}
              format="score"
              className="col-span-12 sm:col-span-6 lg:col-span-3"
            />
            <MetricCard
              title="Total Files"
              value={summary.fileCount}
              breakdown={summary.filesByType}
              format="number"
              className="col-span-12 sm:col-span-6 lg:col-span-3"
            />
            <MetricCard
              title="Critical Issues"
              value={summary.criticalCount}
              severity="critical"
              format="number"
              className="col-span-12 sm:col-span-6 lg:col-span-3"
            />
            <MetricCard
              title="Avg Complexity"
              value={summary.avgComplexity}
              trend={summary.complexityTrend}
              format="decimal"
              className="col-span-12 sm:col-span-6 lg:col-span-3"
            />
          </>
        )}
      </div>

      {/* Charts */}
      <div className="grid grid-cols-12 gap-6 mb-6">
        {showSkeleton ? (
          <>
            <Skeleton variant="chart" className="col-span-12 lg:col-span-8" />
            <Skeleton variant="chart" className="col-span-12 lg:col-span-4" />
          </>
        ) : (
          <>
            <div className="col-span-12 lg:col-span-8 bg-white dark:bg-slate-800 rounded-lg border border-slate-200 dark:border-slate-700 p-6">
              <h2 className="text-lg font-semibold mb-4 text-slate-800 dark:text-slate-100">
                Health Over Time
              </h2>
              <LineChart
                data={summary.healthHistory}
                xKey="date"
                yKey="score"
                height={300}
              />
            </div>
            <div className="col-span-12 lg:col-span-4 bg-white dark:bg-slate-800 rounded-lg border border-slate-200 dark:border-slate-700 p-6">
              <h2 className="text-lg font-semibold mb-4 text-slate-800 dark:text-slate-100">
                Complexity Distribution
              </h2>
              <DoughnutChart
                data={summary.complexityByType}
                labelKey="fileType"
                valueKey="avgComplexity"
                height={300}
              />
            </div>
          </>
        )}
      </div>

      {/* Top issues table */}
      <div className="bg-white dark:bg-slate-800 rounded-lg border border-slate-200 dark:border-slate-700">
        <div className="p-6 border-b border-slate-200 dark:border-slate-700">
          <h2 className="text-lg font-semibold text-slate-800 dark:text-slate-100">
            Top Issues
          </h2>
        </div>
        <TopIssuesTable issues={summary.topIssues} />
      </div>
    </div>
  );
}
```

**acceptance criteria:**

- dashboard loads within 500ms after repository open
- summary cards display current metrics with trend indicators
- charts render with consistent styleguide colors and fonts
- responsive layout: 1 column (mobile), 2 columns (tablet), 4 columns (desktop)
- all data fetched via ipc from main process
- filters update dashboard in real-time
- skeleton loaders shown during initial data fetch (after 100ms delay)
- empty state shown when no files analyzed
- error state shown when analysis fails with retry button

#### 3.3 state management with zustand

**SECURITY NOTE:** All stores must validate IPC inputs/outputs using Zod schemas. See section 3.8.1 for complete validation layer details.

**six zustand stores architecture:**

```typescript
// src/renderer/stores/repository.ts (UPDATED with validation)
import { create } from 'zustand';
import { validateIpcResponse, RepoMetadataSchema, sanitizeFilePath } from '../lib/ipc-validator';
import { z } from 'zod';

interface RepositoryStore {
  activeRepoId: string | null;
  metadata: RepoMetadata | null;
  isOpen: boolean;
  error: string | null;

  openRepo: (path: string) => Promise<void>;
  closeRepo: () => Promise<void>;
  getMetadata: () => Promise<RepoMetadata | null>;
}

export const useRepositoryStore = create<RepositoryStore>((set, get) => ({
  activeRepoId: null,
  metadata: null,
  isOpen: false,
  error: null,

  openRepo: async (path: string) => {
    try {
      // SECURITY: Sanitize file path before sending to main process
      const sanitizedPath = sanitizeFilePath(path);

      // Call IPC
      const rawResult = await window.viprDesktop.repo.open({ path: sanitizedPath });

      // SECURITY: Validate response from main process
      const result = validateIpcResponse(
        z.object({
          repoId: z.string().uuid(),
          metadata: RepoMetadataSchema,
        }),
        rawResult
      );

      set({
        activeRepoId: result.repoId,
        metadata: result.metadata,
        isOpen: true,
        error: null,
      });
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Failed to open repository:', errorMessage);
      set({ error: errorMessage });
      throw error;
    }
  },

  closeRepo: async () => {
    try {
      if (get().activeRepoId) {
        await window.viprDesktop.repo.close({ repoId: get().activeRepoId! });
      }
      set({ activeRepoId: null, metadata: null, isOpen: false, error: null });
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Failed to close repository:', errorMessage);
      set({ error: errorMessage });
      throw error;
    }
  },

  getMetadata: async () => {
    if (!get().activeRepoId) return null;

    try {
      const rawMetadata = await window.viprDesktop.repo.getMetadata({
        repoId: get().activeRepoId!,
      });

      // SECURITY: Validate metadata response
      const metadata = validateIpcResponse(RepoMetadataSchema, rawMetadata);
      set({ metadata, error: null });
      return metadata;
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Failed to get metadata:', errorMessage);
      set({ error: errorMessage });
      return null;
    }
  },
}));
```

```typescript
// src/renderer/stores/analysis.ts (UPDATED with event validation)
import { create } from 'zustand';
import {
  validateIpcResponse,
  validateIpcResponseArray,
  FileRecordSchema,
  AnalysisProgressSchema,
  sanitizeFilePath,
  type FileRecord,
  type AnalysisEvent,
} from '../lib/ipc-validator';

interface AnalysisStore {
  files: Map<string, FileRecord>;
  progress: number;
  isAnalyzing: boolean;
  queueSize: number;
  processedCount: number;
  failedCount: number;
  skippedCount: number;
  error: string | null;

  analyzeFiles: (paths?: string[]) => Promise<void>;
  getFile: (filePath: string) => Promise<FileRecord | null>;
  getFiles: (filters?: Record<string, unknown>) => Promise<FileRecord[]>;
  subscribeToEvents: () => () => void;
  reset: () => void;
}

export const useAnalysisStore = create<AnalysisStore>((set, get) => ({
  files: new Map(),
  progress: 0,
  isAnalyzing: false,
  queueSize: 0,
  processedCount: 0,
  failedCount: 0,
  skippedCount: 0,
  error: null,

  analyzeFiles: async paths => {
    set({
      isAnalyzing: true,
      progress: 0,
      processedCount: 0,
      failedCount: 0,
      skippedCount: 0,
      error: null,
    });

    try {
      // SECURITY: Sanitize all file paths
      const sanitizedPaths = paths?.map(sanitizeFilePath);

      await window.viprDesktop.analysis.run({ paths: sanitizedPaths });
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Analysis failed:', errorMessage);
      set({ isAnalyzing: false, progress: 0, error: errorMessage });
      throw error;
    }
  },

  getFile: async filePath => {
    try {
      // SECURITY: Sanitize file path
      const sanitizedPath = sanitizeFilePath(filePath);

      const rawFile = await window.viprDesktop.db.getFile({ filePath: sanitizedPath });

      if (!rawFile) {
        return null;
      }

      // SECURITY: Validate file record from database
      const file = validateIpcResponse(FileRecordSchema, rawFile);

      set(state => ({
        files: new Map(state.files).set(filePath, file),
        error: null,
      }));

      return file;
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Failed to get file:', errorMessage);
      set({ error: errorMessage });
      return null;
    }
  },

  getFiles: async filters => {
    try {
      const rawFiles = await window.viprDesktop.db.getFiles({ filters });

      // SECURITY: Validate array of file records
      const files = validateIpcResponseArray(FileRecordSchema, rawFiles);

      // Update files map
      set(state => {
        const newFilesMap = new Map(state.files);
        files.forEach(file => newFilesMap.set(file.path, file));
        return { files: newFilesMap, error: null };
      });

      return files;
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.error('Failed to get files:', errorMessage);
      set({ error: errorMessage });
      return [];
    }
  },

  subscribeToEvents: () => {
    // SECURITY: Validate all EventBridge events from Phase 2
    const unsubFileStarted = window.viprDesktop.analysis.on('file-started', (rawData: unknown) => {
      try {
        const data = validateIpcResponse(AnalysisProgressSchema, rawData);
        if (data.type === 'file-started') {
          console.log('File analysis started:', data.filePath);
        }
      } catch (error) {
        console.error('Invalid file-started event data:', error);
      }
    });

    const unsubFileCompleted = window.viprDesktop.analysis.on(
      'file-completed',
      (rawData: unknown) => {
        try {
          const data = validateIpcResponse(AnalysisProgressSchema, rawData);
          if (data.type === 'file-completed') {
            set(state => ({
              processedCount: state.processedCount + 1,
              progress:
                state.queueSize > 0 ? ((state.processedCount + 1) / state.queueSize) * 100 : 0,
            }));
          }
        } catch (error) {
          console.error('Invalid file-completed event data:', error);
        }
      }
    );

    const unsubFileFailed = window.viprDesktop.analysis.on('file-failed', (rawData: unknown) => {
      try {
        const data = validateIpcResponse(AnalysisProgressSchema, rawData);
        if (data.type === 'file-failed') {
          console.error('File analysis failed:', data.filePath, data.error);
          set(state => ({
            failedCount: state.failedCount + 1,
            processedCount: state.processedCount + 1,
          }));
        }
      } catch (error) {
        console.error('Invalid file-failed event data:', error);
      }
    });

    const unsubFileSkipped = window.viprDesktop.analysis.on('file-skipped', (rawData: unknown) => {
      try {
        const data = validateIpcResponse(AnalysisProgressSchema, rawData);
        if (data.type === 'file-skipped') {
          set(state => ({
            skippedCount: state.skippedCount + 1,
            processedCount: state.processedCount + 1,
          }));
        }
      } catch (error) {
        console.error('Invalid file-skipped event data:', error);
      }
    });

    const unsubQueueUpdated = window.viprDesktop.analysis.on(
      'queue-updated',
      (rawData: unknown) => {
        try {
          const data = validateIpcResponse(AnalysisProgressSchema, rawData);
          if (data.type === 'queue-updated') {
            set({ queueSize: data.queueSize || 0 });

            // Check if analysis is complete
            const state = get();
            if (state.queueSize === 0 && state.processedCount > 0) {
              set({ isAnalyzing: false, progress: 100 });
            }
          }
        } catch (error) {
          console.error('Invalid queue-updated event data:', error);
        }
      }
    );

    // Return cleanup function with error handling
    return () => {
      try {
        unsubFileStarted();
        unsubFileCompleted();
        unsubFileFailed();
        unsubFileSkipped();
        unsubQueueUpdated();
      } catch (error) {
        // Window may be destroyed, ignore cleanup errors
        console.warn('Event cleanup warning:', error);
      }
    };
  },

  reset: () => {
    set({
      files: new Map(),
      progress: 0,
      isAnalyzing: false,
      queueSize: 0,
      processedCount: 0,
      failedCount: 0,
      skippedCount: 0,
      error: null,
    });
  },
}));
```

```typescript
// src/renderer/stores/filter.ts (UPDATED with sanitization)
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { sanitizeSearchQuery } from '../lib/security-utils';
import type { FileType } from '@vipr/common';

interface FilterStore {
  fileTypes: Set<FileType>;
  severities: string[];
  searchQuery: string;
  directory: string | null;

  setFileTypes: (types: Set<FileType>) => void;
  setSeverities: (severities: string[]) => void;
  setSearchQuery: (query: string) => void;
  setDirectory: (dir: string | null) => void;
  reset: () => void;
}

export const useFilterStore = create<FilterStore>()(
  persist(
    set => ({
      fileTypes: new Set(),
      severities: [],
      searchQuery: '',
      directory: null,

      setFileTypes: types => set({ fileTypes: types }),
      setSeverities: severities => set({ severities }),

      setSearchQuery: query => {
        // SECURITY: Sanitize search query to prevent injection
        const sanitized = sanitizeSearchQuery(query);
        set({ searchQuery: sanitized });
      },

      setDirectory: dir => set({ directory: dir }),

      reset: () =>
        set({
          fileTypes: new Set(),
          severities: [],
          searchQuery: '',
          directory: null,
        }),
    }),
    {
      name: 'vipr-filters',
      storage: {
        getItem: name => {
          const str = localStorage.getItem(name);
          if (!str) return null;
          const parsed = JSON.parse(str);
          return {
            ...parsed,
            state: {
              ...parsed.state,
              fileTypes: new Set(parsed.state.fileTypes),
            },
          };
        },
        setItem: (name, value) => {
          const str = JSON.stringify({
            ...value,
            state: {
              ...value.state,
              fileTypes: Array.from(value.state.fileTypes),
            },
          });
          localStorage.setItem(name, str);
        },
        removeItem: name => localStorage.removeItem(name),
      },
    }
  )
);
```

```typescript
// src/renderer/stores/ui.ts
import { create } from 'zustand';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error' | 'info' | 'warning';
  duration?: number;
}

interface UIStore {
  sidebarOpen: boolean;
  activeModal: string | null;
  toasts: Toast[];
  breadcrumbs: string[];

  toggleSidebar: () => void;
  openModal: (modalId: string) => void;
  closeModal: () => void;
  showToast: (toast: Omit<Toast, 'id'>) => void;
  dismissToast: (id: string) => void;
  setBreadcrumbs: (crumbs: string[]) => void;
}

export const useUIStore = create<UIStore>((set, get) => ({
  sidebarOpen: true,
  activeModal: null,
  toasts: [],
  breadcrumbs: [],

  toggleSidebar: () => set(state => ({ sidebarOpen: !state.sidebarOpen })),

  openModal: modalId => set({ activeModal: modalId }),

  closeModal: () => set({ activeModal: null }),

  showToast: toast => {
    const id = `toast-${Date.now()}`;
    const newToast = { ...toast, id };
    set(state => ({ toasts: [...state.toasts, newToast] }));

    if (toast.duration !== 0) {
      setTimeout(() => {
        get().dismissToast(id);
      }, toast.duration || 3000);
    }
  },

  dismissToast: id => {
    set(state => ({
      toasts: state.toasts.filter(t => t.id !== id),
    }));
  },

  setBreadcrumbs: crumbs => set({ breadcrumbs: crumbs }),
}));
```

```typescript
// src/renderer/stores/settings.ts
import { create } from 'zustand';

interface Settings {
  theme: 'light' | 'dark' | 'system';
  idePreference: 'vscode' | 'cursor' | 'other';
  mcpEnabled: boolean;
  costPerHour: number;
}

interface SettingsStore extends Settings {
  setSetting: <K extends keyof Settings>(key: K, value: Settings[K]) => Promise<void>;
  getAll: () => Promise<Settings>;
  loadSettings: () => Promise<void>;
}

export const useSettingsStore = create<SettingsStore>((set, get) => ({
  theme: 'system',
  idePreference: 'vscode',
  mcpEnabled: false,
  costPerHour: 100,

  setSetting: async (key, value) => {
    await window.viprDesktop.settings.set({ key, value });
    set({ [key]: value } as Partial<SettingsStore>);
  },

  getAll: async () => {
    const settings = await window.viprDesktop.settings.getAll();
    set(settings);
    return settings;
  },

  loadSettings: async () => {
    await get().getAll();
  },
}));
```

**store hooks with selectors to prevent re-renders:**

```typescript
// src/renderer/hooks/useAnalysis.ts
import { useAnalysisStore } from '../stores/analysis';
import { useEffect } from 'react';

// Selector hooks to prevent unnecessary re-renders
export function useAnalysisProgress() {
  return useAnalysisStore(state => state.progress);
}

export function useIsAnalyzing() {
  return useAnalysisStore(state => state.isAnalyzing);
}

export function useAnalysisStats() {
  return useAnalysisStore(state => ({
    queueSize: state.queueSize,
    processedCount: state.processedCount,
    failedCount: state.failedCount,
    skippedCount: state.skippedCount,
  }));
}

// Hook with automatic event subscription
export function useAnalysisWithEvents() {
  const store = useAnalysisStore();

  useEffect(() => {
    // Subscribe to events on mount
    const unsubscribe = store.subscribeToEvents();

    // Cleanup on unmount
    return unsubscribe;
  }, [store]);

  return store;
}
```

**acceptance criteria:**

- all stores use typescript for type safety
- filter store persists to localstorage between sessions
- settings store syncs with sqlite via ipc
- analysis store subscribes to ALL EventBridge events from Phase 2 (file-started, file-completed, file-failed, file-skipped, queue-updated)
- ui store manages transient state (modals, toasts, sidebar)
- no redux or complex middleware
- devtools integration for debugging
- selector hooks prevent unnecessary re-renders
- event subscriptions properly cleaned up on unmount
- error handling in all async store actions
- all IPC inputs/outputs validated with Zod
- file paths sanitized before IPC calls
- search queries sanitized before storage

---

(Continued in next comment due to length limitations)
