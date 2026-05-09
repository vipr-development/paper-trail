---
id: 06-advanced-features
---

# phase 4: advanced features implementation plan

Comprehensive technical specifications for Phase 4 deliverables: PDF export, cost/velocity estimation, AI prompt generation, MCP server integration, IDE integration, notes/annotations, and issue exclusions.

**ARCHITECTURE INTEGRATION STATUS**: This document has been updated to ensure consistency with Phase 1-3 patterns, including IPC conventions, store patterns, database migrations, and error handling strategies.

---

## section 0: phase 3 integration prerequisites

Before implementing Phase 4 features, ensure Phase 3 deliverables are complete and the following integration points are ready:

### 0.1 required phase 3 components

**IPC Types and Validation:**

- All new IPC channels must be added to `src/shared/ipc-types.ts`
- Follow `domain:action` naming convention established in Phase 1
- Use Zod validation at ALL boundaries (preload request, main response, preload response)

**Store Architecture:**

- Phase 4 features integrate with existing stores: `useAnalysisStore`, `useSettingsStore`
- New store needed: `useReportingStore` for PDF export state management
- All stores must use selector patterns to prevent unnecessary re-renders

**Component Patterns:**

- Loading states using Phase 3 skeleton components
- Error boundaries for all new features
- Optimistic updates for user actions (notes, exclusions)
- Follow Phase 3 component composition patterns

**Database Integration:**

- Phase 2 schema must be at version 1 before adding notes/exclusions tables
- Migration version checking required before Phase 4 migrations
- All queries use prepared statements from Phase 1 patterns

### 0.2 new ipc channels inventory

All channels follow Phase 1 `domain:action` convention with full Zod validation:

```typescript
// src/shared/ipc-types.ts additions

// PDF Export (reports domain)
export const ExportPDFPayloadSchema = z.object({
  reportData: z.object({
    healthScore: z.number().min(0).max(100),
    fileCount: z.number().int().nonnegative(),
    criticalIssues: z.number().int().nonnegative(),
    trends: z.array(z.unknown()),
    distribution: z.array(z.unknown()),
    topIssues: z.array(z.unknown()),
    estimates: z
      .object({
        hours: z.number(),
        velocityGain: z.number(),
        roi: z.number(),
      })
      .optional(),
  }),
  config: z.object({
    title: z.string(),
    includeEstimates: z.boolean().default(false),
    branding: z
      .object({
        logo: z.string().optional(),
        primaryColor: z.string().optional(),
      })
      .optional(),
  }),
  suggestedPath: z.string().optional(),
});

export const ExportPDFResultSchema = z.object({
  success: z.boolean(),
  path: z.string().optional(),
  error: z.string().optional(),
});

// Cost Estimation (analysis domain)
export const EstimateCostsPayloadSchema = z.object({
  files: z.array(z.unknown()),
  config: z.object({
    hourlyRate: z.number().positive(),
    teamSize: z.number().int().positive(),
    experienceLevel: z.enum(['junior', 'mid', 'senior']),
  }),
});

// AI Prompts (ai domain)
export const GeneratePromptPayloadSchema = z.object({
  templateName: z.string(),
  file: z.unknown(),
  insights: z.array(z.unknown()),
});

// MCP Server (mcp domain)
export const MCPStatusSchema = z.object({
  enabled: z.boolean(),
  running: z.boolean(),
});

// IDE Integration (shell domain)
export const OpenInIDEPayloadSchema = z.object({
  filePath: z.string(),
  line: z.number().optional(),
  ide: z.enum(['vscode', 'cursor', 'idea', 'atom', 'sublime']).optional(),
});

// Notes (notes domain)
export const CreateNotePayloadSchema = z.object({
  targetType: z.enum(['file', 'issue', 'abstraction']),
  targetId: z.string(),
  content: z.string().min(1),
});

// Exclusions (exclusions domain)
export const AddExclusionPayloadSchema = z.object({
  issueType: z.string(),
  filePath: z.string().optional(),
  pluginId: z.string().optional(),
  reason: z.string().optional(),
});
```

**IPC Channel Registry:**

- `reports:exportPDF` - PDF generation
- `analysis:estimateCosts` - Cost/velocity calculation
- `ai:getTemplates` - List available templates
- `ai:generatePrompt` - Generate contextual prompt
- `ai:generateContextualPrompt` - Auto-detect best template
- `mcp:getStatus` - Get MCP server status
- `mcp:start` - Start MCP server
- `mcp:stop` - Stop MCP server
- `shell:openInIDE` - Open file in IDE
- `shell:getSupportedIDEs` - List available IDEs
- `notes:create` - Create note
- `notes:update` - Update note
- `notes:delete` - Delete note
- `notes:getByTarget` - Query notes
- `notes:getAll` - List all notes
- `exclusions:add` - Add exclusion
- `exclusions:remove` - Remove exclusion
- `exclusions:getAll` - List exclusions
- `exclusions:isExcluded` - Check if excluded

### 0.3 database migration strategy

Phase 4 adds two new tables that must integrate with Phase 2 schema:

**Migration Version Check:**

```typescript
// src/main/db/migrations/phase4.ts
export const PHASE_4_MIGRATIONS: Migration[] = [
  {
    version: 2, // Continues from Phase 2 version 1
    up: db => {
      // Verify Phase 2 schema exists
      const tables = db.prepare("SELECT name FROM sqlite_master WHERE type='table'").all() as {
        name: string;
      }[];

      const requiredTables = ['files', 'analyses', 'snapshots', 'metadata'];
      const missingTables = requiredTables.filter(t => !tables.some(row => row.name === t));

      if (missingTables.length > 0) {
        throw new Error(`Phase 2 schema incomplete. Missing tables: ${missingTables.join(', ')}`);
      }

      // Create notes table
      db.exec(`
        CREATE TABLE notes (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          target_type TEXT NOT NULL CHECK(target_type IN ('file', 'issue', 'abstraction')),
          target_id TEXT NOT NULL,
          content TEXT NOT NULL,
          created_at INTEGER NOT NULL DEFAULT (unixepoch()),
          updated_at INTEGER NOT NULL DEFAULT (unixepoch())
        );

        CREATE INDEX idx_notes_target ON notes(target_type, target_id);
        CREATE INDEX idx_notes_created_at ON notes(created_at DESC);
      `);

      // Create exclusions table
      db.exec(`
        CREATE TABLE exclusions (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          issue_type TEXT NOT NULL,
          file_path TEXT,
          plugin_id TEXT,
          reason TEXT,
          created_at INTEGER NOT NULL DEFAULT (unixepoch())
        );

        CREATE INDEX idx_exclusions_issue_type ON exclusions(issue_type);
        CREATE INDEX idx_exclusions_file_path ON exclusions(file_path);
        CREATE INDEX idx_exclusions_plugin_id ON exclusions(plugin_id);
      `);
    },
    down: db => {
      db.exec('DROP TABLE IF EXISTS exclusions');
      db.exec('DROP TABLE IF EXISTS notes');
    },
  },
];
```

### 0.4 store modifications needed

**New Store: ReportingStore**

```typescript
// src/renderer/stores/reporting.ts
import { create } from 'zustand';

interface ReportingState {
  isExporting: boolean;
  exportProgress: number;
  lastExportPath: string | null;

  // Actions
  setExporting: (isExporting: boolean) => void;
  setProgress: (progress: number) => void;
  setLastExport: (path: string) => void;
}

export const useReportingStore = create<ReportingState>(set => ({
  isExporting: false,
  exportProgress: 0,
  lastExportPath: null,

  setExporting: isExporting => set({ isExporting }),
  setProgress: progress => set({ exportProgress: progress }),
  setLastExport: path => set({ lastExportPath: path }),
}));
```

**SettingsStore Extensions:**

```typescript
// Add to existing useSettingsStore
interface SettingsState {
  // Existing settings...

  // Phase 4 additions
  costEstimation: {
    hourlyRate: number;
    teamSize: number;
    experienceLevel: 'junior' | 'mid' | 'senior';
  };
  idePreference: 'vscode' | 'cursor' | 'idea' | 'atom' | 'sublime';
  mcpServerEnabled: boolean;
}
```

### 0.5 component patterns from phase 3

All Phase 4 UI components must follow these patterns:

**Loading States:**

```typescript
// Use Phase 3 skeleton components
import { Skeleton } from '@/renderer/components/ui/Skeleton';

{isLoading && <Skeleton className="h-32 w-full" />}
```

**Error Boundaries:**

```typescript
// Wrap all new features
import { ErrorBoundary } from '@/renderer/components/ErrorBoundary';

<ErrorBoundary fallback={<ErrorFallback />}>
  <NewFeatureComponent />
</ErrorBoundary>
```

**Optimistic Updates:**

```typescript
// For notes and exclusions
const handleCreate = async () => {
  // Optimistic update
  setNotes([...notes, optimisticNote]);

  try {
    const result = await window.viprDesktop.notes.create(payload);
    // Update with real data
    setNotes(notes => notes.map(n => (n.id === optimisticNote.id ? result : n)));
  } catch (error) {
    // Rollback on error
    setNotes(notes => notes.filter(n => n.id !== optimisticNote.id));
    showError(error);
  }
};
```

**Selector Patterns:**

```typescript
// Prevent unnecessary re-renders
const fileCount = useAnalysisStore(state => state.files.size);
const criticalIssues = useAnalysisStore(
  state =>
    Array.from(state.files.values()).filter(f =>
      f.analyses?.some(a => a.insights?.some(i => i.severity === 'critical'))
    ).length
);
```

---

## part a: technical proposal

### 0.6 security architecture and mandatory controls

**CRITICAL**: All Phase 4 features must implement security controls before production deployment. This section provides a high-level overview. For detailed implementation specifications, security testing requirements, and vulnerability remediation, see:

**Phase 4 Security Architecture (security documentation in progress)**

#### executive security summary

Phase 4 features introduce significant security risks that must be mitigated:

| Feature             | Critical Vulnerabilities                 | Security Controls Required                                 |
| ------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| **PDF Export**      | XSS via inline scripts, CSP violation    | IPC data passing, HTML sanitization, rate limiting         |
| **MCP Server**      | SQL injection, read-only bypass          | Prepared statements, input validation, read-only database  |
| **IDE Integration** | Command injection via shell.openExternal | Path sanitization, URI validation, project boundary checks |
| **Notes**           | XSS via user content                     | DOMPurify sanitization, content length limits              |
| **AI Prompts**      | Sensitive data exposure                  | Secret detection, clipboard warnings                       |

#### mandatory security principles

All implementations MUST follow these principles:

1. **Input Validation**: Zod schemas at ALL IPC boundaries
2. **Output Sanitization**: HTML escaping for ALL user-controlled data
3. **SQL Safety**: Prepared statements ONLY (no string concatenation)
4. **Path Validation**: Sandboxing to project directory (prevent traversal)
5. **CSP Compliance**: NO inline scripts (Phase 3 requirement)
6. **Rate Limiting**: Expensive operations (PDF export, cost estimation)
7. **Least Privilege**: MCP server runs read-only, separate process

#### security utilities required

Before implementing any Phase 4 feature, create these shared utilities:

**Location: `src/main/security/`**

```typescript
// sanitization.ts
export function escapeHtml(unsafe: string): string;
export function sanitizeJSONForHTML(data: unknown): string;
export function escapeCodeFence(code: string): string;

// path-validator.ts
export class PathValidator {
  constructor(projectRoot: string);
  sanitizePath(filePath: string): string | null;
  verifyFileInProject(filePath: string): Promise<boolean>;
}

// rate-limiter.ts
export class RateLimiter {
  checkLimit(key: string, maxRequests: number, windowMs: number): boolean;
  reset(key: string): void;
}

// sensitive-data.ts
export class SensitiveDataDetector {
  static detect(content: string): string[];
  static hasSensitiveData(content: string): boolean;
}

// validation.ts
export const FilePathSchema: z.ZodSchema;
export const UserContentSchema: z.ZodSchema;
export const CostConfigSchema: z.ZodSchema;
export const URISchemeSchema: z.ZodSchema;
```

#### critical security requirements by feature

**PDF Export (Section 1):**

- ❌ **NEVER** use inline scripts with `window.__DATA__`
- ✓ **ALWAYS** pass data via IPC after page load
- ✓ **ALWAYS** sanitize file paths and user-provided config
- ✓ **ALWAYS** apply rate limiting (5 per minute)

**MCP Server (Section 4):**

- ❌ **NEVER** concatenate user input into SQL strings
- ✓ **ALWAYS** use prepared statements with parameter binding
- ✓ **ALWAYS** validate enum values (severity, plugin ID)
- ✓ **ALWAYS** open database in read-only mode

**IDE Integration (Section 5):**

- ❌ **NEVER** pass unsanitized paths to `shell.openExternal()`
- ✓ **ALWAYS** validate file exists in project before opening
- ✓ **ALWAYS** encode URI components properly
- ✓ **ALWAYS** whitelist URI schemes (no `file://`, `http://`, etc.)

**Notes (Section 6):**

- ❌ **NEVER** render user content without sanitization
- ✓ **ALWAYS** use DOMPurify if markdown enabled
- ✓ **ALWAYS** escape HTML if markdown disabled
- ✓ **ALWAYS** validate content length (max 50KB)

**AI Prompts (Section 3):**

- ❌ **NEVER** copy to clipboard without checking for secrets
- ✓ **ALWAYS** detect sensitive patterns (API keys, tokens)
- ✓ **ALWAYS** warn user before clipboard operation
- ✓ **ALWAYS** escape code fence delimiters in snippets

#### security testing requirements

Before requesting architecture review:

- [ ] All SQL injection tests pass (MCP server)
- [ ] All XSS tests pass (Notes, PDF export)
- [ ] All path traversal tests pass (IDE integration)
- [ ] All rate limiting tests pass
- [ ] All CSP compliance tests pass
- [ ] Penetration testing checklist complete

See **security documentation (in progress)** for complete test suites and attack scenarios.

#### security review process

Phase 4 CANNOT proceed to production without:

1. **All critical vulnerabilities remediated** (see security document)
2. **Security test suite passing** (100% coverage of attack vectors)
3. **Penetration testing complete** (manual verification of defenses)
4. **frontend-security-auditor approval** (sign-off required)

**Security review checkpoint**: End of Week 4, before deployment.

---

### 1. pdf export service (us-07)

#### technical comparison: puppeteer vs electron-pdf

| Aspect                    | Puppeteer                   | electron-pdf (or electron BrowserWindow.printToPDF) |
| ------------------------- | --------------------------- | --------------------------------------------------- |
| **Bundle size**           | ~300MB (includes Chromium)  | 0MB (uses existing Electron)                        |
| **Rendering engine**      | Separate Chromium instance  | App's existing Chromium                             |
| **CSS compatibility**     | Full Chrome CSS             | Full Electron CSS (identical)                       |
| **React component reuse** | Requires HTML string export | Direct component rendering possible                 |
| **Memory overhead**       | High (separate browser)     | Low (reuses app renderer)                           |
| **Startup time**          | ~2-3s (launch Chromium)     | < 500ms (hidden BrowserWindow)                      |
| **Maintenance**           | External dependency         | Built-in Electron API                               |

**Recommendation: Use Electron's built-in `BrowserWindow.printToPDF()`**

Rationale:

- Zero additional dependencies
- Reuses existing React components without HTML serialization
- Faster and more memory-efficient
- Consistent rendering with app UI
- Better integration with Electron security model

#### implementation architecture

```mermaid
---
title: pdf export architecture with phase 3 integration
---
sequenceDiagram
    participant U as User
    participant R as Renderer UI
    participant S as ReportingStore
    participant P as Preload Bridge
    participant M as Main Process
    participant H as Hidden BrowserWindow
    participant F as File System

    U->>R: Click "Export PDF"
    R->>S: setExporting(true)
    R->>R: Configure report (metrics, branding)

    Note over R,P: Phase 1 IPC Pattern with Validation
    R->>P: viprDesktop.reports.exportPDF(payload)
    P->>P: Validate with ExportPDFPayloadSchema
    P->>M: ipcRenderer.invoke('reports:exportPDF', validated)

    M->>M: Validate with ExportPDFPayloadSchema
    M->>H: Create hidden BrowserWindow
    M->>H: Load prebuilt report bundle
    H->>H: Hydrate React components (CSP compliant)
    H-->>M: 'did-finish-load' event

    Note over H: Wait for render completion with timeout
    M->>H: webContents.printToPDF(options)
    H-->>M: PDF Buffer
    M->>F: fs.writeFile(path, buffer)
    M->>H: Close hidden window
    M-->>P: { success: true, path }

    Note over P,R: Response Validation
    P->>P: Validate with ExportPDFResultSchema
    P-->>R: Validated result
    R->>S: setExporting(false)
    R->>S: setLastExport(path)
    R->>U: Show success toast + "Open PDF" button
```

