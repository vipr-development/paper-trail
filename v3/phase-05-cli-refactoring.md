# Phase 5: CLI Refactoring

## Overview

Refactor the CLI to remove direct dependencies on specific analyzers and integrate with the plugin loader system. Transform the CLI from a hardcoded React analyzer wrapper into a flexible, plugin-driven analysis orchestrator with a clean architecture supporting multiple output formats and extensible command structure.

## Objectives

1. Remove direct dependency on `@vipr/react` analyzer package from CLI
2. Integrate plugin loader for automatic analyzer discovery
3. Create a flexible formatter system for multiple output formats (CLI, JSON, SARIF)
4. Refactor CLI entry point for cleaner separation of concerns
5. Update package.json to reflect new architecture
6. Ensure zero breaking changes for existing CLI users
7. Enable future extensibility for new commands and analyzers
8. Improve error handling and user feedback

## Technical Scope

### Current Architecture Analysis

**Anti-Pattern: Hardcoded Dependencies**

```typescript
// clients/cli/package.json
{
  "dependencies": {
    "@vipr/react": "workspace:*",  // Direct coupling to React analyzer
    "@vipr/core": "workspace:*",
    "@vipr/types": "workspace:*"
  }
}
```

**Anti-Pattern: Missing Abstraction**

```typescript
// clients/cli/src/index.ts - Current implementation
import { ReactAnalyzerPlugin } from '@vipr/react'; // Direct import
import { AnalysisEngine } from '@vipr/core';

async function main() {
  const engine = new AnalysisEngine();
  engine.registerPlugin(new ReactAnalyzerPlugin()); // Hardcoded registration
  // ... analysis logic
}
```

**Anti-Pattern: Presentation Logic Mixed with Business Logic**

- No separation between analysis execution and result formatting
- Console output hardcoded in main function
- No abstraction for different output formats
- Color formatting mixed with data logic

**Anti-Pattern: God Function**

- Single `main()` function handles argument parsing, plugin setup, analysis, formatting, and error handling
- Violation of Single Responsibility Principle
- Difficult to test individual concerns

### Technical Debt

1. **Tight Coupling**: CLI cannot work without React analyzer
2. **Limited Extensibility**: Adding new analyzers requires code changes
3. **Poor Testability**: Cannot test formatting without running full analysis
4. **No Output Flexibility**: Only console output supported
5. **Missing Error Handling**: No structured error reporting
6. **No Configuration System**: Cannot customize analyzer behavior

## Refactoring Strategy

### Step 1: Create Plugin Loader Integration

Introduce plugin discovery abstraction to decouple from specific analyzers.

```typescript
// clients/cli/src/plugins/loader.ts

import { AnalysisEngine } from '@vipr/core';
import { ITechnologyPlugin } from '@vipr/types';
import { logger } from '@vipr/logging';

/**
 * Plugin loader for CLI
 *
 * Auto-discovers and loads analyzer plugins from the workspace.
 * In the bundled approach, plugins are statically imported but
 * registered dynamically.
 */
export class CliPluginLoader {
  private engine: AnalysisEngine;
  private loadedPlugins: Map<string, ITechnologyPlugin> = new Map();

  constructor(engine: AnalysisEngine) {
    this.engine = engine;
  }

  /**
   * Load all bundled analyzer plugins
   */
  async loadBundledPlugins(): Promise<void> {
    // Dynamically import bundled analyzers
    const plugins = await this.discoverBundledPlugins();

    for (const plugin of plugins) {
      this.registerPlugin(plugin);
    }

    logger.info(`Loaded ${this.loadedPlugins.size} analyzer plugins`);
  }

  /**
   * Discover plugins bundled with the CLI
   */
  private async discoverBundledPlugins(): Promise<ITechnologyPlugin[]> {
    const plugins: ITechnologyPlugin[] = [];

    // Import React analyzer
    try {
      const { ReactAnalyzerPlugin } = await import('@vipr/react');
      plugins.push(new ReactAnalyzerPlugin());
    } catch (error) {
      logger.warn('React analyzer not available');
    }

    // Import Core analyzer
    try {
      const { CoreAnalyzerPlugin } = await import('@vipr/core');
      plugins.push(new CoreAnalyzerPlugin());
    } catch (error) {
      logger.warn('Core analyzer not available');
    }

    // Future: Import other analyzers as they are created
    // const { VueAnalyzerPlugin } = await import('@vipr/vue');
    // plugins.push(new VueAnalyzerPlugin());

    return plugins;
  }

  /**
   * Register a plugin with the engine
   */
  private registerPlugin(plugin: ITechnologyPlugin): void {
    this.engine.registerPlugin(plugin);
    this.loadedPlugins.set(plugin.id, plugin);
    logger.debug(`Registered plugin: ${plugin.name} (${plugin.id})`);
  }

  /**
   * Get all loaded plugins
   */
  getLoadedPlugins(): ITechnologyPlugin[] {
    return Array.from(this.loadedPlugins.values());
  }

  /**
   * Get plugin by ID
   */
  getPlugin(id: string): ITechnologyPlugin | undefined {
    return this.loadedPlugins.get(id);
  }
}
```

