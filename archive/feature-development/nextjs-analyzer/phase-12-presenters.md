# Phase 12: Report Presenters

## Status

Not Started

## Goals

Implement all report presenters for the Next.js analyzer plugin that transform analysis results into presentation models for CLI and other clients. Each presenter must implement IReportPresenter interface, define complete metadata, transform AnalysisResult to ReportPresentation, and register with the PresenterRegistry.

## Files Created

### Implementation Files

- `presenters/base-presenter.ts` - NextjsBasePresenter extending BaseReportPresenter
- `presenters/overview-presenter.ts` - Composite score and high-level summary
- `presenters/server-client-presenter.ts` - Server/Client component issues
- `presenters/data-fetching-presenter.ts` - Data fetching anti-patterns
- `presenters/migration-presenter.ts` - Migration readiness
- `presenters/security-presenter.ts` - Security vulnerabilities
- `presenters/config-presenter.ts` - Configuration issues
- `presenters/component-presenter.ts` - Component anti-patterns
- `presenters/performance-presenter.ts` - Performance issues
- `presenters/a11y-typescript-presenter.ts` - Accessibility and TypeScript issues
- `presenters/index.ts` - Exports and createNextjsPresenters factory

### Test Files

- `presenters/base-presenter.test.ts` - Base presenter utilities
- `presenters/overview-presenter.test.ts` - Overview presenter tests
- `presenters/server-client-presenter.test.ts` - Server/Client presenter tests
- `presenters/data-fetching-presenter.test.ts` - Data fetching presenter tests
- `presenters/migration-presenter.test.ts` - Migration presenter tests
- `presenters/security-presenter.test.ts` - Security presenter tests
- `presenters/config-presenter.test.ts` - Config presenter tests
- `presenters/component-presenter.test.ts` - Component presenter tests
- `presenters/performance-presenter.test.ts` - Performance presenter tests
- `presenters/a11y-typescript-presenter.test.ts` - A11y/TypeScript presenter tests
- `presenters/index.test.ts` - Factory function tests

## Implementation Details

### NextjsBasePresenter Class

Base class extending BaseReportPresenter with Next.js-specific utilities:

```typescript
import type {
  PluginResult,
  AnalysisResult,
  ReportType,
  PresentationSection,
  PresentationMetric,
  PresentationItem,
  PresentationScore,
  ItemSeverity,
  ItemLocation,
  ScoreThresholds,
} from '@vipr/common';
import { BaseReportPresenter, createAnalysisId } from '@vipr/common';

/**
 * Base class for Next.js report presenters.
 *
 * Provides common utilities for extracting analysis data from plugin results
 * and transforming it into presentation models.
 */
export abstract class NextjsBasePresenter extends BaseReportPresenter<PluginResult> {
  readonly pluginId = 'nextjs';
  abstract readonly reportType: ReportType;
  abstract readonly analysisId: string;

  /**
   * Check if this presenter can handle the given plugin result.
   */
  canPresent(pluginResult: PluginResult): boolean {
    if (pluginResult.pluginId !== 'nextjs') {
      return false;
    }

    const breakdown = pluginResult.analysisBreakdown;
    if (!breakdown) {
      return false;
    }

    return breakdown.has(createAnalysisId(this.analysisId));
  }

  /**
   * Get the analysis result for this presenter's analysis ID.
   */
  protected getAnalysisResult(pluginResult: PluginResult): AnalysisResult | undefined {
    return pluginResult.analysisBreakdown?.get(createAnalysisId(this.analysisId));
  }

  /**
   * Get the analysis data for this presenter's analysis ID.
   */
  protected getAnalysisData<T>(pluginResult: PluginResult): T | undefined {
    const result = this.getAnalysisResult(pluginResult);
    return result?.data as T | undefined;
  }

  /**
   * Create Next.js documentation URL.
   */
  protected createNextjsDocsUrl(path: string): string {
    return `https://nextjs.org/docs${path}`;
  }

  /**
   * Create migration guide URL for specific version.
   */
  protected createMigrationGuideUrl(fromVersion: number, toVersion: number): string {
    return `https://nextjs.org/docs/app/building-your-application/upgrading/version-${toVersion}`;
  }
}
```

### Overview Presenter

Composite score and high-level summary with dimensional breakdown:

```typescript
/**
 * Presenter for Next.js overview reports.
 *
 * Shows composite score and dimensional breakdown of all analyses.
 */
