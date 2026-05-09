# Phase 09: Advanced Features

**Priority:** Low - Nice-to-have  
**Complexity:** High  
**Dependencies:** Phase 00-08  
**Status:** Ready for implementation

**Phase Duration:** 7-10 days  
**Priority:** Medium - Enhanced capabilities  
**Complexity:** High  
**Dependencies:** Phases 00-08

All analyzer integrations use the `ts-morph`-based API.

## Overview

This phase implements advanced features including historical analysis (git integration), custom rules engine, dependency analysis, documentation quality metrics, and plugin architecture for third-party extensions.

## Business Value

- Track code health trends over time
- Enforce team-specific conventions
- Understand component dependencies
- Ensure documentation quality
- Enable community extensions

## Agent Assignments

| Agent                    | Role                           | Capacity  |
| ------------------------ | ------------------------------ | --------- |
| typescript-engineer      | Lead implementer, architecture | Primary   |
| react-engineer           | React patterns, analysis       | Secondary |
| vscode-plugin-engineer   | Extension integration          | Advisory  |
| code-complexity-analyzer | Algorithm design               | Advisory  |

## Execution Strategy

### Milestone 9.1: Git/Historical Analysis (Day 1-3)

**Synchronous Tasks:**

1. Git integration layer (typescript-engineer)
2. Change frequency analysis (code-complexity-analyzer)
3. Trend calculation (Sonnet)

### Milestone 9.2: Custom Rules Engine (Day 3-5)

**Parallel Tasks:**

- Rule configuration system (Opus)
- Custom rule API (Opus)
- Rule validation (Sonnet)

### Milestone 9.3: Dependency Analysis (Day 5-7)

**Parallel Tasks:**

- Import graph builder (Sonnet)
- Circular dependency detection (Opus)
- Coupling metrics (Sonnet)

### Milestone 9.4: Documentation Quality (Day 7-8)

**Parallel Tasks:**

- JSDoc detection (Sonnet)
- Prop documentation scoring (Sonnet)
- Coverage calculation (Haiku)

### Milestone 9.5: Plugin Architecture (Day 8-10)

**Synchronous Tasks:**

1. Plugin loader (Opus)
2. Plugin API (typescript-engineer)
3. Plugin validation (Sonnet)

## Detailed Tasks

### Task 9.1: Git Integration

**Model:** Opus (system integration)
**File:** `src/analyzers/historical-analyzer.ts`

