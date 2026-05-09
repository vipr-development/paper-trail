# Building sophisticated VSCode extensions with ML and rich UI

A **technical debt analysis extension** with CodeLens, visualizations, and ML-powered insights is achievable without a massive bundle. The key is a hybrid architecture: small bundled core with lazy-loaded models, Copilot integration for explanations, and webview dashboards using community UI components. VSCode's extension APIs as of 2025-2026 support all these capabilities, though Microsoft's deprecation of the official webview toolkit and increasing restrictions on fork marketplaces require attention.

## Rich UI views have evolved beyond Microsoft's toolkit

Microsoft's `@vscode/webview-ui-toolkit` was deprecated on January 1, 2025. The recommended replacement is **@vscode-elements/elements**, a Lit-based community library that matches VSCode's design language. For your analysis dashboard, combine this with Chart.js or D3.js for visualizations.

The extension architecture for rich UIs follows this pattern:

```
Extension Host              Webview (Dashboard)
┌──────────────────┐        ┌──────────────────────────────┐
│ Analysis Engine  │ ←───►  │ React/Vue + @vscode-elements │
│ Data Provider    │  msg   │ Chart.js visualizations      │
│ CodeLens Provider│        │ Theme-aware via CSS vars     │
└──────────────────┘        └──────────────────────────────┘
```

**Webviews** are sandboxed iframes with full HTML/CSS/JS capabilities. Always set a Content Security Policy with nonces, use `webview.asWebviewUri()` for resources, and enable `retainContextWhenHidden: true` to preserve chart state. Communication uses `postMessage` bidirectionally.

**TreeViews** work better for navigation and hierarchical data (file lists, debt categories). Use TreeViews for your sidebar navigator and Webviews for the detailed analysis dashboard.

**CodeLens implementation** uses `CodeLensProvider` with an optional `resolveCodeLens` method for lazy-loading expensive commands. Fire `onDidChangeCodeLenses` to refresh displays when analysis results change. Keep titles concise—something like "**3 debt items** • 2 high priority" works well for technical debt annotations.

## Fork compatibility requires multi-marketplace publishing