export class OverviewPresenter extends NextjsBasePresenter {
  readonly reportType = 'nextjs-overview' as const;
  readonly analysisId = 'nextjs-overview'; // Not a real analysis ID

  /**
   * Check if this presenter can handle the given plugin result.
   * Overview is always available if we have Next.js plugin data.
   */
  canPresent(pluginResult: PluginResult): boolean {
    return pluginResult.pluginId === 'nextjs' && pluginResult.metrics !== undefined;
  }

  getMetadata() {
    return this.createMetadata({
      label: 'Next.js Overview',
      hint: 'Composite score and dimensional analysis',
      icon: '▲',
      order: 10,
    });
  }

  present(pluginResult: PluginResult): ReportPresentation {
    const metrics = pluginResult.metrics as NextjsMetrics;

    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Next.js Analysis Overview',
      description:
        'Provides a high-level summary of Next.js-specific issues including router paradigms, component boundaries, data fetching patterns, and migration readiness. This report helps you understand the overall health of your Next.js application and identify critical areas for improvement.',
      sections: [
        this.createOverallSection(pluginResult),
        this.createRouterInfoSection(metrics),
        this.createDimensionsSection(metrics),
        this.createTopIssuesSection(pluginResult),
      ].filter(section => section.score || section.metrics?.length || section.items?.length),
    };
  }

  private createOverallSection(pluginResult: PluginResult): PresentationSection {
    const score = pluginResult.score ?? 0;
    return this.createSection('overall', 'Overall Next.js Health', {
      score: this.createScore(score, 'Score'),
    });
  }

  private createRouterInfoSection(metrics: NextjsMetrics): PresentationSection {
    const routerType = metrics.routerInfo?.type || 'unknown';
    const hasAppRouter = metrics.routerInfo?.hasAppRouter ?? false;
    const hasPagesRouter = metrics.routerInfo?.hasPagesRouter ?? false;
    const coexistence = hasAppRouter && hasPagesRouter;

    const metricsData = [
      this.createMetric('router-type', 'Router Type', routerType, { format: 'text' }),
    ];

    if (coexistence) {
      metricsData.push(
        this.createMetric('coexistence', 'Router Coexistence', 'Yes (mixed)', {
          format: 'text',
          context: 'App and Pages routers both present',
        })
      );
    }

    return this.createSection('router-info', 'Router Configuration', {
      metrics: metricsData,
    });
  }

  private createDimensionsSection(metrics: NextjsMetrics): PresentationSection {
    const dimensions = [
      {
        id: 'server-client',
        name: 'Server/Client Boundaries',
        score: metrics.serverClient?.score ?? 0,
        context: `(${metrics.serverClient?.totalIssues ?? 0} issues)`,
      },
      {
        id: 'data-fetching',
        name: 'Data Fetching',
        score: metrics.dataFetching?.score ?? 0,
        context: `(${metrics.dataFetching?.totalIssues ?? 0} issues)`,
      },
      {
        id: 'migration',
        name: 'Migration Readiness',
        score: metrics.migration?.score ?? 0,
        context: `(${metrics.migration?.blockerCount ?? 0} blockers)`,
      },
      {
        id: 'security',
        name: 'Security',
        score: metrics.security?.score ?? 0,
        context: `(${metrics.security?.totalVulnerabilities ?? 0} vulnerabilities)`,
      },
      {
        id: 'performance',
        name: 'Performance',
        score: metrics.performance?.score ?? 0,
        context: `(${metrics.performance?.totalIssues ?? 0} issues)`,
      },
    ];

    const metricsData = dimensions.map(dim =>
      this.createMetric(`dim-${dim.id}`, dim.name, dim.score, {
        maxValue: 100,
        format: 'score',
        thresholds: { excellent: 80, good: 60, fair: 40 },
        context: dim.context,
      })
    );

    return this.createSection('dimensions', 'Analysis Dimensions', {
      metrics: metricsData,
    });
  }

  private createTopIssuesSection(pluginResult: PluginResult): PresentationSection {
    const insights = pluginResult.insights || [];

    if (insights.length === 0) {
      return this.createSection('top-issues', 'Top Issues', {});
    }

    const sortedInsights = this.sortInsightsBySeverity(insights);
    const items = sortedInsights
      .slice(0, 10)
      .map((insight, idx) => this.createInsightItem(insight, idx));

    return this.createSection('top-issues', `Top 10 Issues (out of ${insights.length})`, {
      items,
      style: 'list',
    });
  }

  private sortInsightsBySeverity(insights: ComplexityInsight[]): ComplexityInsight[] {
    const severityOrder: Record<string, number> = {
      critical: 0,
      error: 1,
      warning: 2,
      info: 3,
    };

    return [...insights].sort(
      (a, b) => (severityOrder[a.severity] ?? 4) - (severityOrder[b.severity] ?? 4)
    );
  }

  private createInsightItem(insight: ComplexityInsight, idx: number): PresentationItem {
    const location = this.mapLocation(insight.location);
    const links: Record<string, string> = {};

    if (location?.file) {
      links.file = this.createFileUrl(location.file, location.line, location.column);
    }

    return this.createItem(`insight-${idx}`, this.mapSeverity(insight.severity), insight.message, {
      suggestion: insight.suggestion,
      location,
      category: insight.category,
      links: Object.keys(links).length > 0 ? links : undefined,
    });
  }
}
```

### Server/Client Presenter

Server/Client component boundary issues:

```typescript
/**
 * Presenter for Server/Client component analysis reports.
 */
