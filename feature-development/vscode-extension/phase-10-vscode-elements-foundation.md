# Phase 10: @vscode-elements Foundation

**Purpose**: Migrate from deprecated @vscode/webview-ui-toolkit to @vscode-elements/elements for webview components.

**Dependencies**: Phase 4 (Sidebar Dashboard)

**Deliverables**: @vscode-elements library setup, webview bundle configuration, CSP headers, base component structure

## Overview

Phase 10 establishes the foundation for rich UI components using @vscode-elements:

1. Add @vscode-elements/elements and Lit dependencies
2. Create esbuild configuration for webview bundle
3. Configure Content Security Policy for webviews
4. Create webview entry point with Lit component base
5. Implement theme-aware styling with CSS variables
6. Set up message passing between extension and webview
7. Migrate dashboard provider to use bundled webview

## Architecture

```mermaid
---
title: @vscode-elements Webview Architecture
config:
  theme: forest
---
graph TB
    Extension[Extension Host] -->|postMessage| Webview[Webview Context]
    Webview -->|postMessage| Extension

    Extension -->|HTML + nonce| WebviewPanel[Webview Panel]
    WebviewPanel --> CSP[Content Security Policy]

    Webview --> LitBase[Lit Component Base]
    LitBase --> VSCodeElements[@vscode-elements]
    VSCodeElements --> Button[vscode-button]
    VSCodeElements --> Dropdown[vscode-dropdown]
    VSCodeElements --> Badge[vscode-badge]
    VSCodeElements --> DataGrid[vscode-data-grid]

    Webview --> Theme[Theme Variables]
    Theme --> CSS[CSS Custom Properties]

    classDef extension fill:#2563eb,stroke:#1e40af,color:#fff
    classDef webview fill:#16a34a,stroke:#15803d,color:#fff
    classDef component fill:#dc2626,stroke:#b91c1c,color:#fff

    class Extension,WebviewPanel extension
    class Webview,LitBase,Theme webview
    class VSCodeElements,Button,Dropdown,Badge,DataGrid component
```

## File Changes

### 1. Add Dependencies

**File**: `clients/vscode-extension/package.json` (additions)

```json
{
  "dependencies": {
    "@vscode-elements/elements": "^1.6.0",
    "lit": "^3.1.0"
  },
  "devDependencies": {
    "esbuild": "^0.19.0"
  }
}
```

### 2. Webview Build Configuration

**File**: `clients/vscode-extension/esbuild.webview.mjs`

```javascript
import * as esbuild from 'esbuild';
import { resolve } from 'path';

const production = process.argv.includes('--production');
const watch = process.argv.includes('--watch');

async function main() {
  const ctx = await esbuild.context({
    entryPoints: ['src/webview/dashboard-app.ts'],
    bundle: true,
    format: 'esm',
    minify: production,
    sourcemap: !production,
    outfile: 'dist/webview/dashboard-app.js',
    platform: 'browser',
    target: 'es2020',
    logLevel: 'info',
  });

  if (watch) {
    await ctx.watch();
    console.log('Watching webview bundle...');
  } else {
    await ctx.rebuild();
    await ctx.dispose();
  }
}

main().catch(e => {
  console.error(e);
  process.exit(1);
});
```

### 3. Update Build Scripts

**File**: `clients/vscode-extension/package.json` (script additions)

```json
{
  "scripts": {
    "build": "pnpm build:extension && pnpm build:webview",
    "build:extension": "esbuild src/extension.ts --bundle --outfile=dist/extension.js --external:vscode --format=cjs --platform=node",
    "build:webview": "node esbuild.webview.mjs --production",
    "watch": "pnpm watch:extension & pnpm watch:webview",
    "watch:extension": "esbuild src/extension.ts --bundle --outfile=dist/extension.js --external:vscode --format=cjs --platform=node --watch",
    "watch:webview": "node esbuild.webview.mjs --watch"
  }
}
```

### 4. Webview Entry Point

**File**: `src/webview/dashboard-app.ts`

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import '@vscode-elements/elements/dist/vscode-button';
import '@vscode-elements/elements/dist/vscode-badge';

/**
 * Message types from extension host
 */
interface ExtensionMessage {
  type: 'init' | 'update' | 'error';
  data?: any;
}

/**
 * Message types to extension host
 */
interface WebviewMessage {
  type: 'ready' | 'command' | 'request';
  command?: string;
  args?: any[];
}

/**
 * Main dashboard application component
 */
@customElement('vipr-dashboard')
export class ViprDashboard extends LitElement {
  @state()
  private analysisData: any = null;

