# History Feature - VSCode Extension Implementation Guide

## Overview

This guide outlines the implementation of git history features in the Vipr VSCode extension. The history features allow developers to view code quality metrics over time directly in their editor.

## Architecture

```
VSCode Extension
├── History Tree View (Sidebar)
│   ├── Commit List with Metrics
│   ├── Expandable Commit Details
│   └── Compare Action Buttons
│
├── File History CodeLens (Inline)
│   ├── "View History" link above file
│   └── Shows metric trend (↑ improving, ↓ declining)
│
└── Status Bar Indicator
    ├── Metrics since last commit
    └── Click to open History view
```

## Phase 1: Add @vipr/history Dependency

**File**: `clients/vscode-extension/package.json`

```json
{
  "dependencies": {
    "@vipr/history": "workspace:*"
    // ... existing dependencies
  }
}
```

Run: `pnpm install`

## Phase 2: History Tree View Provider

**File**: `clients/vscode-extension/src/providers/history-tree-provider.ts`

```typescript
import * as vscode from 'vscode';
import { GitHistoryService, HistoryAnalyzer } from '@vipr/history';
import { AnalysisEngine } from '@vipr/engine';

export class HistoryTreeProvider implements vscode.TreeDataProvider<HistoryTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<HistoryTreeItem | undefined | void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  private gitService: GitHistoryService;
  private historyAnalyzer: HistoryAnalyzer;
  private commits: CommitInfo[] = [];

  constructor(
    private workspaceRoot: string,
    private engine: AnalysisEngine
  ) {
    this.gitService = new GitHistoryService(workspaceRoot);
    // Initialize cache manager and history analyzer
    // ...
  }

  refresh(): void {
    this._onDidChangeTreeData.fire();
  }

  async loadCommits(limit: number = 50): Promise<void> {
    this.commits = await this.gitService.getCommits({ limit });
    this.refresh();
  }

  getTreeItem(element: HistoryTreeItem): vscode.TreeItem {
    return element;
  }

  async getChildren(element?: HistoryTreeItem): Promise<HistoryTreeItem[]> {
    if (!element) {
      // Root level: show commits
      return this.commits.map(commit => new CommitTreeItem(commit));
    }

    if (element instanceof CommitTreeItem) {
      // Commit level: show files changed
      const files = await this.gitService.getFilesChanged(element.commit.sha);
      return files.map(file => new FileTreeItem(file, element.commit.sha));
    }

    return [];
  }
}

class CommitTreeItem extends vscode.TreeItem {
  constructor(
    public readonly commit: CommitInfo,
    public readonly collapsibleState: vscode.TreeItemCollapsibleState = vscode
      .TreeItemCollapsibleState.Collapsed
  ) {
    super(commit.shortSha, collapsibleState);

    this.description = commit.message;
    this.tooltip = `${commit.author} - ${new Date(commit.date).toLocaleString()}`;
    this.contextValue = 'commit';

    // Add health score if available
    // this.iconPath = new vscode.ThemeIcon('git-commit');
  }
}

class FileTreeItem extends vscode.TreeItem {
  constructor(
    public readonly filePath: string,
    public readonly commitSha: string
  ) {
    super(filePath, vscode.TreeItemCollapsibleState.None);

    this.contextValue = 'file';
    this.iconPath = vscode.ThemeIcon.File;

    // Add command to open diff
    this.command = {
      command: 'vipr.history.showFileDiff',
      title: 'Show File Diff',
      arguments: [filePath, commitSha],
    };
  }
}
```

## Phase 3: Register History View

**File**: `clients/vscode-extension/src/extension.ts`

```typescript
import { HistoryTreeProvider } from './providers/history-tree-provider';

export function activate(context: vscode.ExtensionContext) {
  // ... existing activation code

  // Register history tree view
  const workspaceRoot = vscode.workspace.workspaceFolders?.[0]?.uri.fsPath;
  if (workspaceRoot) {
    const historyProvider = new HistoryTreeProvider(workspaceRoot, analysisEngine);

    const historyView = vscode.window.createTreeView('vipr.history', {
      treeDataProvider: historyProvider,
      showCollapseAll: true,
    });

    context.subscriptions.push(historyView);

    // Register commands
    context.subscriptions.push(
      vscode.commands.registerCommand('vipr.history.refresh', () => {
        historyProvider.loadCommits();
      })
    );

    context.subscriptions.push(
      vscode.commands.registerCommand('vipr.history.compareCommits', async (commit1, commit2) => {
        // Show comparison view
      })
    );

    // Load initial commits
    historyProvider.loadCommits();
  }
}
```

## Phase 4: Update package.json Contributions

**File**: `clients/vscode-extension/package.json`

