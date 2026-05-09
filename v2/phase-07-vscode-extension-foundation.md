# Phase 07: VS Code Extension Foundation

**Priority:** High - Developer experience  
**Complexity:** Medium  
**Dependencies:** Phase 00 (Foundation) ✅  
**Status:** Ready for implementation

**Phase Duration:** 5-6 days  
**Priority:** High - Developer experience  
**Complexity:** High  
**Dependencies:** Phase 00-06 (Core analyzers complete)

All analyzer integrations use the `ts-morph`-based API.

## Overview

This phase establishes the VS Code extension foundation using the Language Server Protocol (LSP). It creates the server-client architecture that enables real-time analysis, diagnostics, and integration with VS Code's editor features.

## Business Value

- Real-time complexity feedback during development
- IDE-integrated analysis (no separate CLI step)
- Immediate visibility into code health
- Foundation for advanced features (quick fixes, CodeLens)
- Professional developer tooling experience

## Agent Assignments

| Agent                  | Role                                | Capacity  |
| ---------------------- | ----------------------------------- | --------- |
| vscode-plugin-engineer | Lead implementer, VS Code expertise | Primary   |
| typescript-engineer    | LSP implementation, type safety     | Secondary |
| react-engineer         | Analyzer integration                | Advisory  |

## Execution Strategy

### Milestone 7.1: Project Setup (Day 1)

**Synchronous Tasks:**

1. Initialize extension project (vscode-plugin-engineer)
2. Configure TypeScript and build (typescript-engineer)
3. Set up package.json manifest (Haiku)

### Milestone 7.2: Language Server (Day 2-3)

**Synchronous Tasks:**

1. LSP server implementation (vscode-plugin-engineer)
2. Document analysis integration (typescript-engineer)
3. Diagnostics conversion (Sonnet)

### Milestone 7.3: Extension Client (Day 3-4)

**Parallel Tasks:**

- Client initialization (vscode-plugin-engineer)
- Status bar integration (Sonnet)
- Basic commands (Sonnet)

### Milestone 7.4: Configuration & Testing (Day 5-6)

**Parallel Tasks:**

- Settings implementation (Sonnet)
- Extension tests (Sonnet)
- Documentation (Haiku)

## Detailed Tasks

### Task 7.1: Project Structure

**Model:** Haiku (file creation)

```
vipr-vscode/
├── .vscode/
│   ├── launch.json           # Debug configurations
│   └── tasks.json             # Build tasks
├── src/
│   ├── extension.ts           # Extension entry point
│   ├── client/
│   │   ├── index.ts           # Client initialization
│   │   └── statusBar.ts       # Status bar component
│   └── server/
│       ├── server.ts          # LSP server entry
│       ├── analyzer.ts        # Analyzer wrapper
│       └── diagnostics.ts     # Diagnostics conversion
├── package.json               # Extension manifest
├── tsconfig.json              # TypeScript config
├── tsconfig.server.json       # Server TypeScript config
├── esbuild.js                 # Build configuration
└── README.md                  # Extension documentation
```

### Task 7.2: Extension Manifest

**Model:** Haiku (configuration)
**File:** `vipr-vscode/package.json`

```json
{
  "name": "vipr-react-analyzer",
  "displayName": "Vipr React Analyzer",
  "description": "Real-time React complexity analysis and insights",
  "version": "0.1.0",
  "publisher": "vipr",
  "engines": {
    "vscode": "^1.85.0"
  },
  "categories": ["Linters", "Programming Languages"],
  "keywords": ["react", "complexity", "analysis", "typescript", "javascript"],
  "activationEvents": ["onLanguage:typescriptreact", "onLanguage:javascriptreact"],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "vipr.analyzeWorkspace",
        "title": "Analyze Workspace",
        "category": "Vipr"
      },
      {
        "command": "vipr.showComplexity",
        "title": "Show Complexity Details",
        "category": "Vipr"
      },
      {
        "command": "vipr.generateReport",
        "title": "Generate Complexity Report",
        "category": "Vipr"
      }
    ],
    "configuration": {
      "title": "Vipr React Analyzer",
      "properties": {
        "vipr.enable": {
          "type": "boolean",
          "default": true,
          "description": "Enable/disable Vipr analysis"
        },
        "vipr.showStatusBar": {
          "type": "boolean",
          "default": true,
          "description": "Show complexity score in status bar"
        },
        "vipr.complexityThreshold": {
          "type": "number",
          "default": 50,
          "description": "Complexity score threshold for warnings"
        },
        "vipr.analyzers": {
          "type": "array",
          "default": ["core", "migration", "performance", "antipatterns"],
          "description": "Analyzers to run",
          "items": {
            "type": "string",
            "enum": ["core", "migration", "performance", "antipatterns", "security", "a11y"]
          }
        },
        "vipr.reactVersion": {
          "type": "string",
          "default": "auto",
          "description": "Target React version for migration analysis"
        }
      }
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run build",
    "build": "node esbuild.js",
    "watch": "node esbuild.js --watch",
    "test": "vscode-test",
    "lint": "eslint src --ext ts"
  },
  "dependencies": {
    "vscode-languageclient": "^9.0.1",
    "vscode-languageserver": "^9.0.1",
    "vscode-languageserver-textdocument": "^1.0.11"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "@types/vscode": "^1.85.0",
    "@vscode/test-electron": "^2.3.8",
    "esbuild": "^0.19.0",
    "typescript": "^5.3.0"
  }
}
```

