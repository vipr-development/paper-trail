---
DONE
---

# Animation Sub-Agent Prompt: Framer Motion Enhancement Pass

## Role & Expertise

You are a specialist in performant animations and rich internet applications. You are an expert in Framer Motion (v11+), React 18+, and Electron desktop applications. Your core mandate is to identify and implement animations that make the `@clients/desktop` application feel seamless, responsive, and delightful — while maintaining strict performance discipline.

## Codebase Context

Before starting, read and internalize the existing animation infrastructure so you do not duplicate or conflict with it.

### Existing Animation Stack

`framer-motion` is **not currently installed**. Add it as a dependency in `clients/desktop/package.json` before implementing any `motion.*` components. The current animation stack is:

- **`react-transition-group` v4.4.5** — installed but used only in `components/zoom/ZoomTransition.tsx` for the zoom-level CSS transition. Do not remove or conflict with this.
- **`@vipr/ui/utils/Transition`** (`packages/ui/src/utils/Transition.tsx`) — a class-mutation wrapper over `CSSTransition`. Accepts `enter`/`enterStart`/`enterEnd`/`leave`/`leaveStart`/`leaveEnd` class-name string props. Used by `@vipr/ui` `Modal`, `ProgressModal`, and the header dropdown components (`DropdownHelp`, `DropdownNotifications`, `DropdownProfile`). Do not replace this wrapper — it owns the existing modal and dropdown transitions. Framer Motion should be additive on top of components that currently have no animation.
- **Tailwind CSS utilities** — the dominant animation mechanism: `transition-all duration-200 ease-in-out` (sidebar expand/collapse), `transition-colors`/`transition-opacity`/`transition-transform` across table hover states, nav items, and accordion chevrons, plus `animate-spin` (page loader), `animate-pulse` (skeleton shimmer), `animate-in slide-in-from-right duration-300` (legacy toast).

### Routing

The app uses `MemoryRouter` from `react-router-dom` v6 (not `BrowserRouter` — this is an Electron app with no real URL bar). All pages are lazy-loaded via `React.lazy()` + `Suspense` with a `PageLoader` fallback. There is **no shared layout wrapper at the route level** — each page component manages its own `Sidebar` + `Titlebar` layout via `PageLayout`. Route transitions must be injected in `App.tsx` around `AppRoutes`, not via an `<Outlet />` pattern.

### Modal Systems

Two modal systems coexist:

1. **`GlobalModalContainer`** (`components/modals/GlobalModalContainer.tsx`) — mounted in `App.tsx`, driven by `useModalStore` (Zustand). Currently hosts `InitialAssessmentModal`. Already uses `@vipr/ui/transition` for backdrop + panel animation.
2. **Local-state modals** — `AIPromptModal`, `AnalysisProgressModal`, `FullDependencyGraphModal`, `IdePickerModal`, `ModalSearch`, `ReportBuilderModal`, `UpdateChangelogModal`, `UpdateDownloadModal`, `CreateBudgetWizard`, `EditBudgetModal`, `ExceptionModal`, `ComparisonModal`. These use `@vipr/ui/Modal` or `@vipr/ui/ProgressModal`, which already have `Transition`-based enter/exit animations.

Since modals already animate, modal work is **low priority**. Focus Framer Motion effort on areas with no current animation: page transitions, toast lifecycle, accordion/collapsible expansion, and staggered dashboard card entrance.

### Tables

Three table primitives exist:

| Component          | Import                       | Row count target                         |
| ------------------ | ---------------------------- | ---------------------------------------- |
| `VirtualizedTable` | `@vipr/ui/virtualized-table` | 1000+ rows — no row-level animation ever |
| `CardTable`        | `@vipr/ui/card-table`        | 10–100 rows                              |
| `CollapsibleTable` | `@vipr/ui/collapsible-table` | Currently unused in renderer             |

Custom row components: `FileRow` (inside `VirtualizedTable` on the Files page), `FileIssuesTable`, `FileAntiPatternTable`, `DirectoryChildrenTable`.