  @state()
  private loading = true;

  static styles = css`
    :host {
      display: block;
      padding: 20px;
      color: var(--vscode-foreground);
      font-family: var(--vscode-font-family);
      font-size: var(--vscode-font-size);
    }

    .loading {
      text-align: center;
      padding: 40px;
    }

    .error {
      color: var(--vscode-errorForeground);
      padding: 20px;
      border: 1px solid var(--vscode-errorBorder);
      border-radius: 4px;
    }

    .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 24px;
    }

    h1 {
      margin: 0;
      font-size: 24px;
      font-weight: 600;
    }
  `;

  connectedCallback() {
    super.connectedCallback();
    this.setupMessageListener();
    this.notifyReady();
  }

  private setupMessageListener() {
    window.addEventListener('message', (event: MessageEvent<ExtensionMessage>) => {
      const message = event.data;

      switch (message.type) {
        case 'init':
          this.analysisData = message.data;
          this.loading = false;
          break;
        case 'update':
          this.analysisData = message.data;
          break;
        case 'error':
          this.loading = false;
          console.error('Extension error:', message.data);
          break;
      }
    });
  }

  private notifyReady() {
    this.postMessage({ type: 'ready' });
  }

  private postMessage(message: WebviewMessage) {
    // @ts-ignore - vscode API injected by webview
    if (typeof vscode !== 'undefined') {
      vscode.postMessage(message);
    }
  }

  private handleCommand(command: string, ...args: any[]) {
    this.postMessage({ type: 'command', command, args });
  }

  render() {
    if (this.loading) {
      return html`
        <div class="loading">
          <p>Loading analysis data...</p>
        </div>
      `;
    }

    if (!this.analysisData) {
      return html`
        <div class="error">
          <p>No analysis data available.</p>
          <vscode-button @click=${() => this.handleCommand('vipr.analyzeWorkspace')}>
            Run Analysis
          </vscode-button>
        </div>
      `;
    }

    return html`
      <div class="header">
        <h1>Vipr Analysis Dashboard</h1>
        <vscode-badge>${this.analysisData.fileCount || 0} files</vscode-badge>
      </div>
      <div class="content">
        <p>Dashboard content will be added in Phase 11</p>
      </div>
    `;
  }
}

// Initialize the app when DOM is ready
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', () => {
    const app = document.createElement('vipr-dashboard');
    document.body.appendChild(app);
  });
} else {
  const app = document.createElement('vipr-dashboard');
  document.body.appendChild(app);
}
```

### 5. Update Dashboard Provider

**File**: `src/views/dashboard-provider.ts` (refactored)

```typescript
import * as vscode from 'vscode';
import { getNonce } from '../utils/nonce';

/**
 * Provides the webview dashboard panel
 */
export class DashboardProvider {
  private panel: vscode.WebviewPanel | undefined;
  private disposables: vscode.Disposable[] = [];

  constructor(private readonly context: vscode.ExtensionContext) {}

  /**
   * Show or focus the dashboard panel
   */
  public show() {
    if (this.panel) {
      this.panel.reveal();
      return;
    }

    this.panel = vscode.window.createWebviewPanel(
      'vipr.dashboard',
      'Vipr Dashboard',
      vscode.ViewColumn.One,
      {
        enableScripts: true,
        retainContextWhenHidden: true,
        localResourceRoots: [vscode.Uri.joinPath(this.context.extensionUri, 'dist', 'webview')],
      }
    );

    this.panel.webview.html = this.getHtmlContent(this.panel.webview);

    // Handle messages from webview
    this.panel.webview.onDidReceiveMessage(
      message => this.handleMessage(message),
      null,
      this.disposables
    );

    // Clean up when panel is closed
    this.panel.onDidDispose(
      () => {
        this.panel = undefined;
        this.disposables.forEach(d => d.dispose());
        this.disposables = [];
      },
      null,
      this.disposables
    );
  }

  /**
   * Update dashboard with new data
   */
  public update(data: any) {
    if (this.panel) {
      this.panel.webview.postMessage({ type: 'update', data });
    }
  }

  /**
   * Handle messages from webview
   */
  private async handleMessage(message: any) {
    switch (message.type) {
      case 'ready':
        // Send initial data when webview is ready
        await this.sendInitialData();
        break;
      case 'command':
        // Execute VSCode command
        await vscode.commands.executeCommand(message.command, ...message.args);
        break;
    }
  }

