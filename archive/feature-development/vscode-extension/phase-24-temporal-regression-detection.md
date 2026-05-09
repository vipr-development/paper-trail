This is a great scope to work with. Looking at your CLI output, you have a rich set of metrics that naturally lend themselves to historical tracking. Let me draft a focused, extensible strategy that works with your current architecture.

---

## Vipr Historical Analysis Feature: "When Did This Get Worse?"

### Design Principles

1. **Start with aggregate scores, expand to findings** - Track category scores first (Security: 0%, Performance: 43%), then individual findings
2. **Lazy computation** - Don't analyze entire git history upfront; compute on-demand and cache
3. **Schema-forward design** - Build the SQLite schema to accommodate future metrics without migrations
4. **Attribution over archaeology** - Focus on "who/when introduced this" rather than "show me the entire history"

---

### Data Model (SQLite WASM)

```sql
-- Core tables for historical tracking

-- Stores analysis snapshots tied to git commits
CREATE TABLE analysis_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL,
  commit_hash TEXT NOT NULL,
  commit_date INTEGER NOT NULL,  -- Unix timestamp
  commit_author TEXT,
  commit_message TEXT,
  analyzed_at INTEGER NOT NULL,  -- When we ran the analysis

  UNIQUE(file_path, commit_hash)
);

-- Flexible metric storage - accommodates evolving metrics
CREATE TABLE metric_scores (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  snapshot_id INTEGER NOT NULL REFERENCES analysis_snapshots(id) ON DELETE CASCADE,

  -- Hierarchical metric identification
  category TEXT NOT NULL,        -- 'security', 'performance', 'accessibility', etc.
  metric_name TEXT NOT NULL,     -- 'overall', 'xss_count', 'memoization_effectiveness', etc.

  -- Values (use whichever is appropriate)
  score_value REAL,              -- For percentage/numeric scores (0-100)
  count_value INTEGER,           -- For counts (vulnerabilities: 42)

  UNIQUE(snapshot_id, category, metric_name)
);

-- Individual findings with attribution
CREATE TABLE findings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  snapshot_id INTEGER NOT NULL REFERENCES analysis_snapshots(id) ON DELETE CASCADE,

  category TEXT NOT NULL,
  finding_type TEXT NOT NULL,    -- 'eval_usage', 'missing_useCallback', 'hardcoded_secret'
  severity TEXT NOT NULL,        -- 'critical', 'high', 'medium', 'low'

  line_number INTEGER NOT NULL,
  column_number INTEGER,
  message TEXT,

  -- Attribution (populated via git blame)
  introduced_commit TEXT,        -- Commit that introduced this line
  introduced_date INTEGER,
  introduced_author TEXT
);

-- Cache for expensive git operations
CREATE TABLE git_blame_cache (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL,
  line_number INTEGER NOT NULL,
  as_of_commit TEXT NOT NULL,    -- The commit we ran blame from

  blame_commit TEXT NOT NULL,
  blame_author TEXT,
  blame_date INTEGER,

  cached_at INTEGER NOT NULL,
  UNIQUE(file_path, line_number, as_of_commit)
);

-- Indexes for common queries
CREATE INDEX idx_snapshots_file_date ON analysis_snapshots(file_path, commit_date DESC);
CREATE INDEX idx_metrics_category ON metric_scores(category, metric_name);
CREATE INDEX idx_findings_severity ON findings(severity, category);
CREATE INDEX idx_findings_introduced ON findings(introduced_commit);
```

### TypeScript Interfaces

