# Development Roadmap & To-Do List

## Desktop Application

#### FileDetail.tsx

- Maintainability Index: should be numerical grade (e.g., 39 (D)), not percentage
- Normalize "Fair" vs "Acceptable" terminology across the system
- Complexity tab loader/spinner should display before EmptyState component

### Cleanup and Refactoring

#### P1

- We should be able to take a user to see specific kinds of filters via the URL for deeplinking
- I should be able to take the user directly to a filter showing Circular Dependencies. Right now, there is only per-file discovery. This begs the question: what other metrics are only accessible through drill-down discovery that can be inverted through search, faceting, and filtering? What pages of results and reporting need the ability to show users this?

#### P2

- View rendering and transition animations for all pages, modals, and tabs
- Using React Flow, I want to see a full dependency graph visualization with files containing dependency connections but also shows composite scores and file metadata upon expansion of the sidebar. I should be able to see top issues with a CTA to view all issues, core metrics, and other essential metrics (like what I would see on the Hotspot analysis but with an emphasis on dependencies. Circular dependencies are clearly marked in the React Flow canvas so the user can see it at a birds-eye view). This should be generated in the background after the initial analysis and when the Overview loads. The user can click on a button next to "Browser by Directory" on Files.tsx (secondary button) called "Dependency Graph". It will launch the graph in a modal over the main page, taking up most of the width and height. The header of the modal will hold the stats to the right side of the header but the left side of the close icon that indicates the number of dependencies (upstream, downstream, circular). Not sure if it is possible, but

#### P3

#### NEEDS PRIORITIZATION

- What is the behavior for the Dependency Graph visualizer when there are more then 500 files. Could we show edges that expand over time?
- Is WASM a good candidate for the Dependency Graph Visualizer, to performantly support more than 500 Nodes?
- The dropdowns for comparing branches extend way below the bottom of the screen. They should have a max-height and their own scroll. These should also be compound dropdowns (typeahead dropdowns).
- Creating a budget, users need information to indicate where the thresholds are that they should set. It might be better for us to consider providing a slider instead of a text input so that there is a more clear understanding of whether a metric is good or bad. For example, cyclomatic complexity has a max of 50, but with a text input we don't know that. Being able to provide a range of 0-50 would be helpful. However, there are metrics that, when their thresholds are high, this is a good thing, so we need to also be able to provide a clear user experience. That relay is that these metrics are contextual.
- When creating a budget for a metric, users should be able to explore and find the metric they are setting a budget for. Ideally, a compound dropdown/search with a filter for the different types of analyzers.
- Users should be able to show/hide which columns they want to see on tables
- We should move the issue count column on the Files.tsx view (browse by directory) to be next to the "Name". Order (L-to-R): "Name", "Weighted Score", "Issue Count", "File Count", "Type"
- In place of the project name next to the Zoom Breadcrumb, we should have a dashboard or home icon
- On FileDetail.tsx, we need to see the path to the file underneath the file name
- Alias imports for relative imports inside desktop
- Accept user configuration overrides of CLI and plugin-specific thresholds
- In-repo-based config (vipr.config.json or vipr.config.ts)
- Move desktop eslint config to an `.mjs` file in packages/eslint-config
- Electron Forge audit to ensure desktop implementation conforms to all Electron Forge provides
- Remove deprecated `clients/desktop/src/shared/ipc-types.ts` file
- Offline Google Fonts and remove from CSP headers
- Typescript "types" - should they be co-located from the Desktop application?
- Audit desktop application for type and interface drift from packages/common
- Refactor @vipr/common to separate filesystem and Node.js logic into its own package
- Any way we can use Electron Fuse plugins to enhance or customize the application? https://www.electronjs.org/docs/latest/tutorial/fuses
- Memory leak testing
- Security audit
- Accessibility audit
- Performance profiling (load times, Database → IPC → Renderer → DOM tracing)
- Database file system usage and load testing
- Render time performance analysis
- RUM tracking, error reporting, and user analytics
- Generated docs: flatten or convert markdown to TypeScript only?

