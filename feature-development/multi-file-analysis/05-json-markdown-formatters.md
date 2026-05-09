---
id: 05-json-markdown-formatters
---

# Phase 5: JSON and Markdown Formatters

## Overview

This phase adds batch support to JSON and Markdown formatters, enabling serialization and documentation generation for multi-file analysis results.

## Design Goals

1. **Schema Consistency**: Match existing JSON output structure
2. **Parseable Output**: Enable tooling integration
3. **Human-Readable Markdown**: Professional documentation format
4. **Backward Compatibility**: Single-file formats unchanged
5. **Metadata Inclusion**: Facilitate filtering and parsing

## JSON Formatter

### Batch JSON Schema

The batch JSON output extends existing output envelope types:

```typescript
/**
 * Batch JSON output schema.
 * Extends FullBatchOutput with batch-specific metadata.
 */
export interface BatchJsonOutput extends OutputEnvelope<'full', BatchJsonData> {
  $schema: '1.0.0';
  format: 'full';
  timestamp: string;
  analyzerVersion: string;
  data: BatchJsonData;
}

/**
 * Batch data payload for JSON serialization.
 */
export interface BatchJsonData {
  /** Batch analysis summary */
  summary: {
    rootPath: string;
    analyzedAt: string;
    totalFiles: number;
    analyzedFiles: number;
    skippedFiles: number;
    averageScore: number;
    minScore: number;
    maxScore: number;
    executionTimeMs: number;
  };

  /** Technology breakdown */
  technologies: Array<{
    technology: string;
    fileCount: number;
    averageScore: number;
    minScore: number;
    maxScore: number;
    files: string[];
  }>;

  /** Score distribution */
  scoreDistribution: Array<{
    label: string;
    min: number;
    max: number;
    count: number;
    percentage: number;
    files: string[];
  }>;

  /** Issue statistics */
  issues: {
    totalIssues: number;
    severityCounts: {
      critical: number;
      warning: number;
      info: number;
    };
    byCategory: Array<{
      category: string;
      count: number;
      fileCount: number;
      severityCounts: {
        critical: number;
        warning: number;
        info: number;
      };
      affectedFiles: Array<{
        filePath: string;
        count: number;
      }>;
    }>;
    mostAffectedFiles: Array<{
      filePath: string;
      issueCount: number;
      criticalCount: number;
    }>;
  };

  /** Complexity distribution (optional) */
  complexity?: {
    cyclomatic?: MetricDistributionJson;
    halsteadVolume?: MetricDistributionJson;
    maintainability?: MetricDistributionJson;
    mostComplexFiles: Array<{
      filePath: string;
      score: number;
      primaryComplexity: {
        metric: string;
        value: number;
      };
    }>;
  };

  /** Recommendations */
  recommendations: Array<{
    filePath: string;
    score: number;
    criticalIssues: number;
    totalIssues: number;
    primaryReason: string;
    secondaryReasons: string[];
    estimatedEffort: 'low' | 'medium' | 'high';
  }>;

  /** Individual file results (optional, can be large) */
  files?: Array<FullFileResult>;
}

interface MetricDistributionJson {
  metricName: string;
  min: number;
  max: number;
  mean: number;
  median: number;
  p75: number;
  p90: number;
  p95: number;
  stdDev: number;
  histogram: Array<{
    range: string;
    count: number;
  }>;
}
```

### JSON Formatter Implementation

Update `clients/cli/src/formatters/full-json-formatter.ts`:

```typescript
/**
 * Full JSON formatter with batch support
 */
export class FullJsonFormatter implements IFormatter {
  readonly id = 'json-full';
  readonly name = 'Full JSON Output';
  readonly extension = 'json';

  private includeFiles: boolean;

  constructor(options?: { includeFiles?: boolean }) {
    this.includeFiles = options?.includeFiles ?? false;
  }

  format(result: AggregatedResult | BatchAnalysisResult): string {
    if (isBatchResult(result)) {
      return this.formatBatch(result);
    }
    return this.formatSingle(result);
  }

  private formatSingle(result: AggregatedResult): string {
    // Existing single-file logic
    const output: FullSingleOutput = {
      $schema: '1.0.0',
      format: 'full',
      timestamp: new Date().toISOString(),
      analyzerVersion: getVersion(),
      data: this.serializeSingleResult(result),
    };

    return JSON.stringify(output, null, 2);
  }

  private formatBatch(result: BatchAnalysisResult): string {
    const output: BatchJsonOutput = {
      $schema: '1.0.0',
      format: 'full',
      timestamp: new Date().toISOString(),
      analyzerVersion: getVersion(),
      data: this.serializeBatchResult(result),
    };

    return JSON.stringify(output, null, 2);
  }

  private serializeBatchResult(result: BatchAnalysisResult): BatchJsonData {
    return {
      summary: {
        rootPath: result.rootPath,
        analyzedAt: result.analyzedAt,
        totalFiles: result.summary.totalFiles,
        analyzedFiles: result.summary.analyzedFiles,
        skippedFiles: result.summary.skippedFiles,
        averageScore: result.summary.averageScore,
        minScore: result.summary.minScore,
        maxScore: result.summary.maxScore,
        executionTimeMs: result.summary.executionTimeMs,
      },

      technologies: result.technologies.map(tech => ({
        technology: tech.technology,
        fileCount: tech.fileCount,
        averageScore: tech.averageScore,
        minScore: tech.minScore,
        maxScore: tech.maxScore,
        files: tech.files,
      })),

      scoreDistribution: result.scoreDistribution.map(bucket => ({
        label: bucket.label,
        min: bucket.min,
        max: bucket.max,
        count: bucket.count,
        percentage: bucket.percentage,
        files: bucket.files,
      })),

      issues: {
        totalIssues: result.issues.totalIssues,
        severityCounts: result.issues.severityCounts,
        byCategory: result.issues.byCategory,
        mostAffectedFiles: result.issues.mostAffectedFiles,
      },

      complexity: result.complexity ? this.serializeComplexity(result.complexity) : undefined,

      recommendations: result.recommendations,

      files: this.includeFiles ? result.files.map(f => this.serializeSingleResult(f)) : undefined,
    };
  }

  private serializeComplexity(complexity: ComplexityDistribution): any {
    return {
      cyclomatic: complexity.cyclomatic,
      halsteadVolume: complexity.halsteadVolume,
      maintainability: complexity.maintainability,
      mostComplexFiles: complexity.mostComplexFiles,
    };
  }

  private serializeSingleResult(result: AggregatedResult): FullFileResult {
    // Existing serialization logic
  }
}
```

### JSON Output Examples

#### Batch Summary (Default)

```json
{
  "$schema": "1.0.0",
  "format": "full",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "analyzerVersion": "0.8.0",
  "data": {
    "summary": {
      "rootPath": "/project/src",
      "analyzedAt": "2024-01-15T10:30:00.000Z",
      "totalFiles": 127,
      "analyzedFiles": 127,
      "skippedFiles": 0,
      "averageScore": 75.3,
      "minScore": 35,
      "maxScore": 98,
      "executionTimeMs": 5234
    },
    "technologies": [
      {
        "technology": "react",
        "fileCount": 65,
        "averageScore": 78.2,
        "minScore": 42,
        "maxScore": 96,
        "files": ["src/components/Button.tsx", "..."]
      }
    ],
    "scoreDistribution": [
      {
        "label": "0-20 (Critical)",
        "min": 0,
        "max": 20,
        "count": 5,
        "percentage": 3.9,
        "files": ["src/legacy/old-api.ts"]
      }
    ],
    "issues": {
      "totalIssues": 342,
      "severityCounts": {
        "critical": 28,
        "warning": 134,
        "info": 180
      },
      "byCategory": [
        {
          "category": "complexity",
          "count": 89,
          "fileCount": 45,
          "severityCounts": {
            "critical": 12,
            "warning": 45,
            "info": 32
          },
          "affectedFiles": [
            {
              "filePath": "src/utils/helpers.ts",
              "count": 8
            }
          ]
        }
      ],
      "mostAffectedFiles": [
        {
          "filePath": "src/legacy/old-api.ts",
          "issueCount": 28,
          "criticalCount": 15
        }
      ]
    },
    "recommendations": [
      {
        "filePath": "src/legacy/old-api.ts",
        "score": 22,
        "criticalIssues": 15,
        "totalIssues": 28,
        "primaryReason": "Very low score (22/100)",
        "secondaryReasons": ["28 total issues", "High cyclomatic complexity (52)"],
        "estimatedEffort": "high"
      }
    ]
  }
}
```

#### With Individual Files (`--include-files`)

Add `files` array to the output:

```json
{
  "data": {
    "summary": {
      /* ... */
    },
    "files": [
      {
        "filePath": "src/components/Button.tsx",
        "analyzedAt": "2024-01-15T10:30:00.000Z",
        "score": 85,
        "insights": [
          /* ... */
        ],
        "plugins": {
          /* ... */
        },
        "errors": []
      }
    ]
  }
}
```