```typescript
// types/history.ts

export interface AnalysisSnapshot {
  id: number;
  filePath: string;
  commitHash: string;
  commitDate: Date;
  commitAuthor?: string;
  commitMessage?: string;
  analyzedAt: Date;
}

export interface MetricScore {
  category: MetricCategory;
  metricName: string;
  scoreValue?: number;
  countValue?: number;
}

export type MetricCategory =
  | 'security'
  | 'accessibility'
  | 'performance'
  | 'reliability'
  | 'migration'
  | 'dataflow'
  | 'anti-patterns'
  | 'complexity'; // For cyclomatic, halstead, etc.

export interface Finding {
  id: number;
  category: MetricCategory;
  findingType: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  line: number;
  column?: number;
  message?: string;

  // Attribution
  introducedCommit?: string;
  introducedDate?: Date;
  introducedAuthor?: string;
}

export interface MetricDelta {
  category: MetricCategory;
  metricName: string;
  previousValue: number;
  currentValue: number;
  delta: number; // currentValue - previousValue
  deltaPercent: number; // ((current - previous) / previous) * 100
  direction: 'improved' | 'degraded' | 'unchanged';
}

export interface RegressionReport {
  filePath: string;
  currentCommit: string;
  comparisonCommit: string;

  overallDelta: number; // Change in overall score

  // Category-level changes
  categoryDeltas: MetricDelta[];

  // New findings introduced since comparison commit
  newFindings: Finding[];

  // Findings that were fixed
  resolvedFindings: Finding[];

  // The commit most responsible for degradation (if any)
  primaryRegression?: {
    commit: string;
    author: string;
    date: Date;
    message: string;
    impactScore: number; // How much this commit degraded things
  };
}
```

---

### Analysis Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ANALYSIS PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │ File Change  │───▶│ Run Current  │───▶│ Store Snapshot in SQLite │  │
│  │   Detected   │    │   Analysis   │    │  (metrics + findings)    │  │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘  │
│                                                    │                    │
│                                                    ▼                    │
│                              ┌─────────────────────────────────────┐   │
│                              │  Compare to Last Known Good State   │   │
│                              │  (previous commit or baseline)      │   │
│                              └─────────────────────────────────────┘   │
│                                                    │                    │
│                           ┌────────────────────────┼────────────────┐  │
│                           ▼                        ▼                ▼  │
│                    ┌────────────┐          ┌────────────┐    ┌────────┐│
│                    │ No Change  │          │ Improved   │    │Degraded││
│                    │  (skip)    │          │ (log it)   │    │        ││
│                    └────────────┘          └────────────┘    └────┬───┘│
│                                                                   │    │
│                                                                   ▼    │
│                                            ┌─────────────────────────┐ │
│                                            │   ATTRIBUTION PHASE     │ │
│                                            │   (on-demand, cached)   │ │
│                                            └─────────────────────────┘ │
│                                                         │              │
│                    ┌────────────────────────────────────┼──────────┐   │
│                    ▼                                    ▼          ▼   │
│           ┌──────────────┐                    ┌──────────────┐ ┌─────┐ │
│           │  Git Blame   │                    │  Binary      │ │Cache│ │
│           │  (findings)  │                    │  Search      │ │ Hit │ │
│           └──────────────┘                    │  (scores)    │ └─────┘ │
│                    │                          └──────────────┘         │
│                    └────────────────────┬─────────────┘                │
│                                         ▼                              │
│                              ┌─────────────────────────┐               │
│                              │   Update CodeLens &     │               │
│                              │   Notify User           │               │
│                              └─────────────────────────┘               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Git Integration Service

