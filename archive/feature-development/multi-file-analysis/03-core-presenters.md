---
id: 03-core-presenters
---

# Phase 3: Core Presenters

## Overview

This phase creates four batch presenters in the `@vipr/core` analyzer that transform `BatchAnalysisResult` into presentation models. These presenters follow the existing plugin architecture and are auto-discovered via the presenter registry.

## Batch Presenter Architecture

### Key Design Decisions

| Decision                              | Rationale                                                                  |
| ------------------------------------- | -------------------------------------------------------------------------- |
| **Presenters in `@vipr/core`**        | Core analyzer owns aggregation logic, independent of specific technologies |
| **Input type: `BatchAnalysisResult`** | Distinct from `PluginResult`, batch presenters receive aggregated data     |
| **Category: `['batch']`**             | Enables formatters to filter batch vs single-file reports                  |
| **Auto-discovery**                    | Registered via `getReportPresenters()`, no client-side hardcoding          |
| **Reuse presentation types**          | Same `ReportPresentation` interface as single-file presenters              |

### Presenter Metadata

All batch presenters include `categories: ['batch']` in metadata:

```typescript
getMetadata(): IReportMetadata {
  return {
    reportType: 'project-summary',
    pluginId: 'core',
    label: 'Project Summary',
    hint: 'Overall health and technology breakdown',
    icon: '📊',
    order: 100, // After single-file reports
    categories: ['batch'], // KEY: Identifies batch presenters
  };
}
```

## Report Types

| Report Type        | Purpose                                 | Primary Data                                   |
| ------------------ | --------------------------------------- | ---------------------------------------------- |
| `project-summary`  | Overall health and technology breakdown | `summary`, `technologies`, `scoreDistribution` |
| `batch-issues`     | Issues grouped by category and severity | `issues`                                       |
| `batch-complexity` | Complexity distributions and histograms | `complexity`                                   |
| `worst-files`      | Files needing immediate attention       | `recommendations`                              |

## Presenter 1: Project Summary

### File Location

`analyzers/core/src/presenters/project-summary-presenter.ts`

### Full Implementation