```typescript
import { execSync } from 'child_process';
import * as path from 'path';
import type { CodeLocation } from '../types';

export interface FileHistory {
  /** File path */
  filePath: string;
  /** Commits affecting this file */
  commitCount: number;
  /** Unique authors */
  authorCount: number;
  /** Lines added over time */
  linesAdded: number;
  /** Lines removed over time */
  linesRemoved: number;
  /** Churn rate (changes / total lines) */
  churnRate: number;
  /** Recent commit messages */
  recentCommits: CommitInfo[];
  /** Bug fix commits (based on message patterns) */
  bugFixCount: number;
}

export interface CommitInfo {
  hash: string;
  author: string;
  date: Date;
  message: string;
  isBugFix: boolean;
}

export interface TemporalCoupling {
  /** Files that change together */
  files: [string, string];
  /** Co-change frequency */
  frequency: number;
  /** Confidence (0-1) */
  confidence: number;
}

export interface HistoricalMetrics {
  /** Overall historical score (0-100) */
  score: number;
  /** File histories */
  fileHistories: FileHistory[];
  /** Files with high churn */
  hotspots: FileHistory[];
  /** Temporal couplings */
  temporalCouplings: TemporalCoupling[];
  /** Complexity trend */
  complexityTrend: 'improving' | 'stable' | 'degrading';
  /** Analysis period */
  periodDays: number;
}

export class HistoricalAnalyzer {
  private repoRoot: string;

  constructor(repoRoot?: string) {
    this.repoRoot = repoRoot ?? this.findRepoRoot();
  }

  /**
   * Analyze file history
   */
  analyzeFile(filePath: string, days: number = 90): FileHistory {
    const relativePath = path.relative(this.repoRoot, filePath);
    const since = this.getDateDaysAgo(days);

    // Get commit log for file
    const logOutput = this.git(
      `log --since="${since}" --pretty=format:"%H|%an|%ad|%s" --date=iso -- "${relativePath}"`
    );

    const commits = this.parseCommitLog(logOutput);
    const authors = new Set(commits.map(c => c.author));

    // Get file churn
    const churnOutput = this.git(
      `log --since="${since}" --pretty=format: --numstat -- "${relativePath}"`
    );
    const { linesAdded, linesRemoved } = this.parseChurn(churnOutput);

    // Estimate current file size for churn rate
    const currentLines = this.countLines(filePath);
    const churnRate = currentLines > 0 ? (linesAdded + linesRemoved) / currentLines : 0;

    const bugFixCount = commits.filter(c => c.isBugFix).length;

    return {
      filePath,
      commitCount: commits.length,
      authorCount: authors.size,
      linesAdded,
      linesRemoved,
      churnRate,
      recentCommits: commits.slice(0, 10),
      bugFixCount,
    };
  }

  /**
   * Analyze project for hotspots
   */
  analyzeProject(globPattern: string = '**/*.tsx', days: number = 90): HistoricalMetrics {
    const files = this.findFiles(globPattern);
    const fileHistories = files.map(f => this.analyzeFile(f, days));

    // Identify hotspots (high churn + high complexity = problem)
    const hotspots = fileHistories
      .filter(h => h.churnRate > 1.5 || h.bugFixCount > 3)
      .sort((a, b) => b.churnRate - a.churnRate)
      .slice(0, 10);

    // Find temporal couplings
    const temporalCouplings = this.findTemporalCouplings(days);

    // Calculate overall score
    const avgChurn = fileHistories.reduce((s, f) => s + f.churnRate, 0) / fileHistories.length;
    const avgBugs = fileHistories.reduce((s, f) => s + f.bugFixCount, 0) / fileHistories.length;
    const score = Math.max(0, 100 - avgChurn * 20 - avgBugs * 5);

    // Determine trend (would need multiple snapshots in practice)
    const complexityTrend = this.calculateTrend(fileHistories);

    return {
      score,
      fileHistories,
      hotspots,
      temporalCouplings,
      complexityTrend,
      periodDays: days,
    };
  }

  /**
   * Find files that change together
   */
  private findTemporalCouplings(days: number): TemporalCoupling[] {
    const since = this.getDateDaysAgo(days);
    const couplings: TemporalCoupling[] = [];

    // Get commits with their files
    const logOutput = this.git(`log --since="${since}" --pretty=format:"COMMIT:%H" --name-only`);

    const commits = logOutput.split('COMMIT:').filter(Boolean);
    const coChanges = new Map<string, number>();

    commits.forEach(commit => {
      const files = commit.trim().split('\n').slice(1).filter(Boolean);
      const tsxFiles = files.filter(f => f.endsWith('.tsx'));

      // Count co-changes
      for (let i = 0; i < tsxFiles.length; i++) {
        for (let j = i + 1; j < tsxFiles.length; j++) {
          const key = [tsxFiles[i], tsxFiles[j]].sort().join(':');
          coChanges.set(key, (coChanges.get(key) ?? 0) + 1);
        }
      }
    });

    // Convert to couplings
    coChanges.forEach((count, key) => {
      if (count >= 3) {
        const [file1, file2] = key.split(':');
        couplings.push({
          files: [file1, file2],
          frequency: count,
          confidence: Math.min(1, count / 10),
        });
      }
    });

    return couplings.sort((a, b) => b.frequency - a.frequency).slice(0, 20);
  }

  private git(command: string): string {
    try {
      return execSync(`git ${command}`, {
        cwd: this.repoRoot,
        encoding: 'utf-8',
        maxBuffer: 10 * 1024 * 1024,
      });
    } catch {
      return '';
    }
  }

  private findRepoRoot(): string {
    try {
      return execSync('git rev-parse --show-toplevel', {
        encoding: 'utf-8',
      }).trim();
    } catch {
      return process.cwd();
    }
  }

  private getDateDaysAgo(days: number): string {
    const date = new Date();
    date.setDate(date.getDate() - days);
    return date.toISOString().split('T')[0];
  }

  private parseCommitLog(output: string): CommitInfo[] {
    if (!output.trim()) return [];

    const bugFixPatterns = /\b(fix|bug|issue|error|crash|hotfix)\b/i;

    return output
      .trim()
      .split('\n')
      .map(line => {
        const [hash, author, date, ...messageParts] = line.split('|');
        const message = messageParts.join('|');
        return {
          hash,
          author,
          date: new Date(date),
          message,
          isBugFix: bugFixPatterns.test(message),
        };
      });
  }

  private parseChurn(output: string): { linesAdded: number; linesRemoved: number } {
    let linesAdded = 0;
    let linesRemoved = 0;

    output
      .trim()
      .split('\n')
      .forEach(line => {
        const [added, removed] = line.trim().split(/\s+/);
        if (added && removed) {
          linesAdded += parseInt(added, 10) || 0;
          linesRemoved += parseInt(removed, 10) || 0;
        }
      });

    return { linesAdded, linesRemoved };
  }

  private countLines(filePath: string): number {
    try {
      const output = execSync(`wc -l < "${filePath}"`, { encoding: 'utf-8' });
      return parseInt(output.trim(), 10) || 100;
    } catch {
      return 100;
    }
  }

  private findFiles(pattern: string): string[] {
    try {
      const output = execSync(`find . -name "*.tsx" -type f`, {
        cwd: this.repoRoot,
        encoding: 'utf-8',
      });
      return output
        .trim()
        .split('\n')
        .filter(Boolean)
        .map(f => path.join(this.repoRoot, f));
    } catch {
      return [];
    }
  }

  private calculateTrend(histories: FileHistory[]): 'improving' | 'stable' | 'degrading' {
    const recentBugs = histories.reduce((s, h) => s + h.bugFixCount, 0);
    const avgChurn = histories.reduce((s, h) => s + h.churnRate, 0) / histories.length;

    if (recentBugs > 10 || avgChurn > 2) return 'degrading';
    if (recentBugs < 3 && avgChurn < 0.5) return 'improving';
    return 'stable';
  }
}
```