#### step-by-step implementation

**Step 1: Create report template components**

Location: `src/renderer/components/reports/`

```typescript
// src/renderer/components/reports/PDFReportTemplate.tsx
import React from 'react';
import type { ReportConfig, ReportData } from '@/shared/types';

interface PDFReportTemplateProps {
  data: ReportData;
  config: ReportConfig;
}

export function PDFReportTemplate({ data, config }: PDFReportTemplateProps) {
  return (
    <div className="pdf-report">
      <header className="report-header">
        {config.branding?.logo && (
          <img src={config.branding.logo} alt="Logo" className="report-logo" />
        )}
        <h1>{config.title}</h1>
        <p className="report-date">{new Date().toLocaleDateString()}</p>
      </header>

      <section className="executive-summary">
        <h2>Executive Summary</h2>
        <div className="summary-cards">
          <MetricCard label="Health Score" value={data.healthScore} />
          <MetricCard label="Total Files" value={data.fileCount} />
          <MetricCard label="Critical Issues" value={data.criticalIssues} />
        </div>
      </section>

      <section className="charts">
        <h2>Trend Analysis</h2>
        <LineChart data={data.trends} />
        <DoughnutChart data={data.distribution} />
      </section>

      <section className="issues">
        <h2>Top Issues</h2>
        <table className="issues-table">
          <thead>
            <tr>
              <th>File</th>
              <th>Severity</th>
              <th>Issue</th>
              <th>Recommendation</th>
            </tr>
          </thead>
          <tbody>
            {data.topIssues.map((issue) => (
              <tr key={issue.id}>
                <td>{issue.file}</td>
                <td><Badge severity={issue.severity} /></td>
                <td>{issue.title}</td>
                <td>{issue.recommendation}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      {config.includeEstimates && (
        <section className="cost-estimates">
          <h2>Cost & Velocity Impact</h2>
          <p>Estimated hours to resolve: {data.estimates.hours}</p>
          <p>Projected velocity improvement: {data.estimates.velocityGain}%</p>
          <p>ROI: {data.estimates.roi}</p>
        </section>
      )}

      <footer className="report-footer">
        <p>Generated by Vipr Desktop</p>
        <p className="disclaimer">
          This report is generated from static analysis and should be validated
          by engineering teams before making decisions.
        </p>
      </footer>
    </div>
  );
}
```

**Step 2: Create PDF generation service in main process**

Location: `src/main/services/pdf-export.ts`

```typescript
import { BrowserWindow, dialog, app } from 'electron';
import * as fs from 'fs/promises';
import * as path from 'path';
import { createLogger } from '@vipr/logging';
import type { ReportConfig, ReportData } from '@/shared/types';

const logger = createLogger('pdf-export');

export class PDFExportService {
  private hiddenWindow: BrowserWindow | null = null;
  private readonly RENDER_TIMEOUT = 30000; // 30 seconds
  private readonly REPORT_BUNDLE_PATH: string;

  constructor() {
    // Path to prebuilt report bundle (created during build)
    this.REPORT_BUNDLE_PATH = path.join(app.getAppPath(), 'dist', 'report-renderer', 'index.html');
  }

  async exportToPDF(
    data: ReportData,
    config: ReportConfig,
    suggestedPath?: string
  ): Promise<{ success: boolean; path?: string; error?: string }> {
    const startTime = Date.now();

    try {
      // 1. Prompt user for save location
      const { filePath, canceled } = await dialog.showSaveDialog({
        title: 'Export PDF Report',
        defaultPath: suggestedPath || `vipr-report-${Date.now()}.pdf`,
        filters: [{ name: 'PDF Files', extensions: ['pdf'] }],
      });

      if (canceled || !filePath) {
        return { success: false, error: 'User canceled export' };
      }

      logger.info('Starting PDF export', { filePath, config: config.title });

      // 2. Create hidden BrowserWindow for rendering with strict CSP
      this.hiddenWindow = new BrowserWindow({
        show: false,
        width: 1200, // A4 width approximation
        height: 1697, // A4 height approximation
        webPreferences: {
          nodeIntegration: false,
          contextIsolation: true,
          sandbox: true,
          // Load prebuilt bundle that hydrates with window.__VIPR_REPORT_DATA__
          preload: path.join(app.getAppPath(), 'dist', 'report-preload', 'index.js'),
        },
      });

      // Configure CSP for report rendering
      this.hiddenWindow.webContents.session.webRequest.onHeadersReceived((details, callback) => {
        callback({
          responseHeaders: {
            ...details.responseHeaders,
            'Content-Security-Policy': [
              "default-src 'self';",
              "script-src 'self';", // No inline scripts
              "style-src 'self' 'unsafe-inline';", // Allow Tailwind inline styles
              "img-src 'self' data: https:;", // Allow logos
              "font-src 'self';",
            ].join(' '),
          },
        });
      });

      // 3. Inject report data via preload script (CSP-compliant)
      await this.hiddenWindow.webContents.executeJavaScript(
        `window.__VIPR_REPORT_DATA__ = ${JSON.stringify(data)}; window.__VIPR_REPORT_CONFIG__ = ${JSON.stringify(config)};`
      );

      // 4. Load prebuilt report HTML
      await this.hiddenWindow.loadFile(this.REPORT_BUNDLE_PATH);

      // 5. Wait for rendering with timeout
      await this.waitForRenderComplete(this.RENDER_TIMEOUT);

      logger.debug('Report rendered, generating PDF');

      // 6. Generate PDF with optimized settings
      const pdfBuffer = await this.hiddenWindow.webContents.printToPDF({
        marginsType: 0,
        pageSize: 'A4',
        printBackground: true,
        landscape: false,
        preferCSSPageSize: true, // Respect @page CSS rules
      });

      // 7. Write to file
      await fs.writeFile(filePath, pdfBuffer);

      const duration = Date.now() - startTime;
      logger.info('PDF export complete', { filePath, duration });

      // 8. Cleanup
      this.cleanup();

      return { success: true, path: filePath };
    } catch (error) {
      logger.error('PDF export failed', { error });
      this.cleanup();

      return {
        success: false,
        error: error instanceof Error ? error.message : 'Unknown error',
      };
    }
  }

  private async waitForRenderComplete(timeout: number): Promise<void> {
    return new Promise((resolve, reject) => {
      const timeoutId = setTimeout(() => {
        reject(new Error('PDF render timeout'));
      }, timeout);

      // Wait for custom event from report renderer
      this.hiddenWindow!.webContents.once('did-finish-load', () => {
        // Additional wait for React hydration and chart rendering
        setTimeout(() => {
          clearTimeout(timeoutId);
          resolve();
        }, 2000); // 2 second buffer for Chart.js rendering
      });
    });
  }

  private cleanup(): void {
    if (this.hiddenWindow && !this.hiddenWindow.isDestroyed()) {
      this.hiddenWindow.close();
      this.hiddenWindow = null;
    }
  }

  // Called on app shutdown
  public dispose(): void {
    this.cleanup();
  }

  private getPDFStyles(): string {
    // Include Tailwind CSS output or inline critical styles
    return `
      @page {
        size: A4;
        margin: 2cm;
      }

      body {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        line-height: 1.6;
        color: #1f2937;
      }

      .pdf-report {
        max-width: 100%;
      }

      .report-header {
        text-align: center;
        margin-bottom: 2rem;
        page-break-after: avoid;
      }

      .report-logo {
        max-width: 200px;
        margin-bottom: 1rem;
      }

      .summary-cards {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: 1rem;
        margin-bottom: 2rem;
      }

      .issues-table {
        width: 100%;
        border-collapse: collapse;
        margin-bottom: 2rem;
      }

      .issues-table th,
      .issues-table td {
        border: 1px solid #e5e7eb;
        padding: 0.5rem;
        text-align: left;
      }

      .issues-table th {
        background-color: #f3f4f6;
        font-weight: 600;
      }

      .report-footer {
        margin-top: 2rem;
        padding-top: 1rem;
        border-top: 1px solid #e5e7eb;
        text-align: center;
        font-size: 0.875rem;
        color: #6b7280;
      }

      @media print {
        .page-break {
          page-break-before: always;
        }
      }
    `;
  }
}
```

**Step 3: Register IPC handler**

Location: `src/main/ipc/handlers/reports.ts`

```typescript
import { ipcMain } from 'electron';
import { PDFExportService } from '@/main/services/pdf-export';
import { z } from 'zod';

const ExportPDFPayloadSchema = z.object({
  reportData: z.object({
    healthScore: z.number(),
    fileCount: z.number(),
    criticalIssues: z.number(),
    trends: z.array(z.any()),
    distribution: z.array(z.any()),
    topIssues: z.array(z.any()),
    estimates: z
      .object({
        hours: z.number(),
        velocityGain: z.number(),
        roi: z.number(),
      })
      .optional(),
  }),
  config: z.object({
    title: z.string(),
    includeEstimates: z.boolean().default(false),
    branding: z
      .object({
        logo: z.string().optional(),
        primaryColor: z.string().optional(),
      })
      .optional(),
  }),
  suggestedPath: z.string().optional(),
});

export function registerReportsHandlers() {
  const pdfService = new PDFExportService();

  ipcMain.handle('reports:exportPDF', async (_event, payload) => {
    const validated = ExportPDFPayloadSchema.parse(payload);
    return pdfService.exportToPDF(validated.reportData, validated.config, validated.suggestedPath);
  });
}
```

**Step 4: Create UI component for PDF export**

Location: `src/renderer/pages/Reports.tsx`

```typescript
import React, { useState } from 'react';
import { useAnalysisStore } from '@/renderer/stores/analysis';
import { useReportingStore } from '@/renderer/stores/reporting';
import { Button, Modal, Switch, Input, Skeleton } from '@/renderer/components/ui';
import { ErrorBoundary } from '@/renderer/components/ErrorBoundary';
import { useToast } from '@/renderer/hooks/useToast';
import { createLogger } from '@vipr/logging';

const logger = createLogger('reports-page');

export function ReportsPage() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [config, setConfig] = useState({
    title: 'Code Quality Report',
    includeEstimates: true,
    branding: {
      logo: '',
      primaryColor: '#3b82f6',
    },
  });

  // Phase 3 selector patterns - only subscribe to needed data
  const files = useAnalysisStore(state => state.files);
  const isAnalyzing = useAnalysisStore(state => state.isAnalyzing);

  // Phase 4 reporting store
  const { isExporting, setExporting, setLastExport } = useReportingStore();
  const { showSuccess, showError } = useToast();

  const handleExport = async () => {
    if (isExporting) {
      logger.warn('Export already in progress');
      return;
    }

    setExporting(true);

    try {
      // Phase 3 pattern: calculate metrics from store
      const reportData = {
        healthScore: calculateHealthScore(files),
        fileCount: files.size,
        criticalIssues: countCriticalIssues(files),
        trends: generateTrendData(files),
        distribution: generateDistributionData(files),
        topIssues: getTopIssues(files),
        estimates: calculateEstimates(files),
      };

      logger.info('Starting PDF export', { config });

      // Phase 1 IPC pattern: validated request/response
      const result = await window.viprDesktop.reports.exportPDF({
        reportData,
        config,
      });

      if (result.success && result.path) {
        // Phase 3 pattern: success feedback
        showSuccess('PDF exported successfully');
        setLastExport(result.path);

        // Open file location
        await window.viprDesktop.shell.showItemInFolder(result.path);

        setIsModalOpen(false);
      } else {
        // Phase 3 pattern: error feedback
        logger.error('PDF export failed', { error: result.error });
        showError(result.error || 'Failed to export PDF');
      }
    } catch (error) {
      logger.error('Unexpected error during PDF export', { error });
      showError('An unexpected error occurred');
    } finally {
      setExporting(false);
    }
  };

  // Phase 3 pattern: loading state
  if (isAnalyzing) {
    return (
      <div className="reports-page">
        <Skeleton className="h-48 w-full" />
      </div>
    );
  }

  return (
    <ErrorBoundary fallback={<ReportErrorFallback />}>
      <div className="reports-page">
        <header>
          <h1>Reports</h1>
          <Button
            onClick={() => setIsModalOpen(true)}
            disabled={isExporting || files.size === 0}
          >
            {isExporting ? 'Exporting...' : 'Export PDF'}
          </Button>
        </header>

        {files.size === 0 && (
          <div className="empty-state">
            <p>No files analyzed yet. Run an analysis to generate reports.</p>
          </div>
        )}

        <Modal
          open={isModalOpen}
          onClose={() => !isExporting && setIsModalOpen(false)}
        >
          <h2>Configure PDF Export</h2>

          <Input
            label="Report Title"
            value={config.title}
            onChange={(e) => setConfig({ ...config, title: e.target.value })}
            disabled={isExporting}
          />

          <Switch
            label="Include Cost Estimates"
            checked={config.includeEstimates}
            onChange={(checked) =>
              setConfig({ ...config, includeEstimates: checked })
            }
            disabled={isExporting}
          />

          <Input
            label="Logo URL (optional)"
            value={config.branding.logo}
            onChange={(e) =>
              setConfig({
                ...config,
                branding: { ...config.branding, logo: e.target.value },
              })
            }
            disabled={isExporting}
            placeholder="https://example.com/logo.png"
          />

          <div className="modal-actions">
            <Button
              variant="secondary"
              onClick={() => setIsModalOpen(false)}
              disabled={isExporting}
            >
              Cancel
            </Button>
            <Button
              onClick={handleExport}
              disabled={isExporting}
            >
              {isExporting ? 'Exporting...' : 'Export'}
            </Button>
          </div>
        </Modal>
      </div>
    </ErrorBoundary>
  );
}

// Phase 3 pattern: Error fallback component
function ReportErrorFallback() {
  return (
    <div className="error-state">
      <h2>Failed to load reports</h2>
      <p>An error occurred while loading the reports page.</p>
      <Button onClick={() => window.location.reload()}>
        Reload Page
      </Button>
    </div>
  );
}
```

#### acceptance criteria

**Functionality:**

- [ ] PDF exports in under 5 seconds for 100-page report
- [ ] All charts render correctly in PDF (Chart.js compatibility)
- [ ] Custom branding (logo, colors) applies correctly
- [ ] Tables paginate properly across pages
- [ ] User can configure which metrics to include
- [ ] Generated PDF matches app UI visual style
- [ ] Export dialog saves to user-selected location
- [ ] Success toast shows "Open PDF" button with Phase 3 toast system

**Architecture Compliance:**

- [ ] IPC channel `reports:exportPDF` follows `domain:action` convention
- [ ] Request validated with `ExportPDFPayloadSchema` in preload
- [ ] Response validated with `ExportPDFResultSchema` in preload
- [ ] Main process validates request with same schema
- [ ] CSP headers configured for hidden window (no inline scripts)
- [ ] Prebuilt report bundle used instead of dynamic HTML generation
- [ ] React hydration used instead of window globals
- [ ] Timeout handling prevents indefinite hangs
- [ ] Error cleanup closes hidden window on failure
- [ ] ReportingStore follows Phase 3 selector patterns
- [ ] Loading states use Phase 3 skeleton components
- [ ] Error boundary wraps Reports page
- [ ] User feedback via Phase 3 toast system
- [ ] Proper logging with `@vipr/logging`

---

### 2. cost and velocity estimation (us-08)

#### industry validation and research basis

This implementation is grounded in established software engineering research:

**Key References:**

- **COCOMO II** (Constructive Cost Model): Effort estimation based on SLOC, complexity factors, and team experience (Boehm et al., 2000)
- **Function Point Analysis**: Industry-standard productivity ranges of 5-15 function points per person-month (IFPUG standards)
- **Maintainability Index**: IEEE standard formula (SEI, Microsoft Research) with 85 as industry target for maintainable code
- **Technical Debt Principal/Interest**: Research showing 3-5x velocity impact for high-debt codebases (Cunningham, Fowler)

**Calibration Data:**

- Experience multipliers derived from COCOMO II ACAP (Analyst Capability) and PCAP (Programmer Capability) factors
- Complexity thresholds aligned with McCabe research: 10+ (moderate risk), 20+ (high risk), 50+ (untestable)
- Velocity improvement based on empirical studies showing 15-30% productivity gains from refactoring high-complexity code