### Step 2: Create Formatter System

Extract output formatting into pluggable formatters using the **Strategy Pattern**.

```typescript
// clients/cli/src/formatters/formatter.ts

import type { AggregatedResult } from '@vipr/types';

/**
 * Output formatter interface
 */
export interface IFormatter {
  /** Format identifier */
  readonly id: string;

  /** Format name */
  readonly name: string;

  /** File extension for this format */
  readonly extension: string;

  /**
   * Format analysis results
   */
  format(result: AggregatedResult): string;

  /**
   * Format multiple results
   */
  formatBatch?(results: AggregatedResult[]): string;
}
```

```typescript
// clients/cli/src/formatters/cli-formatter.ts

import type { AggregatedResult, PluginInsight } from '@vipr/types';
import { IFormatter } from './formatter';

const COLORS = {
  reset: '\x1b[0m',
  bright: '\x1b[1m',
  dim: '\x1b[2m',
  red: '\x1b[91m',
  green: '\x1b[92m',
  yellow: '\x1b[93m',
  blue: '\x1b[94m',
  cyan: '\x1b[96m',
};

/**
 * CLI console formatter with colors and formatting
 */
export class CliFormatter implements IFormatter {
  readonly id = 'cli';
  readonly name = 'CLI Console Output';
  readonly extension = 'txt';

  format(result: AggregatedResult): string {
    const lines: string[] = [];

    lines.push(this.formatHeader(result));
    lines.push(this.formatScore(result));
    lines.push(this.formatPluginResults(result));
    lines.push(this.formatInsights(result));

    return lines.join('\n');
  }

  private formatHeader(result: AggregatedResult): string {
    return `
${this.colorize('='.repeat(60), 'cyan')}
${this.colorize('Vipr Code Quality Report', 'bright')}
${this.colorize('='.repeat(60), 'cyan')}
File: ${result.filePath}
Analyzed: ${new Date(result.analyzedAt).toLocaleString()}
`;
  }

  private formatScore(result: AggregatedResult): string {
    const gradeColor = this.getGradeColor(result.overallGrade);
    return `