### Task 9.2: Custom Rules Engine

**Model:** Opus (extensibility architecture)
**File:** `src/plugins/custom-rules.ts`

```typescript
import * as t from '@babel/types';
import type { AntiPatternRule, AntiPattern } from '../rules/types';

/**
 * Custom rule configuration
 */
export interface CustomRuleConfig {
  /** Rule identifier */
  id: string;
  /** Rule name */
  name: string;
  /** Rule description */
  description: string;
  /** Severity */
  severity: 'critical' | 'warning' | 'info';
  /** Pattern to match (regex string or AST pattern) */
  pattern: string | ASTPattern;
  /** Message template */
  message: string;
  /** Fix suggestion */
  fix?: string;
  /** Documentation URL */
  documentation?: string;
}

/**
 * AST pattern for matching
 */
export interface ASTPattern {
  /** Node type to match */
  type: string;
  /** Property matchers */
  properties?: Record<string, any>;
  /** Child patterns */
  children?: ASTPattern[];
}

/**
 * Custom rules loader
 */
export class CustomRulesLoader {
  /**
   * Load custom rules from configuration
   */
  loadFromConfig(configs: CustomRuleConfig[]): AntiPatternRule[] {
    return configs.map(config => this.createRule(config));
  }

  /**
   * Load custom rules from file
   */
  async loadFromFile(filePath: string): Promise<AntiPatternRule[]> {
    const fs = await import('fs/promises');
    const content = await fs.readFile(filePath, 'utf-8');
    const configs = JSON.parse(content) as CustomRuleConfig[];
    return this.loadFromConfig(configs);
  }

  private createRule(config: CustomRuleConfig): AntiPatternRule {
    const rule: AntiPatternRule = {
      id: config.id,
      name: config.name,
      category: 'custom' as any,
      defaultSeverity: config.severity,
      description: config.description,
      references: config.documentation ? [config.documentation] : [],
      fixable: !!config.fix,

      check: (ast: t.File) => {
        const violations: AntiPattern[] = [];

        if (typeof config.pattern === 'string') {
          // Regex pattern matching on source
          // Would need source code access
        } else {
          // AST pattern matching
          violations.push(...this.matchASTPattern(ast, config));
        }

        return violations;
      },
    };

    return rule;
  }

  private matchASTPattern(ast: t.File, config: CustomRuleConfig): AntiPattern[] {
    const violations: AntiPattern[] = [];
    const pattern = config.pattern as ASTPattern;

    // Simple AST traversal for pattern matching
    const traverse = require('@babel/traverse').default;

    traverse(ast, {
      [pattern.type]: (path: any) => {
        if (this.matchesPattern(path.node, pattern)) {
          violations.push({
            id: config.id,
            name: config.name,
            category: 'custom' as any,
            severity: config.severity,
            location: {
              line: path.node.loc?.start.line ?? 0,
              column: path.node.loc?.start.column ?? 0,
            },
            description: config.message,
            impact: config.description,
            fix: config.fix ?? '',
            autoFixable: false,
            references: config.documentation ? [config.documentation] : [],
          });
        }
      },
    });

    return violations;
  }

  private matchesPattern(node: any, pattern: ASTPattern): boolean {
    if (pattern.properties) {
      for (const [key, value] of Object.entries(pattern.properties)) {
        if (typeof value === 'object' && value.regex) {
          const regex = new RegExp(value.regex);
          if (!regex.test(node[key])) return false;
        } else if (node[key] !== value) {
          return false;
        }
      }
    }
    return true;
  }
}

/**
 * Example custom rules configuration
 */
export const EXAMPLE_CUSTOM_RULES: CustomRuleConfig[] = [
  {
    id: 'no-console-log',
    name: 'No console.log',
    description: 'console.log statements should not be in production code',
    severity: 'warning',
    pattern: {
      type: 'CallExpression',
      properties: {
        callee: {
          type: 'MemberExpression',
          object: { name: 'console' },
          property: { name: 'log' },
        },
      },
    },
    message: 'Remove console.log before committing',
    fix: 'Remove the console.log statement or use a logging library',
    documentation: 'https://example.com/coding-standards#no-console',
  },
  {
    id: 'require-error-boundary',
    name: 'Require Error Boundary',
    description: 'Components with async operations should be wrapped in ErrorBoundary',
    severity: 'info',
    pattern: {
      type: 'JSXElement',
      properties: {},
    },
    message: 'Consider wrapping this component with an ErrorBoundary',
  },
];
```

