---
id: 04-cli-formatters
---

# Phase 4: CLI Formatters

## Overview

This phase extends CLI formatters to support batch analysis results with progress indicators, summary rendering, and new CLI options for filtering and pagination.

## Design Principles

1. **Type Detection**: Auto-detect batch vs single-file results
2. **Progressive Disclosure**: Show summaries by default, full details on demand
3. **Reuse Presenters**: Leverage batch presenters from Phase 3
4. **Consistent UX**: Match existing CLI formatter patterns
5. **Backward Compatibility**: Single-file analysis unchanged

## Formatter Detection

### Type Guard Usage

```typescript
import { isBatchResult, type BatchAnalysisResult, type AggregatedResult } from '@vipr/common';

export class CliFormatter extends BaseCliFormatter implements IFormatter {
  format(result: AggregatedResult | BatchAnalysisResult): string {
    if (isBatchResult(result)) {
      return this.formatBatch(result);
    }
    return this.formatSingle(result);
  }

  private formatSingle(result: AggregatedResult): string {
    // Existing single-file logic (unchanged)
  }

  private formatBatch(result: BatchAnalysisResult): string {
    // New batch formatting logic
  }
}
```

## Progress Indicator

### Implementation

Add progress tracking during analysis in `clients/cli/src/commands/analyze-command.ts`:

```typescript
import ora from 'ora';
import type { ProgressCallback } from '@vipr/engine';

async function analyzeDirectory(dirPath: string, options: BatchAnalysisOptions) {
  const spinner = ora('Discovering files...').start();

  const progressCallback: ProgressCallback = {
    onDiscovery: totalFiles => {
      spinner.text = `Found ${totalFiles} files`;
    },

    onFileStart: (filePath, current, total) => {
      spinner.text = `Analyzing ${current}/${total}: ${formatFilePath(filePath)}`;
    },

    onFileComplete: (filePath, result, current, total) => {
      // Update spinner with current progress
      const percentage = Math.round((current / total) * 100);
      spinner.text = `Analyzed ${current}/${total} (${percentage}%) • Last: ${formatFilePath(filePath)}`;
    },

    onFileError: (filePath, error) => {
      spinner.warn(`Failed to analyze ${filePath}: ${error.message}`);
    },

    onComplete: result => {
      spinner.succeed(
        `Analysis complete: ${result.summary.analyzedFiles}/${result.summary.totalFiles} files`
      );
    },
  };

  return engine.analyzeDirectory(dirPath, {
    ...options,
    onProgress: progressCallback,
  });
}

function formatFilePath(path: string): string {
  const parts = path.split('/');
  return parts.length > 3 ? `.../${parts.slice(-3).join('/')}` : path;
}
```

### Progress Output Example

```
⠋ Discovering files...
✔ Found 127 files
⠙ Analyzing 1/127: src/components/Button.tsx
⠹ Analyzing 42/127 (33%) • Last: ...utils/helpers.ts
⠸ Analyzing 127/127 (100%) • Last: ...pages/index.tsx
✔ Analysis complete: 127/127 files
```

## Batch Formatting

### Header Section

```typescript
private formatBatchHeader(result: BatchAnalysisResult): string {
  const { summary } = result;
  const sections: string[] = [];

  // Title box
  sections.push(this.printBoxHeader());
  sections.push(this.printCentered('Project Analysis Summary'));
  sections.push(this.printCentered(this.dim(`${summary.analyzedFiles} files analyzed`)));
  sections.push(this.printBoxDivider());

  // Overall score
  const scoreBar = this.formatScoreBar({
    label: 'Overall Health',
    score: Math.round(summary.averageScore),
    maxScore: 100,
  });
  sections.push(scoreBar);

  // Quick stats
  sections.push('');
  sections.push(this.printTwoColumns(
    `${this.bold('Total Files:')} ${summary.totalFiles}`,
    `${this.bold('Analyzed:')} ${summary.analyzedFiles}`
  ));
  sections.push(this.printTwoColumns(
    `${this.bold('Avg Score:')} ${Math.round(summary.averageScore)}/100`,
    `${this.bold('Range:')} ${summary.minScore}-${summary.maxScore}`
  ));
  sections.push(this.printTwoColumns(
    `${this.bold('Time:')} ${(summary.executionTimeMs / 1000).toFixed(2)}s`,
    `${this.bold('Skipped:')} ${summary.skippedFiles}`
  ));

  return sections.join('\n');
}
```