  /**
   * Send initial data to webview
   */
  private async sendInitialData() {
    // Get analysis data from extension state
    // This will be implemented in later phases
    const data = {
      fileCount: 0,
      message: 'No analysis data yet',
    };

    this.panel?.webview.postMessage({ type: 'init', data });
  }

  /**
   * Generate HTML content for webview
   */
  private getHtmlContent(webview: vscode.Webview): string {
    const nonce = getNonce();
    const scriptUri = webview.asWebviewUri(
      vscode.Uri.joinPath(this.context.extensionUri, 'dist', 'webview', 'dashboard-app.js')
    );

    return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" content="
    default-src 'none';
    style-src ${webview.cspSource} 'unsafe-inline';
    script-src 'nonce-${nonce}';
    font-src ${webview.cspSource};
  ">
  <title>Vipr Dashboard</title>
  <style>
    body {
      margin: 0;
      padding: 0;
      background-color: var(--vscode-editor-background);
    }
  </style>
</head>
<body>
  <script nonce="${nonce}">
    const vscode = acquireVsCodeApi();
  </script>
  <script type="module" nonce="${nonce}" src="${scriptUri}"></script>
</body>
</html>`;
  }

  /**
   * Dispose of resources
   */
  public dispose() {
    this.panel?.dispose();
    this.disposables.forEach(d => d.dispose());
  }
}
```

### 6. Nonce Utility

**File**: `src/utils/nonce.ts`

```typescript
/**
 * Generate a nonce for CSP
 */
export function getNonce(): string {
  let text = '';
  const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  for (let i = 0; i < 32; i++) {
    text += possible.charAt(Math.floor(Math.random() * possible.length));
  }
  return text;
}
```

### 7. Update Extension Activation

**File**: `src/extension.ts` (ensure dashboard is registered)

```typescript
import { DashboardProvider } from './views/dashboard-provider';

let dashboardProvider: DashboardProvider | undefined;

export function activate(context: vscode.ExtensionContext) {
  // Initialize dashboard provider
  dashboardProvider = new DashboardProvider(context);

  // Register show dashboard command
  context.subscriptions.push(
    vscode.commands.registerCommand('vipr.showDashboard', () => {
      dashboardProvider?.show();
    })
  );

  // ... rest of activation code
}

export function deactivate() {
  dashboardProvider?.dispose();
}
```

## Configuration

Update `.vscodeignore` to exclude webview source:

**File**: `clients/vscode-extension/.vscodeignore` (additions)

```
src/webview/**
esbuild.webview.mjs
```

## Acceptance Criteria

- [ ] @vscode-elements/elements and Lit dependencies installed
- [ ] Webview bundle builds successfully with esbuild
- [ ] CSP headers configured with nonce for scripts
- [ ] Dashboard webview loads without CSP violations
- [ ] VSCode theme variables applied correctly to webview
- [ ] Message passing works bidirectionally (extension ↔ webview)
- [ ] Dashboard shows placeholder content with @vscode-elements components
- [ ] Extension bundle size remains under 5MB
- [ ] Hot reload works during development with watch mode
- [ ] No console errors in webview developer tools

## Testing Strategy

### Build Verification

```bash
cd clients/vscode-extension
pnpm install
pnpm build
ls -lh dist/extension.js dist/webview/dashboard-app.js
```

### Manual Verification

1. Open VSCode with extension in debug mode
2. Run command: "Vipr: Show Dashboard"
3. Verify dashboard panel opens
4. Open webview developer tools (Ctrl+Shift+I on webview)
5. Check console for errors
6. Verify:
   - No CSP violations in console
   - Theme variables applied (matches editor theme)
   - @vscode-elements button and badge render correctly
7. Switch VSCode theme (light/dark)
8. Verify webview theme updates automatically
9. Click "Run Analysis" button
10. Verify command executes (check notification)

### Unit Tests

**File**: `src/utils/nonce.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { getNonce } from './nonce';

describe('getNonce', () => {
  it('should generate a 32-character string', () => {
    const nonce = getNonce();
    expect(nonce).toHaveLength(32);
  });

  it('should generate unique nonces', () => {
    const nonce1 = getNonce();
    const nonce2 = getNonce();
    expect(nonce1).not.toBe(nonce2);
  });

  it('should only contain alphanumeric characters', () => {
    const nonce = getNonce();
    expect(nonce).toMatch(/^[A-Za-z0-9]+$/);
  });
});
```

## Summary

Phase 10 establishes a modern, maintainable foundation for rich webview UIs using @vscode-elements and Lit. The architecture supports secure CSP policies, theme-aware styling, and efficient bidirectional communication between extension and webview contexts.