```typescript
// services/git-history.service.ts

import * as vscode from 'vscode';

export class GitHistoryService {
  /**
   * Get blame info for specific lines (batched for efficiency)
   */
  async getBlameForLines(filePath: string, lines: number[]): Promise<Map<number, BlameInfo>> {
    // Check cache first
    const cached = await this.db.getCachedBlame(filePath, lines);
    const uncachedLines = lines.filter(l => !cached.has(l));

    if (uncachedLines.length === 0) return cached;

    // Batch blame call - use line ranges to minimize git calls
    const ranges = this.consolidateToRanges(uncachedLines);
    const results = new Map(cached);

    for (const [start, end] of ranges) {
      const output = await this.exec(`git blame -L ${start},${end} --porcelain "${filePath}"`);
      const parsed = this.parseBlameOutput(output);
      parsed.forEach((info, line) => results.set(line, info));
    }

    // Cache results
    await this.db.cacheBlame(filePath, results);

    return results;
  }

  /**
   * Binary search to find when a metric crossed a threshold
   * Returns the commit that caused the regression
   */
  async findRegressionCommit(
    filePath: string,
    category: MetricCategory,
    metricName: string,
    currentValue: number,
    threshold: number, // Value we consider "good"
    maxDepth: number = 10
  ): Promise<RegressionCommit | null> {
    // Get commit history for file (limited)
    const commits = await this.getFileCommits(filePath, 50);

    if (commits.length < 2) return null;

    let left = 0;
    let right = commits.length - 1;
    let result: RegressionCommit | null = null;

    // Binary search for the first "bad" commit
    while (left < right && maxDepth-- > 0) {
      const mid = Math.floor((left + right) / 2);
      const commit = commits[mid];

      // Check if we have cached analysis for this commit
      let snapshot = await this.db.getSnapshot(filePath, commit.hash);

      if (!snapshot) {
        // Need to analyze this historical version
        const fileContent = await this.getFileAtCommit(filePath, commit.hash);
        if (!fileContent) {
          // File didn't exist at this commit, skip
          left = mid + 1;
          continue;
        }
        snapshot = await this.analyzeAndStore(filePath, commit, fileContent);
      }

      const metricValue = await this.db.getMetricValue(snapshot.id, category, metricName);

      if (metricValue !== null && metricValue > threshold) {
        // This commit is "bad", regression is earlier or at this commit
        result = {
          commit: commit.hash,
          author: commit.author,
          date: commit.date,
          message: commit.message,
          metricValue,
        };
        right = mid;
      } else {
        // This commit is "good", regression is later
        left = mid + 1;
      }
    }

    return result;
  }

  /**
   * Get sampled history for trend visualization
   * Uses intelligent sampling to reduce git operations
   */
  async getMetricTrend(
    filePath: string,
    category: MetricCategory,
    metricName: string,
    sampleCount: number = 10
  ): Promise<TrendPoint[]> {
    const commits = await this.getFileCommits(filePath, 100);

    if (commits.length === 0) return [];

    // Sample commits evenly across history
    const sampleIndices = this.getSampleIndices(commits.length, sampleCount);
    const sampledCommits = sampleIndices.map(i => commits[i]);

    const trend: TrendPoint[] = [];

    for (const commit of sampledCommits) {
      let snapshot = await this.db.getSnapshot(filePath, commit.hash);

      if (!snapshot) {
        // Analyze historical version (expensive, but cached)
        const content = await this.getFileAtCommit(filePath, commit.hash);
        if (!content) continue;
        snapshot = await this.analyzeAndStore(filePath, commit, content);
      }

      const value = await this.db.getMetricValue(snapshot.id, category, metricName);
      if (value !== null) {
        trend.push({
          commit: commit.hash,
          date: commit.date,
          author: commit.author,
          value,
        });
      }
    }

    return trend.sort((a, b) => a.date.getTime() - b.date.getTime());
  }

  private async exec(command: string): Promise<string> {
    const workspaceRoot = vscode.workspace.workspaceFolders?.[0]?.uri.fsPath;
    return new Promise((resolve, reject) => {
      exec(command, { cwd: workspaceRoot }, (error, stdout) => {
        if (error) reject(error);
        else resolve(stdout);
      });
    });
  }

  private async getFileAtCommit(filePath: string, commit: string): Promise<string | null> {
    try {
      return await this.exec(`git show ${commit}:"${filePath}"`);
    } catch {
      return null; // File didn't exist at this commit
    }
  }

  private getSampleIndices(total: number, samples: number): number[] {
    if (total <= samples) return Array.from({ length: total }, (_, i) => i);

    const indices: number[] = [0]; // Always include first
    const step = (total - 1) / (samples - 1);

    for (let i = 1; i < samples - 1; i++) {
      indices.push(Math.round(i * step));
    }

    indices.push(total - 1); // Always include last
    return [...new Set(indices)].sort((a, b) => a - b);
  }
}
```