Overall Score: ${result.overallScore}/100
Grade: ${this.colorize(result.overallGrade, gradeColor)}
`;
  }

  private formatPluginResults(result: AggregatedResult): string {
    const lines: string[] = ['\nPlugin Results:'];

    for (const [pluginId, pluginResult] of result.pluginResults) {
      lines.push(`  ${pluginId}:`);
      lines.push(`    Score: ${pluginResult.score ?? 'N/A'}`);
      lines.push(`    Insights: ${pluginResult.insights.length}`);
      lines.push(`    Time: ${pluginResult.executionTimeMs}ms`);
    }

    return lines.join('\n');
  }

  private formatInsights(result: AggregatedResult): string {
    if (result.insights.length === 0) {
      return '\n' + this.colorize('No issues found!', 'green');
    }

    const lines: string[] = ['\nInsights:'];
    const grouped = this.groupInsightsBySeverity(result.insights);

    for (const [severity, insights] of Object.entries(grouped)) {
      if (insights.length === 0) continue;

      lines.push(
        `\n  ${this.formatSeverity(severity as any)} ${severity.toUpperCase()} (${insights.length})`
      );
      insights.forEach(insight => {
        lines.push(`    - ${insight.message}`);
        if (insight.suggestion) {
          lines.push(`      ${this.colorize('→', 'cyan')} ${insight.suggestion}`);
        }
      });
    }

    return lines.join('\n');
  }

  private groupInsightsBySeverity(insights: PluginInsight[]): Record<string, PluginInsight[]> {
    return insights.reduce(
      (acc, insight) => {
        if (!acc[insight.severity]) {
          acc[insight.severity] = [];
        }
        acc[insight.severity].push(insight);
        return acc;
      },
      {} as Record<string, PluginInsight[]>
    );
  }

  private formatSeverity(severity: string): string {
    switch (severity) {
      case 'critical':
        return this.colorize('●', 'red');
      case 'warning':
        return this.colorize('●', 'yellow');
      case 'info':
        return this.colorize('●', 'blue');
      default:
        return '●';
    }
  }

  private getGradeColor(grade: string): keyof typeof COLORS {
    switch (grade) {
      case 'A':
        return 'green';
      case 'B':
        return 'cyan';
      case 'C':
        return 'yellow';
      case 'D':
        return 'yellow';
      case 'F':
        return 'red';
      default:
        return 'reset';
    }
  }

  private colorize(text: string, color: keyof typeof COLORS): string {
    return `${COLORS[color]}${text}${COLORS.reset}`;
  }
}
```

```typescript
// clients/cli/src/formatters/json-formatter.ts

import type { AggregatedResult } from '@vipr/types';
import { IFormatter } from './formatter';

/**
 * JSON formatter for programmatic consumption
 */
export class JsonFormatter implements IFormatter {
  readonly id = 'json';
  readonly name = 'JSON Output';
  readonly extension = 'json';

  constructor(private pretty: boolean = true) {}

  format(result: AggregatedResult): string {
    const formatted = this.formatResult(result);
    return this.pretty ? JSON.stringify(formatted, null, 2) : JSON.stringify(formatted);
  }

  formatBatch(results: AggregatedResult[]): string {
    const formatted = results.map(r => this.formatResult(r));
    return this.pretty ? JSON.stringify(formatted, null, 2) : JSON.stringify(formatted);
  }

  private formatResult(result: AggregatedResult) {
    return {
      filePath: result.filePath,
      analyzedAt: result.analyzedAt,
      overallScore: result.overallScore,
      overallGrade: result.overallGrade,
      plugins: Array.from(result.pluginResults.entries()).map(([id, pr]) => ({
        id,
        score: pr.score,
        insights: pr.insights.length,
        executionTimeMs: pr.executionTimeMs,
        metrics: pr.metrics,
      })),
      insights: result.insights,
      errors: result.errors,
    };
  }
}
```

```typescript
// clients/cli/src/formatters/index.ts

export { IFormatter } from './formatter';
export { CliFormatter } from './cli-formatter';
export { JsonFormatter } from './json-formatter';

import { IFormatter } from './formatter';
import { CliFormatter } from './cli-formatter';
import { JsonFormatter } from './json-formatter';

/**
 * Get formatter by ID
 */
export function getFormatter(format: string): IFormatter {
  switch (format.toLowerCase()) {
    case 'cli':
    case 'console':
      return new CliFormatter();
    case 'json':
      return new JsonFormatter(true);
    case 'json-compact':
      return new JsonFormatter(false);
    default:
      throw new Error(`Unknown format: ${format}`);
  }
}

/**
 * List available formatters
 */
export function listFormatters(): IFormatter[] {
  return [new CliFormatter(), new JsonFormatter(true)];
}
```