### UI Polish

#### Overview & Initial Assessment

- Improve tooltip labels for metrics
- Complexity Distribution chart: improve labeling (e.g., "84 files are Very Complex (20+)")
- Issues.tsx: ensure pill tabs' numeric values have consistent styling
- `toLocaleString()` for "Issue Distribution" chart hover labels
- Churn Analysis view: clearer default state (not just "No Results")
- Velocity Trends view: clearer default state (not just "No Results")
- Identify usage of `_` to indicate unused code and determine if feature or algorithmic gaps are missing as a result. If not, remove or bypass linter.
- Remove the `__fixtures__` directory in lieu of `fixtures`. Same with `__mocks__`.

### Feature Development

#### Overview & Initial Assessment

- Shepard.js for walkthroughs
- Expand Initial Assessment Modal to be more informative
- Enable pagination for Top 5 Hotspots
- Increase value of "Action Plan" pane (add to-do list feature?)

### Testing

- Consider just isolating tests in `clients/desktop/tests/(unit|integration|acceptance)` instead of co-locating them (since desktop is so large).
- We need click-thru Playwright testing for Electron.
- We need integration tests between packages (unless suggested otherwise)
- We need a loop to test the integrity of metric evaluations through the system to improve the algorithms.

=== BELOW NEEDS TO BE CLEANED UP ===

## VSCode Extension

### Core Features

- Test new async file access changes
- QA MITL engine review
- Increased test coverage (Unit and Integration)
- Explore E2E testing for VSCode extension (e.g., Playwright)
- Refactoring and performance pass
- Reduce and unify docs into a coherent tree
- Audit settings: identify coherent strategy, remove overlaps, align with viper.config.\* approach

### Sidebar & File Analysis

- Sidebar pane: show loader and text when analyzing file via "Analyze File..."
- When starting debugger, reliably show Vipr initialization in status bar footer
- Replace "eye" icon with "V" icon in workbench sidebar
- Set min-width for sidebar pane that the IDE respects (not just CSS)
- When opening Vipr tab and clicking "Analyze Workspace", opening file from analyzed list should:
  - Open analysis for that file
  - Show entire list of problems in bottom Problems pane
  - Show file indicators in gutter

### Analysis Behavior

- Workspace analysis behavior on initialization:
  - If auto-analyze enabled: run on init
  - If disabled: don't analyze, just initialize; clicking status bar button analyzes current file if one is active
- File caching: if file analyzed and unchanged since, show cached analysis immediately on click
- On file save: create new analysis snapshot and update metrics
- When viewing analysis pane, all problems should have "Fix" button to generate AI prompt

### Documentation & Help

- Ensure all links in docs modals are valid
- Ensure links open browser, not blank dashboard sidebar
- Fix empty third columns in `useCallback`, `useMemo`, and `React.memo` tables
- Revise AI prompts: not "Refactor this" but "Analyze, provide examples why metric is elevated, identify improvements"

### Features with LM API

- BIG FEATURE: If VSCode LM API available, provide "Analyze this metric with Co-pilot..."
  - Provide file code + metric-specific prompt
  - Render suggested result in modal chat
  - Allow iterative fixes and metric improvement tracking
  - Re-compute analysis on file change, showcase metric delta

### Diagnostics & Reporting

- Migrate system to use VSCode "DiagnosticSeverity" type
- Feature: Separate analysis button for each metric opening LM modal overlay
- Copy to Markdown feature for action menu on reports
- React Anti-patterns tab: review coherence (values seem mismatched to score)

### Known Issues

- Sidebar pane: "Analyzed Files" list inconsistently shows loaded state

---

## CLI

### Core Features & Fixes