### Task 7.3: Language Server Implementation

**Model:** Opus (core LSP logic)
**File:** `vipr-vscode/src/server/server.ts`

```typescript
import {
  createConnection,
  TextDocuments,
  Diagnostic,
  DiagnosticSeverity,
  ProposedFeatures,
  InitializeParams,
  DidChangeConfigurationNotification,
  TextDocumentSyncKind,
  InitializeResult,
  CodeAction,
  CodeActionKind,
  Command,
} from 'vscode-languageserver/node';

import { TextDocument } from 'vscode-languageserver-textdocument';

// Import analyzer (would be from built package)
// import { analyzeReactComplexity, ReactComplexityResult } from "@vipr/react-analyzer";

interface ViprSettings {
  enable: boolean;
  complexityThreshold: number;
  analyzers: string[];
  reactVersion: string;
}

const defaultSettings: ViprSettings = {
  enable: true,
  complexityThreshold: 50,
  analyzers: ['core', 'migration', 'performance', 'antipatterns'],
  reactVersion: 'auto',
};

// Create LSP connection
const connection = createConnection(ProposedFeatures.all);
const documents: TextDocuments<TextDocument> = new TextDocuments(TextDocument);

let globalSettings: ViprSettings = defaultSettings;
const documentSettings: Map<string, Thenable<ViprSettings>> = new Map();

let hasConfigurationCapability = false;
let hasWorkspaceFolderCapability = false;
let hasDiagnosticRelatedInformationCapability = false;

connection.onInitialize((params: InitializeParams) => {
  const capabilities = params.capabilities;

  hasConfigurationCapability = !!(capabilities.workspace && !!capabilities.workspace.configuration);
  hasWorkspaceFolderCapability = !!(
    capabilities.workspace && !!capabilities.workspace.workspaceFolders
  );
  hasDiagnosticRelatedInformationCapability = !!(
    capabilities.textDocument &&
    capabilities.textDocument.publishDiagnostics &&
    capabilities.textDocument.publishDiagnostics.relatedInformation
  );

  const result: InitializeResult = {
    capabilities: {
      textDocumentSync: TextDocumentSyncKind.Incremental,
      completionProvider: undefined,
      codeActionProvider: {
        codeActionKinds: [CodeActionKind.QuickFix, CodeActionKind.Refactor],
      },
      hoverProvider: true,
      executeCommandProvider: {
        commands: ['vipr.showDetails'],
      },
    },
  };

  if (hasWorkspaceFolderCapability) {
    result.capabilities.workspace = {
      workspaceFolders: {
        supported: true,
      },
    };
  }

  return result;
});

connection.onInitialized(() => {
  if (hasConfigurationCapability) {
    connection.client.register(DidChangeConfigurationNotification.type, undefined);
  }
  if (hasWorkspaceFolderCapability) {
    connection.workspace.onDidChangeWorkspaceFolders(_event => {
      connection.console.log('Workspace folder change event received.');
    });
  }
});

connection.onDidChangeConfiguration(change => {
  if (hasConfigurationCapability) {
    documentSettings.clear();
  } else {
    globalSettings = (change.settings.vipr || defaultSettings) as ViprSettings;
  }

  // Re-analyze all open documents
  documents.all().forEach(analyzeDocument);
});

function getDocumentSettings(resource: string): Thenable<ViprSettings> {
  if (!hasConfigurationCapability) {
    return Promise.resolve(globalSettings);
  }
  let result = documentSettings.get(resource);
  if (!result) {
    result = connection.workspace.getConfiguration({
      scopeUri: resource,
      section: 'vipr',
    });
    documentSettings.set(resource, result);
  }
  return result;
}

// Document lifecycle
documents.onDidClose(e => {
  documentSettings.delete(e.document.uri);
  // Clear diagnostics when document closes
  connection.sendDiagnostics({ uri: e.document.uri, diagnostics: [] });
});

documents.onDidChangeContent(change => {
  analyzeDocument(change.document);
});

async function analyzeDocument(textDocument: TextDocument): Promise<void> {
  const settings = await getDocumentSettings(textDocument.uri);

  if (!settings.enable) {
    return;
  }

  const uri = textDocument.uri;

  // Only analyze React files
  if (!uri.endsWith('.tsx') && !uri.endsWith('.jsx')) {
    return;
  }

  const text = textDocument.getText();
  const diagnostics: Diagnostic[] = [];

  try {
    // Placeholder - would call actual analyzer
    // const result = analyzeReactComplexity(text);
    // diagnostics.push(...convertToDiagnostics(result, textDocument));

    // For now, demonstrate structure
    const mockResult = {
      total: 45,
      grade: 'C',
      insights: [
        {
          severity: 'warning',
          category: 'hooks',
          message: 'Component has 8 hooks',
          line: 10,
          suggestion: 'Consider extracting into custom hooks',
        },
      ],
    };

    mockResult.insights.forEach(insight => {
      const severity =
        insight.severity === 'critical'
          ? DiagnosticSeverity.Error
          : insight.severity === 'warning'
            ? DiagnosticSeverity.Warning
            : DiagnosticSeverity.Information;

      const line = insight.line ? insight.line - 1 : 0;

      const diagnostic: Diagnostic = {
        severity,
        range: {
          start: { line, character: 0 },
          end: { line, character: Number.MAX_SAFE_INTEGER },
        },
        message: insight.message,
        source: 'vipr',
        code: insight.category,
      };

      if (insight.suggestion && hasDiagnosticRelatedInformationCapability) {
        diagnostic.relatedInformation = [
          {
            location: {
              uri: textDocument.uri,
              range: diagnostic.range,
            },
            message: insight.suggestion,
          },
        ];
      }

      diagnostics.push(diagnostic);
    });

    // Add complexity score as information diagnostic
    if (mockResult.total > settings.complexityThreshold) {
      diagnostics.push({
        severity: DiagnosticSeverity.Warning,
        range: {
          start: { line: 0, character: 0 },
          end: { line: 0, character: Number.MAX_SAFE_INTEGER },
        },
        message: `Complexity score: ${mockResult.total} (Grade: ${mockResult.grade}) - Exceeds threshold of ${settings.complexityThreshold}`,
        source: 'vipr',
        code: 'complexity',
      });
    }

    connection.sendDiagnostics({ uri: textDocument.uri, diagnostics });

    // Send custom notification for status bar update
    connection.sendNotification('vipr/complexityUpdate', {
      uri: textDocument.uri,
      score: mockResult.total,
      grade: mockResult.grade,
    });
  } catch (error) {
    connection.console.error(`Error analyzing ${uri}: ${error}`);
  }
}

// Code actions (quick fixes)
connection.onCodeAction(params => {
  const textDocument = documents.get(params.textDocument.uri);
  if (!textDocument) {
    return [];
  }

  const codeActions: CodeAction[] = [];

  params.context.diagnostics.forEach(diagnostic => {
    if (diagnostic.source !== 'vipr') return;

    // Example: Extract to custom hook suggestion
    if (diagnostic.code === 'hooks') {
      codeActions.push({
        title: 'Extract to custom hook',
        kind: CodeActionKind.Refactor,
        diagnostics: [diagnostic],
        command: {
          command: 'vipr.extractHook',
          title: 'Extract to custom hook',
          arguments: [params.textDocument.uri, diagnostic.range],
        },
      });
    }
  });

  return codeActions;
});

// Hover information
connection.onHover(params => {
  // Could show complexity details on hover
  return null;
});

// Make the text document manager listen on the connection
documents.listen(connection);

// Listen on the connection
connection.listen();
```