export class ServerClientPresenter extends NextjsBasePresenter {
  readonly reportType = 'server-client' as const;
  readonly analysisId = 'nextjs-server-client';

  getMetadata() {
    return this.createMetadata({
      label: 'Server/Client Components',
      hint: 'Component boundary and directive issues',
      icon: '🔄',
      order: 20,
      categories: ['correctness', 'performance'],
    });
  }

  present(pluginResult: PluginResult): ReportPresentation {
    const data = this.getAnalysisData<ServerClientComplexity>(pluginResult);

    if (!data) {
      return this.createEmptyPresentation();
    }

    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Server/Client Component Analysis',
      description:
        'Identifies issues with "use client" and "use server" directive placement, unnecessary client components, and non-serializable props crossing Server/Client boundaries. These issues can cause runtime errors, bundle bloat, and hydration failures.',
      score: this.createScore(data.score, 'Boundary Score'),
      sections: [
        this.createSummarySection(data),
        this.createBySeveritySection(data),
        this.createByTypeSection(data),
        this.createIssuesSection(data),
      ].filter(section => section.metrics?.length || section.items?.length),
    };
  }

  private createSummarySection(data: ServerClientComplexity) {
    return this.createSection('summary', 'Summary', {
      metrics: [
        this.createMetric('total-issues', 'Total Issues', data.totalIssues, { format: 'count' }),
        this.createMetric('directive-issues', 'Directive Issues', data.stats.directiveIssues, {
          format: 'count',
        }),
        this.createMetric(
          'unnecessary-client',
          'Unnecessary Client Components',
          data.stats.unnecessaryClient,
          { format: 'count' }
        ),
        this.createMetric(
          'serialization-issues',
          'Serialization Issues',
          data.stats.serializationIssues,
          { format: 'count' }
        ),
      ],
    });
  }

  private createBySeveritySection(data: ServerClientComplexity) {
    const metrics = Object.entries(data.bySeverity)
      .filter(([_, count]) => count > 0)
      .map(([severity, count]) =>
        this.createMetric(`severity-${severity}`, this.formatLabel(severity), count, {
          format: 'count',
        })
      );

    return this.createSection('by-severity', 'Issues by Severity', {
      metrics,
      style: 'summary',
    });
  }

  private createByTypeSection(data: ServerClientComplexity) {
    const metrics = Object.entries(data.byType)
      .filter(([_, count]) => count > 0)
      .map(([type, count]) =>
        this.createMetric(`type-${type}`, this.formatLabel(type), count, {
          format: 'count',
        })
      );

    return this.createSection('by-type', 'Issues by Type', {
      metrics,
      style: 'summary',
    });
  }

  private createIssuesSection(data: ServerClientComplexity) {
    const items = this.sortIssues(data.issues).map(issue => this.createIssueItem(issue));

    return this.createSection('issues', 'Detected Issues', {
      items,
      style: 'list',
    });
  }

  private createIssueItem(issue: ServerClientIssue): PresentationItem {
    const location = this.mapLocation({ line: issue.line, file: issue.file });
    const links: Record<string, string> = {};

    if (location?.file) {
      links.file = this.createFileUrl(location.file, location.line);
      links.docs = this.createNextjsDocsUrl(
        '/app/building-your-application/rendering/server-components'
      );
    }

    return this.createItem(
      `issue-${issue.type}-${issue.line}`,
      this.mapSeverity(issue.severity),
      issue.message,
      {
        description: issue.description,
        location,
        suggestion: issue.recommendation,
        category: issue.type,
        metadata: {
          type: issue.type,
          breaking: issue.breaking,
        },
        links: Object.keys(links).length > 0 ? links : undefined,
      }
    );
  }

  private sortIssues(issues: ServerClientIssue[]): ServerClientIssue[] {
    const severityOrder: Record<string, number> = {
      critical: 0,
      error: 1,
      warning: 2,
      info: 3,
    };

    return [...issues].sort(
      (a, b) => (severityOrder[a.severity] ?? 4) - (severityOrder[b.severity] ?? 4)
    );
  }

  private createEmptyPresentation(): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Server/Client Component Analysis',
      description:
        'Identifies issues with "use client" and "use server" directive placement, unnecessary client components, and non-serializable props crossing Server/Client boundaries.',
      sections: [],
    };
  }
}
```

### Migration Presenter

Migration readiness and breaking changes:

```typescript
/**
 * Presenter for migration analysis reports.
 */