```typescript
/**
 * Project Summary Presenter
 *
 * Presents aggregated statistics for a batch analysis including:
 * - Overall health score
 * - Technology breakdown
 * - Score distribution
 * - Issue counts by severity
 *
 * @module @vipr/core/presenters
 */

import type {
  BatchAnalysisResult,
  ReportPresentation,
  PresentationSection,
  PresentationMetric,
  PresentationItem,
  IReportPresenter,
  IReportMetadata,
} from '@vipr/common';
import { BaseReportPresenter, isBatchResult } from '@vipr/common';

/**
 * Presenter for project summary report.
 */
export class ProjectSummaryPresenter
  extends BaseReportPresenter
  implements IReportPresenter<BatchAnalysisResult>
{
  readonly pluginId = 'core';
  readonly reportType = 'project-summary' as const;

  /**
   * Check if this presenter can handle the given data.
   */
  canPresent(data: unknown): data is BatchAnalysisResult {
    return isBatchResult(data as any);
  }

  /**
   * Get metadata for this report.
   */
  getMetadata(): IReportMetadata {
    return this.createMetadata({
      label: 'Project Summary',
      hint: 'Overall health and technology breakdown',
      icon: '📊',
      order: 100,
      categories: ['batch'],
    });
  }

  /**
   * Transform batch result into presentation model.
   */
  present(data: BatchAnalysisResult): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Project Summary',
      description:
        'Overview of your project health, including overall score, technology breakdown, score distribution, and issue severity counts. Use this report to quickly assess the overall quality of your codebase and identify areas needing attention.',
      sections: [
        this.createOverallSection(data),
        this.createTechnologySection(data),
        this.createDistributionSection(data),
        this.createIssuesSection(data),
      ],
    };
  }

  /**
   * Create overall health section.
   */
  private createOverallSection(data: BatchAnalysisResult): PresentationSection {
    const { summary } = data;

    return {
      id: 'overall-health',
      title: 'Overall Health',
      score: {
        value: Math.round(summary.averageScore),
        maxValue: 100,
        label: 'Average Score',
      },
      metrics: [
        {
          id: 'total-files',
          label: 'Total Files',
          value: summary.totalFiles,
          format: 'count',
        },
        {
          id: 'analyzed-files',
          label: 'Analyzed',
          value: summary.analyzedFiles,
          format: 'count',
        },
        {
          id: 'skipped-files',
          label: 'Skipped',
          value: summary.skippedFiles,
          format: 'count',
        },
        {
          id: 'execution-time',
          label: 'Execution Time',
          value: Math.round(summary.executionTimeMs),
          format: 'milliseconds',
          context: `(${(summary.executionTimeMs / 1000).toFixed(2)}s)`,
        },
        {
          id: 'min-score',
          label: 'Lowest Score',
          value: summary.minScore,
          maxValue: 100,
          format: 'percentage',
        },
        {
          id: 'max-score',
          label: 'Highest Score',
          value: summary.maxScore,
          maxValue: 100,
          format: 'percentage',
        },
      ],
    };
  }

  /**
   * Create technology breakdown section.
   */
  private createTechnologySection(data: BatchAnalysisResult): PresentationSection {
    const items: PresentationItem[] = data.technologies.map(tech => ({
      id: `tech-${tech.technology}`,
      label: this.formatTechnology(tech.technology),
      value: tech.fileCount,
      context: `Avg: ${Math.round(tech.averageScore)}/100`,
      metadata: {
        averageScore: tech.averageScore,
        minScore: tech.minScore,
        maxScore: tech.maxScore,
        files: tech.files,
      },
    }));

    return {
      id: 'technology-breakdown',
      title: 'Technology Breakdown',
      items,
    };
  }

  /**
   * Create score distribution section.
   */
  private createDistributionSection(data: BatchAnalysisResult): PresentationSection {
    const items: PresentationItem[] = data.scoreDistribution.map(bucket => ({
      id: `bucket-${bucket.min}-${bucket.max}`,
      label: bucket.label,
      value: bucket.count,
      context: `${bucket.percentage.toFixed(1)}%`,
      metadata: {
        min: bucket.min,
        max: bucket.max,
        files: bucket.files,
      },
    }));

    return {
      id: 'score-distribution',
      title: 'Score Distribution',
      items,
    };
  }

  /**
   * Create issues summary section.
   */
  private createIssuesSection(data: BatchAnalysisResult): PresentationSection {
    const metrics: PresentationMetric[] = [
      {
        id: 'total-issues',
        label: 'Total Issues',
        value: data.issues.totalIssues,
        format: 'count',
      },
      {
        id: 'critical-count',
        label: 'Critical',
        value: data.issues.severityCounts.critical,
        format: 'count',
        severity: 'critical',
      },
      {
        id: 'warning-count',
        label: 'Warnings',
        value: data.issues.severityCounts.warning,
        format: 'count',
        severity: 'warning',
      },
      {
        id: 'info-count',
        label: 'Info',
        value: data.issues.severityCounts.info,
        format: 'count',
        severity: 'info',
      },
    ];

    return {
      id: 'issues-summary',
      title: 'Issues Summary',
      metrics,
    };
  }

  /**
   * Format technology name for display.
   */
  private formatTechnology(tech: string): string {
    const labels: Record<string, string> = {
      react: 'React',
      typescript: 'TypeScript',
      javascript: 'JavaScript',
      nextjs: 'Next.js',
      unknown: 'Unknown',
    };
    return labels[tech] ?? tech;
  }
}
```

## Presenter 2: Batch Issues

### File Location

`analyzers/core/src/presenters/batch-issues-presenter.ts`

### Implementation