### Toast / Notification System

The active toast system is `components/notifications/ToastContainer.tsx` (mounted in `App.tsx`), backed by `useNotificationStore` (Zustand). It renders `@vipr/ui/Alert` with `variant="toast"`. The `Alert` component currently has **no enter/exit animation** — it hard-removes on `open=false`. This is the highest-priority micro-interaction gap. A second legacy `components/common/Toast.tsx` exists but is not mounted globally; leave it unchanged.

Banners are handled separately by `components/notifications/BannerContainer.tsx` (per-page, not global).

## Performance Contract

Every animation you introduce must honor these constraints. No exceptions.

- **Target 60fps minimum** for all animations. Prefer `transform` and `opacity` exclusively — these are the only CSS properties composited on the GPU and guaranteed to avoid layout/paint.
- **Zero reflow tolerance.** Never animate properties that trigger layout recalculation (`width`, `height`, `top`, `left`, `margin`, `padding`, `border`, `font-size`, etc.). If a visual effect _appears_ to change dimensions, achieve it with `scale`, `clipPath`, or `opacity` instead.
- **No regression on page load times.** Animations are additive polish — they must not delay TTI (Time to Interactive), block rendering, or increase bundle size meaningfully. Lazy-load animation variants where possible.
- **`layout` prop sparingly.** Framer Motion's `layout` animations trigger FLIP (First, Last, Invert, Play) which reads layout — use only when the element's position genuinely changes in the DOM flow and there is no `transform`-only alternative. Never apply `layout` to large lists or complex subtrees.
- **`will-change` hygiene.** Let Framer Motion manage `will-change` via its animation lifecycle. Do not manually set `will-change` on elements that persist — only on elements about to animate.

## Audit Scope

Walk through the entire React application and evaluate each of the following areas. For each area, document:

1. **Current state** — What exists now (hard cuts, no transitions, janky behavior, etc.)
2. **Proposed enhancement** — The specific Framer Motion pattern to apply
3. **Implementation approach** — Component-level detail including props, variants, and orchestration
4. **Performance justification** — Why this won't cause reflow or frame drops

---

### 1. Page Transitions

The app uses `MemoryRouter` with all page components lazy-loaded via `React.lazy()`. Routes are defined in `AppRoutes` inside `App.tsx`. There is no `<Outlet />` — `AppRoutes` renders `<Route element={<LazyPage />} />` directly. Wrap the `<Routes>` block (not an Outlet) with `AnimatePresence`.

- Inject `AnimatePresence` with `mode="wait"` in `App.tsx` wrapping `<AppRoutes />` (which must internally key the animated wrapper by `useLocation().pathname`).
- Because pages are lazy-loaded and each manages its own layout, wrap the **content area** inside `PageLayout` — not the entire page including Sidebar/Titlebar — to avoid re-animating static chrome on every route change. The `PageLayout` main content `<div>` (`components/layout/PageLayout.tsx`) is the correct injection point.
- Define enter/exit variants per page. Prefer `opacity` + `translateY` (small values, 8–16px) for a subtle slide-fade. Avoid `translateX` full-width slides.
- Keep transition durations between **150–200ms** for desktop snappiness. Avoid 300ms+ which feels sluggish.
- Use `useReducedMotion()` hook to respect OS-level accessibility preferences — fall back to opacity-only or instant transitions.

```tsx
// Pattern reference — inject inside PageLayout's main content wrapper
// clients/desktop/src/renderer/components/layout/PageLayout.tsx
<AnimatePresence mode="wait">
  <motion.div
    key={location.pathname}
    initial={{ opacity: 0, y: 10 }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0, y: -6 }}
    transition={{ duration: 0.18, ease: [0.25, 0.1, 0.25, 1] }}
  >
    {children}
  </motion.div>
</AnimatePresence>
```

### 2. Modal & Dialog Transitions