## Markdown Formatter

### Batch Markdown Template

Update `clients/cli/src/formatters/markdown-formatter.ts`:

```typescript
export class MarkdownFormatter implements IFormatter {
  readonly id = 'markdown';
  readonly name = 'Markdown Report';
  readonly extension = 'md';

  private includeFiles: boolean;

  constructor(options?: { includeFiles?: boolean }) {
    this.includeFiles = options?.includeFiles ?? false;
  }

  format(result: AggregatedResult | BatchAnalysisResult): string {
    if (isBatchResult(result)) {
      return this.formatBatch(result);
    }
    return this.formatSingle(result);
  }

  private formatBatch(result: BatchAnalysisResult): string {
    const sections: string[] = [];

    // Header
    sections.push(this.formatBatchHeader(result));

    // Summary
    sections.push(this.formatSummarySection(result));

    // Technology Breakdown
    sections.push(this.formatTechnologySection(result));

    // Score Distribution
    sections.push(this.formatDistributionSection(result));

    // Issues
    sections.push(this.formatIssuesSection(result));

    // Complexity
    if (result.complexity) {
      sections.push(this.formatComplexitySection(result));
    }

    // Recommendations
    sections.push(this.formatRecommendationsSection(result));

    // Individual files (optional)
    if (this.includeFiles) {
      sections.push(this.formatFilesSection(result));
    }

    return sections.join('\n\n');
  }

  private formatBatchHeader(result: BatchAnalysisResult): string {
    return [
      '# Code Analysis Report',
      '',
      `**Generated**: ${new Date(result.analyzedAt).toLocaleString()}`,
      `**Directory**: \`${result.rootPath}\``,
      `**Files Analyzed**: ${result.summary.analyzedFiles} / ${result.summary.totalFiles}`,
      `**Average Score**: ${Math.round(result.summary.averageScore)}/100`,
      '',
      '---',
    ].join('\n');
  }

  private formatSummarySection(result: BatchAnalysisResult): string {
    const { summary } = result;
    return [
      '## Summary',
      '',
      '| Metric | Value |',
      '|--------|-------|',
      `| Total Files | ${summary.totalFiles} |`,
      `| Analyzed | ${summary.analyzedFiles} |`,
      `| Skipped | ${summary.skippedFiles} |`,
      `| Average Score | ${Math.round(summary.averageScore)}/100 |`,
      `| Score Range | ${summary.minScore} - ${summary.maxScore} |`,
      `| Execution Time | ${(summary.executionTimeMs / 1000).toFixed(2)}s |`,
    ].join('\n');
  }

  private formatTechnologySection(result: BatchAnalysisResult): string {
    const lines = [
      '## Technology Breakdown',
      '',
      '| Technology | Files | Avg Score | Score Range |',
      '|------------|-------|-----------|-------------|',
    ];

    for (const tech of result.technologies) {
      lines.push(
        `| ${this.formatTechName(tech.technology)} | ${tech.fileCount} | ${Math.round(tech.averageScore)}/100 | ${tech.minScore}-${tech.maxScore} |`
      );
    }

    return lines.join('\n');
  }

  private formatDistributionSection(result: BatchAnalysisResult): string {
    const lines = [
      '## Score Distribution',
      '',
      '| Score Range | Files | Percentage |',
      '|-------------|-------|------------|',
    ];

    for (const bucket of result.scoreDistribution) {
      const bar = this.createTextBar(bucket.percentage, 20);
      lines.push(`| ${bucket.label} | ${bucket.count} | ${bucket.percentage.toFixed(1)}% ${bar} |`);
    }

    return lines.join('\n');
  }

  private formatIssuesSection(result: BatchAnalysisResult): string {
    const { issues } = result;
    const lines = [
      '## Issues Summary',
      '',
      '### Severity Distribution',
      '',
      '| Severity | Count | Percentage |',
      '|----------|-------|------------|',
    ];

    const total = issues.totalIssues;
    lines.push(
      `| Critical | ${issues.severityCounts.critical} | ${((issues.severityCounts.critical / total) * 100).toFixed(1)}% |`,
      `| Warning | ${issues.severityCounts.warning} | ${((issues.severityCounts.warning / total) * 100).toFixed(1)}% |`,
      `| Info | ${issues.severityCounts.info} | ${((issues.severityCounts.info / total) * 100).toFixed(1)}% |`
    );

    lines.push('', '### Issues by Category', '');
    lines.push('| Category | Count | Files Affected |');
    lines.push('|----------|-------|----------------|');

    for (const cat of issues.byCategory.slice(0, 10)) {
      lines.push(`| ${this.formatCategory(cat.category)} | ${cat.count} | ${cat.fileCount} |`);
    }

    lines.push('', '### Most Affected Files', '');
    lines.push('| File | Total Issues | Critical |');
    lines.push('|------|--------------|----------|');

    for (const file of issues.mostAffectedFiles.slice(0, 20)) {
      lines.push(`| \`${file.filePath}\` | ${file.issueCount} | ${file.criticalCount} |`);
    }

    return lines.join('\n');
  }

  private formatComplexitySection(result: BatchAnalysisResult): string {
    if (!result.complexity) return '';

    const lines = ['## Complexity Analysis', ''];

    // Cyclomatic complexity
    if (result.complexity.cyclomatic) {
      const dist = result.complexity.cyclomatic;
      lines.push(
        '### Cyclomatic Complexity',
        '',
        '| Metric | Value |',
        '|--------|-------|',
        `| Mean | ${dist.mean.toFixed(2)} |`,
        `| Median | ${dist.median.toFixed(2)} |`,
        `| 90th Percentile | ${dist.p90.toFixed(2)} |`,
        `| Max | ${dist.max.toFixed(2)} |`,
        ''
      );
    }

    // Most complex files
    lines.push('### Most Complex Files', '');
    lines.push('| File | Score | Complexity Metric | Value |');
    lines.push('|------|-------|-------------------|-------|');

    for (const file of result.complexity.mostComplexFiles.slice(0, 20)) {
      lines.push(
        `| \`${file.filePath}\` | ${file.score}/100 | ${file.primaryComplexity.metric} | ${file.primaryComplexity.value.toFixed(2)} |`
      );
    }

    return lines.join('\n');
  }

  private formatRecommendationsSection(result: BatchAnalysisResult): string {
    const lines = ['## Recommended Actions', ''];

    const grouped = {
      high: result.recommendations.filter(r => r.estimatedEffort === 'high'),
      medium: result.recommendations.filter(r => r.estimatedEffort === 'medium'),
      low: result.recommendations.filter(r => r.estimatedEffort === 'low'),
    };

    for (const effort of ['high', 'medium', 'low'] as const) {
      const items = grouped[effort];
      if (items.length === 0) continue;

      lines.push(`### ${this.capitalize(effort)} Effort (${items.length} files)`, '');

      for (const rec of items.slice(0, 10)) {
        lines.push(`#### \`${rec.filePath}\``);
        lines.push('');
        lines.push(`- **Score**: ${rec.score}/100`);
        lines.push(`- **Critical Issues**: ${rec.criticalIssues}`);
        lines.push(`- **Total Issues**: ${rec.totalIssues}`);
        lines.push(`- **Primary Reason**: ${rec.primaryReason}`);
        if (rec.secondaryReasons.length > 0) {
          lines.push(`- **Additional Issues**: ${rec.secondaryReasons.join(', ')}`);
        }
        lines.push('');
      }
    }

    return lines.join('\n');
  }

  private formatFilesSection(result: BatchAnalysisResult): string {
    const lines = [
      '## Individual File Results',
      '',
      '<details>',
      '<summary>Click to expand file-by-file breakdown</summary>',
      '',
    ];

    for (const file of result.files) {
      lines.push(`### \`${file.filePath}\``);
      lines.push('');
      lines.push(`**Score**: ${file.overallScore}/100`);
      lines.push(
        `**Issues**: ${file.insights.length} (${file.insights.filter(i => i.severity === 'critical').length} critical)`
      );
      lines.push('');
    }

    lines.push('</details>');

    return lines.join('\n');
  }

  private createTextBar(percentage: number, width: number): string {
    const filled = Math.round((percentage / 100) * width);
    return '█'.repeat(filled) + '░'.repeat(width - filled);
  }

  private formatTechName(tech: string): string {
    const labels: Record<string, string> = {
      react: 'React',
      typescript: 'TypeScript',
      javascript: 'JavaScript',
      nextjs: 'Next.js',
    };
    return labels[tech] ?? tech;
  }

  private formatCategory(category: string): string {
    return category
      .split('-')
      .map(w => w.charAt(0).toUpperCase() + w.slice(1))
      .join(' ');
  }

  private capitalize(str: string): string {
    return str.charAt(0).toUpperCase() + str.slice(1);
  }
}
```

### Markdown Output Example

```markdown
# Code Analysis Report