### Step 3: Create Command Abstraction

Use **Command Pattern** for extensible CLI commands.

```typescript
// clients/cli/src/commands/command.ts

/**
 * Base interface for CLI commands
 */
export interface ICommand {
  /** Command name */
  readonly name: string;

  /** Command description */
  readonly description: string;

  /**
   * Execute the command
   */
  execute(args: string[]): Promise<number>;
}
```

```typescript
// clients/cli/src/commands/analyze-command.ts

import { AnalysisEngine } from '@vipr/core';
import { CliPluginLoader } from '../plugins/loader';
import { getFormatter } from '../formatters';
import { logger } from '@vipr/logging';
import { ICommand } from './command';

/**
 * Analyze command - main analysis command
 */
export class AnalyzeCommand implements ICommand {
  readonly name = 'analyze';
  readonly description = 'Analyze files for complexity and patterns';

  async execute(args: string[]): Promise<number> {
    try {
      // Parse arguments
      const options = this.parseArgs(args);

      // Create engine
      const engine = new AnalysisEngine({
        enableCache: options.cache,
        debug: options.debug,
      });

      // Load plugins
      const pluginLoader = new CliPluginLoader(engine);
      await pluginLoader.loadBundledPlugins();

      logger.info(`Analyzing ${options.files.length} file(s)...`);

      // Run analysis
      const results = await engine.analyzeFiles(options.files);

      // Format output
      const formatter = getFormatter(options.format);
      const output = formatter.formatBatch
        ? formatter.formatBatch(results)
        : results.map(r => formatter.format(r)).join('\n\n');

      // Write output
      if (options.output) {
        await this.writeToFile(options.output, output);
        logger.info(`Results written to ${options.output}`);
      } else {
        console.log(output);
      }

      return this.determineExitCode(results, options);
    } catch (error) {
      logger.error('Analysis failed:', error);
      return 1;
    }
  }

  private parseArgs(args: string[]) {
    // Simple argument parsing (could use a library like yargs/commander)
    return {
      files: args.filter(a => !a.startsWith('--')),
      format: this.getArgValue(args, '--format', 'cli'),
      output: this.getArgValue(args, '--output'),
      cache: !args.includes('--no-cache'),
      debug: args.includes('--debug'),
      failThreshold: parseInt(this.getArgValue(args, '--fail-threshold', '0'), 10),
    };
  }

  private getArgValue(args: string[], flag: string, defaultValue?: string): string | undefined {
    const index = args.indexOf(flag);
    if (index === -1 || index === args.length - 1) {
      return defaultValue;
    }
    return args[index + 1];
  }

  private determineExitCode(results: any[], options: any): number {
    // Exit with error if any file has score below threshold
    if (options.failThreshold > 0) {
      const failedFiles = results.filter(r => r.overallScore < options.failThreshold);
      if (failedFiles.length > 0) {
        logger.warn(`${failedFiles.length} file(s) below threshold`);
        return 1;
      }
    }
    return 0;
  }

  private async writeToFile(path: string, content: string): Promise<void> {
    const fs = await import('fs/promises');
    await fs.writeFile(path, content, 'utf-8');
  }
}
```

### Step 4: Refactor Main Entry Point

Clean architecture with clear separation of concerns.

```typescript
// clients/cli/src/index.ts

#!/usr/bin/env node

import { logger } from '@vipr/logging';
import { AnalyzeCommand } from './commands/analyze-command';

/**
 * CLI Entry Point
 */
async function main() {
  const args = process.argv.slice(2);

  if (args.length === 0 || args.includes('--help') || args.includes('-h')) {
    printHelp();
    return 0;
  }

  if (args.includes('--version') || args.includes('-v')) {
    printVersion();
    return 0;
  }

  // Default command is analyze
  const command = new AnalyzeCommand();
  return await command.execute(args);
}

function printHelp() {
  console.log(`
Vipr - Frontend Code Analysis Tool

USAGE:
  vipr [FILES...] [OPTIONS]

