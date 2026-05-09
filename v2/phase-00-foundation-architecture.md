# Phase 00: Foundation and Architecture

**Phase Duration:** 2-3 days
**Priority:** Critical - Must complete before other phases
**Complexity:** Medium

## Overview

This phase establishes the architectural foundation for all subsequent enhancements. It involves restructuring the codebase to support modular analyzers, creating base abstractions, and setting up the plugin system infrastructure.

## Agent Assignments

| Agent                  | Role                               | Capacity |
| ---------------------- | ---------------------------------- | -------- |
| typescript-engineer    | Lead architect, type system design | Primary  |
| react-engineer         | React-specific patterns validation | Reviewer |
| vscode-plugin-engineer | Extension architecture preview     | Advisory |

## Execution Strategy

### Synchronous Work (Sequential)

1. **Type System Foundation** (typescript-engineer)
2. **Base Analyzer Abstraction** (typescript-engineer)
3. **Review and Validation** (react-engineer)

### Parallel Work (Concurrent)

After step 2 completes:

- Plugin interface design (typescript-engineer)
- Directory structure implementation (Haiku)
- Constants reorganization (Haiku)

## Detailed Tasks

### Task 0.1: Directory Structure Refactoring

**Model:** Haiku (deterministic file operations)

Create the enhanced directory structure:

```
src/
├── analyzers/
│   ├── base-analyzer.ts          # NEW: Abstract base class
│   ├── index.ts                  # NEW: Barrel export
│   ├── type-analyzer.ts          # MOVE from src/
│   ├── dataflow-analyzer.ts      # MOVE from src/
│   └── reliability-analyzer.ts   # MOVE from src/
├── rules/
│   ├── index.ts                  # NEW: Rule registry
│   └── types.ts                  # NEW: Rule type definitions
├── plugins/
│   ├── plugin-interface.ts       # NEW: Plugin contract
│   ├── plugin-loader.ts          # NEW: Dynamic loading
│   └── index.ts                  # NEW: Barrel export
├── constants/
│   ├── weights.ts                # EXTRACT from constants.ts
│   ├── thresholds.ts             # EXTRACT from constants.ts
│   ├── react-api-catalog.ts      # NEW: React version APIs
│   └── index.ts                  # NEW: Barrel export
├── types/
│   ├── core.ts                   # EXTRACT from types.ts
│   ├── analysis.ts               # EXTRACT from types.ts
│   ├── plugin.ts                 # MOVE from types/plugin.ts
│   └── index.ts                  # ENHANCE: Re-exports
└── utils/
    ├── ast-helpers.ts            # EXISTS
    ├── scoring.ts                # EXISTS
    ├── pattern-matchers.ts       # NEW: Pattern matching utilities
    └── index.ts                  # NEW: Barrel export
```

**Files to Create:**

1. `src/analyzers/index.ts` - Barrel export
2. `src/rules/index.ts` - Rule registry
3. `src/rules/types.ts` - Rule type definitions
4. `src/plugins/plugin-interface.ts` - Plugin contract
5. `src/plugins/plugin-loader.ts` - Dynamic loading
6. `src/plugins/index.ts` - Barrel export
7. `src/constants/index.ts` - Barrel export
8. `src/utils/index.ts` - Barrel export

**Files to Move:**

1. `src/type-analyzer.ts` -> `src/analyzers/type-analyzer.ts`
2. `src/dataflow-analyzer.ts` -> `src/analyzers/dataflow-analyzer.ts`
3. `src/reliability-analyzer.ts` -> `src/analyzers/reliability-analyzer.ts`

### Task 0.2: Base Analyzer Abstraction

**Model:** Opus (architectural, business-critical)

Create the abstract base class that all specialized analyzers will extend.

**File:** `src/analyzers/base-analyzer.ts`

````typescript
import * as t from '@babel/types';
import traverse, { NodePath } from '@babel/traverse';
import type { CodeLocation } from '../types';

/**
 * Base configuration for all analyzers
 */
export interface AnalyzerConfig {
  /** Enable debug logging */
  debug?: boolean;
  /** Custom thresholds override */
  thresholds?: Record<string, number>;
  /** Files/patterns to ignore */
  ignorePatterns?: string[];
}

/**
 * Result metadata common to all analyzer results
 */
export interface AnalyzerResultMetadata {
  /** Analyzer identifier */
  analyzer: string;
  /** Analysis timestamp */
  timestamp: string;
  /** Analysis duration in milliseconds */
  durationMs: number;
  /** Source file path if available */
  filePath?: string;
}

/**
 * Base result interface all analyzers must implement
 */
export interface BaseAnalyzerResult {
  /** Normalized score (0-100) */
  score: number;
  /** Result metadata */
  metadata: AnalyzerResultMetadata;
}

/**
 * Abstract base class for all specialized analyzers.
 *
 * Provides common functionality:
 * - AST access
 * - Location extraction
 * - Timing measurement
 * - Configuration management
 *
 * @example
 * ```typescript
 * class MigrationAnalyzer extends BaseAnalyzer<MigrationResult> {
 *   protected analyzeAST(): MigrationResult {
 *     // Implementation
 *   }
 * }
 * ```
 */