- `--fail-on-critical` implementation
- `--no-cache` implementation
- `-o` output: show visual result (not raw ANSI codes); explore markdown/HTML generator
- `-c` (categories) flag: only show selected categories, not whole report
- `-s` (severity) filtering: show only matching severity with consola box + check icon when no results
- `-r` (report) stacking: allow multiple sections (e.g., `-r performance,accessibility`)

### Report Formatting

- `-f json-compact` should be unminified version; `-f json` should be minified
- `-f json` should always include essential complexity metrics (cyclomatic, halstead, LOC)
- When no results for `-s` severity: show message box with check mark icon
- Clicking bar in "Overall" sections should navigate to that report tab
- Fix: `-p` plugin flag only shows header, not overview section with metrics and bar charts

### Documentation & Interactive Mode

- Feature: When viewing analysis pane of file, all problems show CodeLens, squiggly lines, gutter icons
  - Reactive to section changes in side pane
  - Slice file by report results for different views
- Migrate to Ink (React CLI) or Bubbletea (Go-based CLI) for stunning CLI UX
  - Multi-tab report generation and navigation
  - ink-spinner, ink-big-text, ink-divider, ink-gradient, ink-link, ink-chart
- Feature: Combine repetitive report results (e.g., inline function props at multiple lines)
  - Show as grouped result instead of individual listings

### Known Issues

- Vite chunk warning: some chunks >500kB after minification
  - Consider: dynamic import(), rollupOptions.output.manualChunks, chunkSizeWarningLimit
- Dynamic import conflict: prompt-utils.ts dynamically and statically imported
- Radar chart: missing values for Accessibility, Testability, Security

---

## Architecture & Refactoring

### Code Organization

- Duplicate ts-morph assets in vscode-extension should be deduped
- Convert monorepo to ESM entirely
- Remove V2 build system
- Files in `common/src/types` contain functions: move to `common/src/utils`
- `cli-formatter.ts` has React-specific logic: should come from React analyzer
- Review barrel exports in analyzers

### Performance & Optimization

- There are double caches: VSCode extension and Engine - evaluate if Engine cache is needed
- Investigate slow tests:
  - src/formatters/cli-formatter-integration.test.ts (25) 42810ms
  - src/utils/react-helpers.test.ts (35) 1232ms

### Testing & Quality

- Deep algorithmic analysis to verify accuracy with research-backed algorithms
- Battle-test approaches with comprehensive test suites
- Write tests for CLI
- Fix skipped test: src/commands/analyze-command.test.ts
- Increased test coverage (Unit and Integration)

---

## Backlog - Future Features

### Analyzers (Expand Coverage)

- Create analyzers based on popular technologies (per Stack Overflow survey):
  - Next.js
  - Node.js specific
  - jQuery
  - Express
  - Vue.js
  - Svelte
  - Ember.js
  - HTML/CSS

### Platform & DevOps

- Free version vs licensing strategy
- Licensing server implementation
- CI/CD and Github Actions flow
- Changelog automation for semver and version incrementing
- Security/penetration testing
- Auto-updating feature for installed machines
- Analytics research and implementation feasibility

### Marketplace & Distribution

- Marketing website
- Marketing research: pricing, user base, promotion, distribution
- PR/Release strategy: PR, social media posts, blogs, Discord communities
- App store submission: research requirements for various app stores

### Features & Ideas

- Feedback reporting tool
- PDF generation for reports
- Markdown export for reports
- JIRA integration: create tickets with technical details
- Smart tasks: magic button generates tasks for issues (per-file or per-issue)
- Snapshots: long-term analysis trends (prune unnecessary data to reduce storage)
  - Explore pruning: descriptions, prompt info (keep only raw metric data)
- Database improvements: explore info we can prune from SQLite to store longer trends

### Git Integration

- CLI git-bisect hotspot analysis
- Worktree support for analysis
- Branch-aware analysis
- Resilient change detection

### Advanced Analysis

- Dependency graph visualization
- Cross-file dependency analysis
- API surface analysis
- Import cycle detection and visualization