OPTIONS:
  --format <format>        Output format: cli, json (default: cli)
  --output <file>          Write output to file
  --fail-threshold <score> Exit with error if score below threshold
  --no-cache              Disable result caching
  --debug                 Enable debug logging
  --version, -v           Show version
  --help, -h              Show this help

EXAMPLES:
  vipr src/Component.tsx
  vipr src/**/*.tsx --format json --output report.json
  vipr src/App.tsx --fail-threshold 70
  `);
}

function printVersion() {
  const pkg = require('../package.json');
  console.log(`Vipr CLI v${pkg.version}`);
}

// Run CLI
main()
  .then(exitCode => {
    process.exit(exitCode);
  })
  .catch(error => {
    logger.error('Fatal error:', error);
    process.exit(1);
  });
```

## Code Patterns

### Pattern 1: Strategy Pattern for Formatters

**Objective**: Decouple output formatting from analysis logic.

**Before**:

```typescript
// Formatting hardcoded in main function
async function main() {
  const result = await engine.analyzeFile(file);

  // Formatting logic mixed with business logic
  console.log('='.repeat(60));
  console.log('Score:', result.score);
  console.log('Grade:', colorize(result.grade, 'green'));
  // ... more formatting
}
```

**After**:

```typescript
// Strategy pattern with pluggable formatters
async function main() {
  const result = await engine.analyzeFile(file);
  const formatter = getFormatter(options.format); // Strategy selection
  const output = formatter.format(result); // Strategy execution
  console.log(output);
}
```

**Benefits**:

- Easy to add new output formats
- Testable in isolation
- Single responsibility

### Pattern 2: Dependency Injection for Plugin Loading

**Objective**: Decouple CLI from specific analyzer implementations.

**Before**:

```typescript
import { ReactAnalyzerPlugin } from '@vipr/react'; // Tight coupling

async function main() {
  const engine = new AnalysisEngine();
  engine.registerPlugin(new ReactAnalyzerPlugin()); // Hardcoded dependency
}
```

**After**:

```typescript
// Dependency injected through plugin loader
async function main() {
  const engine = new AnalysisEngine();
  const loader = new CliPluginLoader(engine);
  await loader.loadBundledPlugins(); // Discovers and loads available plugins
}
```

**Benefits**:

- No hardcoded analyzer dependencies
- Easy to add new analyzers
- CLI works with any compatible plugin

### Pattern 3: Command Pattern for Extensibility

**Objective**: Enable multiple CLI commands with consistent interface.

**Implementation**:

```typescript
interface ICommand {
  readonly name: string;
  readonly description: string;
  execute(args: string[]): Promise<number>;
}

// Easy to add new commands
class AnalyzeCommand implements ICommand { ... }
class InitCommand implements ICommand { ... }
class ConfigCommand implements ICommand { ... }

// Command registry
const commands = {
  analyze: new AnalyzeCommand(),
  init: new InitCommand(),
  config: new ConfigCommand(),
};
```

**Benefits**:

- Extensible command structure
- Consistent interface
- Easy to test commands

## File Changes

### New Files to Create

#### 1. Plugin Loader

**File**: `clients/cli/src/plugins/loader.ts`

- Discovers bundled analyzer plugins
- Registers plugins with engine
- Provides plugin access methods
- ~120 lines

#### 2. Formatter Interface

**File**: `clients/cli/src/formatters/formatter.ts`

- Defines `IFormatter` interface
- ~30 lines

#### 3. CLI Formatter

**File**: `clients/cli/src/formatters/cli-formatter.ts`

- Console output with colors
- Human-readable formatting
- ~150 lines

#### 4. JSON Formatter

**File**: `clients/cli/src/formatters/json-formatter.ts`

- JSON output for programmatic use
- Pretty and compact modes
- ~80 lines

#### 5. Formatter Registry

**File**: `clients/cli/src/formatters/index.ts`

- Exports formatters
- Provides `getFormatter()` function
- Lists available formatters
- ~30 lines