**Generated**: 1/15/2024, 10:30:00 AM
**Directory**: `/project/src`
**Files Analyzed**: 127 / 127
**Average Score**: 75/100

---

## Summary

| Metric         | Value   |
| -------------- | ------- |
| Total Files    | 127     |
| Analyzed       | 127     |
| Skipped        | 0       |
| Average Score  | 75/100  |
| Score Range    | 35 - 98 |
| Execution Time | 5.23s   |

## Technology Breakdown

| Technology | Files | Avg Score | Score Range |
| ---------- | ----- | --------- | ----------- |
| React      | 65    | 78/100    | 42-96       |
| TypeScript | 42    | 72/100    | 35-95       |
| JavaScript | 20    | 68/100    | 40-88       |

## Score Distribution

| Score Range        | Files | Percentage                 |
| ------------------ | ----- | -------------------------- |
| 0-20 (Critical)    | 5     | 3.9% ███░░░░░░░░░░░░░░░░░  |
| 21-40 (Poor)       | 12    | 9.4% ████████░░░░░░░░░░░░  |
| 41-60 (Fair)       | 28    | 22.0% ████████████████░░░░ |
| 61-80 (Good)       | 45    | 35.4% ███████████████████░ |
| 81-100 (Excellent) | 37    | 29.1% ██████████████████░░ |