```json
{
  "contributes": {
    "views": {
      "vipr": [
        {
          "id": "vipr.history",
          "name": "Git History",
          "icon": "$(history)",
          "contextualTitle": "Vipr History"
        }
      ]
    },
    "commands": [
      {
        "command": "vipr.history.refresh",
        "title": "Refresh History",
        "icon": "$(refresh)"
      },
      {
        "command": "vipr.history.compareCommits",
        "title": "Compare Commits",
        "icon": "$(git-compare)"
      },
      {
        "command": "vipr.history.showFileDiff",
        "title": "Show File at Commit"
      },
      {
        "command": "vipr.history.viewFileHistory",
        "title": "View File History",
        "icon": "$(history)"
      }
    ],
    "menus": {
      "view/title": [
        {
          "command": "vipr.history.refresh",
          "when": "view == vipr.history",
          "group": "navigation"
        }
      ],
      "view/item/context": [
        {
          "command": "vipr.history.compareCommits",
          "when": "view == vipr.history && viewItem == commit",
          "group": "inline"
        }
      ],
      "editor/title": [
        {
          "command": "vipr.history.viewFileHistory",
          "when": "resourceScheme == file",
          "group": "navigation"
        }
      ]
    }
  }
}
```

## Phase 5: File History CodeLens

**File**: `clients/vscode-extension/src/providers/history-codelens-provider.ts`

```typescript
import * as vscode from 'vscode';
import { HistoryAnalyzer } from '@vipr/history';

export class HistoryCodeLensProvider implements vscode.CodeLensProvider {
  private _onDidChangeCodeLenses = new vscode.EventEmitter<void>();
  readonly onDidChangeCodeLenses = this._onDidChangeCodeLenses.event;

  constructor(private historyAnalyzer: HistoryAnalyzer) {}

  async provideCodeLenses(
    document: vscode.TextDocument,
    token: vscode.CancellationToken
  ): Promise<vscode.CodeLens[]> {
    const filePath = document.uri.fsPath;

    // Get file history (last 10 commits)
    const history = await this.historyAnalyzer.getFileHistory(filePath, { limit: 10 });

    if (history.length === 0) {
      return [];
    }

    // Calculate trend
    const latest = history[0];
    const oldest = history[history.length - 1];
    const healthDelta = (latest?.healthScore ?? 0) - (oldest?.healthScore ?? 0);

    const trend = healthDelta > 5 ? '↑ improving' : healthDelta < -5 ? '↓ declining' : '→ stable';
    const trendIcon = healthDelta > 5 ? '✅' : healthDelta < -5 ? '⚠️' : 'ℹ️';

    // Add CodeLens at top of file
    const range = new vscode.Range(0, 0, 0, 0);
    const codeLens = new vscode.CodeLens(range, {
      title: `${trendIcon} Quality ${trend} (${history.length} commits)`,
      command: 'vipr.history.viewFileHistory',
      arguments: [filePath],
    });

    return [codeLens];
  }
}
```

**Register in extension.ts**:

```typescript
const codeLensProvider = new HistoryCodeLensProvider(historyAnalyzer);
context.subscriptions.push(
  vscode.languages.registerCodeLensProvider(
    { scheme: 'file', language: 'typescript' },
    codeLensProvider
  ),
  vscode.languages.registerCodeLensProvider(
    { scheme: 'file', language: 'javascript' },
    codeLensProvider
  )
);
```

## Phase 6: Status Bar Indicator

**File**: `clients/vscode-extension/src/providers/history-status-bar.ts`

```typescript
import * as vscode from 'vscode';
import { HistoryAnalyzer } from '@vipr/history';

export class HistoryStatusBar {
  private statusBarItem: vscode.StatusBarItem;

  constructor(private historyAnalyzer: HistoryAnalyzer) {
    this.statusBarItem = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 100);
    this.statusBarItem.command = 'vipr.history.showSinceLastCommit';
  }

  async update(filePath: string): Promise<void> {
    try {
      // Get last 2 commits for this file
      const history = await this.historyAnalyzer.getFileHistory(filePath, { limit: 2 });

      if (history.length < 2) {
        this.statusBarItem.hide();
        return;
      }

      const current = history[0];
      const previous = history[1];
      const delta = (current?.healthScore ?? 0) - (previous?.healthScore ?? 0);

      const icon = delta > 0 ? '$(arrow-up)' : delta < 0 ? '$(arrow-down)' : '$(dash)';
      const color = delta > 0 ? '#4ade80' : delta < 0 ? '#f87171' : undefined;

      this.statusBarItem.text = `${icon} Health: ${Math.round(current?.healthScore ?? 0)} (${delta > 0 ? '+' : ''}${Math.round(delta)})`;
      this.statusBarItem.tooltip = `Since last commit: ${previous?.commitSha?.substring(0, 7)}`;
      this.statusBarItem.backgroundColor = undefined;

      if (delta < -10) {
        this.statusBarItem.backgroundColor = new vscode.ThemeColor(
          'statusBarItem.warningBackground'
        );
      }

      this.statusBarItem.show();
    } catch (error) {
      this.statusBarItem.hide();
    }
  }

  show(): void {
    this.statusBarItem.show();
  }

  hide(): void {
    this.statusBarItem.hide();
  }

  dispose(): void {
    this.statusBarItem.dispose();
  }
}
```