export class MigrationPresenter extends NextjsBasePresenter {
  readonly reportType = 'migration' as const;
  readonly analysisId = 'nextjs-migration';

  getMetadata() {
    return this.createMetadata({
      label: 'Migration Readiness',
      hint: 'Version compatibility and breaking changes',
      icon: '⚡',
      order: 30,
      categories: ['migration', 'breaking-changes'],
    });
  }

  present(pluginResult: PluginResult): ReportPresentation {
    const data = this.getAnalysisData<MigrationComplexity>(pluginResult);

    if (!data) {
      return this.createEmptyPresentation();
    }

    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Migration Readiness Analysis',
      description:
        'Identifies breaking changes and deprecated patterns across Next.js versions. Focus on blockers and high-priority warnings to ensure smooth migration to newer versions.',
      score: this.createScore(data.score, 'Readiness Score'),
      sections: [
        this.createSummarySection(data),
        this.createVersionSection(data),
        this.createBlockersSection(data),
        this.createWarningsSection(data),
        this.createDeprecationsSection(data),
      ].filter(section => section.metrics?.length || section.items?.length),
    };
  }

  private createSummarySection(data: MigrationComplexity) {
    return this.createSection('summary', 'Summary', {
      metrics: [
        this.createMetric('readiness-score', 'Readiness Score', data.readinessScore, {
          maxValue: 100,
          format: 'score',
          thresholds: { excellent: 80, good: 60, fair: 40 },
        }),
        this.createMetric('blockers', 'Blockers', data.blockerCount, {
          format: 'count',
          context: 'Must be resolved before upgrade',
        }),
        this.createMetric('warnings', 'Warnings', data.warningCount, {
          format: 'count',
          context: 'Should be addressed',
        }),
        this.createMetric('estimated-effort', 'Estimated Effort', data.estimatedEffort, {
          format: 'text',
        }),
      ],
    });
  }

  private createVersionSection(data: MigrationComplexity) {
    const metrics = [
      this.createMetric('current-version', 'Current Version', data.currentVersion || 'Unknown', {
        format: 'text',
      }),
      this.createMetric('target-version', 'Target Version', data.targetVersion || 'Latest', {
        format: 'text',
      }),
    ];

    if (data.detectedFeatures && data.detectedFeatures.length > 0) {
      metrics.push(
        this.createMetric(
          'detected-features',
          'Detected Features',
          data.detectedFeatures.join(', '),
          {
            format: 'text',
          }
        )
      );
    }

    return this.createSection('version-info', 'Version Information', {
      metrics,
    });
  }

  private createBlockersSection(data: MigrationComplexity) {
    if (data.blockers.length === 0) {
      return this.createSection('blockers', 'Migration Blockers', {
        description: 'No blockers detected',
      });
    }

    const items = data.blockers.map((blocker, idx) =>
      this.createItem(`blocker-${idx}`, 'critical', blocker.message, {
        description: blocker.description,
        suggestion: blocker.recommendation,
        location: this.mapLocation({ line: blocker.line, file: blocker.file }),
        category: blocker.type,
        metadata: {
          breaking: blocker.breaking,
          affectedVersion: blocker.affectedVersion,
        },
      })
    );

    return this.createSection('blockers', `Migration Blockers (${data.blockers.length})`, {
      items,
      style: 'list',
    });
  }

  private createWarningsSection(data: MigrationComplexity) {
    if (data.warnings.length === 0) {
      return this.createSection('warnings', 'Migration Warnings', {
        description: 'No warnings',
      });
    }

    const items = data.warnings.slice(0, 20).map((warning, idx) =>
      this.createItem(`warning-${idx}`, this.mapSeverity(warning.severity), warning.message, {
        description: warning.description,
        suggestion: warning.recommendation,
        location: this.mapLocation({ line: warning.line, file: warning.file }),
        category: warning.type,
      })
    );

    return this.createSection(
      'warnings',
      `Migration Warnings (showing ${items.length} of ${data.warnings.length})`,
      {
        items,
        style: 'list',
      }
    );
  }

  private createDeprecationsSection(data: MigrationComplexity) {
    if (!data.deprecatedPatterns || data.deprecatedPatterns.length === 0) {
      return this.createSection('deprecations', 'Deprecated Patterns', {});
    }

    const items = data.deprecatedPatterns.map((pattern, idx) =>
      this.createItem(`deprecation-${idx}`, 'medium', pattern.pattern, {
        description: `Deprecated in Next.js ${pattern.deprecatedIn}`,
        suggestion: pattern.replacement,
        category: 'deprecation',
      })
    );

    return this.createSection('deprecations', 'Deprecated Patterns', {
      items,
      style: 'list',
    });
  }

  private createEmptyPresentation(): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Migration Readiness Analysis',
      description: 'Identifies breaking changes and deprecated patterns across Next.js versions.',
      sections: [],
    };
  }
}
```

### Presenter Factory Function

Factory to create all presenters:

```typescript
/**
 * Create all Next.js report presenters.
 */
