# Commands Reference

Complete reference of all Vipr VSCode Extension commands accessible through the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`).

## Analysis Commands

### Vipr: Analyze File

Analyzes the currently active file and displays results.

**Usage:**

1. Open a TypeScript or React file
2. Run command: `Vipr: Analyze File`
3. Wait for analysis to complete
4. View results in Problems panel and inline

**Behavior:**

- Uses cache if file hasn't changed
- Updates CodeLens hints
- Refreshes diagnostics
- Updates dashboard if open
- Shows notification with score

**Keyboard Shortcut:** None (can be configured in Keyboard Shortcuts)

**Context:**

- Available when a TypeScript/JavaScript file is active
- Grayed out for non-supported file types

---

### Vipr: Analyze Workspace

Analyzes all eligible files in the current workspace.

**Usage:**

1. Open a workspace with TypeScript/React files
2. Run command: `Vipr: Analyze Workspace`
3. Monitor progress in notification
4. Review summary in Output panel

**Behavior:**

- Finds all `.ts`, `.tsx`, `.js`, `.jsx` files
- Excludes `node_modules` and build directories
- Can be cancelled mid-progress
- Shows progress: `45/100 files analyzed`
- Logs hotspots (lowest scored files) to Output
- Updates file tree view with scores

**Options:**

- Incremental mode (default): Only analyzes changed files
- Full mode: Re-analyzes all files (use `vipr.clearCache` first)

**Performance:**

- Uses worker threads for parallel analysis
- Typical speed: 5-10 files per second
- Large workspaces (500+ files): 1-2 minutes

---

### Vipr: Show Problems

Intelligently analyzes and opens Problems panel.

**Usage:**

1. Run command: `Vipr: Show Problems`
2. Extension analyzes current file (if needed)
3. Problems panel opens with Vipr issues highlighted

**Behavior:**

- Checks if current file has been analyzed
- Runs analysis if needed
- Opens Problems panel
- Filters to show Vipr source
- Focuses on first issue

**Useful When:**

- You want to see all issues at once
- Navigating between multiple issues
- Exporting issue list

---

### Vipr: Clear Analysis Cache

Clears all cached analysis results.

**Usage:**

1. Run command: `Vipr: Clear Analysis Cache`
2. Confirmation message appears
3. All files will be re-analyzed on next run

**When to Use:**

- After updating dependencies
- When results seem stale
- After changing configuration
- To force full workspace re-analysis

**Note:** Does not delete historical data (SQLite snapshots remain intact)

---

## Dashboard Commands

### Vipr: Show Dashboard

Opens the Vipr sidebar dashboard panel.

**Usage:**

1. Run command: `Vipr: Show Dashboard`
2. Dashboard opens in sidebar
3. View overall score, plugin breakdown, and issues

**Alternative:** Click the Vipr icon in the Activity Bar (left sidebar)

**Dashboard Features:**

- Overall quality score with trend indicator
- Plugin breakdown (React, Core, etc.)
- Category scores (Security, Performance, etc.)
- Top issues list with navigation
- Historical trend chart
- Export options

---

### Vipr: Open File Picker

Opens quick pick menu to analyze a specific file.

**Usage:**

1. Run command: `Vipr: Open File Picker`
2. Type to filter files
3. Select file to analyze
4. Analysis runs automatically

**Useful For:**

- Analyzing files without opening them
- Quick navigation to problematic files
- Batch analysis workflow

---

### Vipr: Show Problems for File

Shows problems for a specific file path.

**Usage:**

```
vipr.showProblemsForFile(filePath: string)
```

**Programmatic Use:**

```typescript
vscode.commands.executeCommand('vipr.showProblemsForFile', '/path/to/file.tsx');
```

**Behavior:**

- Analyzes file if not already analyzed
- Opens Problems panel
- Filters to show only that file's issues
- Used internally by file tree

---

## AI Commands

### Vipr: Fix with AI

Applies AI-powered fix to selected issue.

**Usage:**

1. Place cursor on line with issue
2. Run command: `Vipr: Fix with AI`
3. Choose AI provider:
   - **Copilot** - GitHub Copilot Chat integration
   - **Cursor** - Cursor AI editor integration
   - **Clipboard** - Copy suggestion to clipboard
   - **Auto** - Detects available provider
4. Review suggested fix
5. Apply or adjust as needed

**Requirements:**

- GitHub Copilot subscription (for Copilot provider)
- Cursor editor (for Cursor provider)
- Language Model API enabled

**Configuration:**

```json
{
  "vipr.aiProvider": "auto" | "copilot" | "cursor" | "clipboard",
  "vipr.enableAiFixing": true
}
```

---

### Vipr: Check Language Model Status

Checks availability of Language Model API.

**Usage:**

1. Run command: `Vipr: Check Language Model Status`
2. Modal shows available models
3. Lists compatible providers

**Output Example:**

```
Language Models Available:
- gpt-4o (GitHub Copilot)
- gpt-3.5-turbo (GitHub Copilot)
```

**Troubleshooting:**
If no models available:

- Install GitHub Copilot extension
- Sign in to Copilot
- Check Copilot subscription status
- Verify VSCode version >= 1.85.0

---

## History Commands

### Vipr: Show History

Displays historical quality trends for current file.

**Usage:**

1. Open a file with analysis history
2. Run command: `Vipr: Show History`
3. History panel opens with trend chart

**Features:**

- Line chart showing score over time
- Commit-level data points
- Author and date information
- Click to view commit details
- Compare current vs. historical scores

**Requirements:**

- Git repository
- Previous analysis snapshots
- SQLite storage enabled

---

### Vipr: Find Regression

Identifies commit that caused quality degradation.

**Usage:**

1. Open a file with decreased score
2. Run command: `Vipr: Find Regression`
3. Extension performs binary search through git history
4. Results show in regression panel

**Output:**

```
Primary Regression Identified
Commit: a1b2c3d
Author: John Doe
Date: 2025-01-15
Message: Refactor authentication flow
Impact: -12 points (from 85 to 73)
```

**Algorithm:**

- Binary search through last 50 commits
- Analyzes file at historical commits
- Identifies first "bad" commit
- Shows metric-level breakdown

**Performance:**

- Max 10 git checkout operations
- Caches historical analysis
- Completes in 10-30 seconds

---

### Vipr: Cleanup History

Removes old analysis snapshots from storage.

**Usage:**

1. Run command: `Vipr: Cleanup History`
2. Select retention period:
   - Keep 7 days
   - Keep 30 days
   - Keep 90 days
   - Keep all
3. Confirm deletion
4. Old snapshots are removed

**Storage Management:**

- Typical snapshot size: 5-50KB
- 1000 snapshots ≈ 10-50MB
- Recommended: Clean up every 3 months

---

## Utility Commands

### Vipr: Show License Status

Displays current license tier and features.

**Usage:**

1. Run command: `Vipr: Show License Status`
2. Modal shows:
   - Current tier (Free, Pro, Enterprise)
   - Enabled features
   - License expiration (if applicable)
   - Upgrade options

**License Validation:**

- Checks license key format
- Verifies tier prefix
- Shows feature access matrix

**Example Output:**

```
License Status: Pro
Key: VIPR-PRO-********
Features Enabled:
✓ All reports
✓ AI-powered fixes
✓ Historical tracking
✓ PDF export
✓ Workspace analysis
```

---

### Vipr: Export Report

Exports analysis report to PDF.

**Usage:**

1. Analyze a file or workspace
2. Run command: `Vipr: Export Report`
3. Choose export scope:
   - **Current File** - Single file report
   - **Workspace** - Multi-file summary
4. Select save location
5. PDF generates with charts and tables

**Report Contents:**

- Executive summary
- Overall scores
- Category breakdown
- Issue details with locations
- Trend charts (if history available)
- Recommendations

**Requirements:**

- Pro or Enterprise license
- pdf-lib dependency (bundled)
- Chart.js for visualizations

---

### Vipr: Refresh File Tree

Refreshes the file navigator tree view.

**Usage:**

1. Open Vipr sidebar
2. Navigate to "Files" tab
3. Run command: `Vipr: Refresh File Tree`
4. Tree updates with latest scores

**Auto-Refresh:**

- Triggers after workspace analysis
- Updates when files are saved (if analyze-on-save enabled)
- Manual refresh for immediate update

---

## Tree View Commands

These commands are available in the file tree view context menu:

### Vipr: Sort by Name

Sorts files alphabetically by name.

### Vipr: Sort by Score

Sorts files by quality score (lowest first).

### Vipr: Filter - Show All Files

Shows all analyzed files.

### Vipr: Filter - Show Critical Only

Shows only files with critical issues.

### Vipr: Filter - Show Warnings and Critical

Shows files with warnings or critical issues.

### Vipr: Open File from Tree

Opens selected file in editor (internal command).

---

## Internal Commands

These commands are used internally and not meant for direct invocation:

- `vipr.fixWithAIFromHover` - Triggered from hover menu
- `vipr.navigateToIssue` - Used by dashboard issue navigation
- `vipr.internal.cleanup` - Memory cleanup trigger

---

## Command Keyboard Shortcuts

You can assign keyboard shortcuts to frequently used commands:

1. Open Keyboard Shortcuts: `Cmd+K Cmd+S` (macOS) / `Ctrl+K Ctrl+S` (Windows/Linux)
2. Search for "Vipr"
3. Click the `+` icon next to a command
4. Press your desired key combination

**Suggested Shortcuts:**

```json
{
  "key": "cmd+shift+a",
  "command": "vipr.analyzeFile",
  "when": "editorTextFocus"
},
{
  "key": "cmd+shift+alt+a",
  "command": "vipr.analyzeWorkspace"
}
```

---

## Next Steps

- [Configuration Guide](./configuration) - Customize command behavior
- [Dashboard Usage](./dashboard) - Master the sidebar interface
- [Testing Commands](../testing/user-testing) - Verify command functionality