export abstract class BaseAnalyzer<TResult extends BaseAnalyzerResult> {
  protected ast: t.File;
  protected config: AnalyzerConfig;
  protected readonly analyzerName: string;

  constructor(ast: t.File, config: AnalyzerConfig = {}) {
    this.ast = ast;
    this.config = config;
    this.analyzerName = this.constructor.name;
  }

  /**
   * Execute analysis and return results
   */
  public analyze(filePath?: string): TResult {
    const startTime = performance.now();

    const result = this.analyzeAST();

    const endTime = performance.now();

    return {
      ...result,
      metadata: {
        analyzer: this.analyzerName,
        timestamp: new Date().toISOString(),
        durationMs: Math.round(endTime - startTime),
        filePath,
      },
    };
  }

  /**
   * Abstract method - implement analysis logic
   */
  protected abstract analyzeAST(): Omit<TResult, 'metadata'>;

  /**
   * Extract code location from AST node path
   */
  protected getLocation(path: NodePath): CodeLocation {
    const loc = path.node.loc;
    return {
      line: loc?.start.line ?? 0,
      column: loc?.start.column ?? 0,
    };
  }

  /**
   * Log debug message if debug mode enabled
   */
  protected debug(message: string, ...args: unknown[]): void {
    if (this.config.debug) {
      console.log(`[${this.analyzerName}] ${message}`, ...args);
    }
  }

  /**
   * Get threshold value with config override support
   */
  protected getThreshold(name: string, defaultValue: number): number {
    return this.config.thresholds?.[name] ?? defaultValue;
  }
}
````

### Task 0.3: Plugin Interface Design

**Model:** Opus (architectural)

Define the plugin contract for extensibility.

**File:** `src/plugins/plugin-interface.ts`

```typescript
import * as t from '@babel/types';
import type { BaseAnalyzerResult, AnalyzerConfig } from '../analyzers/base-analyzer';

/**
 * Plugin metadata for registration and discovery
 */
export interface PluginMetadata {
  /** Unique plugin identifier */
  id: string;
  /** Human-readable name */
  name: string;
  /** Plugin version (semver) */
  version: string;
  /** Plugin description */
  description: string;
  /** Author information */
  author?: string;
  /** Plugin homepage URL */
  homepage?: string;
}

/**
 * Plugin lifecycle hooks
 */
export interface PluginLifecycle {
  /** Called when plugin is registered */
  onRegister?(): void;
  /** Called before analysis starts */
  onBeforeAnalysis?(filePath: string): void;
  /** Called after analysis completes */
  onAfterAnalysis?(result: BaseAnalyzerResult): void;
  /** Called when plugin is unregistered */
  onUnregister?(): void;
}

/**
 * Plugin result contribution
 */
export interface PluginResult {
  /** Plugin identifier */
  pluginId: string;
  /** Plugin-specific metrics */
  metrics: Record<string, unknown>;
  /** Plugin insights/issues found */
  insights: PluginInsight[];
}

export interface PluginInsight {
  /** Unique identifier */
  id: string;
  /** Severity level */
  severity: 'info' | 'warning' | 'critical';
  /** Human-readable message */
  message: string;
  /** Code location */
  location?: { line: number; column: number };
  /** Suggested fix */
  suggestion?: string;
  /** Auto-fixable flag */
  autoFixable?: boolean;
}

/**
 * Plugin interface contract
 *
 * All plugins must implement this interface to integrate
 * with the analyzer ecosystem.
 */
export interface AnalyzerPlugin extends PluginLifecycle {
  /** Plugin metadata */
  readonly metadata: PluginMetadata;

  /**
   * Analyze AST and return plugin-specific results
   *
   * @param ast - Parsed AST
   * @param config - Analyzer configuration
   * @returns Plugin analysis results
   */
  analyze(ast: t.File, config?: AnalyzerConfig): PluginResult;

  /**
   * Optional: Provide code fixes for identified issues
   *
   * @param insight - The insight to fix
   * @param source - Original source code
   * @returns Fixed source code or null if cannot fix
   */
  fix?(insight: PluginInsight, source: string): string | null;
}

/**
 * Plugin registration entry
 */
export interface PluginRegistration {
  plugin: AnalyzerPlugin;
  enabled: boolean;
  priority: number;
}
```

### Task 0.4: Constants Reorganization

**Model:** Haiku (deterministic extraction)

Extract constants from the monolithic `constants.ts` into organized modules.

**Files to Create:**

1. `src/constants/weights.ts` - All weight constants
2. `src/constants/thresholds.ts` - All threshold constants
3. `src/constants/react-api-catalog.ts` - React version API catalog
4. `src/constants/index.ts` - Barrel re-export

### Task 0.5: Update Import Paths

**Model:** Haiku (deterministic refactoring)

Update all import paths throughout the codebase to use new locations:

1. Update `src/analyzer.ts` imports
2. Update `src/cli.ts` imports
3. Update test file imports
4. Ensure backward compatibility for external consumers

## Acceptance Criteria