### Task 9.3: Dependency Analysis

**Model:** Opus (graph algorithms)
**File:** `src/analyzers/dependency-analyzer.ts`

```typescript
import * as t from '@babel/types';
import traverse from '@babel/traverse';
import * as path from 'path';
import { BaseAnalyzer, BaseAnalyzerResult } from './base-analyzer';

export interface DependencyNode {
  filePath: string;
  imports: string[];
  exports: string[];
  isComponent: boolean;
}

export interface DependencyEdge {
  from: string;
  to: string;
  type: 'import' | 'dynamic-import' | 're-export';
}

export interface CircularDependency {
  cycle: string[];
  severity: 'critical' | 'warning';
  impact: string;
}

export interface DependencyMetrics {
  score: number;
  /** Total number of dependencies */
  totalDependencies: number;
  /** Direct dependencies */
  directDependencies: number;
  /** Transitive dependencies */
  transitiveDependencies: number;
  /** Circular dependencies found */
  circularDependencies: CircularDependency[];
  /** Maximum dependency depth */
  dependencyDepth: number;
  /** Fan-in (components depending on this) */
  fanIn: number;
  /** Fan-out (dependencies of this component) */
  fanOut: number;
  /** Instability index (Fan-out / (Fan-in + Fan-out)) */
  instability: number;
  /** Coupling between components */
  couplingScore: number;
}

export class DependencyAnalyzer extends BaseAnalyzer<DependencyMetrics & BaseAnalyzerResult> {
  private filePath?: string;
  private projectFiles: Map<string, DependencyNode> = new Map();

  constructor(ast: t.File, filePath?: string) {
    super(ast);
    this.filePath = filePath;
  }

  /**
   * Set project context for cross-file analysis
   */
  setProjectContext(files: Map<string, DependencyNode>) {
    this.projectFiles = files;
  }

  protected analyzeAST(): Omit<DependencyMetrics & BaseAnalyzerResult, 'metadata'> {
    const node = this.analyzeFile();

    // Calculate metrics
    const directDependencies = node.imports.length;
    const fanOut = directDependencies;

    // Calculate fan-in (requires project context)
    let fanIn = 0;
    if (this.filePath && this.projectFiles.size > 0) {
      this.projectFiles.forEach(otherNode => {
        if (otherNode.imports.includes(this.filePath!)) {
          fanIn++;
        }
      });
    }

    // Instability index (0 = stable, 1 = unstable)
    const instability = fanIn + fanOut > 0 ? fanOut / (fanIn + fanOut) : 0;

    // Find circular dependencies
    const circularDependencies = this.findCircularDependencies(node);

    // Calculate transitive dependencies
    const transitiveDependencies = this.countTransitiveDependencies(node);

    // Dependency depth
    const dependencyDepth = this.calculateDepth(node);

    // Coupling score
    const couplingScore = this.calculateCouplingScore(node);

    // Overall score (lower dependencies = higher score)
    const score = Math.max(
      0,
      100 - directDependencies * 2 - circularDependencies.length * 20 - instability * 20
    );

    return {
      score,
      totalDependencies: directDependencies + transitiveDependencies,
      directDependencies,
      transitiveDependencies,
      circularDependencies,
      dependencyDepth,
      fanIn,
      fanOut,
      instability,
      couplingScore,
    };
  }

  private analyzeFile(): DependencyNode {
    const imports: string[] = [];
    const exports: string[] = [];
    let isComponent = false;

    traverse(this.ast, {
      ImportDeclaration: path => {
        const source = path.node.source.value;
        // Only track local imports
        if (source.startsWith('.') || source.startsWith('/')) {
          imports.push(this.resolveImport(source));
        }
      },

      ExportNamedDeclaration: path => {
        if (path.node.declaration) {
          if (t.isFunctionDeclaration(path.node.declaration)) {
            exports.push(path.node.declaration.id?.name ?? '');
          } else if (t.isVariableDeclaration(path.node.declaration)) {
            path.node.declaration.declarations.forEach(d => {
              if (t.isIdentifier(d.id)) {
                exports.push(d.id.name);
              }
            });
          }
        }
      },

      ExportDefaultDeclaration: path => {
        exports.push('default');
      },

      // Detect if this is a React component
      JSXElement: () => {
        isComponent = true;
      },
    });

    return {
      filePath: this.filePath ?? '',
      imports,
      exports,
      isComponent,
    };
  }

  private resolveImport(importPath: string): string {
    if (!this.filePath) return importPath;

    const dir = path.dirname(this.filePath);
    let resolved = path.resolve(dir, importPath);

    // Add .tsx extension if not present
    if (!resolved.match(/\.(tsx|ts|js|jsx)$/)) {
      resolved += '.tsx';
    }

    return resolved;
  }

  private findCircularDependencies(node: DependencyNode): CircularDependency[] {
    const cycles: CircularDependency[] = [];

    if (this.projectFiles.size === 0) return cycles;

    // DFS for cycle detection
    const visited = new Set<string>();
    const recursionStack = new Set<string>();
    const path: string[] = [];

    const dfs = (current: string) => {
      if (recursionStack.has(current)) {
        // Found cycle
        const cycleStart = path.indexOf(current);
        const cycle = [...path.slice(cycleStart), current];
        cycles.push({
          cycle,
          severity: cycle.length `<=` 2 ? 'critical' : 'warning',
          impact:
            cycle.length `<=` 2
              ? 'Direct circular dependency causes import issues'
              : 'Indirect circular dependency may cause issues',
        });
        return;
      }

      if (visited.has(current)) return;

      visited.add(current);
      recursionStack.add(current);
      path.push(current);

      const node = this.projectFiles.get(current);
      if (node) {
        node.imports.forEach(imp => dfs(imp));
      }

      path.pop();
      recursionStack.delete(current);
    };

    dfs(node.filePath);

    return cycles;
  }

  private countTransitiveDependencies(node: DependencyNode): number {
    const visited = new Set<string>();

    const countDeps = (filePath: string): number => {
      if (visited.has(filePath)) return 0;
      visited.add(filePath);

      const n = this.projectFiles.get(filePath);
      if (!n) return 0;

      let count = 0;
      n.imports.forEach(imp => {
        count += 1 + countDeps(imp);
      });

      return count;
    };

    return countDeps(node.filePath) - node.imports.length;
  }

  private calculateDepth(node: DependencyNode): number {
    const visited = new Set<string>();

    const getDepth = (filePath: string): number => {
      if (visited.has(filePath)) return 0;
      visited.add(filePath);

      const n = this.projectFiles.get(filePath);
      if (!n || n.imports.length === 0) return 0;

      let maxDepth = 0;
      n.imports.forEach(imp => {
        maxDepth = Math.max(maxDepth, 1 + getDepth(imp));
      });

      return maxDepth;
    };

    return getDepth(node.filePath);
  }

  private calculateCouplingScore(node: DependencyNode): number {
    // Higher coupling = more dependencies and more dependents
    const dependencies = node.imports.length;
    let dependents = 0;

    this.projectFiles.forEach(n => {
      if (n.imports.includes(node.filePath)) {
        dependents++;
      }
    });

    // Coupling score (0-100, lower is better)
    return Math.min(100, (dependencies + dependents) * 5);
  }
}
```

