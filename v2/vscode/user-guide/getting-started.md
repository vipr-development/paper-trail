# Getting Started

This guide walks you through installing and using the Vipr VSCode Extension for the first time.

## Prerequisites

- VSCode 1.85.0 or higher
- Node.js 22.22.0 (for development workspaces)
- A TypeScript or React project

## Installation

### From VSCode Marketplace

1. Open VSCode
2. Press `Cmd+Shift+X` (macOS) or `Ctrl+Shift+X` (Windows/Linux) to open Extensions view
3. Search for "Vipr Code Analyzer"
4. Click **Install**
5. Reload VSCode when prompted

### From VSIX File (Local Development)

If you're testing a development build:

```bash
code --install-extension vipr-vscode-{version}.vsix
```

Or in VSCode:

1. Open Extensions view (`Cmd+Shift+X`)
2. Click `...` menu → **Install from VSIX**
3. Select the `.vsix` file

## First-Time Setup

### 1. Open a Project

Open a TypeScript or React project in VSCode:

```bash
cd /path/to/your/project
code .
```

### 2. Verify Activation

The extension activates automatically when you open TypeScript/React files.

Check the status bar (bottom of window) for the Vipr icon:

```
$(pulse) Vipr: Ready
```

### 3. Run Your First Analysis

Open a React component file (e.g., `App.tsx`) and:

**Option A: Command Palette**

1. Press `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows/Linux)
2. Type "Vipr: Analyze File"
3. Press Enter

**Option B: Status Bar**

Click the Vipr status bar item at the bottom of the window.

**What to Expect:**

- Progress notification appears: "Vipr: Analyzing file..."
- Analysis completes in 1-5 seconds (depending on file size)
- Problems panel updates with detected issues
- CodeLens hints appear above components
- Status bar shows file score: `$(pulse) Vipr: 78/100`

### 4. Review Issues

Open the Problems panel to see detected issues:

1. Press `Cmd+Shift+M` (macOS) or `Ctrl+Shift+M` (Windows/Linux)
2. Look for issues with source "Vipr"
3. Click any issue to navigate to its location

### 5. Explore the Dashboard

Open the Vipr sidebar for a comprehensive view:

1. Click the Vipr icon in the Activity Bar (left side)
2. View your overall score
3. Explore plugin breakdown and categories
4. Click top issues to navigate to them

## Basic Workflow

### Analyzing Individual Files

For quick file-level analysis:

1. Open a TypeScript/React file
2. Run `Vipr: Analyze File` from Command Palette
3. Review issues in Problems panel
4. Hover over highlighted code for details
5. Click light bulb for quick fixes

### Analyzing Entire Workspace

For workspace-level analysis:

1. Open Command Palette (`Cmd+Shift+P`)
2. Run `Vipr: Analyze Workspace`
3. Wait for progress notification to complete
4. View summary in Output panel (select "Vipr" from dropdown)
5. Navigate to hotspots (files with lowest scores)

Example output:

```
=== Workspace Analysis Complete ===
Total Files: 45
Analyzed: 45
Failed: 0
Average Score: 72/100

Top 10 Hotspots (Lowest Scores):
  1. src/components/Dashboard.tsx - 42/100
  2. src/utils/helpers.ts - 51/100
  3. src/api/client.ts - 58/100
  ...
```

### Understanding Scores

Vipr uses a 0-100 scoring system:

| Score  | Level     | Description                            |
| ------ | --------- | -------------------------------------- |
| 80-100 | Excellent | High quality, few issues               |
| 60-79  | Good      | Acceptable quality, minor issues       |
| 40-59  | Fair      | Moderate issues, needs improvement     |
| 0-39   | Poor      | Significant issues, requires attention |

Scores are calculated from:

- Issue severity (critical, high, medium, low)
- Issue count
- Code complexity metrics
- Best practice adherence

### Fixing Issues

#### Quick Fixes

For auto-fixable issues:

1. Hover over the issue
2. Click the light bulb icon
3. Select "Vipr: Quick Fix" from the menu
4. The fix applies automatically

#### AI-Assisted Fixes

For complex issues requiring AI:

1. Place cursor on the issue
2. Open Command Palette
3. Run `Vipr: Fix with AI`
4. Choose your AI provider:
   - **Copilot** - Uses GitHub Copilot Chat
   - **Cursor** - Opens in Cursor AI chat
   - **Clipboard** - Copies suggestion to clipboard
5. Review the suggested fix
6. Apply manually or via AI interface

#### Manual Fixes

For issues requiring manual intervention:

1. Read the issue description in hover tooltip
2. Check the "Suggestion" field for guidance
3. Consult documentation links if provided
4. Implement the fix based on recommendations

## Common Usage Patterns

### On-Save Analysis

Enable automatic analysis when you save files:

1. Open Settings (`Cmd+,`)
2. Search for "vipr.analyzeOnSave"
3. Check the box to enable
4. Save any file to trigger analysis

### CodeLens Scores

View component scores inline:

- Scores appear above React component definitions
- Format: `$(pulse) 85/100 (Excellent)`
- Click to run re-analysis
- Disabled components show `$(circle-slash) Not Analyzed`

### Severity Filtering

Focus on specific issue types:

1. Open Settings
2. Configure `vipr.diagnosticSeverity`:
   - **all** - Show all issues (default)
   - **warning** - Show warnings and errors only
   - **error** - Show errors only
3. Re-run analysis to apply filter

### License Configuration

Unlock premium features:

1. Obtain a license key (Pro or Enterprise)
2. Open Settings
3. Find `vipr.licenseKey`
4. Paste your key: `VIPR-PRO-XXXXXXXXXXXX`
5. Run `Vipr: Show License Status` to verify

## Troubleshooting

### Extension Not Activating

**Check activation events:**

1. Open a TypeScript or React file
2. View Extension Host logs:
   - Open Output panel (`Cmd+Shift+U`)
   - Select "Vipr" from dropdown
   - Look for activation messages

**Common fixes:**

- Reload VSCode: Run `Developer: Reload Window`
- Check VSCode version: Must be >= 1.85.0
- Verify file type: Must be `.ts`, `.tsx`, `.js`, or `.jsx`

### No Issues Appearing

**Verify settings:**

```json
{
  "vipr.enableDiagnostics": true,
  "vipr.diagnosticSeverity": "all"
}
```

**Check file eligibility:**

- File must be TypeScript or React
- File must be in workspace (not standalone)
- Analysis must complete successfully (check Output panel)

### Slow Analysis

**For large files:**

- Analysis time scales with file size
- Files > 1000 lines may take 5-10 seconds
- Consider splitting large files

**For workspace analysis:**

- Use incremental analysis (default enabled)
- Limit to specific directories if needed
- Check worker threads enabled: `vipr.performance.useWorkerThreads`

### CodeLens Not Showing

**Enable in settings:**

```json
{
  "vipr.enableCodeLens": true,
  "vipr.showInlineHints": true
}
```

**Verify file contains React components:**

- CodeLens only appears above React components
- Component names must be PascalCase
- Re-run analysis after enabling

## Next Steps

- [Commands Reference](./commands) - Explore all available commands
- [Configuration Guide](./configuration) - Customize extension settings
- [Dashboard Usage](./dashboard) - Master the sidebar panel
- [AI Features](./ai-features) - Use AI-powered analysis