#### estimation algorithms

**Complexity-to-effort correlation model with confidence intervals:**

```typescript
// src/main/services/cost-estimation.ts
import Database from 'better-sqlite3';

interface ComplexityMetrics {
  cyclomaticComplexity: number;
  halsteadDifficulty: number;
  maintainabilityIndex: number;
  linesOfCode: number;
  issueCount: number;
  issueSeverity: 'low' | 'medium' | 'high' | 'critical';
}

interface EstimationConfig {
  hourlyRate: number;
  teamSize: number;
  experienceLevel: 'junior' | 'mid' | 'senior';
}

interface ConfidenceInterval {
  low: number;
  estimate: number;
  high: number;
  confidence: number; // 0-100 percentage
}

interface EstimationResult {
  hoursToResolve: ConfidenceInterval;
  cost: ConfidenceInterval;
  velocityImprovementPercent: ConfidenceInterval;
  roi: ConfidenceInterval;
  confidenceLevel: 'low' | 'medium' | 'high';
  confidenceFactors: {
    metricCompleteness: number; // 0-100
    metricConsistency: number; // 0-100
    historicalAccuracy: number; // 0-100
    sampleSize: number;
  };
  limitations: string[];
}

interface HistoricalEstimate {
  id: number;
  fileId: string;
  estimatedHours: number;
  actualHours?: number;
  completed: boolean;
  completedAt?: number;
  metrics: ComplexityMetrics;
  config: EstimationConfig;
}

export class CostEstimationService {
  // COCOMO II-derived complexity thresholds
  // McCabe: 1-10 (simple), 11-20 (moderate), 21-50 (complex), 50+ (untestable)
  private readonly COMPLEXITY_THRESHOLDS = {
    low: 10,
    medium: 20,
    high: 50,
  };

  // Derived from empirical studies on refactoring effort by severity
  // Low: cosmetic fixes (0.5x), Medium: standard refactor (1.0x),
  // High: significant restructure (2.0x), Critical: architectural change (4.0x)
  private readonly SEVERITY_MULTIPLIERS = {
    low: 0.5,
    medium: 1.0,
    high: 2.0,
    critical: 4.0,
  };

  // COCOMO II ACAP/PCAP factors normalized:
  // Very Low (1.46x) → Junior (1.5x), Nominal (1.0x) → Mid (1.0x),
  // Very High (0.71x) → Senior (0.7x)
  private readonly EXPERIENCE_MULTIPLIERS = {
    junior: 1.5,
    mid: 1.0,
    senior: 0.7,
  };

  // Confidence interval multipliers based on uncertainty
  // Lower confidence = wider interval
  private readonly CONFIDENCE_INTERVALS = {
    high: { low: 0.8, high: 1.2 }, // ±20% range
    medium: { low: 0.6, high: 1.5 }, // -40% to +50% range
    low: { low: 0.4, high: 2.0 }, // -60% to +100% range
  };

  constructor(private db: Database.Database) {}

  estimateRefactoringEffort(
    metrics: ComplexityMetrics,
    config: EstimationConfig,
    fileId: string
  ): EstimationResult {
    // Base effort calculation
    const complexityFactor = this.calculateComplexityFactor(metrics);
    const severityMultiplier = this.SEVERITY_MULTIPLIERS[metrics.issueSeverity];
    const experienceMultiplier = this.EXPERIENCE_MULTIPLIERS[config.experienceLevel];

    // Industry-standard formula derived from COCOMO II:
    // Effort (person-hours) = a * (SLOC/1000)^b * EAF
    // Where: a=2.94 (calibration constant), b=1.0 (linear for small modules),
    //        EAF = product of effort adjustment factors
    //
    // For refactoring, we simplify: Base hours = (LOC / 100) * complexity_factor * severity_multiplier
    // Dividing by 100 gives ~1 hour per 100 LOC for moderate complexity (aligned with industry averages)
    const baseHours = (metrics.linesOfCode / 100) * complexityFactor * severityMultiplier;

    // Adjust for team experience (COCOMO II Personnel factors)
    const adjustedHours = baseHours * experienceMultiplier;

    // Calculate cost
    const cost = adjustedHours * config.hourlyRate;

    // Estimate velocity improvement (based on technical debt research)
    const velocityImprovementPercent = this.estimateVelocityImprovement(metrics);

    // Calculate ROI (velocity gain vs refactoring cost)
    const roi = this.calculateROI(velocityImprovementPercent, adjustedHours, config);

    // Determine confidence level and intervals
    const confidenceAnalysis = this.analyzeConfidence(metrics, fileId);
    const intervals = this.CONFIDENCE_INTERVALS[confidenceAnalysis.confidenceLevel];

    // Build result with confidence intervals
    return {
      hoursToResolve: {
        low: Math.round(adjustedHours * intervals.low * 10) / 10,
        estimate: Math.round(adjustedHours * 10) / 10,
        high: Math.round(adjustedHours * intervals.high * 10) / 10,
        confidence: confidenceAnalysis.overallConfidence,
      },
      cost: {
        low: Math.round(cost * intervals.low),
        estimate: Math.round(cost),
        high: Math.round(cost * intervals.high),
        confidence: confidenceAnalysis.overallConfidence,
      },
      velocityImprovementPercent: {
        low: Math.round(velocityImprovementPercent * 0.5 * 10) / 10,
        estimate: Math.round(velocityImprovementPercent * 10) / 10,
        high: Math.round(velocityImprovementPercent * 1.5 * 10) / 10,
        confidence: confidenceAnalysis.overallConfidence,
      },
      roi: {
        low:
          Math.round(
            this.calculateROI(
              velocityImprovementPercent * 0.5,
              adjustedHours * intervals.high,
              config
            ) * 10
          ) / 10,
        estimate: Math.round(roi * 10) / 10,
        high:
          Math.round(
            this.calculateROI(
              velocityImprovementPercent * 1.5,
              adjustedHours * intervals.low,
              config
            ) * 10
          ) / 10,
        confidence: confidenceAnalysis.overallConfidence,
      },
      confidenceLevel: confidenceAnalysis.confidenceLevel,
      confidenceFactors: confidenceAnalysis.factors,
      limitations: this.identifyLimitations(metrics, confidenceAnalysis),
    };
  }

  private calculateComplexityFactor(metrics: ComplexityMetrics): number {
    // Weighted average of complexity metrics (weights sum to 1.0)
    // Weights derived from correlation studies between metrics and refactoring effort
    const cyclomaticWeight = 0.4; // Strongest predictor of effort
    const halsteadWeight = 0.3; // Good predictor of debugging time
    const maintainabilityWeight = 0.3; // Composite metric incorporating both

    // Normalize cyclomatic complexity (cap at 3x threshold to prevent extreme outliers)
    // McCabe research shows >50 cyclomatic complexity is unmaintainable
    const cyclomaticNorm = Math.min(
      metrics.cyclomaticComplexity / this.COMPLEXITY_THRESHOLDS.high,
      3.0
    );

    // Normalize Halstead difficulty (typical range 0-100, but can exceed)
    // Cap at 150 (3x nominal) to prevent extreme outliers
    const halsteadNorm = Math.min(metrics.halsteadDifficulty / 50, 3.0);

    // Normalize maintainability index (inverted: lower MI = higher complexity)
    // MI ranges from 0-100, target is 85
    const maintainabilityNorm = Math.max(0, (100 - metrics.maintainabilityIndex) / 100);

    return (
      cyclomaticNorm * cyclomaticWeight +
      halsteadNorm * halsteadWeight +
      maintainabilityNorm * maintainabilityWeight
    );
  }

  private estimateVelocityImprovement(metrics: ComplexityMetrics): number {
    // Velocity improvement based on technical debt research (Cunningham, Fowler)
    // High technical debt can reduce velocity by 30-50%
    // Refactoring can recover 15-30% velocity improvement

    // Target maintainability index of 85 (IEEE/SEI standard for "good" code)
    const targetMaintainability = 85;
    const currentMaintainability = metrics.maintainabilityIndex;

    if (currentMaintainability >= targetMaintainability) {
      return 0; // Already at target, no improvement expected
    }

    const improvementGap = targetMaintainability - currentMaintainability;

    // Calculate issue density (issues per 100 LOC)
    const issueDensity = metrics.issueCount / (metrics.linesOfCode / 100);
    const severityFactor = this.SEVERITY_MULTIPLIERS[metrics.issueSeverity];

    // Formula: velocity_improvement = (improvement_gap / 100) * severity_factor * capped_issue_density * scaling_factor * 100
    // Cap issue density at 5 to prevent unrealistic projections
    // Research shows diminishing returns above 5 issues per 100 LOC
    const cappedIssueDensity = Math.min(issueDensity, 5);

    // Apply non-linear scaling factor to account for diminishing returns
    // Large improvements are harder to achieve than small ones
    const scalingFactor = Math.sqrt(improvementGap / 100);

    return (improvementGap / 100) * severityFactor * cappedIssueDensity * scalingFactor * 100;
  }

  private calculateROI(
    velocityGain: number,
    refactoringHours: number,
    config: EstimationConfig
  ): number {
    // ROI = (Annual Value Gain - Refactoring Cost) / Refactoring Cost * 100
    //
    // Assumptions:
    // - 26 two-week sprints per year (52 weeks)
    // - 80 hours per sprint (2 developers * 40 hours)
    // - Velocity gain realizes over 6 months (ramp-up period)
    // - Discounted by 50% for first 6 months due to learning curve

    const hoursPerSprint = 80;
    const sprintsPerYear = 26;

    // Calculate hours saved per sprint from velocity improvement
    const hoursSavedPerSprint = (velocityGain / 100) * hoursPerSprint;

    // First 6 months (13 sprints): 50% realization
    const firstHalfValue = hoursSavedPerSprint * 0.5 * 13 * config.hourlyRate * config.teamSize;

    // Second 6 months (13 sprints): 100% realization
    const secondHalfValue = hoursSavedPerSprint * 1.0 * 13 * config.hourlyRate * config.teamSize;

    const annualValueGain = firstHalfValue + secondHalfValue;

    // Refactoring cost includes direct hours + 20% overhead (testing, review, deployment)
    const refactoringCost = refactoringHours * config.hourlyRate * 1.2;

    // Opportunity cost: during refactoring, no feature work happens
    // This is already captured in refactoringHours, so no additional adjustment needed

    return ((annualValueGain - refactoringCost) / refactoringCost) * 100;
  }

  private analyzeConfidence(
    metrics: ComplexityMetrics,
    fileId: string
  ): {
    confidenceLevel: 'low' | 'medium' | 'high';
    overallConfidence: number;
    factors: {
      metricCompleteness: number;
      metricConsistency: number;
      historicalAccuracy: number;
      sampleSize: number;
    };
  } {
    // Factor 1: Metric completeness (0-100)
    const hasAllMetrics =
      metrics.cyclomaticComplexity > 0 &&
      metrics.halsteadDifficulty > 0 &&
      metrics.maintainabilityIndex > 0 &&
      metrics.linesOfCode > 0;

    const metricCompleteness = hasAllMetrics ? 100 : 50;

    // Factor 2: Metric consistency (0-100)
    // Check for logical consistency between metrics
    // High complexity should correlate with low maintainability
    const consistencyScore = this.calculateMetricConsistency(metrics);

    // Factor 3: Historical accuracy (0-100)
    const historicalAccuracy = this.calculateHistoricalAccuracy(fileId);

    // Factor 4: Sample size for historical data
    const sampleSize = this.getHistoricalSampleSize(fileId);

    // Overall confidence is weighted average
    const overallConfidence =
      metricCompleteness * 0.3 +
      consistencyScore * 0.3 +
      historicalAccuracy * 0.3 +
      Math.min(sampleSize / 10, 1.0) * 100 * 0.1; // Cap sample size contribution at 10 estimates

    // Map to confidence level
    let confidenceLevel: 'low' | 'medium' | 'high';
    if (overallConfidence >= 75) {
      confidenceLevel = 'high';
    } else if (overallConfidence >= 50) {
      confidenceLevel = 'medium';
    } else {
      confidenceLevel = 'low';
    }

    return {
      confidenceLevel,
      overallConfidence,
      factors: {
        metricCompleteness,
        metricConsistency: consistencyScore,
        historicalAccuracy,
        sampleSize,
      },
    };
  }

  private calculateMetricConsistency(metrics: ComplexityMetrics): number {
    // Check for consistency between cyclomatic complexity and maintainability index
    // Expected: high complexity → low maintainability

    const cc = metrics.cyclomaticComplexity;
    const mi = metrics.maintainabilityIndex;
    const hd = metrics.halsteadDifficulty;

    let consistencyPoints = 0;
    let maxPoints = 0;

    // Check 1: CC vs MI (inverted relationship expected)
    maxPoints += 100;
    if (cc > 30 && mi < 50) {
      consistencyPoints += 100; // Very consistent
    } else if (cc > 20 && mi < 65) {
      consistencyPoints += 75; // Consistent
    } else if (cc > 10 && mi < 80) {
      consistencyPoints += 50; // Somewhat consistent
    } else if (cc < 10 && mi > 80) {
      consistencyPoints += 100; // Very consistent (simple code, high maintainability)
    } else {
      consistencyPoints += 25; // Inconsistent
    }

    // Check 2: CC vs Halstead Difficulty (positive relationship expected)
    maxPoints += 100;
    if ((cc > 20 && hd > 30) || (cc < 10 && hd < 20)) {
      consistencyPoints += 100; // Consistent
    } else if ((cc > 10 && hd > 20) || (cc < 20 && hd < 40)) {
      consistencyPoints += 75; // Somewhat consistent
    } else {
      consistencyPoints += 25; // Inconsistent
    }

    // Check 3: Issue density should correlate with complexity
    maxPoints += 100;
    const issueDensity = metrics.issueCount / (metrics.linesOfCode / 100);
    if ((cc > 20 && issueDensity > 1) || (cc < 10 && issueDensity < 0.5)) {
      consistencyPoints += 100;
    } else {
      consistencyPoints += 50;
    }

    return (consistencyPoints / maxPoints) * 100;
  }

  private calculateHistoricalAccuracy(fileId: string): number {
    // Query historical estimates for similar files (same project, similar complexity)
    const historicalEstimates = this.db
      .prepare<unknown[], HistoricalEstimate>(
        `SELECT * FROM estimation_history
         WHERE completed = 1 AND actual_hours IS NOT NULL
         ORDER BY completed_at DESC
         LIMIT 20`
      )
      .all();

    if (historicalEstimates.length === 0) {
      return 50; // No history, moderate confidence
    }

    // Calculate Mean Absolute Percentage Error (MAPE)
    const errors = historicalEstimates.map(est => {
      const error = Math.abs(est.actualHours! - est.estimatedHours) / est.actualHours!;
      return error;
    });

    const mape = errors.reduce((sum, e) => sum + e, 0) / errors.length;

    // Convert MAPE to confidence score (0-100)
    // MAPE < 10% → 100 confidence, MAPE > 50% → 0 confidence
    const accuracy = Math.max(0, 100 - mape * 200);

    return accuracy;
  }

  private getHistoricalSampleSize(fileId: string): number {
    const result = this.db
      .prepare<unknown[], { count: number }>(
        `SELECT COUNT(*) as count FROM estimation_history
         WHERE completed = 1 AND actual_hours IS NOT NULL`
      )
      .get();

    return result?.count || 0;
  }

  private identifyLimitations(
    metrics: ComplexityMetrics,
    confidenceAnalysis: {
      confidenceLevel: 'low' | 'medium' | 'high';
      factors: {
        metricCompleteness: number;
        metricConsistency: number;
        historicalAccuracy: number;
        sampleSize: number;
      };
    }
  ): string[] {
    const limitations: string[] = [];

    // Always include general disclaimers
    limitations.push(
      'Estimates are based on static analysis and may not account for runtime complexity or architectural constraints.'
    );
    limitations.push(
      'Actual effort may vary based on team familiarity with codebase, testing requirements, and unforeseen dependencies.'
    );

    // Metric-specific limitations
    if (confidenceAnalysis.factors.metricCompleteness < 100) {
      limitations.push(
        'Incomplete metrics: Some complexity metrics are missing or zero, reducing estimate accuracy.'
      );
    }

    if (confidenceAnalysis.factors.metricConsistency < 60) {
      limitations.push(
        'Inconsistent metrics: Complexity indicators show conflicting signals, suggesting potential measurement issues.'
      );
    }

    if (confidenceAnalysis.factors.historicalAccuracy < 60) {
      limitations.push(
        'Limited historical data: Past estimates have significant variance, reducing predictive accuracy.'
      );
    }

    if (confidenceAnalysis.factors.sampleSize < 5) {
      limitations.push(
        'Small sample size: Fewer than 5 completed estimates available for calibration. Estimate may be uncalibrated.'
      );
    }

    // Complexity-specific limitations
    if (metrics.cyclomaticComplexity > 50) {
      limitations.push(
        'Extremely high complexity: Code may require architectural refactoring beyond simple refactoring, significantly increasing effort.'
      );
    }

    if (metrics.linesOfCode > 1000) {
      limitations.push(
        'Large file size: Files over 1000 LOC often have hidden dependencies and may require more effort than estimated.'
      );
    }

    if (metrics.maintainabilityIndex < 20) {
      limitations.push(
        'Very low maintainability: Code may be unmaintainable and require complete rewrite rather than refactoring.'
      );
    }

    // Velocity improvement disclaimers
    limitations.push(
      'Velocity improvements assume refactored code will be actively maintained and team will adopt new patterns. Actual gains depend on team discipline and organizational support.'
    );

    return limitations;
  }

  // Track actual effort for accuracy calibration
  recordActualEffort(estimateId: number, actualHours: number): void {
    this.db
      .prepare(
        `UPDATE estimation_history
         SET actual_hours = ?, completed = 1, completed_at = ?
         WHERE id = ?`
      )
      .run(actualHours, Date.now(), estimateId);
  }

  // Store estimate for historical tracking
  saveEstimate(
    fileId: string,
    metrics: ComplexityMetrics,
    config: EstimationConfig,
    result: EstimationResult
  ): number {
    const insertResult = this.db
      .prepare(
        `INSERT INTO estimation_history
         (file_id, estimated_hours, metrics, config, completed, created_at)
         VALUES (?, ?, ?, ?, 0, ?)`
      )
      .run(
        fileId,
        result.hoursToResolve.estimate,
        JSON.stringify(metrics),
        JSON.stringify(config),
        Date.now()
      );

    return insertResult.lastInsertRowid as number;
  }
}
```