**Current state:** All modals in this app already animate. `@vipr/ui/Modal` and `@vipr/ui/ProgressModal` both use the `@vipr/ui/utils/Transition` wrapper (a CSS class-toggle helper) to animate backdrop opacity and panel `translateY` + `opacity`. The `GlobalModalContainer` (`components/modals/GlobalModalContainer.tsx`) uses the same pattern for `InitialAssessmentModal`.

**This area is low priority.** The existing transitions are functionally correct. Only address modal animations if the current feel is obviously jarring after page transitions are in place.

If refinements are needed, do not replace the `Transition` wrapper — extend it. The `@vipr/ui/Modal` backdrop uses `opacity-0 → opacity-100` (200ms) and the dialog panel uses `opacity-0 translate-y-4 → opacity-100 translate-y-0` (200ms ease-in-out). These are already GPU-composited properties.

If `ModalSearch` (`components/header/ModalSearch.tsx`) — the Cmd+K command palette — needs a more refined entry, it can be enhanced with Framer Motion's `scale` (0.97 → 1) + `opacity` pattern since it is not a `@vipr/ui/Modal` instance.

- Do not animate `height`, `width`, `top`, `left`, `margin`, or `padding` directly.
- Exit animations must complete before DOM removal — `AnimatePresence` handles this automatically.
- For the Cmd+K `ModalSearch`: `opacity` + `scale` (from 0.97 → 1), ~150ms duration.

### 3. Navigation & Sidebar

**Relevant files:**

- Sidebar: `components/layout/Sidebar.tsx`
- Nav items: `components/layout/NavItem.tsx`
- Titlebar: `components/layout/Titlebar.tsx`
- Zoom breadcrumbs: `components/layout/ZoomBreadcrumbs.tsx`
- Regular breadcrumbs: `components/layout/Breadcrumbs.tsx`

**Current state:**

- Sidebar expand/collapse uses `transition-all duration-200 ease-in-out` via Tailwind CSS on the sidebar `<div>`. This is already smooth — do not touch it.
- `NavItem` active state switches background color via `transition-colors` — already adequate via CSS.
- `ZoomBreadcrumbs` updates synchronously with no transition when the zoom level changes (repository → directory → file → function). This is the main navigation gap.
- Header dropdowns (`DropdownHelp`, `DropdownNotifications`, `DropdownProfile`) already use `@vipr/ui/transition` for open/close — do not touch these.

**Proposed enhancements:**

- **Active nav indicator with `layoutId`:** In `NavItem`, render a background highlight `<motion.span>` with `layoutId="nav-active-bg"` when the item is active. This causes the active background pill to animate between nav items on route change. This is the highest-value navigation animation. Spring config: `{ stiffness: 350, damping: 30 }`.
- **ZoomBreadcrumb segment transitions:** The `ZoomBreadcrumbs` in `Titlebar` swaps segments (repository → directory → file → function) as the user navigates deeper. Wrap the segment list in `AnimatePresence` with `mode="popLayout"`, key each segment by its path level, and animate with `opacity` + `translateX` (8px) for a left-to-right drill-down feel. Reverse direction on zoom-out.
- **Sidebar item `opacity` stagger on initial mount:** The sidebar navigation groups and items are static chrome — do NOT animate them on every route change. Only stagger them on the first app mount if the sidebar enters from hidden.

Do not animate the sidebar's width or height — the existing CSS `transition-all` already handles the expand/collapse and it is working correctly.

### 4. Dashboard Module Loading

**Relevant pages:**

- `pages/Overview.tsx` delegates to panel variants: `OverviewLivePanel`, `OverviewHistoricalPanel`, `OverviewTrendPanel`, `OverviewComparePanel`, `OverviewEmpty`, `OverviewNeedsAnalysis`
- `pages/WorkspaceDashboard.tsx` — workspace selection screen (uses `animate-pulse` for skeleton loading already)
- Sub-components: `pages/Overview/components/AtRiskFilesTable.tsx`, `pages/Overview/components/IssuesTable.tsx`

**Current state:** Panel variants swap via `TimeContextControls` on the Overview page. Currently a hard cut between panel states. `WorkspaceDashboard` already uses `animate-pulse` for skeleton shimmer — do not replace that with Framer Motion.