#### 6. Command Interface

**File**: `clients/cli/src/commands/command.ts`

- Defines `ICommand` interface
- ~20 lines

#### 7. Analyze Command

**File**: `clients/cli/src/commands/analyze-command.ts`

- Main analysis command implementation
- Argument parsing
- Analysis orchestration
- Output handling
- ~150 lines

#### 8. Command Index

**File**: `clients/cli/src/commands/index.ts`

- Exports commands
- ~10 lines

### Files to Modify

#### 1. CLI Entry Point

**File**: `clients/cli/src/index.ts`

**Current**: 60 lines of mixed concerns
**Target**: 80 lines with clean separation

**Changes**:

- Remove hardcoded analyzer imports
- Remove formatting logic
- Add command delegation
- Add help and version commands
- Improve error handling

**Before**:

```typescript
import { ReactAnalyzerPlugin } from '@vipr/react';

async function main() {
  const engine = new AnalysisEngine();
  engine.registerPlugin(new ReactAnalyzerPlugin());

  const result = await engine.analyzeFile('test.tsx');

  // Formatting inline
  console.log('Score:', result.score);
  console.log('Grade:', result.grade);
}
```

**After**:

```typescript
import { AnalyzeCommand } from './commands/analyze-command';

async function main() {
  const args = process.argv.slice(2);

  if (args.includes('--help')) {
    printHelp();
    return 0;
  }

  const command = new AnalyzeCommand();
  return await command.execute(args);
}
```

#### 2. Package.json

**File**: `clients/cli/package.json`

**Changes**:

```diff
{
  "dependencies": {
    "@vipr/types": "workspace:*",
    "@vipr/core": "workspace:*",
-   "@vipr/react": "workspace:*",
    "@vipr/logging": "workspace:*",
+   "@vipr/plugin-loader": "workspace:*"
  }
}
```

**Note**: Keep `@vipr/react` as an optional dependency for bundling, but not imported directly in CLI code.

#### 3. TypeScript Config

**File**: `clients/cli/tsconfig.json`

**Changes**: None required, existing config should work.

### Files to Delete

None - this is an additive refactoring.

## Dependencies

### Prerequisite Phases

**Phase 1: Type System & Interfaces** (Complete)

- Required: Plugin interfaces defined

**Phase 2: Plugin Discovery & Loading** (Optional)

- Can proceed without the common plugin-loader package
- Implement CLI-specific loader for bundled plugins
- Can migrate to common loader later

**Phase 3: Engine Enhancements** (Not required)

- Engine already supports plugin registration

**Phase 4: Analyzer Refactoring** (Not required)

- Can proceed in parallel
- CLI works with old or new analyzer structure

### External Dependencies

**New Dependencies**:

- None - uses existing workspace packages

**Peer Dependencies**:

- `@vipr/core`: Analysis engine
- `@vipr/types`: Type definitions
- `@vipr/logging`: Logger
- `@vipr/react`: React analyzer (bundled, not imported)

## Acceptance Criteria

### Functional Requirements

1. CLI runs analysis without importing `@vipr/react` directly
2. Plugin loader discovers and registers all bundled analyzers
3. CLI formatter produces identical output to current version
4. JSON formatter produces valid, well-structured JSON
5. Help command displays usage information
6. Version command displays version number
7. Exit codes reflect analysis results correctly
8. File output works for all formatters

### Backward Compatibility Requirements

1. All existing command-line arguments work unchanged
2. Default output format is identical to current CLI
3. Exit codes unchanged (0 for success, 1 for failure)
4. No breaking changes for users
5. Same file patterns supported

### Code Quality Requirements

1. Main entry point under 100 lines
2. Each command under 200 lines
3. Each formatter under 200 lines
4. No direct analyzer imports in CLI code
5. Clear separation of concerns
6. All public methods documented

### Performance Requirements

1. Startup time not increased by more than 50ms
2. Plugin loading completes in under 100ms
3. Formatting overhead under 10ms per file
4. Memory usage not increased by more than 5%