export function createNextjsPresenters(): IReportPresenter[] {
  return [
    new OverviewPresenter(),
    new ServerClientPresenter(),
    new DataFetchingPresenter(),
    new MigrationPresenter(),
    new SecurityPresenter(),
    new ConfigPresenter(),
    new ComponentPresenter(),
    new PerformancePresenter(),
    new A11yTypeScriptPresenter(),
  ];
}
```

### Registration in Plugin

Update `plugin.ts` to register presenters:

```typescript
import { createNextjsPresenters } from './presenters';

export class NextjsAnalyzerPlugin implements ITechnologyPlugin {
  readonly id = 'nextjs';
  readonly name = 'Next.js Analyzer';
  readonly version = '1.0.0';
  readonly filePatterns = [
    '**/*.tsx',
    '**/*.jsx',
    '**/*.ts',
    '**/*.js',
    '**/next.config.{js,mjs,ts}',
  ];

  private analyses: Map<string, IAnalysis> = new Map();
  private presenters: IReportPresenter[] = [];

  constructor() {
    this.registerAnalyses();
    this.presenters = createNextjsPresenters();
  }

  /**
   * Get all report presenters registered by this plugin.
   */
  getReportPresenters(): IReportPresenter[] {
    return this.presenters;
  }
}
```

## Test Coverage

### Test Suite Structure

Each presenter test file should include:

**Metadata Tests (5 tests per presenter):**

- Has correct reportType
- Has correct pluginId
- Has complete metadata (label, hint, icon, order)
- Has appropriate categories
- Metadata fields are non-empty

**CanPresent Tests (3 tests per presenter):**

- Returns true for valid plugin result with analysis data
- Returns false for wrong pluginId
- Returns false when analysis data missing

**Presentation Structure Tests (8 tests per presenter):**

- Returns valid ReportPresentation object
- Includes correct title and description
- Has score when appropriate
- Sections are properly structured
- Items have correct severity mapping
- Locations are properly mapped
- Links are generated when appropriate
- Empty data produces valid empty presentation

**Data Transformation Tests (10 tests per presenter):**

- Correctly transforms analysis data to sections
- Metrics calculated correctly
- Items sorted by severity
- Categories grouped properly
- Context added to metrics
- Suggestions included in items
- File locations linked
- Documentation links added
- Breaking changes flagged
- Edge cases handled (empty arrays, null values)

Total: Approximately 26 tests per presenter × 9 presenters = 234 tests

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 234 tests pass
- [ ] All presenters implement IReportPresenter interface
- [ ] All presenters define complete getMetadata() with reportType, icon, label, categories
- [ ] All presenters transform AnalysisResult to ReportPresentation correctly
- [ ] Sections include proper PresentationItems with severity, location, suggestions
- [ ] Metrics use appropriate format (score, count, text)
- [ ] File locations generate proper links
- [ ] Documentation URLs link to Next.js docs
- [ ] Severity mapping handles all levels (critical, error, warning, info)
- [ ] Empty/missing data produces valid empty presentations
- [ ] No direct imports of presenters in client code
- [ ] All presenters register via createNextjsPresenters() factory
- [ ] Plugin registers presenters in getReportPresenters()

## Technical Notes

### Presenter Metadata Standards

Every presenter must define complete metadata:

```typescript
getMetadata() {
  return this.createMetadata({
    label: 'Human-readable label',
    hint: 'Short description of what this report shows',
    icon: '🔧', // Emoji or symbol
    order: 20,  // Display order (10, 20, 30...)
    categories: ['category1', 'category2'], // Optional
  });
}
```

### Report Types by Presenter

- `nextjs-overview` - Overall health and dimensions
- `server-client` - Server/Client boundary issues
- `data-fetching` - Data fetching anti-patterns
- `migration` - Migration readiness
- `security` - Security vulnerabilities
- `config` - Configuration issues
- `component` - Component anti-patterns
- `performance` - Performance issues
- `a11y-typescript` - Accessibility and TypeScript

### Severity Mapping

Consistent mapping across all presenters:

```typescript
protected mapSeverity(severity: string | undefined): ItemSeverity {
  const normalized = severity?.toLowerCase();
  switch (normalized) {
    case 'critical':
      return 'critical';
    case 'error':
      return 'high';
    case 'warning':
      return 'medium';
    case 'info':
      return 'low';
    default:
      return 'info';
  }
}
```

### Documentation Links

Generate Next.js documentation links:

```typescript
protected createNextjsDocsUrl(path: string): string {
  return `https://nextjs.org/docs${path}`;
}