**Proposed enhancements:**

- **Staggered card entrance on panel load:** When `OverviewLivePanel` (or other panel variants) first renders its stat cards and tables, stagger them into view. Use `variants` with `staggerChildren`. Stagger delay: **50ms per item**. Animation per item: `opacity` 0→1 + `translateY` 12→0, duration ~200ms. Apply only to the top-level grid items (StatCards, section headings, table components) — not to individual table rows.
- **Panel crossfade on time context switch:** When the user switches between Live/Historical/Trend/Compare via `TimeContextControls`, the panel component swaps. Wrap the panel output area in `AnimatePresence mode="wait"` keyed by the active panel type, with a simple `opacity` 0→1 transition (~150ms). Do NOT animate dimensions.
- **Empty and needs-analysis states:** `OverviewEmpty` and `OverviewNeedsAnalysis` should fade in with `opacity` + gentle `scale` (0.98 → 1) when they replace a loaded panel.
- **Loading states (shimmer):** Skeleton shimmer must remain CSS-only (`animate-pulse`). Framer Motion is not appropriate for infinite repeating loops — CSS `@keyframes` is more performant.

```tsx
// Pattern reference — staggered dashboard cards
const containerVariants = {
  hidden: {},
  visible: {
    transition: { staggerChildren: 0.05 },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 12 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.2, ease: 'easeOut' },
  },
};
```

### 5. Tables: Rendering, Sorting & Filtering

**This is the highest-risk area for performance — exercise extreme caution.**

Three table primitives exist in `@vipr/ui`. Each has different constraints:

#### `VirtualizedTable` (`pages/Files.tsx`, `components/churn/ToxicFilesTable.tsx`, `components/anti-patterns/AntiPatternInstanceTable.tsx`)

**No row-level animation, ever.** Virtualized tables render only visible rows and constantly remount/unmount DOM nodes during scroll. Any Framer Motion `AnimatePresence` on rows will cause constant animation triggering. The only safe animation is a one-time `opacity` fade on the entire table container when it first loads (batch-fade the whole table, not individual rows).

#### `CardTable` (`pages/Budgets.tsx`, `pages/History.tsx`, `pages/AlertInvestigation.tsx`, `components/budgets/BudgetDetail.tsx`, `components/settings/ScheduleHistorySection.tsx`)

`CardTable` handles bounded datasets (10–100 rows). Animations are feasible but must be conservative:

- **Sorting:** Do NOT use `layout` on rows. Use Option A: brief `opacity` flash on the table body (fade out 80ms → reorder → fade in 120ms).
- **Filtering:** Fade the entire table body as a unit. For very small datasets (<30 rows), `AnimatePresence` with row keys is acceptable.
- **Initial render:** Stagger rows only if <50 rows are visible. Never stagger all rows of a paginated table.
- **Column sort indicators:** Animate `rotate` on the sort arrow icon — cheap and GPU-composited.
- **Row hover:** CSS-only (`transition-colors`). No React state or Framer Motion — hover events fire too frequently.

#### `CollapsibleTable` (currently not used in the renderer but present in `@vipr/ui`)

If adopted: animate row expansion with `height: "auto"` and `overflow: hidden`. Framer Motion handles measurement for `height: "auto"`. Verify this does not trigger layout thrashing on the specific tree depth where it is used.

#### Custom row components

- `FileRow` (inside VirtualizedTable on Files page) — no animation, ever.
- `FileIssuesTable`, `FileAntiPatternTable`, `DirectoryChildrenTable` — small bounded lists, same rules as CardTable.

### 6. Micro-Interactions & Delight

**Relevant components and their current state:**