### Task 7.4: Extension Client

**Model:** Opus (VS Code API)
**File:** `vipr-vscode/src/extension.ts`

```typescript
import * as path from 'path';
import {
  ExtensionContext,
  window,
  commands,
  StatusBarAlignment,
  StatusBarItem,
  workspace,
  OutputChannel,
} from 'vscode';
import {
  LanguageClient,
  LanguageClientOptions,
  ServerOptions,
  TransportKind,
} from 'vscode-languageclient/node';

let client: LanguageClient;
let statusBarItem: StatusBarItem;
let outputChannel: OutputChannel;

// Complexity data cache
interface ComplexityData {
  score: number;
  grade: string;
}

const complexityCache = new Map<string, ComplexityData>();

export function activate(context: ExtensionContext) {
  outputChannel = window.createOutputChannel('Vipr React Analyzer');
  outputChannel.appendLine('Vipr React Analyzer activating...');

  // Start language server
  startLanguageServer(context);

  // Create status bar
  createStatusBar(context);

  // Register commands
  registerCommands(context);

  outputChannel.appendLine('Vipr React Analyzer activated');
}

function startLanguageServer(context: ExtensionContext) {
  const serverModule = context.asAbsolutePath(path.join('out', 'server', 'server.js'));

  const debugOptions = { execArgv: ['--nolazy', '--inspect=6009'] };

  const serverOptions: ServerOptions = {
    run: { module: serverModule, transport: TransportKind.ipc },
    debug: {
      module: serverModule,
      transport: TransportKind.ipc,
      options: debugOptions,
    },
  };

  const clientOptions: LanguageClientOptions = {
    documentSelector: [
      { scheme: 'file', language: 'typescriptreact' },
      { scheme: 'file', language: 'javascriptreact' },
    ],
    synchronize: {
      fileEvents: workspace.createFileSystemWatcher('**/*.{tsx,jsx}'),
    },
    outputChannel,
  };

  client = new LanguageClient(
    'viprReactAnalyzer',
    'Vipr React Analyzer',
    serverOptions,
    clientOptions
  );

  // Handle custom notifications
  client.onReady().then(() => {
    client.onNotification('vipr/complexityUpdate', (params: any) => {
      const { uri, score, grade } = params;
      complexityCache.set(uri, { score, grade });
      updateStatusBar(score, grade);
    });
  });

  client.start();
}

function createStatusBar(context: ExtensionContext) {
  statusBarItem = window.createStatusBarItem(StatusBarAlignment.Right, 100);
  statusBarItem.command = 'vipr.showComplexity';
  statusBarItem.tooltip = 'Click for complexity details';
  context.subscriptions.push(statusBarItem);

  // Show/hide based on active editor
  context.subscriptions.push(
    window.onDidChangeActiveTextEditor(editor => {
      if (editor) {
        const uri = editor.document.uri.toString();
        const cached = complexityCache.get(uri);
        if (cached) {
          updateStatusBar(cached.score, cached.grade);
        } else {
          statusBarItem.hide();
        }
      } else {
        statusBarItem.hide();
      }
    })
  );

  // Check config for visibility
  const config = workspace.getConfiguration('vipr');
  if (config.get('showStatusBar')) {
    statusBarItem.show();
  }
}

function updateStatusBar(score: number, grade: string) {
  const config = workspace.getConfiguration('vipr');
  if (!config.get('showStatusBar')) {
    statusBarItem.hide();
    return;
  }

  const threshold = config.get('complexityThreshold', 50);

  // Color based on grade
  let icon = '';
  let backgroundColor = '';

  switch (grade) {
    case 'A':
      icon = '$(check)';
      backgroundColor = 'statusBarItem.warningBackground'; // Actually green
      break;
    case 'B':
      icon = '$(info)';
      break;
    case 'C':
      icon = '$(warning)';
      backgroundColor = 'statusBarItem.warningBackground';
      break;
    case 'D':
    case 'F':
      icon = '$(error)';
      backgroundColor = 'statusBarItem.errorBackground';
      break;
    default:
      icon = '$(symbol-misc)';
  }

  statusBarItem.text = `${icon} Vipr: ${score} (${grade})`;
  statusBarItem.backgroundColor = score > threshold ? backgroundColor : undefined;
  statusBarItem.show();
}

function registerCommands(context: ExtensionContext) {
  // Show complexity details
  context.subscriptions.push(
    commands.registerCommand('vipr.showComplexity', () => {
      const editor = window.activeTextEditor;
      if (!editor) {
        window.showInformationMessage('No active editor');
        return;
      }

      const uri = editor.document.uri.toString();
      const cached = complexityCache.get(uri);

      if (cached) {
        window.showInformationMessage(`Complexity Score: ${cached.score} | Grade: ${cached.grade}`);
      } else {
        window.showInformationMessage('No complexity data available');
      }
    })
  );

  // Analyze workspace
  context.subscriptions.push(
    commands.registerCommand('vipr.analyzeWorkspace', async () => {
      window.showInformationMessage('Analyzing workspace...');
      // Would trigger workspace-wide analysis
    })
  );

  // Generate report
  context.subscriptions.push(
    commands.registerCommand('vipr.generateReport', async () => {
      window.showInformationMessage('Generating complexity report...');
      // Would generate and show report
    })
  );
}

export function deactivate(): Thenable<void> | undefined {
  if (!client) {
    return undefined;
  }
  return client.stop();
}
```