---

### CodeLens Integration

```typescript
// providers/history-codelens.provider.ts

import * as vscode from 'vscode';

export class HistoryCodeLensProvider implements vscode.CodeLensProvider {
  private _onDidChangeCodeLenses = new vscode.EventEmitter<void>();
  readonly onDidChangeCodeLenses = this._onDidChangeCodeLenses.event;

  constructor(
    private db: ViprDatabase,
    private gitService: GitHistoryService,
    private analysisService: AnalysisService
  ) {}

  async provideCodeLenses(document: vscode.TextDocument): Promise<vscode.CodeLens[]> {
    const filePath = vscode.workspace.asRelativePath(document.uri);
    const lenses: vscode.CodeLens[] = [];

    // Get current analysis
    const currentAnalysis = await this.analysisService.analyzeFile(document);
    if (!currentAnalysis) return lenses;

    // Get previous snapshot for comparison
    const previousSnapshot = await this.db.getPreviousSnapshot(filePath);

    // File-level summary lens at top
    const summaryLens = this.createSummaryLens(document, currentAnalysis, previousSnapshot);
    if (summaryLens) lenses.push(summaryLens);

    // Finding-level lenses with attribution
    const attributedFindings = await this.attributeFindings(filePath, currentAnalysis.findings);

    for (const finding of attributedFindings) {
      if (finding.introducedCommit) {
        const lens = this.createFindingLens(document, finding);
        lenses.push(lens);
      }
    }

    return lenses;
  }

  private createSummaryLens(
    document: vscode.TextDocument,
    current: AnalysisResult,
    previous?: AnalysisSnapshot
  ): vscode.CodeLens | null {
    const range = new vscode.Range(0, 0, 0, 0);

    if (!previous) {
      // No history yet
      return new vscode.CodeLens(range, {
        title: `📊 Score: ${current.overallScore}/100`,
        command: 'vipr.showAnalysis',
        arguments: [document.uri],
      });
    }

    const delta = current.overallScore - previous.overallScore;
    const icon = delta > 0 ? '📈' : delta < 0 ? '📉' : '➡️';
    const deltaStr = delta > 0 ? `+${delta}` : `${delta}`;

    if (Math.abs(delta) >= 5) {
      // Significant change - show attribution
      return new vscode.CodeLens(range, {
        title: `${icon} Score: ${current.overallScore} (${deltaStr}) • Click for history`,
        command: 'vipr.showHistoryPanel',
        arguments: [document.uri],
      });
    }

    return new vscode.CodeLens(range, {
      title: `📊 Score: ${current.overallScore}/100 (${deltaStr})`,
      command: 'vipr.showAnalysis',
      arguments: [document.uri],
    });
  }

  private createFindingLens(
    document: vscode.TextDocument,
    finding: AttributedFinding
  ): vscode.CodeLens {
    const line = Math.max(0, finding.line - 1);
    const range = new vscode.Range(line, 0, line, 0);

    const age = this.formatAge(finding.introducedDate!);
    const author = this.shortenAuthor(finding.introducedAuthor!);

    // Severity icon
    const icon = {
      critical: '🔴',
      high: '🟠',
      medium: '🟡',
      low: '🔵',
    }[finding.severity];

    return new vscode.CodeLens(range, {
      title: `${icon} ${finding.findingType} • Introduced ${age} by ${author}`,
      command: 'vipr.showFindingDetail',
      arguments: [finding],
    });
  }

  private formatAge(date: Date): string {
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));

    if (diffDays === 0) return 'today';
    if (diffDays === 1) return 'yesterday';
    if (diffDays < 7) return `${diffDays}d ago`;
    if (diffDays < 30) return `${Math.floor(diffDays / 7)}w ago`;
    if (diffDays < 365) return `${Math.floor(diffDays / 30)}mo ago`;
    return `${Math.floor(diffDays / 365)}y ago`;
  }

  private shortenAuthor(author: string): string {
    // "John Smith <john@example.com>" -> "John S."
    const match = author.match(/^([^\s]+)\s+([^\s])/);
    if (match) return `${match[1]} ${match[2]}.`;
    return author.split(' ')[0];
  }
}
```