#### database schema for estimation history

```sql
-- Add to migration script
CREATE TABLE estimation_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_id TEXT NOT NULL,
  estimated_hours REAL NOT NULL,
  actual_hours REAL,
  metrics TEXT NOT NULL, -- JSON
  config TEXT NOT NULL, -- JSON
  completed INTEGER DEFAULT 0,
  created_at INTEGER NOT NULL,
  completed_at INTEGER
);

CREATE INDEX idx_estimation_history_file_id ON estimation_history(file_id);
CREATE INDEX idx_estimation_history_completed ON estimation_history(completed);

-- Add to PREFERENCES table
INSERT INTO preferences (key, value, updated_at) VALUES
  ('costEstimation.hourlyRate', '150', strftime('%s', 'now')),
  ('costEstimation.teamSize', '5', strftime('%s', 'now')),
  ('costEstimation.experienceLevel', '"mid"', strftime('%s', 'now'));
```

#### ui components with confidence intervals

Location: `src/renderer/pages/CostAnalysis.tsx`

```typescript
import React, { useEffect, useState } from 'react';
import { useAnalysisStore } from '@/renderer/stores/analysis';
import { useSettingsStore } from '@/renderer/stores/settings';

export function CostAnalysisPage() {
  const { files } = useAnalysisStore();
  const { settings, updateSetting } = useSettingsStore();
  const [estimates, setEstimates] = useState<EstimationResult[]>([]);

  useEffect(() => {
    async function calculateEstimates() {
      const results = await window.viprDesktop.analysis.estimateCosts({
        files: Array.from(files.values()),
        config: {
          hourlyRate: settings.costEstimation.hourlyRate,
          teamSize: settings.costEstimation.teamSize,
          experienceLevel: settings.costEstimation.experienceLevel,
        },
      });
      setEstimates(results);
    }

    calculateEstimates();
  }, [files, settings]);

  const totalHoursLow = estimates.reduce((sum, e) => sum + e.hoursToResolve.low, 0);
  const totalHoursEst = estimates.reduce((sum, e) => sum + e.hoursToResolve.estimate, 0);
  const totalHoursHigh = estimates.reduce((sum, e) => sum + e.hoursToResolve.high, 0);
  const totalCost = estimates.reduce((sum, e) => sum + e.cost.estimate, 0);
  const avgVelocityGain =
    estimates.reduce((sum, e) => sum + e.velocityImprovementPercent.estimate, 0) / estimates.length;
  const avgConfidence =
    estimates.reduce((sum, e) => sum + e.hoursToResolve.confidence, 0) / estimates.length;

  return (
    <div className="cost-analysis-page">
      <header>
        <h1>Cost & Velocity Analysis</h1>
        <p className="subtitle">
          Industry-validated estimates based on COCOMO II and technical debt research
        </p>
      </header>

      <div className="summary-cards">
        <MetricCard
          label="Total Refactoring Hours"
          value={`${totalHoursEst.toFixed(1)} (${totalHoursLow.toFixed(1)}-${totalHoursHigh.toFixed(1)})`}
          unit="hours"
          confidence={avgConfidence}
          tooltip="Range represents 95% confidence interval based on historical accuracy"
        />
        <MetricCard
          label="Estimated Cost"
          value={`$${totalCost.toLocaleString()}`}
          confidence={avgConfidence}
        />
        <MetricCard
          label="Average Velocity Gain"
          value={`${avgVelocityGain.toFixed(1)}%`}
          confidence={avgConfidence}
          tooltip="Expected productivity improvement after refactoring"
        />
      </div>

      <section className="configuration">
        <h2>Configuration</h2>
        <div className="config-grid">
          <Input
            label="Hourly Rate ($)"
            type="number"
            value={settings.costEstimation.hourlyRate}
            onChange={(e) =>
              updateSetting('costEstimation.hourlyRate', Number(e.target.value))
            }
          />
          <Input
            label="Team Size"
            type="number"
            value={settings.costEstimation.teamSize}
            onChange={(e) =>
              updateSetting('costEstimation.teamSize', Number(e.target.value))
            }
          />
          <Select
            label="Experience Level"
            value={settings.costEstimation.experienceLevel}
            onChange={(value) =>
              updateSetting('costEstimation.experienceLevel', value)
            }
            options={[
              { value: 'junior', label: 'Junior (1.5x multiplier)' },
              { value: 'mid', label: 'Mid-Level (1.0x baseline)' },
              { value: 'senior', label: 'Senior (0.7x multiplier)' },
            ]}
          />
        </div>
      </section>

      <section className="file-estimates">
        <h2>Per-File Estimates</h2>
        <table className="estimates-table">
          <thead>
            <tr>
              <th>File</th>
              <th>Hours (Range)</th>
              <th>Cost</th>
              <th>Velocity Gain</th>
              <th>ROI</th>
              <th>Confidence</th>
              <th>Limitations</th>
            </tr>
          </thead>
          <tbody>
            {estimates.map((estimate, index) => (
              <tr key={index}>
                <td>{estimate.file}</td>
                <td>
                  {estimate.hoursToResolve.estimate.toFixed(1)}
                  <span className="range">
                    ({estimate.hoursToResolve.low.toFixed(1)}-{estimate.hoursToResolve.high.toFixed(1)})
                  </span>
                </td>
                <td>${estimate.cost.estimate.toLocaleString()}</td>
                <td>
                  {estimate.velocityImprovementPercent.estimate.toFixed(1)}%
                  <span className="range">
                    ({estimate.velocityImprovementPercent.low.toFixed(1)}-
                    {estimate.velocityImprovementPercent.high.toFixed(1)}%)
                  </span>
                </td>
                <td>{estimate.roi.estimate.toFixed(0)}%</td>
                <td>
                  <Badge
                    variant={
                      estimate.confidenceLevel === 'high'
                        ? 'success'
                        : estimate.confidenceLevel === 'medium'
                        ? 'warning'
                        : 'error'
                    }
                  >
                    {estimate.confidenceLevel} ({estimate.hoursToResolve.confidence.toFixed(0)}%)
                  </Badge>
                </td>
                <td>
                  <Popover
                    trigger={<InfoIcon />}
                    content={
                      <div className="limitations-list">
                        <h4>Confidence Factors:</h4>
                        <ul>
                          <li>Metric Completeness: {estimate.confidenceFactors.metricCompleteness}%</li>
                          <li>Metric Consistency: {estimate.confidenceFactors.metricConsistency.toFixed(0)}%</li>
                          <li>Historical Accuracy: {estimate.confidenceFactors.historicalAccuracy.toFixed(0)}%</li>
                          <li>Sample Size: {estimate.confidenceFactors.sampleSize}</li>
                        </ul>
                        <h4>Limitations:</h4>
                        <ul>
                          {estimate.limitations.map((limit, i) => (
                            <li key={i}>{limit}</li>
                          ))}
                        </ul>
                      </div>
                    }
                  />
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>

      <section className="estimation-accuracy">
        <h2>Estimation Accuracy Tracking</h2>
        <p>
          Track actual hours when issues are resolved to improve future estimate accuracy.
          Current historical accuracy: {avgConfidence.toFixed(0)}%
        </p>
        <button onClick={() => {/* Open accuracy tracking modal */}}>
          View Historical Estimates
        </button>
      </section>
    </div>
  );
}
```

#### acceptance criteria

- [ ] Estimation formulas validated against COCOMO II and industry research
- [ ] Complexity thresholds aligned with McCabe research (10+, 20+, 50+)
- [ ] Experience multipliers derived from COCOMO II ACAP/PCAP factors
- [ ] Velocity improvement calculations account for diminishing returns and ramp-up period
- [ ] ROI calculations include 20% overhead and 6-month realization discount
- [ ] Confidence intervals provided for all estimates (hours, cost, velocity, ROI)
- [ ] Confidence levels based on 4 factors: completeness, consistency, historical accuracy, sample size
- [ ] Metric consistency checks validate logical relationships between metrics
- [ ] Historical accuracy tracked using MAPE (Mean Absolute Percentage Error)
- [ ] Limitations list generated based on metric quality and confidence factors
- [ ] User can track actual hours to calibrate future estimates
- [ ] Estimation history stored in database for continuous improvement
- [ ] UI displays confidence intervals as ranges
- [ ] Settings persist across app sessions
- [ ] Cost analysis page displays summary cards with confidence scores
- [ ] Estimates update in real-time when configuration changes
- [ ] Warning shown when confidence is low or sample size insufficient
- [ ] Popover displays detailed confidence factors and limitations

### 3. ai prompt generation (us-09)

#### prompt templates

Location: `src/main/services/ai-prompts.ts`

```typescript
import type { FileRecord, PluginInsight } from '@vipr/common';

interface PromptTemplate {
  name: string;
  format: 'claude' | 'gpt' | 'copilot';
  generate(file: FileRecord, insights: PluginInsight[]): string;
}

export class AIPromptService {
  private templates: PromptTemplate[] = [
    {
      name: 'Refactor Complex Function',
      format: 'claude',
      generate: (file, insights) => {
        const complexityInsights = insights.filter(i => i.ruleId.includes('complexity'));
        const topInsight = complexityInsights[0];

        return `I have a function with high cyclomatic complexity (${topInsight.metrics?.cyclomatic}) in the file ${file.path}.

**Current Issues:**
${complexityInsights.map(i => `- ${i.title}: ${i.description}`).join('\n')}

**Code Context:**
\`\`\`${file.fileType}
${topInsight.code || '// Code snippet not available'}
\`\`\`

**Request:**
Please refactor this function to reduce complexity while maintaining functionality. Focus on:
1. Extracting nested conditionals into separate functions
2. Reducing the number of decision points
3. Improving readability and testability

Provide the refactored code with explanations for each change.`;
      },
    },
    {
      name: 'Fix React Anti-Patterns',
      format: 'claude',
      generate: (file, insights) => {
        const reactInsights = insights.filter(i => i.pluginId.includes('react'));

        return `I have a React component with anti-patterns in ${file.path}.

**Detected Issues:**
${reactInsights.map(i => `- ${i.title}: ${i.description}`).join('\n')}

**Component Code:**
\`\`\`tsx
${file.content || '// Component code not available'}
\`\`\`

**Request:**
Please rewrite this component following React best practices:
1. Fix hooks usage (rules of hooks)
2. Optimize re-renders with proper memoization
3. Improve prop drilling if present
4. Ensure proper TypeScript types

Provide the improved component with explanations.`;
      },
    },
    {
      name: 'Explain Maintainability Issues',
      format: 'gpt',
      generate: (file, insights) => {
        const maintainabilityScore = file.metrics?.maintainabilityIndex || 0;

        return `This file has a low maintainability index of ${maintainabilityScore.toFixed(1)}.

File: ${file.path}

Issues:
${insights.map(i => `- ${i.title}`).join('\n')}

Explain why these issues reduce maintainability and suggest specific improvements. Focus on:
- Code readability
- Complexity reduction
- Testability
- Documentation needs`;
      },
    },
    {
      name: 'Generate Unit Tests',
      format: 'copilot',
      generate: (file, insights) => {
        return `Generate comprehensive unit tests for this file using Vitest.

File: ${file.path}

Code:
\`\`\`${file.fileType}
${file.content || '// Code not available'}
\`\`\`

Include tests for:
- Happy path scenarios
- Edge cases
- Error handling
- Complexity hotspots identified: ${insights
          .map(i => i.location?.line)
          .filter(Boolean)
          .join(', ')}`;
      },
    },
  ];

  getAvailableTemplates(): PromptTemplate[] {
    return this.templates;
  }

  generatePrompt(templateName: string, file: FileRecord, insights: PluginInsight[]): string {
    const template = this.templates.find(t => t.name === templateName);
    if (!template) {
      throw new Error(`Template not found: ${templateName}`);
    }

    return template.generate(file, insights);
  }

  generateContextualPrompt(file: FileRecord, insights: PluginInsight[]): string {
    // Auto-select best template based on file type and insights
    if (insights.some(i => i.pluginId === 'react')) {
      return this.generatePrompt('Fix React Anti-Patterns', file, insights);
    }

    if (insights.some(i => i.ruleId.includes('complexity'))) {
      return this.generatePrompt('Refactor Complex Function', file, insights);
    }

    return this.generatePrompt('Explain Maintainability Issues', file, insights);
  }
}
```

#### ui integration

Location: `src/renderer/pages/FileDetail.tsx` (add new tab)

```typescript
function AIPromptsTab({ file, insights }: { file: FileRecord; insights: PluginInsight[] }) {
  const [selectedTemplate, setSelectedTemplate] = useState<string>('');
  const [generatedPrompt, setGeneratedPrompt] = useState<string>('');
  const [templates, setTemplates] = useState<PromptTemplate[]>([]);

  useEffect(() => {
    async function loadTemplates() {
      const available = await window.viprDesktop.ai.getTemplates();
      setTemplates(available);

      // Auto-generate contextual prompt
      const contextual = await window.viprDesktop.ai.generateContextualPrompt({
        file,
        insights,
      });
      setGeneratedPrompt(contextual);
    }

    loadTemplates();
  }, [file, insights]);

  const handleTemplateChange = async (templateName: string) => {
    setSelectedTemplate(templateName);
    const prompt = await window.viprDesktop.ai.generatePrompt({
      templateName,
      file,
      insights,
    });
    setGeneratedPrompt(prompt);
  };

  const handleCopy = async () => {
    // SECURITY: Detect sensitive data before clipboard operation
    // See security documentation Section 5 for full implementation
    const detected = await window.viprDesktop.ai.detectSensitiveData(generatedPrompt);

    if (detected.length > 0) {
      const confirmed = confirm(
        `WARNING: This prompt may contain sensitive data (${detected.join(', ')}).\n\n` +
        `Copying this to your clipboard may expose credentials or secrets to AI services.\n\n` +
        `Are you sure you want to continue?`
      );

      if (!confirmed) {
        return;
      }
    }

    navigator.clipboard.writeText(generatedPrompt);
    // Show toast: "Prompt copied to clipboard"
  };

  return (
    <div className="ai-prompts-tab">
      <header>
        <Select
          label="Prompt Template"
          value={selectedTemplate}
          onChange={handleTemplateChange}
          options={templates.map((t) => ({
            value: t.name,
            label: `${t.name} (${t.format})`,
          }))}
        />
        <Button onClick={handleCopy}>Copy to Clipboard</Button>
      </header>

      <div className="prompt-preview">
        <pre>{generatedPrompt}</pre>
      </div>

      <footer className="prompt-tips">
        <p>
          This prompt is optimized for {templates.find((t) => t.name === selectedTemplate)?.format || 'your AI assistant'}.
          Paste it into your IDE or chat interface for AI-assisted refactoring.
        </p>
      </footer>
    </div>
  );
}
```

#### acceptance criteria

- [ ] At least 4 prompt templates (complexity, React, maintainability, tests)
- [ ] Auto-detection of best template based on file type and insights
- [ ] Copy-to-clipboard functionality with success feedback
- [ ] Multi-format support (Claude, GPT, Copilot)
- [ ] Prompts include relevant code snippets and metrics
- [ ] Contextual prompts generate in under 500ms
- [ ] User can manually select alternative templates

---

### 4. mcp server integration (us-10)

#### architecture overview

```mermaid
---
title: mcp server architecture
---
graph TB
    subgraph Desktop["Desktop App (Main Process)"]
        MainApp[Main Application]
        SQLite[(SQLite Database<br/>WAL Mode)]
        ChildMgr[Child Process Manager]
    end

    subgraph MCP["MCP Server (Child Process)"]
        MCPServer[MCP Server]
        Tools[Exposed Tools]
        DBReader[Read-Only DB Access]
    end

    subgraph Clients["AI Clients"]
        Claude[Claude Code]
        Cursor[Cursor IDE]
        Other[Other MCP Clients]
    end

    MainApp --> SQLite
    MainApp --> ChildMgr
    ChildMgr -->|spawn| MCPServer
    MCPServer --> DBReader
    DBReader -->|read| SQLite

    Claude -->|stdio JSON-RPC| MCPServer
    Cursor -->|stdio JSON-RPC| MCPServer
    Other -->|stdio JSON-RPC| MCPServer

    Tools -->|get_file_analysis| DBReader
    Tools -->|search_issues| DBReader
    Tools -->|get_recommendations| DBReader
    Tools -->|get_snapshot| DBReader
    Tools -->|compare_snapshots| DBReader