### Task 9.4: Documentation Quality

**Model:** Sonnet (pattern detection)
**File:** `src/analyzers/documentation-analyzer.ts`

```typescript
import * as t from '@babel/types';
import traverse from '@babel/traverse';
import { BaseAnalyzer, BaseAnalyzerResult } from './base-analyzer';

export interface DocumentationMetrics {
  score: number;
  coverage: {
    components: number;
    props: number;
    functions: number;
  };
  quality: DocumentationQuality[];
  missingDocumentation: MissingDoc[];
}

export interface DocumentationQuality {
  element: string;
  hasDescription: boolean;
  hasExamples: boolean;
  hasTypeAnnotations: boolean;
  completeness: number;
}

export interface MissingDoc {
  element: string;
  type: 'component' | 'prop' | 'function' | 'hook';
  location: { line: number; column: number };
  importance: 'high' | 'medium' | 'low';
}

export class DocumentationAnalyzer extends BaseAnalyzer<DocumentationMetrics & BaseAnalyzerResult> {
  protected analyzeAST(): Omit<DocumentationMetrics & BaseAnalyzerResult, 'metadata'> {
    const components = this.findComponents();
    const functions = this.findFunctions();
    const props = this.findProps();

    const documentedComponents = components.filter(c => c.hasJSDoc);
    const documentedFunctions = functions.filter(f => f.hasJSDoc);
    const documentedProps = props.filter(p => p.hasDescription);

    const coverage = {
      components:
        components.length > 0 ? (documentedComponents.length / components.length) * 100 : 100,
      functions: functions.length > 0 ? (documentedFunctions.length / functions.length) * 100 : 100,
      props: props.length > 0 ? (documentedProps.length / props.length) * 100 : 100,
    };

    const quality = this.assessQuality(components, functions);
    const missingDocumentation = this.findMissingDocs(components, functions, props);

    // Overall score
    const score = Math.round(
      coverage.components * 0.4 + coverage.functions * 0.3 + coverage.props * 0.3
    );

    return {
      score,
      coverage,
      quality,
      missingDocumentation,
    };
  }

  private findComponents(): Array<{ name: string; hasJSDoc: boolean; jsDoc?: string }> {
    const components: Array<{ name: string; hasJSDoc: boolean; jsDoc?: string }> = [];

    traverse(this.ast, {
      FunctionDeclaration: path => {
        const name = path.node.id?.name ?? '';
        if (/^[A-Z]/.test(name)) {
          const leadingComments = path.node.leadingComments;
          const jsDocComment = leadingComments?.find(
            c => c.type === 'CommentBlock' && c.value.startsWith('*')
          );

          components.push({
            name,
            hasJSDoc: !!jsDocComment,
            jsDoc: jsDocComment?.value,
          });
        }
      },

      VariableDeclarator: path => {
        if (!t.isIdentifier(path.node.id)) return;

        const name = path.node.id.name;
        if (/^[A-Z]/.test(name) && t.isArrowFunctionExpression(path.node.init)) {
          // Check for JSDoc on parent VariableDeclaration
          const parent = path.parentPath?.parentPath;
          const leadingComments = parent?.node.leadingComments;
          const jsDocComment = leadingComments?.find(
            c => c.type === 'CommentBlock' && c.value.startsWith('*')
          );

          components.push({
            name,
            hasJSDoc: !!jsDocComment,
            jsDoc: jsDocComment?.value,
          });
        }
      },
    });

    return components;
  }

  private findFunctions(): Array<{ name: string; hasJSDoc: boolean; complexity: string }> {
    const functions: Array<{ name: string; hasJSDoc: boolean; complexity: string }> = [];

    traverse(this.ast, {
      FunctionDeclaration: path => {
        const name = path.node.id?.name ?? '';
        if (!/^[A-Z]/.test(name) && name.length > 0) {
          const hasJSDoc = !!path.node.leadingComments?.some(
            c => c.type === 'CommentBlock' && c.value.startsWith('*')
          );

          // Estimate complexity
          let branches = 0;
          traverse(t.file(t.program([path.node])), {
            IfStatement: () => branches++,
            ConditionalExpression: () => branches++,
            LogicalExpression: () => branches++,
          });

          functions.push({
            name,
            hasJSDoc,
            complexity: branches > 5 ? 'high' : branches > 2 ? 'medium' : 'low',
          });
        }
      },
    });

    return functions;
  }

  private findProps(): Array<{ name: string; hasDescription: boolean }> {
    const props: Array<{ name: string; hasDescription: boolean }> = [];

    traverse(this.ast, {
      TSPropertySignature: path => {
        if (t.isIdentifier(path.node.key)) {
          const name = path.node.key.name;
          const hasComment = !!path.node.leadingComments?.length;

          props.push({ name, hasDescription: hasComment });
        }
      },
    });

    return props;
  }

  private assessQuality(
    components: Array<{ name: string; hasJSDoc: boolean; jsDoc?: string }>,
    functions: Array<{ name: string; hasJSDoc: boolean; complexity: string }>
  ): DocumentationQuality[] {
    const quality: DocumentationQuality[] = [];

    components.forEach(c => {
      if (c.hasJSDoc && c.jsDoc) {
        quality.push({
          element: c.name,
          hasDescription: c.jsDoc.includes('@description') || c.jsDoc.length > 20,
          hasExamples: c.jsDoc.includes('@example'),
          hasTypeAnnotations: true, // TypeScript
          completeness: this.calculateCompleteness(c.jsDoc),
        });
      }
    });

    return quality;
  }

  private calculateCompleteness(jsDoc: string): number {
    let score = 0;

    if (jsDoc.length > 50) score += 25;
    if (jsDoc.includes('@param')) score += 25;
    if (jsDoc.includes('@returns') || jsDoc.includes('@return')) score += 25;
    if (jsDoc.includes('@example')) score += 25;

    return score;
  }

  private findMissingDocs(
    components: Array<{ name: string; hasJSDoc: boolean }>,
    functions: Array<{ name: string; hasJSDoc: boolean; complexity: string }>,
    props: Array<{ name: string; hasDescription: boolean }>
  ): MissingDoc[] {
    const missing: MissingDoc[] = [];

    components
      .filter(c => !c.hasJSDoc)
      .forEach(c => {
        missing.push({
          element: c.name,
          type: 'component',
          location: { line: 0, column: 0 },
          importance: 'high',
        });
      });

    functions
      .filter(f => !f.hasJSDoc && f.complexity !== 'low')
      .forEach(f => {
        missing.push({
          element: f.name,
          type: 'function',
          location: { line: 0, column: 0 },
          importance: f.complexity === 'high' ? 'high' : 'medium',
        });
      });

    return missing;
  }
}
```

