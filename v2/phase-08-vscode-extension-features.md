# Phase 08: VS Code Extension Features

**Priority:** High - Developer productivity  
**Complexity:** High  
**Dependencies:** Phase 00 (Foundation) ✅, Phase 07 (Extension Foundation)  
**Status:** Ready for implementation

**Phase Duration:** 5-7 days  
**Priority:** High - Developer experience enhancement  
**Complexity:** High  
**Dependencies:** Phase 07 (Extension Foundation)

All analyzer integrations use the `ts-morph`-based API.

## Overview

This phase builds advanced features on top of the VS Code extension foundation, including CodeLens integration, quick fixes, sidebar panel, editor decorations, and comprehensive refactoring support.

## Business Value

- Quick fixes reduce time to address issues
- CodeLens provides at-a-glance complexity visibility
- Sidebar dashboard for project-wide insights
- Integrated refactoring improves code quality
- Professional IDE experience

## Agent Assignments

| Agent                  | Role                               | Capacity  |
| ---------------------- | ---------------------------------- | --------- |
| vscode-plugin-engineer | Lead implementer, VS Code features | Primary   |
| react-engineer         | Refactoring logic                  | Secondary |
| typescript-engineer    | Type-safe implementations          | Advisory  |

## Execution Strategy

### Milestone 8.1: CodeLens Integration (Day 1-2)

**Synchronous Tasks:**

1. CodeLens provider implementation (vscode-plugin-engineer)
2. Complexity display above components (Sonnet)

### Milestone 8.2: Quick Fixes (Day 2-4)

**Parallel Tasks:**

- Extract hook quick fix (Opus)
- Add memo quick fix (Sonnet)
- Fix dependencies quick fix (Opus)
- Add cleanup quick fix (Sonnet)

### Milestone 8.3: Sidebar Panel (Day 4-5)

**Synchronous Tasks:**

1. Webview panel design (vscode-plugin-engineer)
2. Project analysis aggregation (Sonnet)
3. Interactive dashboard (Opus)

### Milestone 8.4: Editor Decorations & Polish (Day 6-7)

**Parallel Tasks:**

- Complexity highlighting (Sonnet)
- Gutter icons (Haiku)
- Settings refinement (Haiku)
- Documentation (Haiku)

## Detailed Tasks

### Task 8.1: CodeLens Provider

**Model:** Opus (VS Code API complexity)
**File:** `vipr-vscode/src/providers/codeLensProvider.ts`

```typescript
import * as vscode from 'vscode';
import {
  CodeLens,
  CodeLensProvider,
  TextDocument,
  CancellationToken,
  ProviderResult,
  Range,
  Command,
} from 'vscode';

interface ComponentInfo {
  name: string;
  range: Range;
  score: number;
  grade: string;
}

export class ViprCodeLensProvider implements CodeLensProvider {
  private components: Map<string, ComponentInfo[]> = new Map();
  private onDidChangeCodeLensesEmitter = new vscode.EventEmitter<void>();

  public readonly onDidChangeCodeLenses = this.onDidChangeCodeLensesEmitter.event;

  /**
   * Update component analysis data
   */
  updateAnalysis(uri: string, components: ComponentInfo[]) {
    this.components.set(uri, components);
    this.onDidChangeCodeLensesEmitter.fire();
  }

  provideCodeLenses(document: TextDocument, token: CancellationToken): ProviderResult<CodeLens[]> {
    const components = this.components.get(document.uri.toString());

    if (!components || token.isCancellationRequested) {
      return [];
    }

    return components.map(component => {
      const lens = new CodeLens(component.range);
      lens.command = {
        title: `Complexity: ${component.score} (${component.grade})`,
        command: 'vipr.showComponentDetails',
        arguments: [document.uri, component.name],
        tooltip: `Click for detailed complexity breakdown of ${component.name}`,
      };
      return lens;
    });
  }

  resolveCodeLens(codeLens: CodeLens, token: CancellationToken): ProviderResult<CodeLens> {
    return codeLens;
  }
}

/**
 * Register CodeLens provider
 */
export function registerCodeLensProvider(
  context: vscode.ExtensionContext,
  provider: ViprCodeLensProvider
) {
  const selector = [
    { language: 'typescriptreact', scheme: 'file' },
    { language: 'javascriptreact', scheme: 'file' },
  ];

  context.subscriptions.push(vscode.languages.registerCodeLensProvider(selector, provider));

  return provider;
}
```