### Batch Report Sections

```typescript
private formatBatch(result: BatchAnalysisResult): string {
  const sections: string[] = [];

  // Header
  sections.push(this.formatBatchHeader(result));

  // Get batch presenters
  const batchReports = this.registry
    .getAvailableReports()
    .filter(r => r.categories?.includes('batch'));

  // Render each batch report
  for (const reportMetadata of batchReports) {
    const presenter = this.registry.get(reportMetadata.pluginId, reportMetadata.reportType);

    if (presenter?.canPresent(result)) {
      const presentation = presenter.present(result);

      if (this.presentationHasContent(presentation)) {
        sections.push('');
        sections.push(this.formatBatchReportSection(presentation));
      }
    }
  }

  // Footer
  sections.push(this.printBoxFooter());

  return sections.join('\n');
}

private formatBatchReportSection(presentation: ReportPresentation): string {
  const sections: string[] = [];

  // Section header
  sections.push(this.printSectionHeader(presentation.title));
  if (presentation.description) {
    sections.push(this.dim(presentation.description));
  }
  sections.push('');

  // Render presentation sections
  for (const section of presentation.sections) {
    sections.push(this.renderer.renderSection(section));
  }

  return sections.join('\n');
}
```

## CLI Options

### New Options

Add to `clients/cli/src/commands/analyze-command.ts`:

```typescript
interface AnalyzeCommandOptions {
  // Existing options
  format?: 'cli' | 'json' | 'markdown';
  output?: string;
  reportType?: string;

  // New batch options
  limit?: number; // Items per section (default: 20)
  page?: number; // Page number for pagination (default: 1)
  belowThreshold?: number; // Show only files below this score
  showAll?: boolean; // Show all files, not just summaries
  minSeverity?: 'critical' | 'warning' | 'info'; // Minimum severity
}
```

### Command Implementation

```typescript
program
  .command('analyze <path>')
  .description('Analyze a file or directory')
  .option('-f, --format <format>', 'Output format (cli, json, markdown)', 'cli')
  .option('-o, --output <file>', 'Output file path')
  .option('-r, --report-type <type>', 'Specific report type to show')
  .option('--limit <number>', 'Items per section (batch mode)', '20')
  .option('--page <number>', 'Page number for pagination', '1')
  .option('--below-threshold <score>', 'Show only files below this score')
  .option('--show-all', 'Show all files, not just summaries')
  .option('--min-severity <level>', 'Minimum severity (critical, warning, info)')
  .action(async (path: string, options: AnalyzeCommandOptions) => {
    const stat = await fs.stat(path);

    if (stat.isDirectory()) {
      // Batch mode
      const batchOptions: BatchAnalysisOptions = {
        patterns: ['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx'],
        maxScore: options.belowThreshold,
        minSeverity: options.minSeverity as any,
        includeCleanFiles: !options.belowThreshold,
      };

      const result = await engine.analyzeDirectory(path, batchOptions);

      // Apply pagination to result if needed
      const paginatedResult = options.limit
        ? applyPagination(result, parseInt(options.limit), parseInt(options.page ?? '1'))
        : result;

      const formatter = createFormatter(options.format, {
        showAll: options.showAll,
      });

      const output = formatter.format(paginatedResult);
      if (options.output) {
        await fs.writeFile(options.output, output);
      } else {
        console.log(output);
      }
    } else {
      // Single file mode (existing logic)
      const result = await engine.analyzeFile(path);
      const formatter = createFormatter(options.format, { reportType: options.reportType });
      const output = formatter.format(result);
      console.log(output);
    }
  });
```

### Pagination Helper