```typescript
/**
 * Batch Issues Presenter
 *
 * Presents aggregated issue statistics including:
 * - Issues grouped by category
 * - Severity distribution per category
 * - Files affected by each issue type
 * - Most affected files
 *
 * @module @vipr/core/presenters
 */

import type {
  BatchAnalysisResult,
  ReportPresentation,
  PresentationSection,
  PresentationItem,
  IReportPresenter,
  IReportMetadata,
} from '@vipr/common';
import { BaseReportPresenter, isBatchResult } from '@vipr/common';

export class BatchIssuesPresenter
  extends BaseReportPresenter
  implements IReportPresenter<BatchAnalysisResult>
{
  readonly pluginId = 'core';
  readonly reportType = 'batch-issues' as const;

  canPresent(data: unknown): data is BatchAnalysisResult {
    return isBatchResult(data as any);
  }

  getMetadata(): IReportMetadata {
    return this.createMetadata({
      label: 'Issues Report',
      hint: 'Issues grouped by category and severity',
      icon: '⚠️',
      order: 101,
      categories: ['batch'],
    });
  }

  present(data: BatchAnalysisResult): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Issues Report',
      description:
        'Detailed breakdown of all issues found across your codebase, grouped by category and severity. Identify patterns in code quality problems and prioritize fixes based on severity and affected file counts.',
      sections: [
        this.createSeveritySummarySection(data),
        this.createCategorySection(data),
        this.createMostAffectedSection(data),
      ],
    };
  }

  private createSeveritySummarySection(data: BatchAnalysisResult): PresentationSection {
    const { severityCounts } = data.issues;
    const total = severityCounts.critical + severityCounts.warning + severityCounts.info;

    return {
      id: 'severity-summary',
      title: 'Severity Breakdown',
      metrics: [
        {
          id: 'critical',
          label: 'Critical',
          value: severityCounts.critical,
          format: 'count',
          severity: 'critical',
          context: total > 0 ? `${((severityCounts.critical / total) * 100).toFixed(1)}%` : '0%',
        },
        {
          id: 'warning',
          label: 'Warning',
          value: severityCounts.warning,
          format: 'count',
          severity: 'warning',
          context: total > 0 ? `${((severityCounts.warning / total) * 100).toFixed(1)}%` : '0%',
        },
        {
          id: 'info',
          label: 'Info',
          value: severityCounts.info,
          format: 'count',
          severity: 'info',
          context: total > 0 ? `${((severityCounts.info / total) * 100).toFixed(1)}%` : '0%',
        },
      ],
    };
  }

  private createCategorySection(data: BatchAnalysisResult): PresentationSection {
    const items: PresentationItem[] = data.issues.byCategory
      .sort((a, b) => b.count - a.count)
      .map(category => {
        const criticalPct =
          category.count > 0
            ? ((category.severityCounts.critical / category.count) * 100).toFixed(0)
            : '0';

        return {
          id: `category-${category.category}`,
          label: this.formatCategory(category.category),
          value: category.count,
          context: `${category.fileCount} files • ${criticalPct}% critical`,
          metadata: {
            category: category.category,
            fileCount: category.fileCount,
            severityCounts: category.severityCounts,
            affectedFiles: category.affectedFiles,
          },
        };
      });

    return {
      id: 'issues-by-category',
      title: 'Issues by Category',
      items,
    };
  }

  private createMostAffectedSection(data: BatchAnalysisResult): PresentationSection {
    const items: PresentationItem[] = data.issues.mostAffectedFiles.slice(0, 20).map(file => ({
      id: `affected-${file.filePath}`,
      label: this.formatFilePath(file.filePath),
      value: file.issueCount,
      context: file.criticalCount > 0 ? `${file.criticalCount} critical` : 'No critical issues',
      severity: file.criticalCount > 0 ? 'critical' : undefined,
      metadata: {
        filePath: file.filePath,
        issueCount: file.issueCount,
        criticalCount: file.criticalCount,
      },
    }));

    return {
      id: 'most-affected-files',
      title: 'Most Affected Files (Top 20)',
      items,
    };
  }

  private formatCategory(category: string): string {
    return category
      .split('-')
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ');
  }

  private formatFilePath(path: string): string {
    const parts = path.split('/');
    return parts.length > 3 ? `.../${parts.slice(-3).join('/')}` : path;
  }
}
```

## Presenter 3: Batch Complexity

### File Location

`analyzers/core/src/presenters/batch-complexity-presenter.ts`

### Implementation