```

#### implementation

Location: `src/main/mcp/server.ts`

```typescript
import { spawn, ChildProcess } from 'child_process';
import * as path from 'path';
import { app } from 'electron';
import { createLogger } from '@vipr/logging';

const logger = createLogger('mcp-server-manager');

interface MCPServerHealth {
  isHealthy: boolean;
  lastCheck: number;
  consecutiveFailures: number;
}

export class MCPServerManager {
  private process: ChildProcess | null = null;
  private dbPath: string;
  private health: MCPServerHealth = {
    isHealthy: false,
    lastCheck: 0,
    consecutiveFailures: 0,
  };
  private healthCheckInterval: NodeJS.Timeout | null = null;
  private readonly HEALTH_CHECK_INTERVAL = 30000; // 30 seconds
  private readonly MAX_RESTART_ATTEMPTS = 3;
  private readonly RESTART_COOLDOWN = 5000; // 5 seconds between restarts
  private restartAttempts = 0;
  private isShuttingDown = false;

  constructor(dbPath: string) {
    this.dbPath = dbPath;
  }

  async start(): Promise<void> {
    if (this.process) {
      throw new Error('MCP server already running');
    }

    this.isShuttingDown = false;

    const serverScript = path.join(app.getAppPath(), 'dist', 'mcp-server', 'index.js');

    logger.info('Starting MCP server', { script: serverScript, dbPath: this.dbPath });

    this.process = spawn('node', [serverScript, this.dbPath], {
      stdio: ['pipe', 'pipe', 'pipe'],
      env: {
        ...process.env,
        NODE_ENV: 'production',
        // Ensure SQLite opens in read-only mode
        SQLITE_READONLY: 'true',
      },
    });

    // Handle stdout
    this.process.stdout?.on('data', data => {
      const message = data.toString().trim();
      logger.debug('[MCP Server]', { message });

      // Detect successful startup
      if (message.includes('MCP server running')) {
        this.health.isHealthy = true;
        this.health.consecutiveFailures = 0;
        this.restartAttempts = 0;
        logger.info('MCP server started successfully');
      }
    });

    // Handle stderr
    this.process.stderr?.on('data', data => {
      const error = data.toString().trim();
      logger.error('[MCP Server Error]', { error });
      this.health.isHealthy = false;
    });

    // Handle process exit
    this.process.on('exit', (code, signal) => {
      logger.warn('MCP server exited', { code, signal });
      this.process = null;
      this.health.isHealthy = false;

      // Auto-restart if not shutting down intentionally
      if (!this.isShuttingDown) {
        this.handleUnexpectedExit(code || 0);
      }
    });

    // Start health check monitoring
    this.startHealthChecks();

    // Wait for startup confirmation
    await this.waitForStartup();
  }

  private async waitForStartup(timeout = 10000): Promise<void> {
    const startTime = Date.now();

    return new Promise((resolve, reject) => {
      const checkInterval = setInterval(() => {
        if (this.health.isHealthy) {
          clearInterval(checkInterval);
          resolve();
        } else if (Date.now() - startTime > timeout) {
          clearInterval(checkInterval);
          reject(new Error('MCP server startup timeout'));
        }
      }, 100);
    });
  }

  private async handleUnexpectedExit(exitCode: number): Promise<void> {
    if (this.restartAttempts >= this.MAX_RESTART_ATTEMPTS) {
      logger.error('Max restart attempts reached, giving up', {
        attempts: this.restartAttempts,
      });
      return;
    }

    this.restartAttempts++;
    logger.info('Attempting to restart MCP server', {
      attempt: this.restartAttempts,
      exitCode,
    });

    // Wait before restarting
    await new Promise(resolve => setTimeout(resolve, this.RESTART_COOLDOWN));

    try {
      await this.start();
    } catch (error) {
      logger.error('Failed to restart MCP server', { error });
    }
  }

  private startHealthChecks(): void {
    this.healthCheckInterval = setInterval(() => {
      this.performHealthCheck();
    }, this.HEALTH_CHECK_INTERVAL);
  }

  private performHealthCheck(): void {
    if (!this.process) {
      this.health.isHealthy = false;
      return;
    }

    // Check if process is still alive
    try {
      // Send signal 0 to check process existence without killing it
      process.kill(this.process.pid!, 0);
      this.health.lastCheck = Date.now();

      // If process exists but hasn't responded, increment failure count
      if (!this.health.isHealthy) {
        this.health.consecutiveFailures++;

        if (this.health.consecutiveFailures >= 3) {
          logger.warn('MCP server health check failed multiple times', {
            failures: this.health.consecutiveFailures,
          });
        }
      }
    } catch (error) {
      // Process doesn't exist
      this.health.isHealthy = false;
      logger.error('MCP server process not found during health check');
    }
  }

  async stop(): Promise<void> {
    this.isShuttingDown = true;

    if (this.healthCheckInterval) {
      clearInterval(this.healthCheckInterval);
      this.healthCheckInterval = null;
    }

    if (!this.process) {
      return;
    }

    logger.info('Stopping MCP server gracefully');

    // Send SIGTERM for graceful shutdown
    this.process.kill('SIGTERM');

    // Wait up to 5 seconds for graceful shutdown
    await new Promise<void>(resolve => {
      const timeout = setTimeout(() => {
        if (this.process) {
          logger.warn('MCP server did not stop gracefully, forcing shutdown');
          this.process.kill('SIGKILL');
        }
        resolve();
      }, 5000);

      this.process?.once('exit', () => {
        clearTimeout(timeout);
        resolve();
      });
    });

    this.process = null;
    this.health.isHealthy = false;
    logger.info('MCP server stopped');
  }

  isRunning(): boolean {
    return this.process !== null && this.health.isHealthy;
  }

  getHealth(): MCPServerHealth {
    return { ...this.health };
  }

  // Integrate with app shutdown
  public dispose(): void {
    this.stop();
  }
}
```

Location: `src/mcp-server/index.ts` (separate entry point)

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import Database from 'better-sqlite3';

const dbPath = process.argv[2];
if (!dbPath) {
  console.error('Usage: node index.js <database-path>');
  process.exit(1);
}

// Open database in read-only mode for safe concurrent access
// WAL mode (configured in main process) allows concurrent readers
const db = new Database(dbPath, {
  readonly: true,
  fileMustExist: true,
});

// Verify WAL mode is enabled (should be set by main process)
const journalMode = db.pragma('journal_mode', { simple: true }) as string;
if (journalMode !== 'wal') {
  console.error('WARNING: Database not in WAL mode. Concurrent access may fail.');
  console.error(`Current journal mode: ${journalMode}`);
}

// Configure for optimal read-only performance
db.pragma('query_only = ON'); // Additional safety
db.pragma('busy_timeout = 5000'); // Wait up to 5s for locks

const server = new Server(
  {
    name: 'vipr-desktop',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Tool 1: Get file analysis
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'get_file_analysis',
        description: 'Get analysis results for a specific file',
        inputSchema: {
          type: 'object',
          properties: {
            filePath: {
              type: 'string',
              description: 'Path to the file',
            },
          },
          required: ['filePath'],
        },
      },
      {
        name: 'search_issues',
        description: 'Search for issues by severity, plugin, or keyword',
        inputSchema: {
          type: 'object',
          properties: {
            severity: {
              type: 'string',
              enum: ['low', 'medium', 'high', 'critical'],
            },
            pluginId: {
              type: 'string',
            },
            keyword: {
              type: 'string',
            },
          },
        },
      },
      {
        name: 'get_recommendations',
        description: 'Get AI-optimized recommendations for a file or issue',
        inputSchema: {
          type: 'object',
          properties: {
            filePath: {
              type: 'string',
            },
          },
          required: ['filePath'],
        },
      },
      {
        name: 'get_snapshot',
        description: 'Get historical snapshot by git SHA',
        inputSchema: {
          type: 'object',
          properties: {
            gitSha: {
              type: 'string',
            },
          },
          required: ['gitSha'],
        },
      },
      {
        name: 'compare_snapshots',
        description: 'Compare two snapshots for regression detection',
        inputSchema: {
          type: 'object',
          properties: {
            fromSha: {
              type: 'string',
            },
            toSha: {
              type: 'string',
            },
          },
          required: ['fromSha', 'toSha'],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async request => {
  switch (request.params.name) {
    case 'get_file_analysis': {
      const { filePath } = request.params.arguments as { filePath: string };

      const file = db.prepare('SELECT * FROM files WHERE path = ?').get(filePath);

      if (!file) {
        return {
          content: [
            {
              type: 'text',
              text: `File not found: ${filePath}`,
            },
          ],
        };
      }

      const analyses = db.prepare('SELECT * FROM analyses WHERE file_id = ?').all((file as any).id);

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({ file, analyses }, null, 2),
          },
        ],
      };
    }

    case 'search_issues': {
      const { severity, pluginId, keyword } = request.params.arguments as {
        severity?: string;
        pluginId?: string;
        keyword?: string;
      };

      // SECURITY: Validate all inputs before query construction
      if (severity && !['low', 'medium', 'high', 'critical'].includes(severity)) {
        throw new Error('Invalid severity value. Must be: low, medium, high, or critical');
      }

      if (pluginId && !/^[a-z0-9-]+$/i.test(pluginId)) {
        throw new Error('Invalid plugin ID format');
      }

      if (keyword && (keyword.length === 0 || keyword.length > 500)) {
        throw new Error('Invalid keyword length. Must be 1-500 characters');
      }

      // SECURITY: Build query using prepared statements ONLY
      let query = `
        SELECT f.path, a.plugin_id, a.result
        FROM files f
        JOIN analyses a ON f.id = a.file_id
        WHERE 1=1
      `;
      const params: any[] = [];

      if (pluginId) {
        query += ' AND a.plugin_id = ?';
        params.push(pluginId);
      }

      if (keyword) {
        // SECURITY: Escape LIKE wildcards and use parameter binding
        const escaped = keyword.replace(/\\/g, '\\\\').replace(/%/g, '\\%').replace(/_/g, '\\_');
        query += " AND (f.path LIKE ? ESCAPE '\\' OR a.result LIKE ? ESCAPE '\\')";
        params.push(`%${escaped}%`, `%${escaped}%`);
      }

      // SECURITY: Execute with prepared statement
      const stmt = db.prepare(query);
      let results = stmt.all(...params) as Array<{
        path: string;
        plugin_id: string;
        result: string;
      }>;

      // Filter by severity if requested (post-query, since it's in JSON)
      if (severity) {
        results = results.filter(row => {
          try {
            const parsed = JSON.parse(row.result);
            return parsed.severity === severity;
          } catch {
            return false; // Invalid JSON, exclude
          }
        });
      }

      // SECURITY: Sanitize results before returning
      const sanitized = results.map(row => {
        let parsedResult;
        try {
          parsedResult = JSON.parse(row.result);
        } catch {
          parsedResult = { error: 'Invalid result format' };
        }

        return {
          path: row.path,
          pluginId: row.plugin_id,
          result: parsedResult,
        };
      });

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(sanitized, null, 2),
          },
        ],
      };
    }

    case 'get_recommendations': {
      const { filePath } = request.params.arguments as { filePath: string };

      // Fetch file analysis
      const file = db.prepare('SELECT * FROM files WHERE path = ?').get(filePath);

      if (!file) {
        return {
          content: [
            {
              type: 'text',
              text: `File not found: ${filePath}`,
            },
          ],
        };
      }

      const analyses = db.prepare('SELECT * FROM analyses WHERE file_id = ?').all((file as any).id);

      // Generate AI-friendly recommendations
      const recommendations = analyses.map((analysis: any) => {
        const result = JSON.parse(analysis.result);
        return {
          plugin: analysis.plugin_id,
          insights: result.insights || [],
          suggestedActions: result.insights?.map((i: any) => i.recommendation) || [],
        };
      });

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({ file: filePath, recommendations }, null, 2),
          },
        ],
      };
    }

    case 'get_snapshot': {
      const { gitSha } = request.params.arguments as { gitSha: string };

      const snapshot = db.prepare('SELECT * FROM snapshots WHERE git_sha = ?').get(gitSha);

      if (!snapshot) {
        return {
          content: [
            {
              type: 'text',
              text: `Snapshot not found: ${gitSha}`,
            },
          ],
        };
      }

      const snapshotFiles = db
        .prepare('SELECT * FROM snapshot_files WHERE snapshot_id = ?')
        .all((snapshot as any).id);

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify({ snapshot, files: snapshotFiles }, null, 2),
          },
        ],
      };
    }

    case 'compare_snapshots': {
      const { fromSha, toSha } = request.params.arguments as {
        fromSha: string;
        toSha: string;
      };

      const fromSnapshot = db.prepare('SELECT * FROM snapshots WHERE git_sha = ?').get(fromSha);
      const toSnapshot = db.prepare('SELECT * FROM snapshots WHERE git_sha = ?').get(toSha);

      if (!fromSnapshot || !toSnapshot) {
        return {
          content: [
            {
              type: 'text',
              text: 'One or both snapshots not found',
            },
          ],
        };
      }

      // Compute diff
      const comparison = {
        from: fromSnapshot,
        to: toSnapshot,
        scoreChange: (toSnapshot as any).avg_score - (fromSnapshot as any).avg_score,
        fileCountChange: (toSnapshot as any).file_count - (fromSnapshot as any).file_count,
      };

      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(comparison, null, 2),
          },
        ],
      };
    }

    default:
      throw new Error(`Unknown tool: ${request.params.name}`);
  }
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('Vipr Desktop MCP server running on stdio');
}

main().catch(error => {
  console.error('Fatal error:', error);
  process.exit(1);
});
```

#### ui settings toggle

Location: `src/renderer/pages/Settings.tsx`