---

## Completed ✅

### FunctionDetail.tsx

- In the first badge, discern between function expression, function declaration, and named function export
- Add info icon in "Complexity Score" header with hover tooltip explaining composition
- Resize the four metrics next to the radial gauge (currently too large)
- Copy function name: show icon to right of function name instead of menu item
- Function Signature should collapse by default (or consider removing in favor of syntax-highlighted source)
- Increase width of search input for issues (or expand on focus)
- Add "Copy AI Prompt" option to individual issues in the issues table
- Replace line number column in issues table with more meaningful metadata
- Deep linking: "View in file context" should link to function in file context, not just file
- Normalize severity column placement across all issue tables (leftmost/first column)

### 0.13.0 - Next.js Analyzer

- Draft complete phase documentation (Phases 12-14)
- Implementation (Phases 1-14)
- Move markdown files out of analyzer into documentation tree
- Ensure subagents for VSCode accommodate new CLI architecture
- Feature: In-IDE prompts generated from "Fix" button

### 0.8.0 - Post-CLI Cleanup

- Grade boundaries: switched to 10-point scale (9.5/10)
- Removed `__snapshots__` underscore syntax
- Analyzer plugins provide thresholds via configuration
- Investigated slow tests
- Manual code review and TODO sweep
- Evaluated `packages/testing` necessity
- Added linting for all eligible packages
- Added descriptions for reporting sub-sections
- Removed `@deprecated` items
- Removed unused properties prefixed with `_`
- CLI test coverage
- TypeScript audit for shared types
- Linked file paths in overview header to file open action
- Replaced `makeLink` with `linkToFile` utility with proper colorization
- Moved functions from `common/src/types` to `common/src/utils`
- Separated React-specific logic in cli-formatter
- Markdown export functionality
- Vipr.config.json/ts support with thresholds and plugin configs
- Custom color library (utils/colors.ts)

### 0.7.0 - CLI Polish

- Verified `--help` flags work as intended
- JSON output with stripped and standard schema versions
- A11y spacing improvements for bar charts
- Architecture review and refactoring
- Feature: `--interactive` mode with clack for report selection, formatting, clipboard, export
- Aliases for relative imports inside packages
- Deep algorithmic analysis verification with tests

### 0.6.0 - CLI Port & Cleanup

- CLI UX design
- JSON output for all flags with full metrics
- ASCII logo rendering
- Report formatting consistency (visual)
- Decouple formatters from CLI
- Individual analysis flag verification
- `--help` flag accuracy
- Logo version number increment automation

### 0.5.0 - Analyzer Polish

- Added sample components for each analyzer
- Unit, integration, and benchmark testing
- Analyzer cleanup and refinement (React in particular)
- Co-located weights and constants
- Optimized aggregateResults function
- Node + TypeScript code review
- Evaluated formatter abstraction in plugins
- Removed analyzeTraditional
- Removed barrel exports in analyzers

### 0.4.0 - Turbo Monorepo Port

- Ported all analyzers (Performance, Reliability, Technical Debt, Type)
- SME review and refinement of analyzer measurements

---

## Known Bugs & Issues

### Development Environment

- [x] After dev mode start, opening FileDetail.tsx or navigating between pages causes `✨ optimized dependencies changed. reloading`

### Analysis & Caching

- Initial analysis of workspace on startup should be cached, not re-analyzed on next startup
- Radar chart missing values for Accessibility, Testability, Security metrics

### Database & Performance

- Double caches (VSCode extension + Engine) causing brittle results with clients

---

## Documentation & Configuration

### Needed Documentation

- Full architecture diagrams
- Plugin system documentation
- Configuration guide for vipr.config.\*
- CLI command reference
- VSCode extension configuration guide
- Desktop application user guide

### Settings & Configuration

- Audit and consolidate all settings
- Align with viper.config.\* approach
- Ensure synergy across desktop app, MCP server, and CLI