## Phase 7: Webview Panel for Detailed History

**File**: `clients/vscode-extension/src/views/history-panel.ts`

```typescript
import * as vscode from 'vscode';
import { HistoryAnalyzer } from '@vipr/history';

export class HistoryPanel {
  public static currentPanel: HistoryPanel | undefined;
  private readonly _panel: vscode.WebviewPanel;
  private _disposables: vscode.Disposable[] = [];

  private constructor(
    panel: vscode.WebviewPanel,
    private historyAnalyzer: HistoryAnalyzer,
    private filePath: string
  ) {
    this._panel = panel;
    this._update();

    this._panel.onDidDispose(() => this.dispose(), null, this._disposables);
  }

  public static createOrShow(
    extensionUri: vscode.Uri,
    historyAnalyzer: HistoryAnalyzer,
    filePath: string
  ) {
    // Show existing panel or create new one
    if (HistoryPanel.currentPanel) {
      HistoryPanel.currentPanel._panel.reveal(vscode.ViewColumn.Two);
      return;
    }

    const panel = vscode.window.createWebviewPanel(
      'viprHistory',
      'File History',
      vscode.ViewColumn.Two,
      {
        enableScripts: true,
        retainContextWhenHidden: true,
      }
    );

    HistoryPanel.currentPanel = new HistoryPanel(panel, historyAnalyzer, filePath);
  }

  private async _update() {
    const history = await this.historyAnalyzer.getFileHistory(this.filePath, { limit: 50 });

    this._panel.webview.html = this._getHtmlForWebview(history);
  }

  private _getHtmlForWebview(history: any[]): string {
    return `
      <!DOCTYPE html>
      <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>File History</title>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        <style>
          body { padding: 20px; }
          .chart-container { height: 400px; margin-bottom: 30px; }
          table { width: 100%; border-collapse: collapse; }
          th, td { padding: 8px; text-align: left; border-bottom: 1px solid #ddd; }
        </style>
      </head>
      <body>
        <h1>File History: ${this.filePath}</h1>

        <div class="chart-container">
          <canvas id="healthChart"></canvas>
        </div>

        <table>
          <thead>
            <tr>
              <th>Commit</th>
              <th>Date</th>
              <th>Author</th>
              <th>Health Score</th>
              <th>Issues</th>
            </tr>
          </thead>
          <tbody>
            ${history
              .map(
                h => `
              <tr>
                <td><code>${h.commitSha.substring(0, 7)}</code></td>
                <td>${new Date(h.commitDate).toLocaleDateString()}</td>
                <td>${h.author}</td>
                <td>${Math.round(h.healthScore ?? 0)}</td>
                <td>${h.issueCount}</td>
              </tr>
            `
              )
              .join('')}
          </tbody>
        </table>

        <script>
          const ctx = document.getElementById('healthChart').getContext('2d');
          const data = ${JSON.stringify(history)};

          new Chart(ctx, {
            type: 'line',
            data: {
              labels: data.map(d => new Date(d.commitDate).toLocaleDateString()),
              datasets: [{
                label: 'Health Score',
                data: data.map(d => d.healthScore),
                borderColor: 'rgb(75, 192, 192)',
                tension: 0.1
              }]
            },
            options: {
              responsive: true,
              maintainAspectRatio: false
            }
          });
        </script>
      </body>
      </html>
    `;
  }

  public dispose() {
    HistoryPanel.currentPanel = undefined;
    this._panel.dispose();
    while (this._disposables.length) {
      const disposable = this._disposables.pop();
      if (disposable) {
        disposable.dispose();
      }
    }
  }
}
```

## Testing

1. **Unit Tests**: Test GitHistoryService, HistoryAnalyzer with mock git data
2. **Integration Tests**: Test TreeView updates, CodeLens rendering
3. **Manual Testing**:
   - Open a git repository in VSCode
   - Verify History view shows commits
   - Click on commit to see files changed
   - Open a file and verify CodeLens appears
   - Check status bar shows metric delta
   - Run "View File History" command

## Performance Considerations

1. **Cache Results**: Use VSCode's Memento API to cache commit lists
2. **Lazy Loading**: Only analyze commits when expanded in tree view
3. **Debounce Updates**: Debounce status bar updates on file changes
4. **Background Processing**: Use VSCode's Progress API for long operations

## Future Enhancements

1. **Interactive Charts**: Add clickable chart to jump to specific commit
2. **Blame Integration**: Show metric history in git blame view
3. **Trend Notifications**: Alert when file quality declines significantly
4. **Comparison View**: Side-by-side diff with metric overlay
5. **Team Insights**: Show team-wide metric trends