```typescript
/**
 * Batch Complexity Presenter
 *
 * Presents complexity distribution statistics including:
 * - Cyclomatic complexity distribution
 * - Halstead metrics distribution
 * - Maintainability index distribution
 * - Most complex files
 *
 * @module @vipr/core/presenters
 */

import type {
  BatchAnalysisResult,
  ReportPresentation,
  PresentationSection,
  PresentationMetric,
  PresentationItem,
  IReportPresenter,
  IReportMetadata,
  MetricDistribution,
} from '@vipr/common';
import { BaseReportPresenter, isBatchResult } from '@vipr/common';

export class BatchComplexityPresenter
  extends BaseReportPresenter
  implements IReportPresenter<BatchAnalysisResult>
{
  readonly pluginId = 'core';
  readonly reportType = 'batch-complexity' as const;

  canPresent(data: unknown): data is BatchAnalysisResult {
    return isBatchResult(data as any) && (data as BatchAnalysisResult).complexity !== undefined;
  }

  getMetadata(): IReportMetadata {
    return this.createMetadata({
      label: 'Complexity Distribution',
      hint: 'Complexity metrics across all files',
      icon: '📈',
      order: 102,
      categories: ['batch'],
    });
  }

  present(data: BatchAnalysisResult): ReportPresentation {
    if (!data.complexity) {
      return {
        reportType: this.reportType,
        pluginId: this.pluginId,
        title: 'Complexity Distribution',
        description: 'No complexity data available for this project.',
        sections: [],
      };
    }

    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Complexity Distribution',
      description:
        'Statistical analysis of complexity metrics across your codebase. Identify outliers, understand distribution patterns, and prioritize complex files for refactoring.',
      sections: [
        this.createDistributionSection(data.complexity.cyclomatic, 'Cyclomatic Complexity'),
        this.createDistributionSection(data.complexity.halsteadVolume, 'Halstead Volume'),
        this.createDistributionSection(data.complexity.maintainability, 'Maintainability Index'),
        this.createMostComplexSection(data),
      ].filter(section => section.metrics?.length || section.items?.length),
    };
  }

  private createDistributionSection(
    distribution: MetricDistribution | undefined,
    title: string
  ): PresentationSection {
    if (!distribution) {
      return {
        id: `${title.toLowerCase().replace(/\s+/g, '-')}`,
        title,
        metrics: [],
      };
    }

    const metrics: PresentationMetric[] = [
      {
        id: 'mean',
        label: 'Mean',
        value: Math.round(distribution.mean * 100) / 100,
        format: 'count',
      },
      {
        id: 'median',
        label: 'Median (p50)',
        value: Math.round(distribution.median * 100) / 100,
        format: 'count',
      },
      {
        id: 'p75',
        label: '75th Percentile',
        value: Math.round(distribution.p75 * 100) / 100,
        format: 'count',
      },
      {
        id: 'p90',
        label: '90th Percentile',
        value: Math.round(distribution.p90 * 100) / 100,
        format: 'count',
      },
      {
        id: 'p95',
        label: '95th Percentile',
        value: Math.round(distribution.p95 * 100) / 100,
        format: 'count',
      },
      {
        id: 'min',
        label: 'Min',
        value: Math.round(distribution.min * 100) / 100,
        format: 'count',
      },
      {
        id: 'max',
        label: 'Max',
        value: Math.round(distribution.max * 100) / 100,
        format: 'count',
      },
      {
        id: 'stddev',
        label: 'Std Dev',
        value: Math.round(distribution.stdDev * 100) / 100,
        format: 'count',
      },
    ];

    return {
      id: title.toLowerCase().replace(/\s+/g, '-'),
      title,
      metrics,
    };
  }

  private createMostComplexSection(data: BatchAnalysisResult): PresentationSection {
    if (!data.complexity) {
      return {
        id: 'most-complex-files',
        title: 'Most Complex Files',
        items: [],
      };
    }

    const items: PresentationItem[] = data.complexity.mostComplexFiles.map(file => ({
      id: `complex-${file.filePath}`,
      label: this.formatFilePath(file.filePath),
      value: Math.round(file.primaryComplexity.value),
      context: `${file.primaryComplexity.metric} • Score: ${file.score}/100`,
      severity: file.score < 50 ? 'critical' : file.score < 70 ? 'warning' : undefined,
      metadata: {
        filePath: file.filePath,
        score: file.score,
        complexityMetric: file.primaryComplexity.metric,
        complexityValue: file.primaryComplexity.value,
      },
    }));

    return {
      id: 'most-complex-files',
      title: 'Most Complex Files (Top 20)',
      items,
    };
  }

  private formatFilePath(path: string): string {
    const parts = path.split('/');
    return parts.length > 3 ? `.../${parts.slice(-3).join('/')}` : path;
  }
}
```