### Testing Requirements

1. Unit tests for each formatter
2. Integration tests for commands
3. CLI argument parsing tests
4. Plugin loader tests
5. End-to-end tests for common workflows

### Documentation Requirements

1. Updated README with new options
2. Formatter documentation
3. Plugin integration guide
4. Command development guide

## Recommended Claude Model

**Primary Model**: Claude Sonnet 4.5

- Straightforward refactoring with clear patterns
- Well-defined interfaces and structure
- Lower risk than analyzer refactoring
- Good balance of capability and cost

**Secondary Model**: Claude Opus 4.5

- Only if complex edge cases arise
- Command pattern implementation review
- Architecture validation

## Assigned Subagents

### CLI Refactoring Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Refactor main entry point
- Implement command pattern
- Extract formatting logic
- Update package.json

### Formatter Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Implement CLI formatter
- Implement JSON formatter
- Create formatter registry
- Ensure output compatibility

### Plugin Integration Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Create CLI plugin loader
- Integrate with engine
- Handle plugin registration
- Test plugin discovery

### Testing Agent

**Model**: Sonnet 4.5
**Responsibilities**:

- Create CLI integration tests
- Test formatters
- Test commands
- End-to-end testing

## Risk Mitigation

### High-Risk Areas

1. **Breaking CLI Interface**: Changes to command-line arguments
   - **Mitigation**: Maintain exact same arguments, add new ones only

2. **Output Format Changes**: Users may parse CLI output
   - **Mitigation**: Keep default output identical, new formats are opt-in

3. **Plugin Loading Failures**: Missing or broken analyzers
   - **Mitigation**: Graceful degradation, clear error messages

4. **Performance Regression**: Plugin loading overhead
   - **Mitigation**: Lazy loading, benchmarking

### Breaking Change Prevention

1. Keep default behavior identical
2. New features are opt-in via flags
3. Maintain exit code conventions
4. Preserve file path handling

### Rollback Strategy

1. Feature flag for new architecture
2. Keep old code path as fallback
3. Gradual rollout
4. Easy revert in package.json

## Metrics for Success

### Code Quality Metrics

- Lines in main entry: Target < 100 (currently 60)
- Cyclomatic complexity: Keep under 10 per function
- Class cohesion: > 0.8
- Formatter independence: 100%

### Performance Metrics

- Startup time increase: < 50ms
- Plugin loading time: < 100ms
- Formatting overhead: < 10ms
- Memory overhead: < 5%

### User Experience Metrics

- Help text clarity: User testing
- Error message quality: User testing
- Command discoverability: Documentation review

### Maintainability Metrics

- Time to add new formatter: < 30 minutes
- Time to add new command: < 1 hour
- Code review time: Reduce by 40%
- Developer onboarding: < 2 hours

## Migration Path

### Phase 1: Add New Architecture (Non-Breaking)

1. Create formatters alongside existing code
2. Create commands alongside existing code
3. Add plugin loader
4. Keep existing entry point

### Phase 2: Gradual Migration

1. Add feature flag to use new architecture
2. Test with alpha users
3. Collect feedback

### Phase 3: Switch Default

1. Make new architecture default
2. Keep fallback option
3. Monitor for issues

### Phase 4: Cleanup

1. Remove old code after stable period
2. Remove feature flags
3. Update documentation

## Future Extensibility

### Planned Enhancements

1. **Configuration File Support**
   - `.viprrc` file for defaults
   - Support JSON, YAML, TOML

2. **Watch Mode**
   - Watch files for changes
   - Re-analyze on change

3. **Interactive Mode**
   - REPL for exploration
   - Interactive report browsing

4. **Additional Formatters**
   - HTML report generator
   - Markdown formatter
   - SARIF for security tools
   - GitLab/GitHub CI integrations

5. **Multi-Command Support**
   - `vipr init` - Initialize config
   - `vipr config` - Manage configuration
   - `vipr plugins` - List available plugins
   - `vipr rules` - Manage analysis rules