---

### Lit Webview Component for History Panel

```typescript
// webview/components/history-panel.ts

import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { TrendPoint, RegressionReport, MetricDelta } from '../../types/history';

@customElement('vipr-history-panel')
export class HistoryPanel extends LitElement {
  @property({ type: String }) filePath = '';
  @property({ type: Object }) report?: RegressionReport;
  @property({ type: Array }) trend: TrendPoint[] = [];

  @state() private selectedCategory = 'overall';
  @state() private loading = false;

  static styles = css`
    :host {
      display: block;
      padding: var(--vscode-padding);
      font-family: var(--vscode-font-family);
    }

    .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 16px;
    }

    .score-delta {
      display: flex;
      align-items: center;
      gap: 8px;
    }

    .delta-badge {
      padding: 4px 8px;
      border-radius: 4px;
      font-weight: 600;
    }

    .delta-badge.improved {
      background: var(--vscode-testing-iconPassed);
      color: var(--vscode-editor-background);
    }

    .delta-badge.degraded {
      background: var(--vscode-testing-iconFailed);
      color: var(--vscode-editor-background);
    }

    .regression-card {
      background: var(--vscode-editor-inactiveSelectionBackground);
      border-left: 3px solid var(--vscode-testing-iconFailed);
      padding: 12px;
      margin: 16px 0;
      border-radius: 0 4px 4px 0;
    }

    .regression-card h4 {
      margin: 0 0 8px 0;
      color: var(--vscode-testing-iconFailed);
    }

    .commit-info {
      display: grid;
      grid-template-columns: auto 1fr;
      gap: 4px 12px;
      font-size: 12px;
    }

    .commit-info dt {
      color: var(--vscode-descriptionForeground);
    }

    .category-tabs {
      display: flex;
      gap: 4px;
      margin-bottom: 16px;
      flex-wrap: wrap;
    }

    .category-tab {
      padding: 6px 12px;
      border: 1px solid var(--vscode-button-border);
      background: transparent;
      color: var(--vscode-foreground);
      cursor: pointer;
      border-radius: 4px;
      font-size: 12px;
    }

    .category-tab.active {
      background: var(--vscode-button-background);
      color: var(--vscode-button-foreground);
    }

    .category-tab .delta {
      margin-left: 4px;
      opacity: 0.8;
    }

    .trend-chart {
      height: 200px;
      margin: 16px 0;
    }

    .findings-list {
      list-style: none;
      padding: 0;
      margin: 0;
    }

    .finding-item {
      display: flex;
      align-items: flex-start;
      gap: 12px;
      padding: 8px;
      border-bottom: 1px solid var(--vscode-panel-border);
    }

    .finding-item:hover {
      background: var(--vscode-list-hoverBackground);
      cursor: pointer;
    }

    .severity-icon {
      font-size: 16px;
    }

    .finding-details {
      flex: 1;
    }

    .finding-type {
      font-weight: 500;
    }

    .finding-attribution {
      font-size: 11px;
      color: var(--vscode-descriptionForeground);
      margin-top: 4px;
    }

    .empty-state {
      text-align: center;
      padding: 32px;
      color: var(--vscode-descriptionForeground);
    }
  `;

  render() {
    if (this.loading) {
      return html`<vscode-progress-ring></vscode-progress-ring>`;
    }

    if (!this.report) {
      return html`
        <div class="empty-state">
          <p>No history available for this file yet.</p>
          <p>History will be tracked after the next commit.</p>
        </div>
      `;
    }

    return html`
      <div class="header">
        <h3>${this.getFileName()}</h3>
        ${this.renderScoreDelta()}
      </div>

      ${this.report.primaryRegression ? this.renderRegressionCard() : ''}
      ${this.renderCategoryTabs()}

      <vipr-trend-chart .data=${this.trend} .category=${this.selectedCategory}></vipr-trend-chart>

      ${this.renderNewFindings()}
    `;
  }

  private renderScoreDelta() {
    const delta = this.report!.overallDelta;
    const direction = delta > 0 ? 'improved' : delta < 0 ? 'degraded' : '';
    const sign = delta > 0 ? '+' : '';

    return html`
      <div class="score-delta">
        <span>Overall Score Change:</span>
        <span class="delta-badge ${direction}">${sign}${delta}</span>
      </div>
    `;
  }

  private renderRegressionCard() {
    const reg = this.report!.primaryRegression!;

    return html`
      <div class="regression-card">
        <h4>⚠️ Primary Regression Identified</h4>
        <dl class="commit-info">
          <dt>Commit</dt>
          <dd><code>${reg.commit.slice(0, 7)}</code></dd>

          <dt>Author</dt>
          <dd>${reg.author}</dd>

          <dt>Date</dt>
          <dd>${this.formatDate(reg.date)}</dd>

          <dt>Message</dt>
          <dd>${reg.message}</dd>

          <dt>Impact</dt>
          <dd>${reg.impactScore > 0 ? '+' : ''}${reg.impactScore} points</dd>
        </dl>

        <vscode-button @click=${() => this.viewCommit(reg.commit)}> View Commit </vscode-button>
        <vscode-button appearance="secondary" @click=${() => this.viewDiff(reg.commit)}>
          View Diff
        </vscode-button>
      </div>
    `;
  }

  private renderCategoryTabs() {
    const categories = [
      { id: 'overall', label: 'Overall' },
      ...this.report!.categoryDeltas.map(d => ({
        id: d.category,
        label: this.formatCategoryLabel(d.category),
        delta: d.delta,
      })),
    ];

    return html`
      <div class="category-tabs">
        ${categories.map(
          cat => html`
            <button
              class="category-tab ${this.selectedCategory === cat.id ? 'active' : ''}"
              @click=${() => (this.selectedCategory = cat.id)}
            >
              ${cat.label}
              ${cat.delta !== undefined
                ? html` <span class="delta">(${cat.delta > 0 ? '+' : ''}${cat.delta})</span> `
                : ''}
            </button>
          `
        )}
      </div>
    `;
  }

  private renderNewFindings() {
    const findings = this.report!.newFindings.filter(
      f => this.selectedCategory === 'overall' || f.category === this.selectedCategory
    );

    if (findings.length === 0) {
      return html`<p class="empty-state">No new findings in this category.</p>`;
    }

    return html`
      <h4>New Findings (${findings.length})</h4>
      <ul class="findings-list">
        ${findings.map(
          f => html`
            <li class="finding-item" @click=${() => this.goToFinding(f)}>
              <span class="severity-icon">${this.getSeverityIcon(f.severity)}</span>
              <div class="finding-details">
                <div class="finding-type">${f.findingType}</div>
                <div class="finding-attribution">
                  Line ${f.line} • Introduced ${this.formatDate(f.introducedDate!)} by
                  ${f.introducedAuthor}
                </div>
              </div>
            </li>
          `
        )}
      </ul>
    `;
  }

  private getSeverityIcon(severity: string): string {
    return { critical: '🔴', high: '🟠', medium: '🟡', low: '🔵' }[severity] || '⚪';
  }

  private formatCategoryLabel(category: string): string {
    return category.charAt(0).toUpperCase() + category.slice(1).replace(/-/g, ' ');
  }

  private formatDate(date: Date): string {
    return new Intl.DateTimeFormat('en-US', {
      month: 'short',
      day: 'numeric',
      year: 'numeric',
    }).format(date);
  }

  private getFileName(): string {
    return this.filePath.split('/').pop() || this.filePath;
  }

  private viewCommit(hash: string) {
    this.dispatchEvent(new CustomEvent('view-commit', { detail: { hash } }));
  }

  private viewDiff(hash: string) {
    this.dispatchEvent(new CustomEvent('view-diff', { detail: { hash } }));
  }

  private goToFinding(finding: Finding) {
    this.dispatchEvent(new CustomEvent('go-to-finding', { detail: finding }));
  }
}
```