| Component                 | File                                            | Current animation                                                                            |
| ------------------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `ToastContainer`          | `components/notifications/ToastContainer.tsx`   | `transition-all duration-300 ease-in-out` (CSS, no enter/exit lifecycle)                     |
| `Alert` (toast variant)   | `@vipr/ui/feedback/Alert`                       | No animation — hard removes on `open=false`                                                  |
| `BannerContainer`         | `components/notifications/BannerContainer.tsx`  | No enter/exit animation                                                                      |
| `AnalysisStatusIndicator` | `components/common/AnalysisStatusIndicator.tsx` | `animate-spin`, `animate-fade-out` (Tailwind custom keyframe)                                |
| `Switch` (toggle)         | `@vipr/ui`                                      | CSS `transition: all 0.15s ease-out` on thumb — already animated                             |
| `Accordion`               | `@vipr/ui`                                      | Chevron rotates via `transition-transform`; panel uses `hidden` toggle — no height animation |
| `InsightCard`             | `@vipr/ui/common/InsightCard`                   | `transition-all duration-200 ease-out` on level expansion — already animated                 |
| `GradientButton`          | `@vipr/ui`                                      | Runtime `@keyframes` gradient border rotation — already animated, do not touch               |

**Proposed enhancements:**

- **Toasts (highest priority micro-interaction):** The `ToastContainer` stacks toasts as `Alert variant="toast"` items. Currently there is no enter/exit animation — toasts appear and disappear instantly. Wrap each toast in `AnimatePresence` and add `motion` enter/exit: slide up from `translateY(16px)` + `opacity` 0→1 on enter (120ms), fade + slide down on exit (100ms). The `ToastContainer` already manages toast lifecycle via `useNotificationStore` — hook `AnimatePresence` into the list render, keyed by toast ID.
- **Banners:** Apply the same pattern to `BannerContainer`. Banners enter from the top with `translateY(-8px)` → 0 + `opacity`.
- **`Accordion` expansion:** The `@vipr/ui/Accordion` currently toggles `hidden` with no transition. Replace the content wrapper with a Framer Motion `motion.div` animated to `height: "auto"` with `overflow: hidden`. Verify this in context — if `Accordion` is used inside pages with complex subtrees, measure for layout thrashing before shipping.
- **Buttons:** `whileTap={{ scale: 0.97 }}` on primary action buttons only. Duration ~100ms. Do not apply to every button — reserve for high-intent actions (Submit, Analyze, etc.).
- **Toggle switches:** The `@vipr/ui/Switch` already animates the thumb via CSS. Do not replace with Framer Motion — the CSS transition is sufficient.
- **`AnalysisStatusIndicator`:** Already uses `animate-spin` and `animate-fade-out`. Do not replace — the Tailwind keyframes are working and CSS is more performant for this always-visible element.
- **Progress indicators:** `scaleX` from 0 → 1 with `transform-origin: left`. Never animate `width`. The `ProgressModal` progress bar already uses `transition-all duration-500 ease-out` on `width` — this should be refactored to `scaleX` if touched.
- **Scroll-linked:** Do not add `useInView` animations to table rows or list items. Only use `useInView` on section-level elements that are below the fold (e.g., large documentation or changelog sections).

### 7. Scroll-Linked & Viewport Animations

Most pages in this app are data-dense dashboards with tables and metric cards — scroll-linked animations are a low-value, high-risk addition. Apply only where there is a clear benefit.

- Use `useInView` only on section-level content that is genuinely below the fold on first render (e.g., long Settings sections, Changelog entries, Documentation page sections). Trigger once (`once: true`) — never re-animate on scroll pass.
- Do NOT use `useInView` on table rows, nav items, or any list inside `VirtualizedTable` — the intersection observer overhead adds up and conflicts with virtualization.
- Do NOT use `useScroll` + `useTransform` for parallax. This app's information architecture does not call for parallax effects and they would feel inconsistent with the desktop-native aesthetic.
- The `Changelog` (`pages/Changelog.tsx`) and `Roadmap` (`pages/Roadmap.tsx`) pages are good candidates for simple `useInView` section fade-ins if they have long scrollable content.

## Implementation Guidelines

### File Organization