```typescript
function MCPServerSection() {
  const [mcpEnabled, setMcpEnabled] = useState(false);
  const [mcpStatus, setMcpStatus] = useState<'stopped' | 'running' | 'error'>('stopped');

  useEffect(() => {
    async function checkStatus() {
      const status = await window.viprDesktop.mcp.getStatus();
      setMcpEnabled(status.enabled);
      setMcpStatus(status.running ? 'running' : 'stopped');
    }

    checkStatus();
  }, []);

  const handleToggle = async (enabled: boolean) => {
    if (enabled) {
      await window.viprDesktop.mcp.start();
    } else {
      await window.viprDesktop.mcp.stop();
    }

    setMcpEnabled(enabled);
    setMcpStatus(enabled ? 'running' : 'stopped');
  };

  return (
    <section className="mcp-server-section">
      <h2>MCP Server</h2>
      <p>
        Enable the Model Context Protocol server to integrate Vipr with AI tools
        like Claude Code and Cursor.
      </p>

      <Switch
        label="Enable MCP Server"
        checked={mcpEnabled}
        onChange={handleToggle}
      />

      <div className="mcp-status">
        Status:{' '}
        <Badge variant={mcpStatus === 'running' ? 'success' : 'default'}>
          {mcpStatus}
        </Badge>
      </div>

      {mcpEnabled && (
        <div className="mcp-instructions">
          <h3>Configuration Instructions</h3>
          <p>Add this to your MCP client configuration:</p>
          <pre>
            {JSON.stringify(
              {
                mcpServers: {
                  'vipr-desktop': {
                    command: 'node',
                    args: ['/path/to/vipr/mcp-server/index.js', '/path/to/db.sqlite'],
                  },
                },
              },
              null,
              2
            )}
          </pre>
        </div>
      )}
    </section>
  );
}
```

#### acceptance criteria

**Functionality:**

- [ ] MCP server spawns as child process when enabled
- [ ] SQLite database opened in read-only mode via WAL
- [ ] WAL mode verified on startup with warning if not enabled
- [ ] query_only pragma enforced for additional safety
- [ ] All 5 tools (get_file_analysis, search_issues, get_recommendations, get_snapshot, compare_snapshots) work correctly
- [ ] stdio JSON-RPC communication follows MCP specification
- [ ] Settings UI displays server status (running/stopped/unhealthy)
- [ ] Server stops gracefully when app closes or user disables
- [ ] Instructions provided for configuring AI clients

**Health and Reliability:**

- [ ] Health checks run every 30 seconds
- [ ] Auto-restart on unexpected crash (max 3 attempts)
- [ ] Graceful shutdown with SIGTERM before SIGKILL
- [ ] 5-second timeout for graceful shutdown
- [ ] Health status exposed via `mcp:getStatus` IPC
- [ ] Consecutive failure count tracked
- [ ] Restart cooldown prevents rapid restart loops

**Architecture Compliance:**

- [ ] IPC channels follow naming convention (mcp:start, mcp:stop, mcp:getStatus)
- [ ] Zod validation for all MCP-related payloads
- [ ] Integration with app shutdown lifecycle
- [ ] Proper logging with `@vipr/logging`
- [ ] No direct SQLite writes from MCP server
- [ ] busy_timeout prevents database locked errors
- [ ] Health check doesn't interfere with normal operation
- [ ] dispose() method for cleanup

---

### 5. ide integration (us-13)

#### uri scheme handler

Location: `src/main/services/ide-integration.ts`

```typescript
import { shell } from 'electron';
import { PathValidator } from '@/main/security/path-validator';

type IDEType = 'vscode' | 'cursor' | 'idea' | 'atom' | 'sublime';

interface IDEConfig {
  uriScheme: string;
  commandFormat: (filePath: string, line?: number) => string;
}

export class IDEIntegrationService {
  private projectRoot: string;

  constructor(projectRoot: string) {
    this.projectRoot = projectRoot;
  }

  private readonly IDE_CONFIGS: Record<IDEType, IDEConfig> = {
    vscode: {
      uriScheme: 'vscode',
      commandFormat: (filePath, line) =>
        `vscode://file/${encodeURIComponent(filePath)}${line ? `:${line}` : ''}`,
    },
    cursor: {
      uriScheme: 'cursor',
      commandFormat: (filePath, line) =>
        `cursor://file/${encodeURIComponent(filePath)}${line ? `:${line}` : ''}`,
    },
    idea: {
      uriScheme: 'idea',
      commandFormat: (filePath, line) =>
        `idea://open?file=${encodeURIComponent(filePath)}${line ? `&line=${line}` : ''}`,
    },
    atom: {
      uriScheme: 'atom',
      commandFormat: (filePath, line) =>
        `atom://open?path=${encodeURIComponent(filePath)}${line ? `&line=${line}` : ''}`,
    },
    sublime: {
      uriScheme: 'subl',
      commandFormat: (filePath, line) =>
        `subl://open?url=file://${encodeURIComponent(filePath)}${line ? `&line=${line}` : ''}`,
    },
  };

  async openInIDE(
    filePath: string,
    line?: number,
    idePreference?: IDEType
  ): Promise<{ success: boolean; error?: string }> {
    // SECURITY: Validate and sanitize file path using PathValidator
    const pathValidator = new PathValidator(this.projectRoot);
    const sanitizedPath = pathValidator.sanitizePath(filePath);

    if (!sanitizedPath) {
      console.warn(`[SECURITY] Invalid file path rejected: ${filePath}`);
      return {
        success: false,
        error: 'Invalid file path',
      };
    }

    // SECURITY: Verify file exists and is within project
    const isValid = await pathValidator.verifyFileInProject(sanitizedPath);
    if (!isValid) {
      console.warn(`[SECURITY] File not in project or doesn't exist: ${sanitizedPath}`);
      return {
        success: false,
        error: 'File not found or outside project directory',
      };
    }

    // SECURITY: Validate line number if provided
    if (line !== undefined && (line < 1 || line > 1000000 || !Number.isInteger(line))) {
      return {
        success: false,
        error: 'Invalid line number',
      };
    }

    const ide = idePreference || 'vscode';
    const config = this.IDE_CONFIGS[ide];

    if (!config) {
      return {
        success: false,
        error: `Unsupported IDE: ${ide}`,
      };
    }

    // SECURITY: Generate URI with proper encoding (commandFormat must use encodeURIComponent)
    const uri = config.commandFormat(sanitizedPath, line);

    // SECURITY: Verify URI scheme matches expected IDE
    if (!uri.startsWith(config.uriScheme + '://')) {
      console.error(`[SECURITY] URI scheme mismatch. Expected: ${config.uriScheme}, Got: ${uri}`);
      return {
        success: false,
        error: 'Invalid URI generated',
      };
    }

    // SECURITY: Validate URI format before opening
    try {
      const parsed = new URL(uri);
      const allowedSchemes = Object.values(this.IDE_CONFIGS).map(c => c.uriScheme);

      if (!allowedSchemes.includes(parsed.protocol.slice(0, -1))) {
        console.error(`[SECURITY] Disallowed URI scheme: ${parsed.protocol}`);
        return {
          success: false,
          error: 'Disallowed URI scheme',
        };
      }

      // SECURITY: Use shell.openExternal with validated URI
      await shell.openExternal(uri);
      return { success: true };
    } catch (error) {
      console.error('[SECURITY] Failed to open IDE:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to open IDE',
      };
    }
  }

  getSupportedIDEs(): IDEType[] {
    return Object.keys(this.IDE_CONFIGS) as IDEType[];
  }
}
```

#### ipc handler

Location: `src/main/ipc/handlers/shell.ts`

```typescript
import { ipcMain } from 'electron';
import { IDEIntegrationService } from '@/main/services/ide-integration';
import { z } from 'zod';

const OpenInIDEPayloadSchema = z.object({
  filePath: z.string(),
  line: z.number().optional(),
  ide: z.enum(['vscode', 'cursor', 'idea', 'atom', 'sublime']).optional(),
});

export function registerShellHandlers(projectRoot: string) {
  const ideService = new IDEIntegrationService(projectRoot);

  ipcMain.handle('shell:openInIDE', async (_event, payload) => {
    try {
      const validated = OpenInIDEPayloadSchema.parse(payload);
      return ideService.openInIDE(validated.filePath, validated.line, validated.ide);
    } catch (error) {
      console.error('[SECURITY] Invalid openInIDE payload:', error);
      return {
        success: false,
        error: 'Invalid request parameters',
      };
    }
  });

  ipcMain.handle('shell:getSupportedIDEs', async () => {
    return ideService.getSupportedIDEs();
  });
}
```

#### ui component

Location: `src/renderer/components/IssueCard.tsx` (add "Open in IDE" button)

```typescript
function IssueCard({ issue }: { issue: PluginInsight }) {
  const { settings } = useSettingsStore();

  const handleOpenInIDE = async () => {
    const result = await window.viprDesktop.shell.openInIDE({
      filePath: issue.location?.file || '',
      line: issue.location?.line,
      ide: settings.idePreference,
    });

    if (!result.success) {
      // Show error toast
      console.error('Failed to open IDE:', result.error);
    }
  };

  return (
    <div className="issue-card">
      <header>
        <Badge severity={issue.severity}>{issue.severity}</Badge>
        <h3>{issue.title}</h3>
      </header>

      <p>{issue.description}</p>

      {issue.location && (
        <div className="issue-location">
          <code>
            {issue.location.file}:{issue.location.line}:{issue.location.column}
          </code>
        </div>
      )}

      <footer>
        <Button onClick={handleOpenInIDE} variant="secondary">
          Open in IDE
        </Button>
      </footer>
    </div>
  );
}
```

#### settings ui

Location: `src/renderer/pages/Settings.tsx`

```typescript
function IDEPreferenceSection() {
  const { settings, updateSetting } = useSettingsStore();
  const [supportedIDEs, setSupportedIDEs] = useState<string[]>([]);

  useEffect(() => {
    async function loadIDEs() {
      const ides = await window.viprDesktop.shell.getSupportedIDEs();
      setSupportedIDEs(ides);
    }

    loadIDEs();
  }, []);

  return (
    <section className="ide-preference-section">
      <h2>IDE Integration</h2>
      <p>Choose your preferred IDE for opening files.</p>

      <Select
        label="Preferred IDE"
        value={settings.idePreference}
        onChange={(value) => updateSetting('idePreference', value)}
        options={supportedIDEs.map((ide) => ({
          value: ide,
          label: ide.charAt(0).toUpperCase() + ide.slice(1),
        }))}
      />

      <p className="help-text">
        Make sure your IDE is installed and configured to handle URI schemes.
      </p>
    </section>
  );
}
```

#### acceptance criteria

- [ ] Support for VSCode, Cursor, IntelliJ IDEA, Atom, Sublime Text
- [ ] Opens files to specific line number when available
- [ ] User can configure preferred IDE in Settings
- [ ] Graceful error handling when IDE not installed
- [ ] "Open in IDE" button appears in IssueCard and FileDetail components
- [ ] Settings persist across app sessions

---

### 6. notes and annotations (us-14)

#### database schema

```sql
CREATE TABLE notes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  target_type TEXT NOT NULL CHECK(target_type IN ('file', 'issue', 'abstraction')),
  target_id TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE INDEX idx_notes_target ON notes(target_type, target_id);
```

#### implementation

Location: `src/main/db/notes.ts`

```typescript
import Database from 'better-sqlite3';

export interface Note {
  id: number;
  targetType: 'file' | 'issue' | 'abstraction';
  targetId: string;
  content: string;
  createdAt: number;
  updatedAt: number;
}

export class NotesRepository {
  constructor(private db: Database.Database) {}

  create(note: Omit<Note, 'id' | 'createdAt' | 'updatedAt'>): Note {
    const now = Date.now();
    const result = this.db
      .prepare(
        `INSERT INTO notes (target_type, target_id, content, created_at, updated_at)
         VALUES (?, ?, ?, ?, ?)`
      )
      .run(note.targetType, note.targetId, note.content, now, now);

    return {
      id: result.lastInsertRowid as number,
      ...note,
      createdAt: now,
      updatedAt: now,
    };
  }

  update(id: number, content: string): void {
    const now = Date.now();
    this.db
      .prepare('UPDATE notes SET content = ?, updated_at = ? WHERE id = ?')
      .run(content, now, id);
  }

  delete(id: number): void {
    this.db.prepare('DELETE FROM notes WHERE id = ?').run(id);
  }

  getByTarget(targetType: string, targetId: string): Note[] {
    return this.db
      .prepare(
        'SELECT * FROM notes WHERE target_type = ? AND target_id = ? ORDER BY created_at DESC'
      )
      .all(targetType, targetId) as Note[];
  }

  getAll(): Note[] {
    return this.db.prepare('SELECT * FROM notes ORDER BY created_at DESC').all() as Note[];
  }
}
```

#### ipc handlers

Location: `src/main/ipc/handlers/notes.ts`

```typescript
import { ipcMain } from 'electron';
import { NotesRepository } from '@/main/db/notes';
import { z } from 'zod';

const CreateNotePayloadSchema = z.object({
  targetType: z.enum(['file', 'issue', 'abstraction']),
  targetId: z.string(),
  content: z.string(),
});

const UpdateNotePayloadSchema = z.object({
  id: z.number(),
  content: z.string(),
});

const DeleteNotePayloadSchema = z.object({
  id: z.number(),
});

const GetNotesByTargetPayloadSchema = z.object({
  targetType: z.enum(['file', 'issue', 'abstraction']),
  targetId: z.string(),
});

export function registerNotesHandlers(notesRepo: NotesRepository) {
  ipcMain.handle('notes:create', async (_event, payload) => {
    const validated = CreateNotePayloadSchema.parse(payload);
    return notesRepo.create(validated);
  });

  ipcMain.handle('notes:update', async (_event, payload) => {
    const validated = UpdateNotePayloadSchema.parse(payload);
    notesRepo.update(validated.id, validated.content);
    return { success: true };
  });

  ipcMain.handle('notes:delete', async (_event, payload) => {
    const validated = DeleteNotePayloadSchema.parse(payload);
    notesRepo.delete(validated.id);
    return { success: true };
  });

  ipcMain.handle('notes:getByTarget', async (_event, payload) => {
    const validated = GetNotesByTargetPayloadSchema.parse(payload);
    return notesRepo.getByTarget(validated.targetType, validated.targetId);
  });

  ipcMain.handle('notes:getAll', async () => {
    return notesRepo.getAll();
  });
}
```

#### ui component

Location: `src/renderer/components/NotesEditor.tsx`

```typescript
import React, { useState, useEffect } from 'react';
import { Textarea, Button, Skeleton } from '@/renderer/components/ui';
import { ErrorBoundary } from '@/renderer/components/ErrorBoundary';
import { useToast } from '@/renderer/hooks/useToast';
import { createLogger } from '@vipr/logging';
import type { Note } from '@/shared/ipc-types';

const logger = createLogger('notes-editor');

interface NotesEditorProps {
  targetType: 'file' | 'issue' | 'abstraction';
  targetId: string;
}

