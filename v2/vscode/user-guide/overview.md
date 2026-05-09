# Features Overview

The Vipr VSCode Extension provides comprehensive code quality analysis for React and TypeScript applications, delivering real-time feedback, diagnostics, and AI-powered insights directly in your editor.

## Core Capabilities

### Real-Time Analysis

Analyze your codebase with powerful static analysis that detects:

- **Security vulnerabilities** - XSS risks, dangerous patterns, hardcoded secrets
- **Accessibility issues** - ARIA violations, keyboard navigation problems, missing labels
- **Performance concerns** - Inefficient rendering, missing memoization, expensive operations
- **Code complexity** - Cyclomatic complexity, maintainability index, Halstead metrics
- **React best practices** - Hook violations, anti-patterns, migration issues
- **Data flow problems** - Prop drilling, state management issues, unnecessary re-renders

### Visual Feedback

#### Problems Panel Integration

Issues appear automatically in VSCode's Problems panel with:

- Severity-based filtering (Error, Warning, Info)
- File grouping and quick navigation
- Clickable links to issue locations
- Detailed descriptions and suggestions

#### Inline Diagnostics

See issues directly in your code with:

- Colored squiggly underlines (red for errors, yellow for warnings, blue for info)
- Hover tooltips showing full issue details
- Suggested fixes and explanations
- Related information links

#### CodeLens Hints

Get at-a-glance insights with inline hints showing:

- Component quality scores above React components
- File-level metrics at the top of files
- Historical trend indicators (improved/degraded)
- Click-to-navigate actions

#### Editor Decorations

Visual indicators include:

- Gutter icons showing issue severity
- Background highlighting for problematic code ranges
- Overview ruler markers for quick file scanning
- Hover tooltips with actionable suggestions

### Interactive Dashboard

A dedicated sidebar panel provides:

- **Overall Score** - File and workspace quality metrics (0-100 scale)
- **Plugin Breakdown** - Detailed scores by analyzer (React, Core, etc.)
- **Issue Categories** - Security, accessibility, performance, complexity
- **Top Issues List** - Prioritized problems with click-to-navigate
- **Trend Visualization** - Historical quality tracking over time
- **File Navigator** - Tree view with score-based filtering and sorting

### AI-Powered Features

#### Copilot Chat Integration

Interact with Vipr analysis through natural language:

```
@vipr explain this issue
@vipr suggest a fix
@vipr analyze security problems
@vipr show me performance issues
```

#### Language Model API

Get AI-generated insights for:

- Detailed issue explanations
- Step-by-step fix suggestions
- Workspace quality summaries
- Root cause analysis

#### AI-Assisted Fixes

Automatic and semi-automatic fixes via:

- Quick Fix actions (light bulb menu)
- "Fix with AI" command for complex issues
- Clipboard-based suggestions for manual review
- Integration with Cursor AI and GitHub Copilot

### Historical Analysis

Track code quality over time with:

#### SQLite-based Storage

- Persistent analysis snapshots tied to git commits
- Efficient querying of historical metrics
- Category-level trend data
- Individual finding attribution

#### Git Integration

- Blame-based attribution showing who introduced issues
- Commit-level quality comparisons
- Regression detection identifying quality degradations
- Historical trend visualization

#### Temporal Regression Detection

Identify when quality degraded:

- Binary search through git history to find regression commits
- "Primary Regression" identification with commit details
- Before/after metric comparisons
- Author and date attribution for accountability

### Performance Optimized

Built for large codebases with:

- **Lazy Loading** - Heavy dependencies load on-demand
- **Worker Threads** - CPU-intensive analysis off the main thread
- **Result Caching** - Content-based invalidation for unchanged files
- **Incremental Analysis** - Only analyze modified files
- **Memory Monitoring** - Automatic cleanup when usage is high
- **Bundle Optimization** - Minimal activation footprint

Activation time: < 2 seconds
Bundle size: < 5MB
Memory usage: < 200MB for typical workspaces

### License Tiers

#### Free Tier

- Core Overview and React Overview reports
- Basic diagnostics and CodeLens
- File-level analysis
- Standard issue detection

#### Pro Tier

- All reports including advanced analysis
- AI-powered explanations and fixes
- Workspace-level analysis
- Historical tracking and trends
- PDF report export

#### Enterprise Tier

- All Pro features
- Priority support
- Custom analyzer configurations
- Team collaboration features

## Next Steps

- [Getting Started](./getting-started) - Installation and basic usage
- [Commands Reference](./commands) - All available commands
- [Configuration Guide](./configuration) - Customize extension behavior
- [Testing Guide](../testing/user-testing) - How to test features