### Task 8.2: Quick Fix Provider

**Model:** Opus (refactoring logic)
**File:** `vipr-vscode/src/providers/quickFixProvider.ts`

```typescript
import * as vscode from 'vscode';
import {
  CodeAction,
  CodeActionKind,
  CodeActionProvider,
  TextDocument,
  Range,
  WorkspaceEdit,
  Position,
} from 'vscode';

interface QuickFixContext {
  diagnosticCode: string;
  range: Range;
  document: TextDocument;
}

export class ViprQuickFixProvider implements CodeActionProvider {
  public static readonly providedCodeActionKinds = [
    CodeActionKind.QuickFix,
    CodeActionKind.Refactor,
    CodeActionKind.RefactorExtract,
  ];

  provideCodeActions(
    document: TextDocument,
    range: Range,
    context: vscode.CodeActionContext,
    token: vscode.CancellationToken
  ): vscode.ProviderResult<(vscode.CodeAction | vscode.Command)[]> {
    const actions: CodeAction[] = [];

    // Process each Vipr diagnostic
    for (const diagnostic of context.diagnostics) {
      if (diagnostic.source !== 'vipr') continue;

      const ctx: QuickFixContext = {
        diagnosticCode: String(diagnostic.code),
        range: diagnostic.range,
        document,
      };

      switch (diagnostic.code) {
        case 'hooks':
          actions.push(this.createExtractHookAction(ctx));
          break;

        case 'missing-effect-deps':
          actions.push(...this.createFixDependenciesActions(ctx, diagnostic));
          break;

        case 'missing-effect-cleanup':
          actions.push(this.createAddCleanupAction(ctx));
          break;

        case 'inline-function-props':
          actions.push(this.createExtractCallbackAction(ctx));
          break;

        case 'component-in-render':
          actions.push(this.createMoveComponentAction(ctx));
          break;

        case 'keyboard-access':
          actions.push(this.createAddKeyboardHandlerAction(ctx));
          break;
      }
    }

    return actions;
  }

  private createExtractHookAction(ctx: QuickFixContext): CodeAction {
    const action = new CodeAction('Extract to custom hook', CodeActionKind.RefactorExtract);

    action.command = {
      command: 'vipr.extractHook',
      title: 'Extract to custom hook',
      arguments: [ctx.document.uri, ctx.range],
    };

    action.isPreferred = true;

    return action;
  }

  private createFixDependenciesActions(
    ctx: QuickFixContext,
    diagnostic: vscode.Diagnostic
  ): CodeAction[] {
    const actions: CodeAction[] = [];

    // Extract missing deps from diagnostic message
    const match = diagnostic.message.match(/missing dependencies?: (.+)/i);
    if (match) {
      const missingDeps = match[1].split(', ');

      // Add missing dependencies
      const addDepsAction = new CodeAction(
        `Add ${missingDeps.join(', ')} to dependencies`,
        CodeActionKind.QuickFix
      );
      addDepsAction.isPreferred = true;
      addDepsAction.command = {
        command: 'vipr.addDependencies',
        title: 'Add missing dependencies',
        arguments: [ctx.document.uri, ctx.range, missingDeps],
      };
      actions.push(addDepsAction);

      // Alternative: Wrap in useCallback/useRef
      if (missingDeps.length === 1) {
        const wrapAction = new CodeAction(
          `Wrap ${missingDeps[0]} in useRef`,
          CodeActionKind.QuickFix
        );
        wrapAction.command = {
          command: 'vipr.wrapInRef',
          title: 'Wrap in useRef',
          arguments: [ctx.document.uri, ctx.range, missingDeps[0]],
        };
        actions.push(wrapAction);
      }
    }

    return actions;
  }

  private createAddCleanupAction(ctx: QuickFixContext): CodeAction {
    const action = new CodeAction('Add cleanup function', CodeActionKind.QuickFix);

    action.command = {
      command: 'vipr.addCleanup',
      title: 'Add cleanup function',
      arguments: [ctx.document.uri, ctx.range],
    };

    return action;
  }

  private createExtractCallbackAction(ctx: QuickFixContext): CodeAction {
    const action = new CodeAction('Extract to useCallback', CodeActionKind.RefactorExtract);

    action.command = {
      command: 'vipr.extractCallback',
      title: 'Extract to useCallback',
      arguments: [ctx.document.uri, ctx.range],
    };

    return action;
  }

  private createMoveComponentAction(ctx: QuickFixContext): CodeAction {
    const action = new CodeAction('Move component outside', CodeActionKind.RefactorExtract);

    action.command = {
      command: 'vipr.moveComponent',
      title: 'Move component outside render',
      arguments: [ctx.document.uri, ctx.range],
    };

    return action;
  }

  private createAddKeyboardHandlerAction(ctx: QuickFixContext): CodeAction {
    const action = new CodeAction('Add keyboard handler', CodeActionKind.QuickFix);

    // Create workspace edit to add onKeyDown
    const edit = new WorkspaceEdit();

    // This is simplified - real implementation would parse JSX
    const line = ctx.document.lineAt(ctx.range.start.line);
    const onClickMatch = line.text.match(/onClick=\{([^}]+)\}/);

    if (onClickMatch) {
      const callback = onClickMatch[1];
      const insertPosition = new Position(
        ctx.range.start.line,
        onClickMatch.index! + onClickMatch[0].length
      );

      edit.insert(
        ctx.document.uri,
        insertPosition,
        ` onKeyDown={(e) => { if (e.key === 'Enter' || e.key === ' ') ${callback}(e); }} tabIndex={0} role="button"`
      );
    }

    action.edit = edit;
    action.isPreferred = true;

    return action;
  }
}

/**
 * Register Quick Fix provider
 */
export function registerQuickFixProvider(context: vscode.ExtensionContext) {
  const selector = [
    { language: 'typescriptreact', scheme: 'file' },
    { language: 'javascriptreact', scheme: 'file' },
  ];

  context.subscriptions.push(
    vscode.languages.registerCodeActionsProvider(selector, new ViprQuickFixProvider(), {
      providedCodeActionKinds: ViprQuickFixProvider.providedCodeActionKinds,
    })
  );
}
```