### Task 7.5: Build Configuration

**Model:** Haiku (configuration)
**File:** `vipr-vscode/esbuild.js`

```javascript
const esbuild = require('esbuild');

const watch = process.argv.includes('--watch');

async function build() {
  const ctx = await esbuild.context({
    entryPoints: ['src/extension.ts', 'src/server/server.ts'],
    bundle: true,
    outdir: 'out',
    external: ['vscode'],
    format: 'cjs',
    platform: 'node',
    sourcemap: true,
    minify: !watch,
  });

  if (watch) {
    await ctx.watch();
    console.log('Watching for changes...');
  } else {
    await ctx.rebuild();
    await ctx.dispose();
  }
}

build().catch(() => process.exit(1));
```

## Acceptance Criteria

### Extension Requirements

- [ ] Extension activates on .tsx/.jsx files
- [ ] Language server starts and connects
- [ ] Diagnostics appear in Problems panel
- [ ] Status bar shows complexity score
- [ ] Commands registered and functional
- [ ] Settings configurable

### Performance

- [ ] Activation < 1 second
- [ ] Analysis < 200ms after file change
- [ ] No memory leaks on document close

### Quality

- [ ] No console errors
- [ ] Graceful error handling
- [ ] Respects user settings

## Testing Instructions

### Manual Testing