### Must Pass

- [ ] All existing tests pass without modification
- [ ] `npm run checks:types` passes with zero errors
- [ ] `npm run checks:linting` passes
- [ ] `npm test` passes with same coverage
- [ ] CLI commands work identically to before refactoring
- [ ] JSON output schema remains unchanged

### Code Quality

- [ ] BaseAnalyzer has 100% test coverage
- [ ] Plugin interface has JSDoc documentation
- [ ] All barrel exports are correctly configured
- [ ] No circular dependencies introduced
- [ ] Import paths are consistent (use `@/` alias or relative)

### Documentation

- [ ] `src/analyzers/README.md` created explaining architecture
- [ ] `src/plugins/README.md` created explaining plugin system
- [ ] Type exports are documented

## Testing Instructions

### Manual Testing Steps

1. **Verify Build**

   ```bash
   npm run build
   # Expected: Successful compilation, no errors
   ```

2. **Run Type Checks**

   ```bash
   npm run checks:types
   # Expected: No TypeScript errors
   ```

3. **Run Linting**

   ```bash
   npm run checks:linting
   # Expected: No ESLint errors
   ```

4. **Run Tests**

   ```bash
   npm test
   # Expected: All tests pass, coverage maintained
   ```

5. **Test CLI Unchanged**

   ```bash
   npm run analyze -- src/sample-components/DataTable.tsx
   # Expected: Same output as before refactoring

   npm run analyze -- src/sample-components/ --json
   # Expected: Valid JSON output matching schema
   ```

6. **Verify Imports Work**

   ```bash
   # Create a test file that imports from new locations
   cat > /tmp/import-test.ts << 'EOF'
   import { BaseAnalyzer } from './src/analyzers/base-analyzer';
   import { AnalyzerPlugin } from './src/plugins';
   import { COMPLEXITY_WEIGHTS } from './src/constants';
   console.log('Imports work!');
   EOF
   npx tsx /tmp/import-test.ts
   # Expected: "Imports work!" printed
   ```

7. **Schema Validation**
   ```bash
   npm run analyze -- src/sample-components/DataTable.tsx --json | npx ajv validate -s schema.json -d -
   # Expected: Valid
   ```

### Automated Test Requirements

Add the following test file: `src/analyzers/base-analyzer.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { BaseAnalyzer, BaseAnalyzerResult, AnalyzerConfig } from './base-analyzer';
import * as parser from '@babel/parser';

// Concrete implementation for testing
class TestAnalyzer extends BaseAnalyzer<BaseAnalyzerResult & { testValue: number }> {
  protected analyzeAST() {
    return {
      score: 50,
      testValue: 42,
    };
  }
}

describe('BaseAnalyzer', () => {
  const parseCode = (code: string) =>
    parser.parse(code, {
      sourceType: 'module',
      plugins: ['jsx', 'typescript'],
    });

  it('should return result with metadata', () => {
    const ast = parseCode('const x = 1;');
    const analyzer = new TestAnalyzer(ast);
    const result = analyzer.analyze();

    expect(result.score).toBe(50);
    expect(result.testValue).toBe(42);
    expect(result.metadata).toBeDefined();
    expect(result.metadata.analyzer).toBe('TestAnalyzer');
    expect(result.metadata.durationMs).toBeGreaterThanOrEqual(0);
  });

  it('should include file path in metadata when provided', () => {
    const ast = parseCode('const x = 1;');
    const analyzer = new TestAnalyzer(ast);
    const result = analyzer.analyze('test.tsx');

    expect(result.metadata.filePath).toBe('test.tsx');
  });

  it('should support custom thresholds', () => {
    const ast = parseCode('const x = 1;');
    const config: AnalyzerConfig = {
      thresholds: { custom: 100 },
    };
    const analyzer = new TestAnalyzer(ast, config);

    // Access protected method via any cast for testing
    const threshold = (analyzer as any).getThreshold('custom', 50);
    expect(threshold).toBe(100);
  });
});
```

## Rollback Plan

If issues arise after this phase:

1. All changes are isolated to new files and import paths
2. Existing `constants.ts` and `types.ts` remain functional
3. Git revert to pre-phase commit is safe

## Dependencies

- None (this is the foundation phase)

## Downstream Impact

All subsequent phases depend on this foundation:

- Phase 01-09 will use `BaseAnalyzer`
- Phase 07-08 (VS Code) will use plugin system
- All phases will use reorganized constants

## Estimated Effort

| Task                    | Model  | Estimated Time |
| ----------------------- | ------ | -------------- |
| 0.1 Directory Structure | Haiku  | 30 minutes     |
| 0.2 Base Analyzer       | Opus   | 2 hours        |
| 0.3 Plugin Interface    | Opus   | 1 hour         |
| 0.4 Constants Reorg     | Haiku  | 45 minutes     |
| 0.5 Import Updates      | Haiku  | 1 hour         |
| Testing & Validation    | Sonnet | 1 hour         |
| **Total**               |        | **6-7 hours**  |

---

**Document Version:** 1.0
**Created:** January 10, 2026
**Status:** Ready for Implementation
