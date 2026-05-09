# User Testing Guide

This guide provides step-by-step testing procedures for end users to verify Vipr extension functionality.

## Prerequisites

- VSCode 1.85.0 or higher installed
- Vipr extension installed
- A React/TypeScript test project (can use create-react-app)

## Test Project Setup

### Create Test Project

```bash
npx create-react-app test-vipr --template typescript
cd test-vipr
code .
```

### Add Test Files

Create `src/components/ProblemComponent.tsx` with intentional issues:

```typescript
import React, { useState } from 'react';

// Missing accessibility: no label
export function ProblemComponent() {
  const [count, setCount] = useState(0);

  // Missing useCallback - performance issue
  const handleClick = () => {
    setCount(count + 1);
  };

  return (
    <div>
      {/* Missing alt text - accessibility */}
      <img src="/logo.png" />

      {/* Inline event handler - performance */}
      <button onClick={() => console.log('clicked')}>
        Count: {count}
      </button>

      {/* Using eval - security issue */}
      <button onClick={() => eval('alert("test")')}>
        Dangerous
      </button>

      {/* dangerouslySetInnerHTML - security */}
      <div dangerouslySetInnerHTML={{ __html: '<script>alert("xss")</script>' }} />
    </div>
  );
}
```

---

## Feature Testing

### 1. Basic Analysis

**Test: Analyze Single File**

1. Open `src/components/ProblemComponent.tsx`
2. Open Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`)
3. Run `Vipr: Analyze File`
4. Wait for progress notification

**Expected Results:**

- ✓ Progress notification appears: "Vipr: Analyzing file..."
- ✓ Notification completes: "Analysis complete: XX/100"
- ✓ Status bar updates: `$(pulse) Vipr: XX/100`
- ✓ Problems panel shows issues (open with `Cmd+Shift+M`)
- ✓ CodeLens hint appears above component

**Verify Issues Found:**

- Security: eval usage, dangerouslySetInnerHTML
- Accessibility: missing alt text, missing label
- Performance: inline event handler, missing useCallback

---

**Test: Analyze Workspace**

1. Open Command Palette
2. Run `Vipr: Analyze Workspace`
3. Monitor progress notification

**Expected Results:**

- ✓ Progress shows: "Analyzing workspace... X/Y files"
- ✓ Can cancel mid-progress (click X on notification)
- ✓ Output panel logs summary (View → Output → select "Vipr")
- ✓ Summary shows:
  - Total files analyzed
  - Average score
  - Top 10 hotspots (lowest scores)

**Verify Output:**

```
=== Workspace Analysis Complete ===
Total Files: 15
Analyzed: 15
Failed: 0
Average Score: 68/100

Top 10 Hotspots (Lowest Scores):
  1. src/components/ProblemComponent.tsx - 42/100
  ...