## Acceptance Criteria

### Feature Requirements

- [ ] Git integration provides file history
- [ ] Circular dependencies detected
- [ ] Custom rules load from config
- [ ] Documentation coverage calculated
- [ ] All features integrate with main analyzer

### Performance

- [ ] Git analysis < 5 seconds per file
- [ ] Dependency graph builds in < 30 seconds
- [ ] Custom rules don't impact analysis speed

## Testing Instructions

### Manual Testing

1. **Test Git Analysis**

   ```bash
   npm run analyze -- src/analyzer.ts --historical
   # Expected: Shows commit history, churn rate
   ```

2. **Test Circular Dependencies**

   ```bash
   npm run analyze -- src/ --dependencies
   # Expected: Lists any circular dependencies
   ```

3. **Test Documentation Quality**
   ```bash
   npm run analyze -- src/analyzer.ts --documentation
   # Expected: Shows JSDoc coverage
   ```

## Estimated Effort

| Task                      | Model  | Estimated Time  |
| ------------------------- | ------ | --------------- |
| 9.1 Git Integration       | Opus   | 5 hours         |
| 9.2 Custom Rules Engine   | Opus   | 4 hours         |
| 9.3 Dependency Analysis   | Opus   | 4 hours         |
| 9.4 Documentation Quality | Sonnet | 3 hours         |
| 9.5 Plugin Architecture   | Opus   | 4 hours         |
| Integration Testing       | Sonnet | 3 hours         |
| Documentation             | Haiku  | 2 hours         |
| **Total**                 |        | **25-27 hours** |

---

**Document Version:** 1.0
**Created:** January 10, 2026
**Status:** Ready for Implementation