### Task 8.3: Refactoring Commands

**Model:** Opus (AST manipulation)
**File:** `vipr-vscode/src/commands/refactorCommands.ts`

```typescript
import * as vscode from 'vscode';

/**
 * Register all refactoring commands
 */
export function registerRefactorCommands(context: vscode.ExtensionContext) {
  // Extract to custom hook
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'vipr.extractHook',
      async (uri: vscode.Uri, range: vscode.Range) => {
        const editor = vscode.window.activeTextEditor;
        if (!editor) return;

        const hookName = await vscode.window.showInputBox({
          prompt: 'Enter custom hook name',
          placeHolder: 'useCustomHook',
          validateInput: value => {
            if (!value.startsWith('use')) {
              return "Hook name must start with 'use'";
            }
            if (!/^use[A-Z]/.test(value)) {
              return 'Hook name should be camelCase starting with use';
            }
            return null;
          },
        });

        if (!hookName) return;

        // Would call refactoring engine
        vscode.window.showInformationMessage(
          `Extracting to ${hookName}... (Implementation pending)`
        );
      }
    )
  );

  // Add missing dependencies
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'vipr.addDependencies',
      async (uri: vscode.Uri, range: vscode.Range, deps: string[]) => {
        const editor = vscode.window.activeTextEditor;
        if (!editor) return;

        // Find the dependency array on this line
        const line = editor.document.lineAt(range.start.line);
        const text = line.text;

        // Simple regex to find empty deps array or deps with items
        const emptyDepsMatch = text.match(/\[\s*\]/);
        const depsMatch = text.match(/\[([^\]]+)\]/);

        if (emptyDepsMatch) {
          // Replace empty array with deps
          const newText = text.replace(/\[\s*\]/, `[${deps.join(', ')}]`);
          await editor.edit(builder => {
            builder.replace(line.range, newText);
          });
        } else if (depsMatch) {
          // Add to existing deps
          const existingDeps = depsMatch[1].trim();
          const newDeps = existingDeps ? `${existingDeps}, ${deps.join(', ')}` : deps.join(', ');
          const newText = text.replace(depsMatch[0], `[${newDeps}]`);
          await editor.edit(builder => {
            builder.replace(line.range, newText);
          });
        }

        vscode.window.showInformationMessage(`Added dependencies: ${deps.join(', ')}`);
      }
    )
  );

  // Add cleanup function
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'vipr.addCleanup',
      async (uri: vscode.Uri, range: vscode.Range) => {
        const editor = vscode.window.activeTextEditor;
        if (!editor) return;

        // This is a simplified implementation
        // Real version would parse the effect and generate appropriate cleanup
        const cleanupTemplate = `
    return () => {
      // Cleanup logic here
    };`;

        // Would insert before the closing brace of the effect
        vscode.window.showInformationMessage('Add cleanup function... (Implementation pending)');
      }
    )
  );

  // Extract to useCallback
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'vipr.extractCallback',
      async (uri: vscode.Uri, range: vscode.Range) => {
        const editor = vscode.window.activeTextEditor;
        if (!editor) return;

        // Would extract inline function to useCallback
        vscode.window.showInformationMessage(
          'Extracting to useCallback... (Implementation pending)'
        );
      }
    )
  );

  // Move component outside
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'vipr.moveComponent',
      async (uri: vscode.Uri, range: vscode.Range) => {
        const editor = vscode.window.activeTextEditor;
        if (!editor) return;

        // Would move nested component definition outside parent
        vscode.window.showInformationMessage('Moving component... (Implementation pending)');
      }
    )
  );
}
```