```

---

### 2. Visual Feedback

**Test: Problems Panel Integration**

1. After analyzing, open Problems panel (`Cmd+Shift+M`)
2. Filter to show only Vipr issues

**Expected Results:**

- ✓ Issues appear with "Vipr" source
- ✓ Severity icons match:
  - Red circle (Error) for critical
  - Yellow triangle (Warning) for warnings
  - Blue circle (Info) for info
- ✓ Clicking issue navigates to line
- ✓ Issues grouped by file

---

**Test: Inline Diagnostics**

1. Open file with issues
2. Look for squiggly underlines

**Expected Results:**

- ✓ Red squiggles under critical issues
- ✓ Yellow squiggles under warnings
- ✓ Blue squiggles under info issues
- ✓ Hover over squiggle shows tooltip with:
  - Issue message
  - Suggestion (if available)
  - "Fix with AI" link (if supported)

---

**Test: CodeLens Hints**

1. Open React component file
2. Look above component definition

**Expected Results:**

- ✓ CodeLens shows: `$(pulse) XX/100 (Level)`
- ✓ Levels: Excellent (80-100), Good (60-79), Fair (40-59), Poor (0-39)
- ✓ Clicking hint re-runs analysis
- ✓ Hint updates after analysis

**Test: Enable/Disable CodeLens**

1. Open Settings (`Cmd+,`)
2. Search "vipr.enableCodeLens"
3. Uncheck the box
4. Verify CodeLens disappears
5. Re-enable and verify it reappears

---

**Test: Editor Decorations**

1. Analyze file with issues
2. Look for gutter icons (left margin)

**Expected Results:**

- ✓ Icons appear in gutter next to problematic lines
- ✓ Background highlighting on issue ranges
- ✓ Overview ruler markers (right scrollbar area)
- ✓ Hovering shows detailed tooltip

**Test: Enable/Disable Decorations**

1. Settings: `vipr.showDecorations`
2. Disable → decorations disappear
3. Enable → decorations reappear

---

### 3. Dashboard Testing

**Test: Open Dashboard**

1. Click Vipr icon in Activity Bar (left sidebar)
   OR
2. Run `Vipr: Show Dashboard` from Command Palette

**Expected Results:**

- ✓ Dashboard panel opens in sidebar
- ✓ Shows overall score with color-coded circle:
  - Green (80-100)
  - Blue (60-79)
  - Yellow (40-59)
  - Red (0-39)
- ✓ Plugin breakdown section shows scores by analyzer
- ✓ Top issues list appears
- ✓ Refresh button re-runs analysis
- ✓ Settings button opens VSCode settings

---

**Test: Dashboard Navigation**

1. Open dashboard
2. Click an issue in "Top Issues" list

**Expected Results:**

- ✓ Editor navigates to issue location
- ✓ Line is highlighted
- ✓ Cursor positioned at issue

---

**Test: File Tree Navigator**

1. In dashboard, switch to "Files" tab
2. View file tree

**Expected Results:**

- ✓ Files listed with scores
- ✓ Icons indicate severity:
  - Red dot: Critical issues
  - Yellow dot: Warnings
  - Blue dot: Info only
  - Green check: No issues
- ✓ Clicking file opens it in editor

**Test: Tree Sorting**

1. Click "Sort by Score" button
2. Verify files sorted by score (lowest first)
3. Click "Sort by Name" button
4. Verify alphabetical sorting

**Test: Tree Filtering**

1. Click "Show Critical Only"
2. Verify only files with critical issues visible
3. Click "Show Warnings and Critical"
4. Verify warning+ files visible
5. Click "Show All Files"
6. Verify all analyzed files visible

---

### 4. Quick Fixes

**Test: Auto-Fix**

1. Create a file with auto-fixable issue:

```typescript
// Missing return type
function add(a: number, b: number) {
  return a + b;
}
```

2. Hover over the issue
3. Click the light bulb icon
4. Select "Vipr: Quick Fix"

**Expected Results:**

- ✓ Light bulb appears on auto-fixable issues
- ✓ Clicking applies fix automatically
- ✓ Code updates to:

```typescript
function add(a: number, b: number): number {
  return a + b;
}
```

---

### 5. AI Features

**Test: AI-Powered Fix**

**Prerequisites:** GitHub Copilot installed and active

1. Place cursor on an issue line
2. Run `Vipr: Fix with AI`
3. Select provider: "Copilot"

**Expected Results:**

- ✓ Copilot Chat opens with context
- ✓ Shows issue details
- ✓ Provides fix suggestion
- ✓ Can apply suggestion via Copilot interface

---

**Test: Chat Participant**

**Prerequisites:** GitHub Copilot Chat enabled

1. Open Copilot Chat panel
2. Type: `@vipr explain this issue`
3. Press Enter

**Expected Results:**

- ✓ Vipr chat participant responds
- ✓ Explanation includes:
  - What the issue is
  - Why it's problematic
  - How to fix it

**Test Chat Commands:**

```
@vipr analyze security problems
@vipr suggest a fix for [issue type]
@vipr show performance issues
```

---

**Test: Language Model Status**

1. Run `Vipr: Check Language Model Status`

**Expected Results:**

- ✓ Modal shows available models
- ✓ Lists: gpt-4o, gpt-3.5-turbo (if Copilot active)
- ✓ Shows "No models available" if Copilot inactive

---

### 6. Historical Features

**Test: SQLite Storage**

**Prerequisites:** Git repository

1. Analyze workspace
2. Check database exists:
   - macOS: `~/Library/Application Support/Code/User/globalStorage/<extension-id>/vipr.db`
   - Windows: `%APPDATA%\Code\User\globalStorage\<extension-id>\vipr.db`
   - Linux: `~/.config/Code/User/globalStorage/<extension-id>/vipr.db`

**Expected Results:**

- ✓ Database file created
- ✓ Size increases with each analysis
- ✓ Typical size: 5-50KB per snapshot

---

**Test: Show History**

**Prerequisites:** Multiple analysis snapshots (run analysis, commit, change file, analyze again)

1. Open a file with history
2. Run `Vipr: Show History`

**Expected Results:**

- ✓ History panel opens
- ✓ Shows trend chart with score over time
- ✓ Data points show commit information
- ✓ Can click points to view commit details

---

**Test: Find Regression**

**Prerequisites:** File with degraded score

1. Make changes that lower score
2. Run analysis
3. Run `Vipr: Find Regression`

**Expected Results:**

- ✓ Extension searches git history
- ✓ Progress: "Searching commits... X/Y"
- ✓ Identifies commit that caused degradation
- ✓ Shows:
  - Commit hash
  - Author
  - Date
  - Message
  - Score impact

---

**Test: Cleanup History**

1. Run `Vipr: Cleanup History`
2. Select "Keep 7 days"
3. Confirm deletion

**Expected Results:**

- ✓ Quick pick menu appears
- ✓ Options: 7 days, 30 days, 90 days, Keep all
- ✓ Confirmation shows number deleted
- ✓ Database size decreases

---

### 7. Configuration Testing

**Test: License Configuration**

1. Open Settings
2. Find `vipr.licenseKey`
3. Enter: `VIPR-FREE-12345678`
4. Run `Vipr: Show License Status`

**Expected Results:**

- ✓ Shows "Free" tier
- ✓ Lists available features
- ✓ Shows locked premium features

**Test Pro License:**

1. Update to: `VIPR-PRO-12345678`
2. Check license status

**Expected Results:**

- ✓ Shows "Pro" tier
- ✓ All features unlocked
- ✓ Can export reports

---

**Test: Analyze on Save**

1. Enable: `vipr.analyzeOnSave: true`
2. Open and edit a file
3. Save (`Cmd+S` / `Ctrl+S`)

**Expected Results:**

- ✓ Analysis triggers automatically
- ✓ No progress notification (background)
- ✓ Problems panel updates
- ✓ Status bar updates

---

**Test: Diagnostic Severity Filter**

1. Set `vipr.diagnosticSeverity: "error"`
2. Analyze file

**Expected Results:**

- ✓ Only critical (error) issues appear
- ✓ Warnings and info hidden
- ✓ Problems panel shows fewer issues

---

### 8. Performance Testing

**Test: Large File Analysis**

1. Create file with 1000+ lines
2. Run `Vipr: Analyze File`
3. Measure time

**Expected Results:**

- ✓ Completes in < 10 seconds
- ✓ UI remains responsive
- ✓ No freezing or lag

---

**Test: Workspace Analysis (100+ files)**

1. Open large React project
2. Run `Vipr: Analyze Workspace`
3. Monitor memory usage

**Expected Results:**

- ✓ Progress updates smoothly
- ✓ Can cancel anytime
- ✓ Memory stays < 500MB
- ✓ VSCode remains responsive

---

**Test: Cache Effectiveness**

1. Analyze a file (note time)
2. Re-analyze same file without changes
3. Compare times

**Expected Results:**

- ✓ Second analysis much faster (< 50ms)
- ✓ Status bar shows "(cached)" briefly
- ✓ Results identical

---

**Test: Worker Threads**

1. Enable `vipr.performance.useWorkerThreads: true`
2. Run workspace analysis
3. Open Task Manager / Activity Monitor
4. Check CPU usage

**Expected Results:**

- ✓ Multiple cores utilized
- ✓ Analysis faster than single-threaded
- ✓ UI thread not blocked

---

### 9. Error Handling

**Test: Invalid File Type**

1. Open a `.txt` or `.md` file
2. Try to run analysis

**Expected Results:**

- ✓ Command disabled or shows warning
- ✓ Message: "File type not supported"
- ✓ No crashes

---

**Test: Network Errors (AI Features)**

1. Disable internet connection
2. Try `Vipr: Fix with AI` with Copilot

**Expected Results:**

- ✓ Graceful error message
- ✓ No crash
- ✓ Suggests checking connection

---

**Test: Missing Dependencies**

1. Uninstall GitHub Copilot
2. Run `Vipr: Check Language Model Status`

**Expected Results:**

- ✓ Message: "No language models available"
- ✓ Suggests installing Copilot
- ✓ Provides settings link

---

## Regression Testing Checklist

After each update, verify:

- [ ] Extension activates in < 2 seconds
- [ ] Analyze File works on .ts, .tsx, .js, .jsx
- [ ] Workspace analysis completes without errors
- [ ] Problems panel shows issues
- [ ] CodeLens appears above components
- [ ] Dashboard opens and displays data
- [ ] Quick fixes apply correctly
- [ ] License validation works
- [ ] History tracking persists across sessions
- [ ] Cleanup removes old snapshots
- [ ] Settings changes take effect
- [ ] No memory leaks after 10+ analyses
- [ ] Cached results load instantly

---

## Reporting Issues

When reporting bugs, include:

1. **Extension Version:** Check in Extensions view
2. **VSCode Version:** Help → About
3. **Steps to Reproduce:**
   ```
   1. Open file X
   2. Run command Y
   3. Observe Z
   ```
4. **Expected Behavior:** What should happen
5. **Actual Behavior:** What actually happened
6. **Logs:** From Output panel (Vipr channel)
7. **Screenshots:** If visual issue
8. **Test Project:** Minimal reproduction case

**Submit to:** https://github.com/glorioustephan/vipr-app/issues

---

## Next Steps

- [Developer Testing Guide](./developer-testing) - For contributors
- [Troubleshooting](../user-guide/troubleshooting) - Common issues and fixes