// Usage:
links.docs = this.createNextjsDocsUrl('/app/building-your-application/rendering/server-components');
```

### Sorting and Prioritization

Always sort items by severity before display:

```typescript
private sortIssues(issues: Issue[]): Issue[] {
  const severityOrder: Record<string, number> = {
    critical: 0,
    error: 1,
    warning: 2,
    info: 3,
  };

  return [...issues].sort(
    (a, b) => (severityOrder[a.severity] ?? 4) - (severityOrder[b.severity] ?? 4)
  );
}
```

### Empty Presentation Handling

Always provide valid presentation even with no data:

```typescript
private createEmptyPresentation(): ReportPresentation {
  return {
    reportType: this.reportType,
    pluginId: this.pluginId,
    title: 'Report Title',
    description: 'Report description',
    sections: [],
  };
}
```

### Location Mapping

Use inherited mapLocation utility:

```typescript
const location = this.mapLocation({
  line: issue.line,
  column: issue.column,
  file: issue.file,
});
```

### Metrics Context

Add context to metrics for clarity:

```typescript
this.createMetric('dimension-score', 'Dimension Name', score, {
  maxValue: 100,
  format: 'score',
  thresholds: { excellent: 80, good: 60, fair: 40 },
  context: `(${issueCount} issues)`,
});
```

### Breaking Changes Flag

Flag breaking changes in metadata:

```typescript
metadata: {
  breaking: issue.breaking,
  affectedVersion: issue.affectedVersion,
}
```

## Next Steps

After completing Phase 12, proceed to:

- Phase 13: Integration Testing - Create comprehensive test fixtures and integration tests
- Phase 14: CLI Integration - Integrate with CLI plugin loader and create documentation