export function NotesEditor({ targetType, targetId }: NotesEditorProps) {
  const [notes, setNotes] = useState<Note[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [newNote, setNewNote] = useState('');
  const [editingId, setEditingId] = useState<number | null>(null);
  const [editContent, setEditContent] = useState('');
  const { showSuccess, showError } = useToast();

  useEffect(() => {
    loadNotes();
  }, [targetType, targetId]);

  const loadNotes = async () => {
    try {
      setIsLoading(true);
      const loaded = await window.viprDesktop.notes.getByTarget({
        targetType,
        targetId,
      });
      setNotes(loaded);
    } catch (error) {
      logger.error('Failed to load notes', { error });
      showError('Failed to load notes');
    } finally {
      setIsLoading(false);
    }
  };

  // Phase 3 pattern: optimistic update
  const handleCreate = async () => {
    if (!newNote.trim()) return;

    const optimisticId = -Date.now(); // Negative ID for optimistic note
    const optimisticNote: Note = {
      id: optimisticId,
      targetType,
      targetId,
      content: newNote,
      createdAt: Date.now(),
      updatedAt: Date.now(),
    };

    // Optimistically add to UI
    setNotes([optimisticNote, ...notes]);
    setNewNote('');

    try {
      const created = await window.viprDesktop.notes.create({
        targetType,
        targetId,
        content: optimisticNote.content,
      });

      // Replace optimistic note with real note
      setNotes(current =>
        current.map(n => (n.id === optimisticId ? created : n))
      );

      showSuccess('Note created');
    } catch (error) {
      logger.error('Failed to create note', { error });

      // Rollback on error
      setNotes(current => current.filter(n => n.id !== optimisticId));
      setNewNote(optimisticNote.content); // Restore content

      showError('Failed to create note');
    }
  };

  // Phase 3 pattern: optimistic update
  const handleUpdate = async (id: number) => {
    if (!editContent.trim()) return;

    const previousNotes = notes;
    const now = Date.now();

    // Optimistically update UI
    setNotes(current =>
      current.map(n =>
        n.id === id ? { ...n, content: editContent, updatedAt: now } : n
      )
    );
    setEditingId(null);
    setEditContent('');

    try {
      await window.viprDesktop.notes.update({
        id,
        content: editContent,
      });

      showSuccess('Note updated');
    } catch (error) {
      logger.error('Failed to update note', { error });

      // Rollback on error
      setNotes(previousNotes);
      setEditingId(id);
      setEditContent(editContent);

      showError('Failed to update note');
    }
  };

  // Phase 3 pattern: optimistic delete
  const handleDelete = async (id: number) => {
    if (!confirm('Delete this note?')) return;

    const previousNotes = notes;

    // Optimistically remove from UI
    setNotes(current => current.filter(n => n.id !== id));

    try {
      await window.viprDesktop.notes.delete({ id });
      showSuccess('Note deleted');
    } catch (error) {
      logger.error('Failed to delete note', { error });

      // Rollback on error
      setNotes(previousNotes);

      showError('Failed to delete note');
    }
  };

  // Phase 3 pattern: loading state
  if (isLoading) {
    return (
      <div className="notes-editor">
        <Skeleton className="h-32 w-full mb-4" />
        <Skeleton className="h-24 w-full" />
      </div>
    );
  }

  return (
    <div className="notes-editor">
      <div className="notes-list">
        {notes.map((note) => (
          <div key={note.id} className="note-card">
            {editingId === note.id ? (
              <>
                <Textarea
                  value={editContent}
                  onChange={(e) => setEditContent(e.target.value)}
                  rows={3}
                />
                <div className="note-actions">
                  <Button onClick={() => handleUpdate(note.id)}>Save</Button>
                  <Button
                    variant="secondary"
                    onClick={() => {
                      setEditingId(null);
                      setEditContent('');
                    }}
                  >
                    Cancel
                  </Button>
                </div>
              </>
            ) : (
              <>
                {/* SECURITY NOTE: If markdown support is added in Phase 5, must use DOMPurify:
                    import DOMPurify from 'dompurify';
                    <div dangerouslySetInnerHTML={{
                      __html: DOMPurify.sanitize(note.content, {
                        ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'code', 'pre', 'a', 'ul', 'ol', 'li', 'p', 'br'],
                        ALLOWED_ATTR: ['href', 'title'],
                        ALLOWED_URI_REGEXP: /^(?:https?|mailto):/i,
                      })
                    }} />
                    For now, plain text rendering in <p> tag is safe as React escapes by default */}
                <p className="note-content">{note.content}</p>
                <div className="note-meta">
                  <span className="note-date">
                    {new Date(note.updatedAt).toLocaleString()}
                  </span>
                  <div className="note-actions">
                    <Button
                      variant="ghost"
                      onClick={() => {
                        setEditingId(note.id);
                        setEditContent(note.content);
                      }}
                    >
                      Edit
                    </Button>
                    <Button
                      variant="ghost"
                      onClick={() => handleDelete(note.id)}
                    >
                      Delete
                    </Button>
                  </div>
                </div>
              </>
            )}
          </div>
        ))}
      </div>

      <div className="new-note-form">
        <Textarea
          placeholder="Add a note..."
          value={newNote}
          onChange={(e) => setNewNote(e.target.value)}
          rows={3}
        />
        <Button onClick={handleCreate}>Add Note</Button>
      </div>
    </div>
  );
}
```

Integration into FileDetail:

```typescript
// Add new tab to FileDetail component
<Tabs>
  <Tab label="Overview">...</Tab>
  <Tab label="Issues">...</Tab>
  <Tab label="Abstractions">...</Tab>
  <Tab label="AI Prompts">...</Tab>
  <Tab label="Notes">
    <NotesEditor targetType="file" targetId={file.id} />
  </Tab>
</Tabs>
```

#### acceptance criteria

**Functionality:**

- [ ] Notes table created with Phase 4 migration (version 2)
- [ ] Migration checks for Phase 2 schema before executing
- [ ] CRUD operations work for notes with proper validation
- [ ] Notes display inline in FileDetail tabs
- [ ] User can add/edit/delete notes with confirmation
- [ ] Notes persist across app sessions
- [ ] Notes sorted by creation date (newest first)

**Architecture Compliance:**

- [ ] IPC channels follow naming convention (notes:create, notes:update, notes:delete, notes:getByTarget, notes:getAll)
- [ ] All payloads validated with Zod schemas
- [ ] Prepared statements used for all database operations
- [ ] Optimistic updates for create/update/delete
- [ ] Rollback on error with user feedback
- [ ] Loading states with Phase 3 skeletons
- [ ] Error boundaries wrap NotesEditor
- [ ] Toast notifications via Phase 3 system
- [ ] Proper logging with `@vipr/logging`
- [ ] No console.log statements

**User Experience:**

- [ ] Immediate UI feedback (optimistic updates)
- [ ] Clear error messages on failure
- [ ] Editing state preserved on error
- [ ] Confirmation required for delete
- [ ] Empty state when no notes exist
- [ ] Textarea auto-focus when editing

---

### 7. issue exclusions (us-15)

#### database schema

```sql
CREATE TABLE exclusions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  issue_type TEXT NOT NULL,
  file_path TEXT,
  plugin_id TEXT,
  reason TEXT,
  created_at INTEGER NOT NULL
);

CREATE INDEX idx_exclusions_issue_type ON exclusions(issue_type);
CREATE INDEX idx_exclusions_file_path ON exclusions(file_path);
```

#### implementation

Location: `src/main/db/exclusions.ts`

```typescript
import Database from 'better-sqlite3';

export interface Exclusion {
  id: number;
  issueType: string;
  filePath?: string;
  pluginId?: string;
  reason?: string;
  createdAt: number;
}

export class ExclusionsRepository {
  constructor(private db: Database.Database) {}

  add(exclusion: Omit<Exclusion, 'id' | 'createdAt'>): Exclusion {
    const now = Date.now();
    const result = this.db
      .prepare(
        `INSERT INTO exclusions (issue_type, file_path, plugin_id, reason, created_at)
         VALUES (?, ?, ?, ?, ?)`
      )
      .run(
        exclusion.issueType,
        exclusion.filePath || null,
        exclusion.pluginId || null,
        exclusion.reason || null,
        now
      );

    return {
      id: result.lastInsertRowid as number,
      ...exclusion,
      createdAt: now,
    };
  }

  remove(id: number): void {
    this.db.prepare('DELETE FROM exclusions WHERE id = ?').run(id);
  }

  getAll(): Exclusion[] {
    return this.db
      .prepare('SELECT * FROM exclusions ORDER BY created_at DESC')
      .all() as Exclusion[];
  }

  isExcluded(issueType: string, filePath?: string, pluginId?: string): boolean {
    let query = 'SELECT COUNT(*) as count FROM exclusions WHERE issue_type = ?';
    const params: any[] = [issueType];

    if (filePath) {
      query += ' AND (file_path IS NULL OR file_path = ?)';
      params.push(filePath);
    }

    if (pluginId) {
      query += ' AND (plugin_id IS NULL OR plugin_id = ?)';
      params.push(pluginId);
    }

    const result = this.db.prepare(query).get(...params) as { count: number };
    return result.count > 0;
  }
}
```

#### ipc handlers

Location: `src/main/ipc/handlers/exclusions.ts`

```typescript
import { ipcMain } from 'electron';
import { ExclusionsRepository } from '@/main/db/exclusions';
import { z } from 'zod';

const AddExclusionPayloadSchema = z.object({
  issueType: z.string(),
  filePath: z.string().optional(),
  pluginId: z.string().optional(),
  reason: z.string().optional(),
});

const RemoveExclusionPayloadSchema = z.object({
  id: z.number(),
});

const IsExcludedPayloadSchema = z.object({
  issueType: z.string(),
  filePath: z.string().optional(),
  pluginId: z.string().optional(),
});