### Task 8.4: Sidebar Panel

**Model:** Opus (Webview API)
**File:** `vipr-vscode/src/views/sidebarPanel.ts`

```typescript
import * as vscode from 'vscode';

interface ProjectMetrics {
  totalFiles: number;
  averageScore: number;
  gradeDistribution: Record<string, number>;
  hotspots: Array<{
    file: string;
    score: number;
    grade: string;
  }>;
}

export class ViprSidebarProvider implements vscode.WebviewViewProvider {
  public static readonly viewType = 'vipr.sidebarView';

  private view?: vscode.WebviewView;
  private metrics?: ProjectMetrics;

  constructor(private readonly extensionUri: vscode.Uri) {}

  public resolveWebviewView(
    webviewView: vscode.WebviewView,
    context: vscode.WebviewViewResolveContext,
    token: vscode.CancellationToken
  ) {
    this.view = webviewView;

    webviewView.webview.options = {
      enableScripts: true,
      localResourceRoots: [this.extensionUri],
    };

    webviewView.webview.html = this.getHtmlContent();

    // Handle messages from webview
    webviewView.webview.onDidReceiveMessage(async message => {
      switch (message.command) {
        case 'openFile':
          const doc = await vscode.workspace.openTextDocument(message.path);
          vscode.window.showTextDocument(doc);
          break;
        case 'refresh':
          this.refresh();
          break;
      }
    });
  }

  public updateMetrics(metrics: ProjectMetrics) {
    this.metrics = metrics;
    if (this.view) {
      this.view.webview.postMessage({
        command: 'updateMetrics',
        metrics,
      });
    }
  }

  public refresh() {
    if (this.view) {
      this.view.webview.html = this.getHtmlContent();
    }
  }

  private getHtmlContent(): string {
    const metrics = this.metrics;

    return `<!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Vipr Dashboard</title>
      <style>
        body {
          font-family: var(--vscode-font-family);
          padding: 10px;
          color: var(--vscode-foreground);
        }
        .header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 15px;
        }
        .refresh-btn {
          background: var(--vscode-button-background);
          color: var(--vscode-button-foreground);
          border: none;
          padding: 5px 10px;
          cursor: pointer;
          border-radius: 3px;
        }
        .metric-card {
          background: var(--vscode-editor-background);
          border: 1px solid var(--vscode-panel-border);
          border-radius: 5px;
          padding: 10px;
          margin-bottom: 10px;
        }
        .metric-value {
          font-size: 24px;
          font-weight: bold;
        }
        .metric-label {
          font-size: 12px;
          opacity: 0.8;
        }
        .grade-a { color: #4caf50; }
        .grade-b { color: #8bc34a; }
        .grade-c { color: #ffc107; }
        .grade-d { color: #ff9800; }
        .grade-f { color: #f44336; }
        .hotspot-list {
          list-style: none;
          padding: 0;
        }
        .hotspot-item {
          display: flex;
          justify-content: space-between;
          padding: 8px;
          border-bottom: 1px solid var(--vscode-panel-border);
          cursor: pointer;
        }
        .hotspot-item:hover {
          background: var(--vscode-list-hoverBackground);
        }
        .section-title {
          font-size: 14px;
          font-weight: bold;
          margin: 15px 0 10px;
        }
      </style>
    </head>
    <body>
      <div class="header">
        <h3>Vipr Dashboard</h3>
        <button class="refresh-btn" onclick="refresh()">Refresh</button>
      </div>

      ${
        metrics
          ? `
        <div class="metric-card">
          <div class="metric-value">${metrics.totalFiles}</div>
          <div class="metric-label">React Files</div>
        </div>

        <div class="metric-card">
          <div class="metric-value">${metrics.averageScore.toFixed(1)}</div>
          <div class="metric-label">Average Complexity</div>
        </div>

        <div class="section-title">Grade Distribution</div>
        <div class="metric-card">
          ${Object.entries(metrics.gradeDistribution)
            .map(
              ([grade, count]) =>
                `<span class="grade-${grade.toLowerCase()}">${grade}: ${count}</span> `
            )
            .join('')}
        </div>

        <div class="section-title">Hotspots</div>
        <ul class="hotspot-list">
          ${metrics.hotspots
            .slice(0, 10)
            .map(
              hotspot => `
            <li class="hotspot-item" onclick="openFile('${hotspot.file}')">
              <span>${hotspot.file.split('/').pop()}</span>
              <span class="grade-${hotspot.grade.toLowerCase()}">${hotspot.score} (${hotspot.grade})</span>
            </li>
          `
            )
            .join('')}
        </ul>
      `
          : `
        <p>Run workspace analysis to see metrics.</p>
        <button class="refresh-btn" onclick="refresh()">Analyze Workspace</button>
      `
      }

      <script>
        const vscode = acquireVsCodeApi();

        function refresh() {
          vscode.postMessage({ command: 'refresh' });
        }

        function openFile(path) {
          vscode.postMessage({ command: 'openFile', path });
        }

        window.addEventListener('message', event => {
          const message = event.data;
          if (message.command === 'updateMetrics') {
            // Would re-render with new metrics
            location.reload();
          }
        });
      </script>
    </body>
    </html>`;
  }
}

/**
 * Register sidebar panel
 */
export function registerSidebarPanel(context: vscode.ExtensionContext) {
  const provider = new ViprSidebarProvider(context.extensionUri);

  context.subscriptions.push(
    vscode.window.registerWebviewViewProvider(ViprSidebarProvider.viewType, provider)
  );

  return provider;
}
```

### Task 8.5: Editor Decorations

**Model:** Sonnet (decoration patterns)
**File:** `vipr-vscode/src/decorations/complexityDecorations.ts`

```typescript
import * as vscode from 'vscode';

// Decoration types for different complexity levels
const lowComplexityDecoration = vscode.window.createTextEditorDecorationType({
  backgroundColor: 'rgba(76, 175, 80, 0.1)',
  isWholeLine: true,
});

const mediumComplexityDecoration = vscode.window.createTextEditorDecorationType({
  backgroundColor: 'rgba(255, 193, 7, 0.1)',
  isWholeLine: true,
});

const highComplexityDecoration = vscode.window.createTextEditorDecorationType({
  backgroundColor: 'rgba(244, 67, 54, 0.1)',
  isWholeLine: true,
});

// Gutter icons
const issueGutterDecoration = vscode.window.createTextEditorDecorationType({
  gutterIconPath: vscode.Uri.parse('data:image/svg+xml,...'), // SVG icon
  gutterIconSize: 'contain',
});

interface ComplexityRange {
  range: vscode.Range;
  score: number;
  message: string;
}

export class ComplexityDecorationManager {
  private disposables: vscode.Disposable[] = [];

  constructor() {
    // Listen for editor changes
    this.disposables.push(
      vscode.window.onDidChangeActiveTextEditor(editor => {
        if (editor) {
          this.updateDecorations(editor);
        }
      })
    );
  }

  updateDecorations(editor: vscode.TextEditor, ranges?: ComplexityRange[]) {
    if (!ranges) {
      // Clear decorations
      editor.setDecorations(lowComplexityDecoration, []);
      editor.setDecorations(mediumComplexityDecoration, []);
      editor.setDecorations(highComplexityDecoration, []);
      return;
    }

    const low: vscode.DecorationOptions[] = [];
    const medium: vscode.DecorationOptions[] = [];
    const high: vscode.DecorationOptions[] = [];

    ranges.forEach(({ range, score, message }) => {
      const decoration: vscode.DecorationOptions = {
        range,
        hoverMessage: new vscode.MarkdownString(`**Complexity:** ${score}\n\n${message}`),
      };

      if (score < 30) {
        low.push(decoration);
      } else if (score < 50) {
        medium.push(decoration);
      } else {
        high.push(decoration);
      }
    });

    editor.setDecorations(lowComplexityDecoration, low);
    editor.setDecorations(mediumComplexityDecoration, medium);
    editor.setDecorations(highComplexityDecoration, high);
  }

  dispose() {
    this.disposables.forEach(d => d.dispose());
    lowComplexityDecoration.dispose();
    mediumComplexityDecoration.dispose();
    highComplexityDecoration.dispose();
  }
}
```

## Acceptance Criteria

### Feature Requirements

- [ ] CodeLens shows complexity above components
- [ ] Quick fixes available for common issues
- [ ] Sidebar shows project-wide metrics
- [ ] Editor decorations highlight complex areas
- [ ] All quick fixes apply correct transformations

### User Experience

- [ ] CodeLens clickable and shows details
- [ ] Quick fixes appear in lightbulb menu
- [ ] Sidebar updates automatically
- [ ] Decorations don't impact performance

## Testing Instructions

### Manual Testing

1. **Test CodeLens**
   - Open a .tsx file with components
   - Verify complexity shown above function declarations
   - Click CodeLens to see details

2. **Test Quick Fixes**
   - Create file with complexity issues
   - Click lightbulb on diagnostic
   - Verify quick fix options appear
   - Apply fix and verify result

3. **Test Sidebar**
   - Open View > Vipr Dashboard
   - Run workspace analysis
   - Verify metrics display
   - Click hotspot to navigate

4. **Test Decorations**
   - Open complex component
   - Verify background highlighting
   - Hover for complexity info

## Estimated Effort

| Task                     | Model  | Estimated Time  |
| ------------------------ | ------ | --------------- |
| 8.1 CodeLens Provider    | Opus   | 3 hours         |
| 8.2 Quick Fix Provider   | Opus   | 4 hours         |
| 8.3 Refactoring Commands | Opus   | 4 hours         |
| 8.4 Sidebar Panel        | Opus   | 4 hours         |
| 8.5 Editor Decorations   | Sonnet | 2 hours         |
| Integration Testing      | Sonnet | 2 hours         |
| Documentation            | Haiku  | 1 hour          |
| **Total**                |        | **20-22 hours** |

---

**Document Version:** 1.0
**Created:** January 10, 2026
**Status:** Ready for Implementation