```typescript
function applyPagination(
  result: BatchAnalysisResult,
  limit: number,
  page: number
): BatchAnalysisResult {
  // Calculate pagination for lists in the result
  const start = (page - 1) * limit;
  const end = start + limit;

  return {
    ...result,
    recommendations: result.recommendations.slice(start, end),
    issues: {
      ...result.issues,
      mostAffectedFiles: result.issues.mostAffectedFiles.slice(start, end),
      byCategory: result.issues.byCategory.map(cat => ({
        ...cat,
        affectedFiles: cat.affectedFiles.slice(0, limit), // Limit per category
      })),
    },
    complexity: result.complexity
      ? {
          ...result.complexity,
          mostComplexFiles: result.complexity.mostComplexFiles.slice(start, end),
        }
      : undefined,
  };
}
```

## Summary vs Detail Modes

### Default: Summary Mode

By default, show summaries and statistics:

```typescript
private formatBatch(result: BatchAnalysisResult): string {
  if (this.options.showAll) {
    return this.formatBatchDetailed(result);
  }
  return this.formatBatchSummary(result);
}

private formatBatchSummary(result: BatchAnalysisResult): string {
  // Show only aggregated statistics
  // - Overall health
  // - Technology breakdown
  // - Score distribution
  // - Top issues
  // - Worst files (limited)
}
```

### Detail Mode: `--show-all`

Show complete file lists and details:

```typescript
private formatBatchDetailed(result: BatchAnalysisResult): string {
  const sections: string[] = [];

  // Header
  sections.push(this.formatBatchHeader(result));

  // All batch reports
  sections.push(this.formatBatchReports(result));

  // File-by-file breakdown
  sections.push(this.printSectionHeader('Individual File Results'));
  for (const file of result.files) {
    sections.push(this.formatFileSummary(file));
  }

  sections.push(this.printBoxFooter());
  return sections.join('\n');
}

private formatFileSummary(file: AggregatedResult): string {
  return [
    `${this.bold(file.filePath)} - Score: ${file.overallScore}/100`,
    `  Issues: ${file.insights.length} (${file.insights.filter(i => i.severity === 'critical').length} critical)`,
  ].join('\n');
}
```

## Example Output

### Summary Mode (Default)

```
┌────────────────────────────────────────────────────────────┐
│                Project Analysis Summary                    │
│                   127 files analyzed                       │
├────────────────────────────────────────────────────────────┤
│ Overall Health                                    75/100   │
│ ████████████████████████████████████░░░░░░░░░░░░░          │
├────────────────────────────────────────────────────────────┤
│ Total Files: 127              Analyzed: 127                │
│ Avg Score: 75/100             Range: 35-98                 │
│ Time: 5.23s                   Skipped: 0                   │
└────────────────────────────────────────────────────────────┘

Project Summary
Overview of your project health...

┌────────────────────────────────────────────────────────────┐
│ Technology Breakdown                                       │
├────────────────────────────────────────────────────────────┤
│ React                         65 files   Avg: 78/100       │
│ TypeScript                    42 files   Avg: 72/100       │
│ JavaScript                    20 files   Avg: 68/100       │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ Score Distribution                                         │
├────────────────────────────────────────────────────────────┤
│ 0-20 (Critical)               5 files    3.9%              │
│ 21-40 (Poor)                  12 files   9.4%              │
│ 41-60 (Fair)                  28 files   22.0%             │
│ 61-80 (Good)                  45 files   35.4%             │
│ 81-100 (Excellent)            37 files   29.1%             │
└────────────────────────────────────────────────────────────┘

Worst Files
Files needing immediate attention...

Low Effort (8 files)
  src/utils/helpers.ts                           Score: 45/100
    Low score (45/100) • 12 total issues

Medium Effort (15 files)
  src/components/LegacyForm.tsx                  Score: 38/100
    15 critical issues • High cyclomatic complexity (45)

High Effort (3 files)
  src/legacy/old-api.ts                          Score: 22/100
    Very low score (22/100) • 28 critical issues
```

### Detail Mode (`--show-all`)

Same as above, plus:

```
Individual File Results

src/components/Button.tsx - Score: 85/100
  Issues: 2 (0 critical)

src/components/Form.tsx - Score: 72/100
  Issues: 8 (1 critical)

... (127 total files)
```

## Testing Strategy

### Unit Tests

Create `clients/cli/src/formatters/cli-formatter-batch.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { CliFormatter } from './cli-formatter';
import { PresenterRegistry } from '@vipr/common';
import type { BatchAnalysisResult } from '@vipr/common';

describe('CliFormatter - Batch Mode', () => {
  const registry = new PresenterRegistry();
  // Register batch presenters...

  const mockBatchResult: BatchAnalysisResult = {
    // ... mock data
  };

  it('should detect batch results', () => {
    const formatter = new CliFormatter(registry);
    const output = formatter.format(mockBatchResult);

    expect(output).toContain('Project Analysis Summary');
  });

  it('should render batch presenters', () => {
    const formatter = new CliFormatter(registry);
    const output = formatter.format(mockBatchResult);

    expect(output).toContain('Technology Breakdown');
    expect(output).toContain('Score Distribution');
  });

  it('should respect showAll option', () => {
    const formatter = new CliFormatter(registry, { showAll: true });
    const output = formatter.format(mockBatchResult);

    expect(output).toContain('Individual File Results');
  });
});
```

### Integration Tests

```typescript
describe('Analyze Command - Batch Mode', () => {
  it('should analyze directory with progress', async () => {
    const progressEvents: string[] = [];

    await analyzeDirectory('fixtures/test-project', {
      onProgress: {
        onDiscovery: () => progressEvents.push('discovery'),
        onComplete: () => progressEvents.push('complete'),
      },
    });

    expect(progressEvents).toContain('discovery');
    expect(progressEvents).toContain('complete');
  });

  it('should apply pagination', () => {
    const result = createMockBatchResult(100);
    const paginated = applyPagination(result, 20, 2);

    expect(paginated.recommendations.length).toBe(20);
  });
});
```

## Help Text Updates

Update command help to document batch features:

```typescript
program
  .command('analyze <path>')
  .description('Analyze a file or directory')
  .addHelpText(
    'after',
    `
Examples:
  Single file:
    $ vipr analyze src/components/Button.tsx
    $ vipr analyze src/App.tsx --report-type hooks

  Directory (batch mode):
    $ vipr analyze src/
    $ vipr analyze src/ --format json -o results.json
    $ vipr analyze src/ --below-threshold 50
    $ vipr analyze src/ --limit 20 --page 2
    $ vipr analyze src/ --show-all

Batch Options:
  --limit         Number of items per section (default: 20)
  --page          Page number for paginated results (default: 1)
  --below-threshold Only show files below this score
  --show-all      Show all files, not just summaries
  --min-severity  Minimum severity (critical, warning, info)
  `
  );
```

## Next Steps

1. **Implement batch detection** in CLI formatter
2. **Add progress indicator** to analyze command
3. **Create batch formatting methods**
4. **Add new CLI options**
5. **Write comprehensive tests**
6. **Update help text**
7. **Proceed to Phase 5**: JSON/Markdown formatters

## Success Criteria

- [ ] Auto-detects batch vs single-file results
- [ ] Progress indicator works during analysis
- [ ] Batch header shows summary statistics
- [ ] Batch presenters render correctly
- [ ] `--show-all` shows detailed file list
- [ ] Pagination works with `--limit` and `--page`
- [ ] Filtering works with `--below-threshold` and `--min-severity`
- [ ] Help text documents all new options
- [ ] Full test coverage for batch mode

## Files Modified

- `clients/cli/src/formatters/cli-formatter.ts` - Add batch detection and formatting
- `clients/cli/src/commands/analyze-command.ts` - Add batch options, progress
- `clients/cli/src/formatters/cli-formatter-batch.test.ts` - New batch formatter tests

## Files Created

- `clients/cli/src/utils/pagination.ts` - Pagination helper
- `clients/cli/src/utils/progress.ts` - Progress callback factory