## Issues Summary

### Severity Distribution

| Severity | Count | Percentage |
| -------- | ----- | ---------- |
| Critical | 28    | 8.2%       |
| Warning  | 134   | 39.2%      |
| Info     | 180   | 52.6%      |

### Most Affected Files

| File                            | Total Issues | Critical |
| ------------------------------- | ------------ | -------- |
| `src/legacy/old-api.ts`         | 28           | 15       |
| `src/components/LegacyForm.tsx` | 22           | 8        |

## Recommended Actions

### High Effort (3 files)

#### `src/legacy/old-api.ts`

- **Score**: 22/100
- **Critical Issues**: 15
- **Total Issues**: 28
- **Primary Reason**: Very low score (22/100)
- **Additional Issues**: 28 total issues, High cyclomatic complexity (52)
```

## CLI Integration

Update command to support options:

```bash
# JSON with file details
vipr analyze src/ --format json --include-files -o report.json

# Markdown report
vipr analyze src/ --format markdown -o ANALYSIS.md

# Markdown with individual files
vipr analyze src/ --format markdown --include-files -o FULL_REPORT.md
```

## Testing Strategy

### Unit Tests

```typescript
describe('JSON Formatter - Batch Mode', () => {
  it('should serialize batch results to JSON', () => {
    const formatter = new FullJsonFormatter();
    const json = formatter.format(mockBatchResult);
    const parsed = JSON.parse(json);

    expect(parsed.format).toBe('full');
    expect(parsed.data.summary).toBeDefined();
    expect(parsed.data.technologies).toBeInstanceOf(Array);
  });

  it('should include files when requested', () => {
    const formatter = new FullJsonFormatter({ includeFiles: true });
    const json = formatter.format(mockBatchResult);
    const parsed = JSON.parse(json);

    expect(parsed.data.files).toBeDefined();
  });
});

describe('Markdown Formatter - Batch Mode', () => {
  it('should generate markdown report', () => {
    const formatter = new MarkdownFormatter();
    const markdown = formatter.format(mockBatchResult);

    expect(markdown).toContain('# Code Analysis Report');
    expect(markdown).toContain('## Summary');
    expect(markdown).toContain('## Technology Breakdown');
  });
});
```

## Next Steps

1. **Implement JSON batch serialization**
2. **Create Markdown template**
3. **Add `--include-files` option**
4. **Write comprehensive tests**
5. **Update documentation**
6. **Proceed to Phase 6**: Interactive mode

## Success Criteria

- [ ] JSON output is valid and parseable
- [ ] JSON schema matches specification
- [ ] Markdown is well-formatted and readable
- [ ] `--include-files` option works
- [ ] Both formatters detect batch results automatically
- [ ] Full test coverage
- [ ] Documentation examples provided

## Files Modified

- `clients/cli/src/formatters/full-json-formatter.ts` - Add batch support
- `clients/cli/src/formatters/markdown-formatter.ts` - Add batch template
- `packages/common/src/types/output/batch.ts` - Export JSON types

## Files Created

- `clients/cli/src/formatters/json-batch.test.ts` - JSON batch tests
- `clients/cli/src/formatters/markdown-batch.test.ts` - Markdown batch tests