---

### Sparkline Chart Component

```typescript
// webview/components/trend-chart.ts

import { LitElement, html, css, svg } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('vipr-trend-chart')
export class TrendChart extends LitElement {
  @property({ type: Array }) data: TrendPoint[] = [];
  @property({ type: String }) category = 'overall';
  @property({ type: Number }) width = 400;
  @property({ type: Number }) height = 150;

  static styles = css`
    :host {
      display: block;
    }

    svg {
      width: 100%;
      height: auto;
    }

    .trend-line {
      fill: none;
      stroke: var(--vscode-charts-blue);
      stroke-width: 2;
    }

    .trend-area {
      fill: var(--vscode-charts-blue);
      opacity: 0.1;
    }

    .data-point {
      fill: var(--vscode-charts-blue);
      cursor: pointer;
    }

    .data-point:hover {
      fill: var(--vscode-focusBorder);
      r: 6;
    }

    .axis-line {
      stroke: var(--vscode-panel-border);
      stroke-width: 1;
    }

    .axis-label {
      fill: var(--vscode-descriptionForeground);
      font-size: 10px;
    }

    .tooltip {
      position: absolute;
      background: var(--vscode-editorHoverWidget-background);
      border: 1px solid var(--vscode-editorHoverWidget-border);
      padding: 8px;
      border-radius: 4px;
      font-size: 12px;
      pointer-events: none;
      z-index: 100;
    }
  `;

  render() {
    if (this.data.length < 2) {
      return html`<p>Not enough data points for trend visualization.</p>`;
    }

    const padding = { top: 20, right: 20, bottom: 30, left: 40 };
    const chartWidth = this.width - padding.left - padding.right;
    const chartHeight = this.height - padding.top - padding.bottom;

    const values = this.data.map(d => d.value);
    const minVal = Math.min(...values);
    const maxVal = Math.max(...values);
    const valueRange = maxVal - minVal || 1;

    const xScale = (i: number) => (i / (this.data.length - 1)) * chartWidth;
    const yScale = (v: number) => chartHeight - ((v - minVal) / valueRange) * chartHeight;

    const linePath = this.data
      .map((d, i) => `${i === 0 ? 'M' : 'L'} ${xScale(i)} ${yScale(d.value)}`)
      .join(' ');

    const areaPath = `${linePath} L ${chartWidth} ${chartHeight} L 0 ${chartHeight} Z`;

    return html`
      <svg viewBox="0 0 ${this.width} ${this.height}">
        <g transform="translate(${padding.left}, ${padding.top})">
          <!-- Area fill -->
          <path class="trend-area" d="${areaPath}"></path>

          <!-- Line -->
          <path class="trend-line" d="${linePath}"></path>

          <!-- Data points -->
          ${this.data.map(
            (d, i) => svg`
            <circle 
              class="data-point"
              cx="${xScale(i)}" 
              cy="${yScale(d.value)}"
              r="4"
              @mouseenter=${(e: MouseEvent) => this.showTooltip(e, d)}
              @mouseleave=${() => this.hideTooltip()}
            ></circle>
          `
          )}

          <!-- X axis -->
          <line
            class="axis-line"
            x1="0"
            y1="${chartHeight}"
            x2="${chartWidth}"
            y2="${chartHeight}"
          ></line>

          <!-- Y axis -->
          <line class="axis-line" x1="0" y1="0" x2="0" y2="${chartHeight}"></line>

          <!-- Y axis labels -->
          <text class="axis-label" x="-5" y="5" text-anchor="end">${maxVal}</text>
          <text class="axis-label" x="-5" y="${chartHeight}" text-anchor="end">${minVal}</text>

          <!-- X axis labels (first and last date) -->
          <text class="axis-label" x="0" y="${chartHeight + 15}" text-anchor="start">
            ${this.formatDateShort(this.data[0].date)}
          </text>
          <text class="axis-label" x="${chartWidth}" y="${chartHeight + 15}" text-anchor="end">
            ${this.formatDateShort(this.data[this.data.length - 1].date)}
          </text>
        </g>
      </svg>
    `;
  }

  private formatDateShort(date: Date): string {
    return new Intl.DateTimeFormat('en-US', { month: 'short', day: 'numeric' }).format(date);
  }

  private showTooltip(event: MouseEvent, point: TrendPoint) {
    // Emit event for parent to handle tooltip
    this.dispatchEvent(
      new CustomEvent('point-hover', {
        detail: { point, x: event.clientX, y: event.clientY },
      })
    );
  }

  private hideTooltip() {
    this.dispatchEvent(new CustomEvent('point-leave'));
  }
}
```