1. **Install Extension**

   ```bash
   cd vipr-vscode
   npm install
   npm run build
   # Press F5 in VS Code to launch Extension Development Host
   ```

2. **Test Activation**
   - Open a .tsx file
   - Check Output panel for "Vipr React Analyzer"
   - Verify status bar item appears

3. **Test Diagnostics**
   - Open sample component with complexity issues
   - Check Problems panel for diagnostics
   - Hover over underlined code for suggestions

4. **Test Commands**
   - Open Command Palette (Ctrl+Shift+P)
   - Type "Vipr" and verify commands appear
   - Run "Show Complexity Details"

5. **Test Settings**
   - Open Settings (Ctrl+,)
   - Search "Vipr"
   - Change threshold and verify behavior

### Automated Tests

**File:** `vipr-vscode/src/test/extension.test.ts`

```typescript
import * as assert from 'assert';
import * as vscode from 'vscode';

suite('Extension Test Suite', () => {
  test('Extension should be present', () => {
    assert.ok(vscode.extensions.getExtension('vipr.vipr-react-analyzer'));
  });

  test('Commands should be registered', async () => {
    const commands = await vscode.commands.getCommands();
    assert.ok(commands.includes('vipr.showComplexity'));
    assert.ok(commands.includes('vipr.analyzeWorkspace'));
  });
});
```

## Dependencies

- Phases 00-06 must be complete (analyzers to integrate)
- Node.js 20.18.1+ for build
- VS Code 1.85+ for testing

## Downstream Impact

- Phase 08 builds on this foundation
- Extension becomes primary analyzer interface

## Estimated Effort

| Task                   | Model  | Estimated Time  |
| ---------------------- | ------ | --------------- |
| 7.1 Project Setup      | Haiku  | 1 hour          |
| 7.2 Extension Manifest | Haiku  | 1 hour          |
| 7.3 Language Server    | Opus   | 4 hours         |
| 7.4 Extension Client   | Opus   | 3 hours         |
| 7.5 Build Config       | Haiku  | 0.5 hours       |
| 7.6 Status Bar         | Sonnet | 1.5 hours       |
| 7.7 Commands           | Sonnet | 1.5 hours       |
| Extension Tests        | Sonnet | 2 hours         |
| Documentation          | Haiku  | 1 hour          |
| **Total**              |        | **15-17 hours** |

---

**Document Version:** 1.0
**Created:** January 10, 2026
**Status:** Ready for Implementation