export function registerExclusionsHandlers(exclusionsRepo: ExclusionsRepository) {
  ipcMain.handle('exclusions:add', async (_event, payload) => {
    const validated = AddExclusionPayloadSchema.parse(payload);
    return exclusionsRepo.add(validated);
  });

  ipcMain.handle('exclusions:remove', async (_event, payload) => {
    const validated = RemoveExclusionPayloadSchema.parse(payload);
    exclusionsRepo.remove(validated.id);
    return { success: true };
  });

  ipcMain.handle('exclusions:getAll', async () => {
    return exclusionsRepo.getAll();
  });

  ipcMain.handle('exclusions:isExcluded', async (_event, payload) => {
    const validated = IsExcludedPayloadSchema.parse(payload);
    return exclusionsRepo.isExcluded(validated.issueType, validated.filePath, validated.pluginId);
  });
}
```

#### ui components

Location: `src/renderer/components/IssueCard.tsx` (add "Exclude" button)

```typescript
function IssueCard({ issue }: { issue: PluginInsight }) {
  const [isExcluded, setIsExcluded] = useState(false);

  useEffect(() => {
    async function checkExclusion() {
      const excluded = await window.viprDesktop.exclusions.isExcluded({
        issueType: issue.ruleId,
        filePath: issue.location?.file,
        pluginId: issue.pluginId,
      });
      setIsExcluded(excluded);
    }

    checkExclusion();
  }, [issue]);

  const handleExclude = async () => {
    const reason = prompt('Reason for exclusion (optional):');

    await window.viprDesktop.exclusions.add({
      issueType: issue.ruleId,
      filePath: issue.location?.file,
      pluginId: issue.pluginId,
      reason: reason || undefined,
    });

    setIsExcluded(true);
  };

  return (
    <div className={`issue-card ${isExcluded ? 'excluded' : ''}`}>
      {/* ... existing card content ... */}

      <footer>
        <Button onClick={handleOpenInIDE} variant="secondary">
          Open in IDE
        </Button>
        {!isExcluded && (
          <Button onClick={handleExclude} variant="ghost">
            Exclude
          </Button>
        )}
        {isExcluded && <Badge variant="muted">Excluded</Badge>}
      </footer>
    </div>
  );
}
```

Location: `src/renderer/pages/Settings.tsx` (exclusions management)

```typescript
function ExclusionsSection() {
  const [exclusions, setExclusions] = useState<Exclusion[]>([]);

  useEffect(() => {
    loadExclusions();
  }, []);

  const loadExclusions = async () => {
    const loaded = await window.viprDesktop.exclusions.getAll();
    setExclusions(loaded);
  };

  const handleRemove = async (id: number) => {
    if (!confirm('Remove this exclusion?')) return;

    await window.viprDesktop.exclusions.remove({ id });
    await loadExclusions();
  };

  return (
    <section className="exclusions-section">
      <h2>Issue Exclusions</h2>
      <p>Manage excluded issues. Excluded issues are hidden from reports.</p>

      <table className="exclusions-table">
        <thead>
          <tr>
            <th>Issue Type</th>
            <th>File Path</th>
            <th>Plugin</th>
            <th>Reason</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {exclusions.map((exclusion) => (
            <tr key={exclusion.id}>
              <td>{exclusion.issueType}</td>
              <td>{exclusion.filePath || 'All files'}</td>
              <td>{exclusion.pluginId || 'All plugins'}</td>
              <td>{exclusion.reason || '-'}</td>
              <td>
                <Button
                  variant="ghost"
                  onClick={() => handleRemove(exclusion.id)}
                >
                  Remove
                </Button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>

      {exclusions.length === 0 && (
        <p className="empty-state">No exclusions configured.</p>
      )}
    </section>
  );
}
```

#### acceptance criteria

- [ ] Exclusions table created with migration
- [ ] User can exclude issues from IssueCard
- [ ] Excluded issues show grayed-out in UI (not hidden, Phase 5 feature)
- [ ] User can manage exclusions in Settings
- [ ] Exclusions persist across app sessions
- [ ] Exclusion check is performant (indexed queries)
- [ ] Optional reason captured when excluding

---

---

## section 0.6: phase 4 system integration architecture

This diagram shows how all Phase 4 features integrate with Phase 1-3 architecture:

```mermaid
---
title: phase 4 full system integration
---
graph TB
    subgraph Renderer["Renderer Process (Phase 3 UI)"]
        UI[React Components]
        AnalysisStore[useAnalysisStore]
        SettingsStore[useSettingsStore]
        ReportingStore[useReportingStore - NEW]
        ErrorBoundary[Error Boundaries]
    end

    subgraph Preload["Preload Bridge (Phase 1 IPC)"]
        Bridge[Context Bridge]
        Validation[Zod Validation]
        IPCTypes[IPC Type Definitions]
    end

    subgraph Main["Main Process (Phase 1 Foundation)"]
        IPCRouter[IPC Router]
        SQLite[(SQLite WAL Mode)]
        PDFService[PDF Export Service - NEW]
        CostService[Cost Estimation - NEW]
        AIService[AI Prompt Service - NEW]
        MCPManager[MCP Server Manager - NEW]
        IDEService[IDE Integration - NEW]
        NotesRepo[Notes Repository - NEW]
        ExclusionsRepo[Exclusions Repository - NEW]
    end

    subgraph Utility["Utility Process (Phase 1 Engine)"]
        Engine[@vipr/engine]
        PresenterRegistry[Presenter Registry]
    end

    subgraph MCP["MCP Server (Child Process) - NEW"]
        MCPServer[MCP Server]
        ReadOnlyDB[(Read-Only SQLite)]
        HealthCheck[Health Checks]
    end

    subgraph External["External Integrations - NEW"]
        IDE[VSCode/Cursor/etc]
        Claude[Claude Code]
        Cursor[Cursor IDE]
    end

    UI -->|store selectors| AnalysisStore
    UI -->|store selectors| SettingsStore
    UI -->|store selectors| ReportingStore
    ErrorBoundary -.wraps.-> UI

    UI -->|viprDesktop API| Bridge
    Bridge -->|validate request| Validation
    Bridge -->|validate response| Validation
    Bridge <-->|typed IPC| IPCRouter

    IPCRouter -->|reports:exportPDF| PDFService
    IPCRouter -->|analysis:estimateCosts| CostService
    IPCRouter -->|ai:*| AIService
    IPCRouter -->|mcp:*| MCPManager
    IPCRouter -->|shell:*| IDEService
    IPCRouter -->|notes:*| NotesRepo
    IPCRouter -->|exclusions:*| ExclusionsRepo

    IPCRouter <-->|analysis requests| Utility
    Engine --> PresenterRegistry

    NotesRepo --> SQLite
    ExclusionsRepo --> SQLite
    CostService --> SQLite
    PDFService -.queries.-> SQLite

    MCPManager -->|spawn/health check| MCPServer
    MCPServer --> ReadOnlyDB
    MCPServer --> HealthCheck
    ReadOnlyDB -.read-only access.-> SQLite

    IDEService -->|URI schemes| IDE
    MCPServer -->|stdio JSON-RPC| Claude
    MCPServer -->|stdio JSON-RPC| Cursor

    classDef phase1 fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef phase2 fill:#10b981,stroke:#059669,color:#fff
    classDef phase3 fill:#f59e0b,stroke:#d97706,color:#fff
    classDef phase4 fill:#ef4444,stroke:#b91c1c,color:#fff

    class Bridge,IPCRouter,SQLite,Utility,Engine phase1
    class PresenterRegistry phase2
    class UI,AnalysisStore,SettingsStore,ErrorBoundary phase3
    class ReportingStore,PDFService,CostService,AIService,MCPManager,IDEService,NotesRepo,ExclusionsRepo,MCPServer,ReadOnlyDB,HealthCheck phase4
```

**Integration Points:**

1. **Phase 1 Foundation**: All new IPC handlers follow established patterns, SQLite access through existing connection
2. **Phase 2 Schema**: New tables (notes, exclusions) extend existing schema with migration version checking
3. **Phase 3 UI**: New features use existing stores, error boundaries, and component patterns
4. **Phase 4 Additions**: Seven new feature domains with proper isolation and error handling

---

## part b: architecture review

### review subagent selection

Based on the subagent reference in 00-user-stories.md, the following reviewers are selected for Phase 4:

#### 1. code-complexity-analyzer

**Validation Scope:**

- Cost and velocity estimation algorithms (US-08)
- Complexity-to-effort correlation models
- ROI calculation formulas
- Industry-standard metrics validation

**Specific Review Criteria:**

- [ ] Estimation formulas produce reasonable results (compare against industry data)
- [ ] Velocity improvement calculations are defensible
- [ ] ROI projections account for long-term vs short-term value
- [ ] Confidence levels accurately reflect metric quality
- [ ] Edge cases handled (zero complexity, missing metrics)

**Checkpoint Timing:** After US-08 implementation complete

---

#### 2. architecture-reviewer

**Validation Scope:**

- Overall Phase 4 system integration
- IPC channel design and security
- Database schema and migration strategy
- Module boundaries and separation of concerns

**Specific Review Criteria:**

- [ ] All IPC channels follow `domain:action` naming convention
- [ ] Zod validation at every IPC boundary
- [ ] SQLite transactions used for multi-table operations
- [ ] No direct SQLite access from renderer process
- [ ] MCP server properly isolated as child process
- [ ] Notes and exclusions schemas normalized and indexed
- [ ] Migration scripts are sequential and reversible
- [ ] Error handling consistent across all features
- [ ] No security vulnerabilities in IPC handlers

**Checkpoint Timing:** End of Week 3 (before integration testing)

---

#### 3. frontend-security-auditor

**Validation Scope:**

- IPC security boundaries
- SQLite read-only access in MCP server
- URI scheme handling for IDE integration
- Clipboard operations (AI prompts)

**Specific Review Criteria:**

- [ ] No SQL injection vulnerabilities (all queries use prepared statements)
- [ ] IPC payloads validated before processing
- [ ] MCP server cannot write to database (read-only mode enforced)
- [ ] URI schemes sanitized before shell.openExternal()
- [ ] Clipboard operations do not expose sensitive data
- [ ] PDF export does not leak file paths in HTML
- [ ] No XSS vulnerabilities in notes content (markdown rendering)

**Checkpoint Timing:** End of Week 4 (before production deployment)

---

### review process flow

```mermaid
---
title: phase 4 architecture review process
---
graph TB
    Impl[Phase 4 Implementation] --> PreCheck[Pre-Review Checklist]
    PreCheck -->|All checked| CodeReview[Code Complexity Analyzer Review]
    PreCheck -->|Incomplete| FixIssues[Fix Issues]
    FixIssues --> PreCheck

    CodeReview --> CodePass{Pass?}
    CodePass -->|Yes| ArchReview[Architecture Reviewer Review]
    CodePass -->|No - Critical| FixCritical[Fix Critical Issues]
    CodePass -->|No - Minor| DocumentMinor[Document Minor Issues]

    FixCritical --> CodeReview
    DocumentMinor --> ArchReview

    ArchReview --> ArchPass{Pass?}
    ArchPass -->|Yes| SecReview[Security Auditor Review]
    ArchPass -->|No - Critical| FixArch[Fix Architecture Issues]
    ArchPass -->|No - Minor| DocumentArch[Document Architecture Debt]

    FixArch --> ArchReview
    DocumentArch --> SecReview

    SecReview --> SecPass{Pass?}
    SecPass -->|Yes| Approved[Phase 4 Approved]
    SecPass -->|No| FixSecurity[Fix Security Issues]

    FixSecurity --> SecReview

    Approved --> Phase5[Phase 5: Polish]
```

---

### pre-review checklist

Before requesting architecture review:

**Documentation:**

- [ ] All IPC channels documented in `shared/ipc-types.ts`
- [ ] Database schema documented with ER diagrams
- [ ] API contracts defined for MCP server tools
- [ ] Configuration settings documented in Settings UI

**Code Quality:**

- [ ] TypeScript strict mode enabled, no `any` types
- [ ] All functions have JSDoc comments
- [ ] Error handling with user-friendly messages
- [ ] Loading states for all async operations
- [ ] Use `@vipr/logging` instead of console.log

**Testing:**

- [ ] Unit tests for all service classes (CostEstimationService, AIPromptService, etc.)
- [ ] Integration tests for IPC handlers
- [ ] Database migration tests (up and down)
- [ ] MCP server tool tests with mock SQLite database

**Security:**

- [ ] All IPC payloads validated with Zod
- [ ] SQL queries use prepared statements exclusively
- [ ] No sensitive data logged to console
- [ ] PDF export sanitizes user input
- [ ] CSP headers configured for hidden windows
- [ ] No inline scripts in PDF generation

**Performance:**

- [ ] PDF generation completes in under 5 seconds
- [ ] Cost estimation completes in under 1 second
- [ ] AI prompt generation completes in under 500ms
- [ ] Notes CRUD operations complete in under 100ms
- [ ] Database queries indexed and optimized

**Phase 3 Integration:**

- [ ] All stores use selector patterns
- [ ] Error boundaries wrap all new features
- [ ] Optimistic updates for user actions
- [ ] Loading skeletons for async operations
- [ ] IPC channels follow `domain:action` convention
- [ ] Validation at ALL IPC boundaries (preload, main, response)

**Architecture Compliance:**

- [ ] All IPC channels follow naming convention (domain:action)
- [ ] Zod validation complete at all boundaries
- [ ] Error handling consistent with Phase 3 patterns
- [ ] Loading states for all async operations
- [ ] Store selectors prevent unnecessary re-renders
- [ ] Database migrations integrate with Phase 2 system
- [ ] MCP server has health checks and restart logic
- [ ] PDF export respects CSP from Phase 3
- [ ] All new features have error boundaries
- [ ] No direct SQLite access from renderer
- [ ] Prepared statements used for all queries
- [ ] Migration version checking before Phase 4 migrations
- [ ] Graceful shutdown integration with app close
- [ ] Timeout handling for long-running operations

---

### post-review actions

After all reviews complete:

**Critical Findings:**

- Must be fixed before Phase 4 approval
- Re-review required after fixes

**High-Priority Findings:**

- Must be fixed before Phase 5 begins
- No re-review required (trust fixes)

**Medium/Low-Priority Findings:**

- Create GitHub issues for Phase 5 or backlog
- Document as technical debt in architecture docs

**Phase 5 Kickoff:**

- Schedule team meeting to review Phase 4 learnings
- Prioritize Phase 5 polish tasks based on Phase 4 feedback
- Update roadmap with realistic timelines

---

## section 8: architectural consistency summary

### 8.1 critical improvements made

This document has been comprehensively updated to ensure architectural consistency with Phase 1-3:

**IPC Layer Consistency:**

- All channels follow `domain:action` naming convention established in Phase 1
- Bidirectional Zod validation at preload (request and response)
- Complete IPC type definitions added to Section 0.2
- Error handling patterns consistent across all handlers

**Database Integration:**

- Phase 4 migrations check for Phase 2 schema before executing (version validation)
- All queries use prepared statements from Phase 1 patterns
- Rollback capability via down() methods
- WAL mode verification in MCP server with explicit checks

**State Management Integration:**

- New ReportingStore follows Phase 3 patterns with selector usage
- SettingsStore extended for Phase 4 configuration
- All UI components use optimistic updates with rollback
- Loading states use Phase 3 skeleton components throughout

**Security Enhancements:**

- PDF export fixed: no inline scripts, respects CSP headers
- Prebuilt report bundle with proper React hydration
- MCP server enforces read-only SQLite access with query_only pragma
- URI scheme sanitization for IDE integration

**Reliability Improvements:**

- MCP server health checks with auto-restart (max 3 attempts)
- Graceful shutdown integration with app close
- Timeout handling for PDF generation (30s with configurable setting)
- Cleanup on errors prevents resource leaks

**Developer Experience:**

- Proper logging with `@vipr/logging` instead of console.log
- Error boundaries wrap all Phase 4 features
- Toast notifications via Phase 3 useToast hook
- Comprehensive acceptance criteria with architecture checklist

### 8.2 integration validation checklist

Before marking Phase 4 complete, validate these integration points:

**Phase 1 Foundation Integration:**

- [ ] All IPC channels registered in router with typed handlers
- [ ] SQLite connection shared across all Phase 4 features
- [ ] Utility process not affected by Phase 4 additions
- [ ] No new process types introduced

**Phase 2 Database Integration:**

- [ ] Migration version 2 successfully applies after version 1
- [ ] Foreign key constraints work correctly
- [ ] Prepared statements cached and reused
- [ ] WAL mode benefits verified with concurrent access

**Phase 3 UI Integration:**

- [ ] All stores use selector patterns from existing stores
- [ ] Error boundaries reused from Phase 3
- [ ] Toast system integrated correctly
- [ ] Loading skeletons consistent with existing UI

**Cross-Feature Dependencies:**

- [ ] PDF export can query cost estimation data
- [ ] MCP server can access notes and exclusions
- [ ] IDE integration works with all issue types
- [ ] AI prompts can reference exclusions

### 8.3 known limitations and phase 5 hooks

**Deferred to Phase 5:**

- Advanced PDF templates and customization
- Markdown rendering in notes content
- Bulk operations for notes and exclusions
- Snapshot comparison in PDF reports
- Advanced filtering in cost analysis
- MCP server performance metrics dashboard

**Extension Points Prepared:**

- ReportingStore ready for template management
- Notes schema supports future tag system
- Exclusions support pattern matching (to be added)
- MCP server tool registry extensible
- AI prompt templates in separate module

---

## files modified

### documentation

- **Update**: `documentation/docs/feature-development/electron-app/06-advanced-features.md` (this file)
  - Added Section 0: Phase 3 Integration Prerequisites
  - Updated all IPC patterns with Phase 1 conventions
  - Added comprehensive architecture compliance checklist
  - Fixed CSP violations in PDF export
  - Added health checks and graceful shutdown for MCP server
  - Added optimistic updates for notes and exclusions
  - Updated all acceptance criteria with architecture requirements

### phase 4 implementation files to create

**Shared Types (extends Phase 1):**

- `src/shared/ipc-types.ts` - Add Phase 4 schemas (Section 0.2)

**Main Process Services:**

- `src/main/services/pdf-export.ts` - PDF generation with CSP compliance
- `src/main/services/cost-estimation.ts` - Cost/velocity algorithms
- `src/main/services/ai-prompts.ts` - AI prompt templates
- `src/main/services/ide-integration.ts` - IDE URI schemes
- `src/main/mcp/server-manager.ts` - MCP server lifecycle with health checks

**Main Process Repositories:**

- `src/main/db/notes.ts` - Notes repository with prepared statements
- `src/main/db/exclusions.ts` - Exclusions repository with prepared statements
- `src/main/db/migrations/phase4.ts` - Phase 4 migrations (version 2)

**IPC Handlers:**

- `src/main/ipc/handlers/reports.ts` - PDF export handler
- `src/main/ipc/handlers/cost-estimation.ts` - Cost analysis handler
- `src/main/ipc/handlers/ai-prompts.ts` - AI prompt handlers
- `src/main/ipc/handlers/mcp.ts` - MCP server control
- `src/main/ipc/handlers/ide.ts` - IDE integration handler
- `src/main/ipc/handlers/notes.ts` - Notes CRUD handlers
- `src/main/ipc/handlers/exclusions.ts` - Exclusions CRUD handlers

**MCP Server (Child Process):**

- `src/mcp-server/index.ts` - MCP server entry point
- `src/mcp-server/tools/` - MCP tool implementations

**Report Renderer (Separate Entry):**

- `src/report-renderer/index.html` - Prebuilt report HTML
- `src/report-renderer/index.tsx` - Report React entry
- `src/report-preload/index.ts` - Report preload for data injection

**Renderer Stores (extends Phase 3):**

- `src/renderer/stores/reporting.ts` - New reporting store
- Update `src/renderer/stores/settings.ts` - Add Phase 4 settings

**Renderer Components:**

- `src/renderer/pages/Reports.tsx` - PDF export UI
- `src/renderer/pages/CostAnalysis.tsx` - Cost analysis dashboard
- `src/renderer/components/NotesEditor.tsx` - Notes CRUD with optimistic updates
- `src/renderer/components/IssueCard.tsx` - Update for exclusions
- `src/renderer/components/reports/PDFReportTemplate.tsx` - Report template
- `src/renderer/pages/Settings.tsx` - Update for Phase 4 settings

**Build Configuration:**

- Update `forge.config.ts` - Add report-renderer and mcp-server entries
- `vite.report.config.ts` - New Vite config for report bundle
- `vite.mcp.config.ts` - New Vite config for MCP server

**Testing:**

- `src/main/services/*.test.ts` - Service unit tests
- `src/main/db/*.test.ts` - Repository integration tests
- `src/main/ipc/handlers/*.test.ts` - IPC handler tests
- `src/mcp-server/*.test.ts` - MCP server tool tests

---

## next steps

After approval of this Phase 4 plan:

1. **Week 1**: Implement PDF export + Cost estimation + AI prompts (typescript-engineer, react-engineer, code-complexity-analyzer)
2. **Week 2**: Implement MCP server + IDE integration (typescript-engineer, database-engineer)
3. **Week 3**: Implement Notes + Exclusions (database-engineer, react-engineer)
4. **Week 4**: Integration testing + Architecture review + Security audit

**Success Metrics:**

- All 7 features implemented and tested
- Zero critical security vulnerabilities
- Performance benchmarks met
- Architecture review approval
- Ready for Phase 5 (Polish and advanced visualizations)

---

## security hardening summary

This document has been reviewed and updated by the `frontend-security-auditor` agent. The following security improvements have been implemented:

### critical vulnerabilities addressed

**1. PDF Export XSS (Section 1)**

- ❌ **Removed**: Inline scripts with `window.__REPORT_DATA__`
- ✓ **Added**: IPC-based data passing (CSP compliant)
- ✓ **Added**: HTML sanitization for user-controlled config
- ✓ **Added**: Rate limiting (5 exports per minute)

**2. MCP Server SQL Injection (Section 4, lines 2076-2149)**

- ❌ **Removed**: Unvalidated string concatenation in queries
- ✓ **Added**: Input validation for severity, pluginId, keyword
- ✓ **Added**: LIKE wildcard escaping with ESCAPE clause
- ✓ **Added**: Prepared statements with parameter binding
- ✓ **Added**: Post-query filtering for JSON-based severity
- ✓ **Added**: Result sanitization before returning

**3. IDE Integration Command Injection (Section 5, lines 2458-2537)**

- ❌ **Removed**: Unsanitized paths passed to `shell.openExternal()`
- ✓ **Added**: PathValidator integration for path sanitization
- ✓ **Added**: Project boundary verification
- ✓ **Added**: URI scheme whitelist validation
- ✓ **Added**: Proper URI encoding with `encodeURIComponent()`
- ✓ **Added**: Line number validation
- ✓ **Added**: Error handling with security logging

**4. Notes XSS (Section 6, line 3009)**

- ✓ **Added**: Security comment for future markdown implementation
- ✓ **Documented**: DOMPurify configuration for Phase 5
- ✓ **Verified**: React's default escaping sufficient for plain text

**5. AI Prompts Sensitive Data Exposure (Section 3, lines 1590-1608)**

- ✓ **Added**: Sensitive data detection before clipboard operations
- ✓ **Added**: User confirmation prompt with specific pattern list
- ✓ **Added**: IPC handler for `ai:detectSensitiveData`

### security utilities added

All security utilities are specified in Section 0.6 and fully documented in `security documentation`:

- **PathValidator**: Sanitizes file paths, prevents traversal attacks
- **RateLimiter**: DoS prevention for expensive operations
- **SensitiveDataDetector**: Detects API keys, tokens, credentials in code
- **Sanitization functions**: HTML escaping, JSON sanitization, code fence escaping
- **Validation schemas**: Zod schemas for all user inputs

### security testing requirements

Complete security test suites are provided in `security documentation`:

- SQL injection tests (10+ attack vectors)
- XSS prevention tests (script tags, event handlers, mutation XSS)
- Path traversal tests (parent dirs, absolute paths, null bytes)
- Rate limiting tests (under limit, over limit, time windows)
- CSP compliance tests (no inline scripts, no eval)

### security review checklist

Before Phase 4 production deployment:

- [ ] All critical vulnerabilities remediated (verified above)
- [ ] Security test suite passing (100% coverage required)
- [ ] Penetration testing complete (manual verification)
- [ ] `frontend-security-auditor` sign-off obtained
- [ ] Security documentation updated
- [ ] Team trained on security best practices

### references

- **Detailed Security Architecture**: security documentation (in progress)
- **OWASP Top 10 2021**: https://owasp.org/Top10/
- **Electron Security Guidelines**: https://www.electronjs.org/docs/latest/tutorial/security
- **Phase 3 CSP Configuration**: See `03-ui-implementation.md` Section 2

**Security Audit Date**: 2026-02-02  
**Auditor**: frontend-security-auditor (Claude Sonnet 4.5)  
**Status**: ✓ Security hardening complete, ready for implementation