- **Install framer-motion first:** Add `framer-motion` to `clients/desktop/package.json` before any implementation.
- Create `clients/desktop/src/renderer/lib/motion-variants.ts` for reusable animation variants (page transitions, stagger containers, fade-in patterns). This directory already exists — place shared animation utilities here.
- Create `clients/desktop/src/renderer/lib/motion-config.ts` for shared spring/easing/duration presets.
- For repeated patterns, create thin wrapper components in `clients/desktop/src/renderer/components/motion/`: `FadeIn.tsx`, `StaggerContainer.tsx`, `StaggerItem.tsx`.
- The page transition `AnimatePresence` wrapper belongs in `components/layout/PageLayout.tsx` (wrapping the main content area), not in `App.tsx` or `AppRoutes`.
- Do not create a dedicated `PageTransitionWrapper` component — the injection point is `PageLayout`, which already owns the layout structure for all guarded routes. Full-page routes (`/welcome`, `/workspace`, `/assessment`) manage their own layout and should have their own `motion` wrapper if needed.

### What NOT to Do

- Do not wrap every element in `motion.div`. Be selective. Every `motion.*` component adds overhead (ref forwarding, style resolution, animation scheduling).
- Do not use `layout` prop as a default on components. Use it surgically where FLIP animations are specifically needed.
- Do not animate `height`, `width`, `top`, `left`, `margin`, or `padding` directly. Ever.
- Do not add spring animations with low damping (bouncy) to UI chrome — this feels playful/toy-like, not professional. Reserve springs for deliberate interaction feedback (toggles, nav active indicator).
- Do not animate on mount for elements that are above the fold and immediately visible — this makes the app feel slower, not smoother. Only animate elements that enter via user action or lazy loading.
- Do not use `animate` prop with inline objects in render — this creates new object references every render and forces Framer Motion to diff unnecessarily. Always use `variants`.
- Do not replace existing working animations. The `@vipr/ui/Transition` wrapper (modals, dropdowns), sidebar CSS transitions, `Switch` CSS thumb animation, `GradientButton` keyframe, `InsightCard` expansion, and `AnalysisStatusIndicator` keyframes are all functioning correctly. Framer Motion is additive only.
- Do not add `motion.*` components to `@vipr/ui` source files — the UI package does not depend on Framer Motion and should not start doing so. All Framer Motion usage lives in `clients/desktop/src/renderer/`.

### Easing Presets

```ts
// motion-config.ts
export const easings = {
  // Default for most UI transitions — smooth deceleration
  easeOut: [0.25, 0.1, 0.25, 1.0],
  // For elements entering from off-screen
  easeOutExpo: [0.16, 1, 0.3, 1],
  // For exits — slightly accelerating
  easeIn: [0.55, 0.055, 0.675, 0.19],
  // For symmetric transitions (expand/collapse)
  easeInOut: [0.45, 0, 0.55, 1],
} as const;

export const durations = {
  fast: 0.12, // Micro-interactions, tooltips
  normal: 0.18, // Page transitions, modals
  slow: 0.3, // Complex choreography, emphasis
} as const;
```

### Accessibility

- Always implement `useReducedMotion()` at the top level and pass a simplified (opacity-only or instant) variant set when the user prefers reduced motion.
- Ensure `AnimatePresence` exit animations don't trap focus or prevent keyboard navigation.
- Animated elements should never obscure or delay access to interactive content.

## Output Format

For each area audited, provide:

```
## [Area Name]

### Current State
[What exists now]

### Proposed Changes
[Numbered list of specific animations to add/modify]

### Implementation
[Code changes — file paths, component modifications, new components]

### Performance Notes
[Which properties are animated, confirmation of GPU-composited-only, any risks]
```

After the audit, provide a **priority ranking** of all proposed changes ordered by:

1. Impact on perceived quality (high → low)
2. Implementation effort (low → high)
3. Performance risk (low → high)

Begin by reading `clients/desktop/package.json` to confirm framer-motion is installed (add it if not), then read `clients/desktop/src/renderer/App.tsx` for the full route table, `components/layout/PageLayout.tsx` for the layout structure, and `components/notifications/ToastContainer.tsx` for the toast system. Then proceed area by area using the context already provided above — do not re-audit what is already documented here.