---

### Implementation Phases

#### Phase 1: Foundation (Week 1-2)

- [ ] SQLite schema setup with migrations
- [ ] Basic snapshot storage after each analysis run
- [ ] Store current HEAD commit hash with each snapshot
- [ ] Simple comparison: current vs. previous snapshot

**Deliverable:** CodeLens shows `📊 Score: 45 (+3)` at file top

#### Phase 2: Attribution (Week 3-4)

- [ ] Git blame integration for findings
- [ ] Blame caching in SQLite
- [ ] CodeLens on individual findings with "Introduced X ago by Y"
- [ ] Click-to-navigate to finding line

**Deliverable:** Findings show attribution in CodeLens

#### Phase 3: Regression Detection (Week 5-6)

- [ ] Binary search implementation for finding regression commits
- [ ] Historical analysis (analyze file at past commits)
- [ ] Primary regression identification
- [ ] Regression card in webview

**Deliverable:** "⚠️ Primary Regression Identified" card with commit details

#### Phase 4: Trend Visualization (Week 7-8)

- [ ] Trend chart component (Lit)
- [ ] Sampled history analysis
- [ ] Category breakdown tabs
- [ ] Export trend data

**Deliverable:** Full history panel with interactive trend charts

#### Phase 5: Polish & Integration (Week 9-10)

- [ ] Performance optimization (worker threads for git operations)
- [ ] Copilot integration: "Explain this regression" / "Suggest fix"
- [ ] Settings: history depth, sampling rate
- [ ] Telemetry for feature usage

---

### Configuration Options

```typescript
// Default configuration
export const historyConfig = {
  // How many commits to search during binary search
  maxBisectDepth: 10,

  // How many commits to sample for trend charts
  trendSampleCount: 10,

  // Maximum commits to fetch from git log
  maxCommitHistory: 100,

  // Minimum score delta to highlight as "significant"
  significantDeltaThreshold: 5,

  // Categories to track (user can disable some)
  enabledCategories: ['security', 'accessibility', 'performance', 'reliability', 'anti-patterns'],

  // Severity levels to show attribution for
  attributionSeverities: ['critical', 'high', 'medium'],

  // Cache TTL for git blame results (ms)
  blameCacheTTL: 24 * 60 * 60 * 1000, // 24 hours
};
```

---

This gives you a complete, extensible foundation. The schema handles evolving metrics gracefully (just add rows to `metric_scores`), the git integration is cached and efficient, and the UI components are modular Lit elements that integrate with your existing architecture.

Want me to expand on any specific part—perhaps the database service implementation or the worker thread architecture for git operations?