## Presenter 4: Worst Files

### File Location

`analyzers/core/src/presenters/worst-files-presenter.ts`

### Implementation

```typescript
/**
 * Worst Files Presenter
 *
 * Presents files recommended for refactoring based on:
 * - Low scores
 * - Critical issues
 * - High complexity
 * - Estimated effort
 *
 * @module @vipr/core/presenters
 */

import type {
  BatchAnalysisResult,
  ReportPresentation,
  PresentationSection,
  PresentationItem,
  IReportPresenter,
  IReportMetadata,
  FileRecommendation,
} from '@vipr/common';
import { BaseReportPresenter, isBatchResult } from '@vipr/common';

export class WorstFilesPresenter
  extends BaseReportPresenter
  implements IReportPresenter<BatchAnalysisResult>
{
  readonly pluginId = 'core';
  readonly reportType = 'worst-files' as const;

  canPresent(data: unknown): data is BatchAnalysisResult {
    return isBatchResult(data as any);
  }

  getMetadata(): IReportMetadata {
    return this.createMetadata({
      label: 'Worst Files',
      hint: 'Files needing immediate attention',
      icon: '🔥',
      order: 103,
      categories: ['batch'],
    });
  }

  present(data: BatchAnalysisResult): ReportPresentation {
    return {
      reportType: this.reportType,
      pluginId: this.pluginId,
      title: 'Worst Files',
      description:
        'Files that need immediate attention, prioritized by critical issues, low scores, and high complexity. Focus your refactoring efforts on these files to maximize code quality improvements.',
      sections: [
        this.createRecommendationsByEffort(data, 'low'),
        this.createRecommendationsByEffort(data, 'medium'),
        this.createRecommendationsByEffort(data, 'high'),
      ].filter(section => section.items && section.items.length > 0),
    };
  }

  private createRecommendationsByEffort(
    data: BatchAnalysisResult,
    effort: 'low' | 'medium' | 'high'
  ): PresentationSection {
    const filtered = data.recommendations.filter(rec => rec.estimatedEffort === effort);

    const items: PresentationItem[] = filtered.map(rec => ({
      id: `rec-${rec.filePath}`,
      label: this.formatFilePath(rec.filePath),
      value: rec.score,
      context: this.formatRecommendationContext(rec),
      severity: rec.criticalIssues > 0 ? 'critical' : rec.score < 50 ? 'warning' : undefined,
      metadata: {
        filePath: rec.filePath,
        score: rec.score,
        criticalIssues: rec.criticalIssues,
        totalIssues: rec.totalIssues,
        primaryReason: rec.primaryReason,
        secondaryReasons: rec.secondaryReasons,
        estimatedEffort: rec.estimatedEffort,
      },
    }));

    return {
      id: `recommendations-${effort}`,
      title: `${this.formatEffort(effort)} Effort`,
      items,
    };
  }

  private formatRecommendationContext(rec: FileRecommendation): string {
    const parts: string[] = [rec.primaryReason];

    if (rec.secondaryReasons.length > 0) {
      parts.push(rec.secondaryReasons[0]); // Show first secondary reason
    }

    return parts.join(' • ');
  }

  private formatEffort(effort: string): string {
    return effort.charAt(0).toUpperCase() + effort.slice(1);
  }

  private formatFilePath(path: string): string {
    const parts = path.split('/');
    return parts.length > 3 ? `.../${parts.slice(-3).join('/')}` : path;
  }
}
```

## Registration

Update `analyzers/core/src/presenters/index.ts`:

```typescript
/**
 * Core analyzer presenters
 *
 * @module @vipr/core/presenters
 */

export { CoreOverviewPresenter } from './core-overview-presenter';
export { ProjectSummaryPresenter } from './project-summary-presenter';
export { BatchIssuesPresenter } from './batch-issues-presenter';
export { BatchComplexityPresenter } from './batch-complexity-presenter';
export { WorstFilesPresenter } from './worst-files-presenter';
```

Update `analyzers/core/src/plugin.ts` to register batch presenters:

```typescript
import {
  CoreOverviewPresenter,
  ProjectSummaryPresenter,
  BatchIssuesPresenter,
  BatchComplexityPresenter,
  WorstFilesPresenter,
} from './presenters';

export class CoreAnalyzerPlugin implements ITechnologyPlugin {
  // ... existing code ...

  getReportPresenters(): IReportPresenter[] {
    return [
      new CoreOverviewPresenter(),
      new ProjectSummaryPresenter(),
      new BatchIssuesPresenter(),
      new BatchComplexityPresenter(),
      new WorstFilesPresenter(),
    ];
  }
}
```

## Testing Strategy

### Unit Tests

Create `analyzers/core/src/presenters/batch-presenters.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { ProjectSummaryPresenter } from './project-summary-presenter';
import { BatchIssuesPresenter } from './batch-issues-presenter';
import type { BatchAnalysisResult } from '@vipr/common';

describe('Batch Presenters', () => {
  const mockBatchResult: BatchAnalysisResult = {
    type: 'batch',
    analyzedAt: '2024-01-01T00:00:00Z',
    rootPath: '/test',
    files: [],
    summary: {
      totalFiles: 100,
      analyzedFiles: 95,
      skippedFiles: 5,
      averageScore: 75,
      minScore: 20,
      maxScore: 98,
      executionTimeMs: 5000,
    },
    technologies: [
      {
        technology: 'react',
        fileCount: 50,
        averageScore: 80,
        minScore: 40,
        maxScore: 95,
        files: [],
      },
    ],
    scoreDistribution: [],
    issues: {
      totalIssues: 200,
      byCategory: [],
      severityCounts: { critical: 10, warning: 50, info: 140 },
      mostAffectedFiles: [],
    },
    recommendations: [],
  };

  describe('ProjectSummaryPresenter', () => {
    const presenter = new ProjectSummaryPresenter();

    it('should accept batch results', () => {
      expect(presenter.canPresent(mockBatchResult)).toBe(true);
    });

    it('should have batch category in metadata', () => {
      const metadata = presenter.getMetadata();
      expect(metadata.categories).toContain('batch');
    });

    it('should generate presentation with sections', () => {
      const presentation = presenter.present(mockBatchResult);
      expect(presentation.sections.length).toBeGreaterThan(0);
      expect(presentation.sections[0].id).toBe('overall-health');
    });
  });

  describe('BatchIssuesPresenter', () => {
    const presenter = new BatchIssuesPresenter();

    it('should format severity breakdown', () => {
      const presentation = presenter.present(mockBatchResult);
      const severitySection = presentation.sections.find(s => s.id === 'severity-summary');
      expect(severitySection).toBeDefined();
      expect(severitySection?.metrics?.length).toBe(3); // critical, warning, info
    });
  });
});
```

## Integration with Formatters

Formatters detect batch presenters via `categories`:

```typescript
// In CLI formatter
const availableReports = registry.getAvailableReports();
const batchReports = availableReports.filter(r => r.categories?.includes('batch'));

if (isBatchResult(result)) {
  // Use batch reports only
  for (const report of batchReports) {
    const presenter = registry.get(report.pluginId, report.reportType);
    if (presenter?.canPresent(result)) {
      const presentation = presenter.present(result);
      this.renderPresentation(presentation);
    }
  }
}
```

## Next Steps

1. **Implement all four presenters**
2. **Write comprehensive unit tests**
3. **Register presenters in `CoreAnalyzerPlugin`**
4. **Test with real batch results**
5. **Proceed to Phase 4**: CLI formatters

## Success Criteria

- [ ] All presenters implement `IReportPresenter<BatchAnalysisResult>`
- [ ] Metadata includes `categories: ['batch']`
- [ ] Presenters generate valid `ReportPresentation` objects
- [ ] Full test coverage for all presenters
- [ ] Auto-discovered via `getReportPresenters()`
- [ ] Formatters can filter batch vs single-file presenters

## Files Created

- `analyzers/core/src/presenters/project-summary-presenter.ts`
- `analyzers/core/src/presenters/batch-issues-presenter.ts`
- `analyzers/core/src/presenters/batch-complexity-presenter.ts`
- `analyzers/core/src/presenters/worst-files-presenter.ts`
- `analyzers/core/src/presenters/batch-presenters.test.ts`

## Files Modified

- `analyzers/core/src/presenters/index.ts` - Export new presenters
- `analyzers/core/src/plugin.ts` - Register new presenters in `getReportPresenters()`