Microsoft now blocks proprietary extensions (C/C++, Pylance, C# Dev Kit) from running in Cursor, Windsurf, and other forks. For community extensions like yours, the core APIs work across forks, but distribution requires attention.

**Publish to both marketplaces** using `vsce publish` for VS Code Marketplace and `ovsx publish` for Open VSX Registry. Open VSX serves VSCodium, Eclipse Theia, Gitpod, and increasingly Cursor/Windsurf users who can't access certain Microsoft extensions.

Environment detection enables fork-specific behavior:

```typescript
const appName = vscode.env.appName;
const isCursor = appName.includes('Cursor');
const isWindsurf = appName.includes('Windsurf');
```

Set a conservative engine version in `package.json` (like `"vscode": "^1.74.0"`) for maximum compatibility since forks often lag behind VS Code releases. Test webviews across all target editors—CORS issues can appear in remote development scenarios.

## Extension size has no hard limit but bundling is critical

The historical **20MB marketplace limit** no longer applies. Extensions like Microsoft Live Share ship at ~150MB. However, size directly impacts user experience: installation time, disk I/O, and especially cold activation time.

**Target under 5MB bundled** for your extension core (without ML models). Use esbuild—it's 10-100x faster than webpack:

```javascript
esbuild.build({
  entryPoints: ['src/extension.ts'],
  bundle: true,
  platform: 'node',
  external: ['vscode'],
  outfile: 'dist/extension.js',
  minify: true,
});
```

A well-configured `.vscodeignore` excludes source files, node_modules (when bundled), tests, and documentation. The Azure Account extension reduced from 6.2MB to 840KB with bundling, cutting activation time by 50%. The Docker extension went from 20 seconds cold activation to 2 seconds.

For ML models, **never bundle them in the VSIX**. Download on first use to `globalStorageUri`, which persists across sessions.

## SQLite works for storing analysis reports

VSCode extensions can use SQLite through WASM-based packages. **node-sqlite3-wasm** is the best choice—it provides direct filesystem access without loading the entire database into memory, and requires no platform-specific compilation.

```typescript
const { Database } = require('node-sqlite3-wasm');
const dbPath = vscode.Uri.joinPath(context.globalStorageUri, 'reports.db').fsPath;
const db = new Database(dbPath);

db.exec(`CREATE TABLE IF NOT EXISTS reports (
  id TEXT PRIMARY KEY,
  workspace_path TEXT,
  created_at INTEGER,
  summary TEXT,
  details TEXT
)`);
```

**Storage selection by data size:**

| Data Size      | Best Storage                     |
| -------------- | -------------------------------- |
| Under 100KB    | `globalState` (key-value)        |
| 100KB–10MB     | JSON files in `globalStorageUri` |
| Over 10MB      | SQLite via node-sqlite3-wasm     |
| Sensitive data | `context.secrets` (encrypted)    |

Avoid storing more than ~3MB in `globalState`—it can cause UI freezes. For your analysis reports, SQLite with JSON columns handles structured queries well while keeping the schema simple.

## PDF generation works with pure JavaScript libraries

**pdf-lib** is ideal for VSCode extensions: pure TypeScript, no native dependencies, works in both Node.js and browser contexts, and supports creating and modifying PDFs.

```typescript
import { PDFDocument, rgb } from 'pdf-lib';

const doc = await PDFDocument.create();
const page = doc.addPage([595, 842]); // A4
page.drawText('Technical Debt Report', { x: 50, y: 800, size: 24 });
const bytes = await doc.save();
await vscode.workspace.fs.writeFile(reportUri, Buffer.from(bytes));
```

**pdfmake** offers a declarative JSON-based approach excellent for tabular reports. For pixel-perfect HTML rendering, Puppeteer works but downloads ~200MB of Chromium—acceptable only if PDF quality is paramount.

## Copilot and AI agent APIs are now production-ready

VSCode's **Language Model API** (stable since July 2024) gives extensions direct access to Copilot's models:

```typescript
const [model] = await vscode.lm.selectChatModels({ vendor: 'copilot', family: 'gpt-4o' });
const response = await model.sendRequest([
  vscode.LanguageModelChatMessage.User('Analyze this technical debt pattern...'),
]);
```

For your extension, implement a **Chat Participant** (like `@techdebt`) for interactive Q&A:

```typescript
vscode.chat.createChatParticipant(
  'techdebt.analyzer',
  async (request, context, response, token) => {
    const debtAnalysis = await runLocalAnalysis();
    // Use Copilot to explain findings in natural language
  }
);
```

Register **Language Model Tools** so Copilot's agent mode can invoke your analysis:

```json
"contributes": {
  "languageModelTools": [{
    "name": "analyzeDebt",
    "displayName": "Analyze Technical Debt",
    "modelDescription": "Scans workspace for code anti-patterns, complexity hotspots, and debt indicators"
  }]
}
```

**MCP (Model Context Protocol)** reached general availability in VS Code 1.102. Users with local LLMs (via Ollama or llama.cpp) can connect to your analysis tools through MCP servers. This is optional but attracts power users.

Cursor and Windsurf don't expose their AI APIs to extensions—their AI features are proprietary. Your extension will work in these IDEs but won't integrate with their AI; users get Copilot integration only in VS Code proper.

## Performance requires lazy activation and worker threads

**Activation events** are critical. Never use `"*"` (startup activation). For a technical debt extension:

```json
"activationEvents": [
  "onCommand:techdebt.analyze",
  "onView:techdebt.dashboard",
  "workspaceContains:**/.git"
]
```

As of VS Code 1.74+, contributions auto-trigger activation without explicit declarations.

**Worker threads** are supported for CPU-bound tasks. Run model inference off the main thread:

```typescript
import { Worker } from 'worker_threads';

const worker = new Worker(path.join(__dirname, 'inference-worker.js'), {
  workerData: { modelPath, codeToAnalyze },
});
worker.on('message', result => updateUI(result));
```

For heavy processing, consider a **Language Server** in a separate process. It handles cancellation tokens well and can be written in any language.

Memory management matters because all extensions share a single Node.js process. Use `WeakMap` for caches, stream large files, and dispose resources via `context.subscriptions.push(disposable)`.

## Dynamic artifact downloads are fully supported

Extensions routinely download binaries, models, and data files post-installation. The C# extension downloads OmniSharp on first use; language servers commonly download platform-specific binaries.

Store downloads in `globalStorageUri`:

```typescript
const modelDir = vscode.Uri.joinPath(context.globalStorageUri, 'models');
await vscode.workspace.fs.createDirectory(modelDir);
```

**Always verify checksums** for downloaded binaries:

```typescript
const hash = crypto.createHash('sha256').update(fileBuffer).digest('hex');
if (hash !== expectedSHA256) throw new Error('Integrity check failed');
```

Show progress with cancellation support:

```typescript
await vscode.window.withProgress(
  {
    location: vscode.ProgressLocation.Notification,
    title: 'Downloading analysis models',
    cancellable: true,
  },
  async (progress, token) => {
    // Download with progress.report({ increment, message })
  }
);
```

**Security context**: VSCode extensions have no sandboxing. Downloaded binaries execute with full user permissions. Be transparent about what you download and why.

## ML inference works best with Transformers.js and ONNX Runtime

**Transformers.js** (Hugging Face) is ideal for VSCode extensions. It uses ONNX Runtime Web internally, supports WebGPU acceleration, and provides a clean pipeline API:

```typescript
import { pipeline } from '@huggingface/transformers';

const embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
const embeddings = await embedder(codeSnippet);
```

For custom models, use **onnxruntime-node** for better performance:

```typescript
import * as ort from 'onnxruntime-node';
const session = await ort.InferenceSession.create(modelPath);
const results = await session.run({ input: tensor });
```

**TensorFlow.js** works but is limited to CPU inference in extensions (no WebGL backend available in the extension host).

### Recommended architecture for your technical debt extension

```
┌─────────────────────────────────────────────────────────────────┐
│                  Extension Package (~3MB bundled)               │
├─────────────────────────────────────────────────────────────────┤
│ CORE (always loaded)                                            │
│ • CodeLens provider, commands, TreeView                         │
│ • Rule-based pattern detection (no ML)                          │
│ • Webview dashboard shell                                       │
├─────────────────────────────────────────────────────────────────┤
│ LAZY-LOADED                                                     │
│ • Chart.js bundle (when dashboard opened)                       │
│ • Analysis engine (when analyze command invoked)                │
├─────────────────────────────────────────────────────────────────┤
│ DOWNLOADED ON DEMAND (to globalStorageUri)                      │
│ • Embedding model (~25MB) - Xenova/all-MiniLM-L6-v2            │
│ • Complexity classifier (~5MB) - custom ONNX model             │
├─────────────────────────────────────────────────────────────────┤
│ COPILOT INTEGRATION (when available)                            │
│ • Chat Participant @techdebt for explanations                   │
│ • Language Model API for generating remediation suggestions     │
│ • Tool registration for agent mode invocation                   │
└─────────────────────────────────────────────────────────────────┘
```

**The hybrid approach wins**: Use local models for embeddings and classification (fast, private, works offline), Copilot APIs for explanations and suggestions (better quality, no model hosting), and gracefully degrade when Copilot isn't available.

A companion desktop app with MCP server makes sense only if your models exceed ~500MB or you need GPU acceleration that extensions can't provide. For most technical debt analysis, the in-extension approach with downloaded models works well.

## Conclusion

Building a sophisticated technical debt analysis extension is entirely feasible within VSCode's current architecture. The key decisions:

- **UI**: Use @vscode-elements/elements for forms, Chart.js in webviews for visualizations, TreeViews for navigation
- **Storage**: SQLite via node-sqlite3-wasm in globalStorageUri for reports, globalState for preferences
- **ML**: Download Transformers.js models post-install (~25-30MB), run inference in worker threads
- **AI**: Implement Chat Participant and Language Model Tools for Copilot integration
- **Distribution**: Publish to both VS Code Marketplace and Open VSX for fork compatibility
- **Performance**: Narrow activation events, esbuild bundling, lazy module loading

This architecture keeps your initial VSIX under 5MB while delivering ML-powered insights through a combination of local inference and cloud-powered explanations.
