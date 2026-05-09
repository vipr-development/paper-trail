# Phase 00: Babel to ts-morph Migration - COMPLETED ✅

**Status:** ✅ Complete (January 2026)

## Summary

The React analyzer has been fully migrated from Babel to ts-morph. All Babel dependencies have been removed, and the codebase now exclusively uses ts-morph for static analysis with full TypeScript type information.

## What Was Accomplished

✅ All analyzers migrated to ts-morph
✅ Type-aware metrics implemented
✅ Plugin system supports ts-morph
✅ All 115 tests passing
✅ Babel dependencies removed
✅ Legacy code cleaned up
✅ Type safety improved (no `any` types)

## Current Implementation

### File Structure

```
src/
├── analyzers/
│   ├── base-analyzer-tsmorph.ts       (Base class for all analyzers)
│   ├── type-analyzer.ts                (Type complexity analysis)
│   ├── reliability-analyzer-tsmorph.ts (Reliability metrics)
│   ├── dataflow-analyzer-tsmorph.ts   (Data flow analysis)
│   └── index.ts                        (Barrel exports)
├── plugins/
│   ├── plugin-interface-tsmorph.ts    (Plugin interface)
│   └── plugin-loader-tsmorph.ts       (Plugin manager)
├── types/
│   ├── index.ts                        (Core types barrel)
│   ├── plugin-tsmorph.ts              (Plugin types)
│   └── rules.ts                        (Rule types)
├── utils/
│   ├── tsmorph-helpers.ts             (ts-morph utilities)
│   └── scoring.ts                      (Scoring functions)
├── tsmorph-analyzer.ts                 (Main analyzer)
├── analyzer.ts                         (Legacy-compatible exports)
└── cli.ts                              (CLI interface)
```

## ts-morph API Reference

For future phase development, here are the key patterns used throughout the codebase:

### Basic Setup

```typescript
import { Project, SourceFile, Node, SyntaxKind } from 'ts-morph';

// Create project
const project = new Project({
  tsConfigFilePath: './tsconfig.json',
  // OR for in-memory:
  useInMemoryFileSystem: true,
  compilerOptions: {
    jsx: 2, // JsxEmit.React
    strict: true,
  },
});

// Add file
const sourceFile = project.createSourceFile('temp.tsx', code);
```

### Traversal Patterns

```typescript
// Traverse all descendants
sourceFile.forEachDescendant(node => {
  if (node.getKind() === SyntaxKind.FunctionDeclaration) {
    // Process function
  }
});

// Type-safe traversal
sourceFile.forEachDescendant(node => {
  const funcDecl = node.asKind(SyntaxKind.FunctionDeclaration);
  if (funcDecl) {
    // funcDecl is typed as FunctionDeclaration
  }
});

// Specific getters (more efficient)
const functions = sourceFile.getFunctions();
const classes = sourceFile.getClasses();
const imports = sourceFile.getImportDeclarations();
const variables = sourceFile.getVariableDeclarations();
```

### Type Guards

```typescript
if (Node.isFunctionDeclaration(node)) {
  // node is now FunctionDeclaration
}
if (Node.isCallExpression(node)) {
  const expr = node.getExpression();
  if (Node.isIdentifier(expr)) {
    const name = expr.getText();
  }
}
```

### Location Information

```typescript
const line = node.getStartLineNumber();
const col = node.getStartColumn();
const endLine = node.getEndLineNumber();
const text = node.getText();
const fullText = node.getFullText(); // includes trivia
```

### Type Analysis

```typescript
// Get type information
const type = node.getType();
const typeText = type.getText();
const isAny = typeText === 'any';

// Check for explicit type annotation
const returnTypeNode = funcDecl.getReturnTypeNode();
const hasExplicitType = !!returnTypeNode;

// Get type parameters
const typeParams = funcDecl.getTypeParameters();
```

### Common Patterns

```typescript
// Find React components
sourceFile.getFunctions().forEach(fn => {
  if (hasJsxReturn(fn) && isPascalCase(fn.getName() || '')) {
    // It's a React component
  }
});

// Find hook calls
sourceFile.forEachDescendant(node => {
  if (Node.isCallExpression(node)) {
    const expr = node.getExpression();
    if (Node.isIdentifier(expr) && expr.getText().startsWith('use')) {
      // It's a hook call
    }
  }
});

// Analyze JSX
sourceFile.forEachDescendant(node => {
  if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
    // Process JSX element
  }
});
```

## For More Details

See the original migration guide below for comprehensive examples and patterns.

---

# Original Migration Guide (Historical Reference)

## Migration Requirements

**Before (Babel pattern):**

```typescript
import * as parser from '@babel/parser';
import traverse from '@babel/traverse';

const ast = parser.parse(code, {
  sourceType: 'module',
  plugins: ['typescript', 'jsx'],
});

traverse(ast, {
  FunctionDeclaration(path) {
    // analysis
  },
  ArrowFunctionExpression(path) {
    // analysis
  },
});
```

**After (ts-morph pattern):**

```typescript
import { Project, SyntaxKind, Node, SourceFile } from 'ts-morph';

const project = new Project({ tsConfigFilePath: './tsconfig.json' });
// Or for single file without tsconfig:
// const project = new Project({ compilerOptions: { jsx: JsxEmit.React, strict: true } });

const sourceFile = project.addSourceFileAtPath(filePath);
// Or from string: project.createSourceFile('temp.tsx', code);

sourceFile.forEachDescendant(node => {
  if (Node.isFunctionDeclaration(node)) {
    // analysis
  }
  if (Node.isArrowFunction(node)) {
    // analysis
  }
});
```

### 2. Metric Calculation Translations

Translate all existing Babel-based metric calculations to ts-morph equivalents. The formulas remain identical; only the traversal API changes.

**Cyclomatic Complexity:**

```typescript
// Babel version
let complexity = 1;
traverse(ast, {
  IfStatement() {
    complexity++;
  },
  ConditionalExpression() {
    complexity++;
  },
  LogicalExpression({ node }) {
    if (node.operator === '&&' || node.operator === '||') complexity++;
  },
  SwitchCase() {
    complexity++;
  },
  ForStatement() {
    complexity++;
  },
  ForInStatement() {
    complexity++;
  },
  ForOfStatement() {
    complexity++;
  },
  WhileStatement() {
    complexity++;
  },
  DoWhileStatement() {
    complexity++;
  },
  CatchClause() {
    complexity++;
  },
});

// ts-morph version
function calculateCyclomaticComplexity(node: Node): number {
  let complexity = 1;

  node.forEachDescendant(descendant => {
    switch (descendant.getKind()) {
      case SyntaxKind.IfStatement:
      case SyntaxKind.ConditionalExpression:
      case SyntaxKind.ForStatement:
      case SyntaxKind.ForInStatement:
      case SyntaxKind.ForOfStatement:
      case SyntaxKind.WhileStatement:
      case SyntaxKind.DoStatement:
      case SyntaxKind.CatchClause:
        complexity++;
        break;
      case SyntaxKind.CaseClause:
        complexity++;
        break;
      case SyntaxKind.BinaryExpression:
        const binExpr = descendant.asKind(SyntaxKind.BinaryExpression);
        const op = binExpr?.getOperatorToken().getKind();
        if (
          op === SyntaxKind.AmpersandAmpersandToken ||
          op === SyntaxKind.BarBarToken ||
          op === SyntaxKind.QuestionQuestionToken
        ) {
          complexity++;
        }
        break;
    }
  });

  return complexity;
}
```

**Lines of Code / Parameter Count:**

```typescript
// ts-morph makes these trivial
function getLOC(node: Node): number {
  return node.getEndLineNumber() - node.getStartLineNumber() + 1;
}

function getParameterCount(fn: FunctionDeclaration | ArrowFunction): number {
  return fn.getParameters().length;
}
```

**Finding React Components:**

```typescript
import {
  Node,
  SourceFile,
  FunctionDeclaration,
  VariableDeclaration,
  ArrowFunction,
  FunctionExpression,
  SyntaxKind,
  JsxElement,
  JsxSelfClosingElement,
} from 'ts-morph';

interface ReactComponent {
  name: string;
  node: FunctionDeclaration | ArrowFunction | FunctionExpression;
  kind: 'function' | 'arrow' | 'expression';
}

function findReactComponents(sourceFile: SourceFile): ReactComponent[] {
  const components: ReactComponent[] = [];

  // Function declarations: function MyComponent() { return <div/> }
  sourceFile.getFunctions().forEach(fn => {
    if (hasJsxReturn(fn) && isPascalCase(fn.getName() || '')) {
      components.push({ name: fn.getName()!, node: fn, kind: 'function' });
    }
  });

  // Variable declarations: const MyComponent = () => <div/>
  sourceFile.getVariableDeclarations().forEach(varDecl => {
    const init = varDecl.getInitializer();
    if (Node.isArrowFunction(init) || Node.isFunctionExpression(init)) {
      if (hasJsxReturn(init) && isPascalCase(varDecl.getName())) {
        components.push({
          name: varDecl.getName(),
          node: init,
          kind: Node.isArrowFunction(init) ? 'arrow' : 'expression',
        });
      }
    }
  });

  return components;
}

function hasJsxReturn(node: Node): boolean {
  let hasJsx = false;
  node.forEachDescendant(desc => {
    if (Node.isJsxElement(desc) || Node.isJsxSelfClosingElement(desc) || Node.isJsxFragment(desc)) {
      hasJsx = true;
    }
  });
  return hasJsx;
}

function isPascalCase(name: string): boolean {
  return /^[A-Z][a-zA-Z0-9]*$/.test(name);
}
```

### 3. Add Type-Aware Metrics (NEW - ts-morph exclusive)

These metrics are impossible with Babel. Add them to your analysis output:

```typescript
import {
  Node,
  Type,
  Symbol,
  FunctionDeclaration,
  ArrowFunction,
  ParameterDeclaration,
  PropertySignature,
} from 'ts-morph';

interface TypeAwareMetrics {
  // Count of 'any' types in component (props, state, variables, returns)
  anyCount: number;

  // Whether return type is explicit vs inferred
  hasExplicitReturnType: boolean;
  inferredReturnType: string;

  // Props analysis
  propsTypeCoverage: number; // 0-100% of props with explicit types
  anyPropsCount: number; // props typed as 'any'
  optionalPropsCount: number;
  requiredPropsCount: number;

  // Hooks type safety
  untypedUseStateCount: number; // useState() without generic
  untypedUseRefCount: number; // useRef() without generic
  untypedUseReducerCount: number;

  // Generic component detection
  isGenericComponent: boolean;
  typeParameterCount: number;
}

function analyzeTypeMetrics(component: FunctionDeclaration | ArrowFunction): TypeAwareMetrics {
  const metrics: TypeAwareMetrics = {
    anyCount: 0,
    hasExplicitReturnType: false,
    inferredReturnType: '',
    propsTypeCoverage: 0,
    anyPropsCount: 0,
    optionalPropsCount: 0,
    requiredPropsCount: 0,
    untypedUseStateCount: 0,
    untypedUseRefCount: 0,
    untypedUseReducerCount: 0,
    isGenericComponent: false,
    typeParameterCount: 0,
  };

  // Check for explicit return type
  const returnTypeNode = component.getReturnTypeNode();
  metrics.hasExplicitReturnType = !!returnTypeNode;
  metrics.inferredReturnType = component.getReturnType().getText();

  // Check for generic type parameters
  const typeParams = component.getTypeParameters();
  metrics.isGenericComponent = typeParams.length > 0;
  metrics.typeParameterCount = typeParams.length;

  // Analyze props parameter
  const propsParam = component.getParameters()[0];
  if (propsParam) {
    const propsType = propsParam.getType();
    const properties = propsType.getProperties();

    let typedProps = 0;
    properties.forEach(prop => {
      const propType = prop.getTypeAtLocation(component);
      const typeText = propType.getText();

      if (typeText === 'any') {
        metrics.anyPropsCount++;
      } else {
        typedProps++;
      }

      // Check if optional (has ? or includes undefined)
      const declarations = prop.getDeclarations();
      const isOptional =
        declarations.some(d => Node.isPropertySignature(d) && d.hasQuestionToken()) ||
        propType.isNullable();

      if (isOptional) {
        metrics.optionalPropsCount++;
      } else {
        metrics.requiredPropsCount++;
      }
    });

    metrics.propsTypeCoverage =
      properties.length > 0 ? Math.round((typedProps / properties.length) * 100) : 100;
  }

  // Count all 'any' usage in component body
  component.forEachDescendant(node => {
    // Check type annotations
    if (Node.isTypeReference(node) || Node.isTypeNode(node)) {
      if (node.getText() === 'any') {
        metrics.anyCount++;
      }
    }

    // Check variable declarations with inferred 'any'
    if (Node.isVariableDeclaration(node)) {
      const type = node.getType();
      if (type.getText() === 'any' && !node.getTypeNode()) {
        metrics.anyCount++;
      }
    }

    // Check untyped hooks
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr)) {
        const hookName = expr.getText();
        const typeArgs = node.getTypeArguments();

        if (hookName === 'useState' && typeArgs.length === 0) {
          // Check if initial value provides type
          const args = node.getArguments();
          if (args.length === 0 || args[0].getType().getText() === 'undefined') {
            metrics.untypedUseStateCount++;
          }
        }
        if (hookName === 'useRef' && typeArgs.length === 0) {
          metrics.untypedUseRefCount++;
        }
        if (hookName === 'useReducer' && typeArgs.length === 0) {
          metrics.untypedUseReducerCount++;
        }
      }
    }
  });

  return metrics;
}
```

### 4. React-Specific Analysis

```typescript
interface ReactAnalysis {
  // Hook usage
  hooks: {
    name: string;
    line: number;
    hasCleanup?: boolean; // for useEffect
    dependencyCount?: number; // deps array length
    hasMissingDeps?: boolean; // common bug pattern
  }[];

  // JSX complexity
  jsxDepth: number; // max nesting level
  jsxElementCount: number;
  conditionalRenderCount: number;
  listRenderCount: number; // .map() in JSX

  // Props patterns
  propsSpreadCount: number; // {...props} usage
  inlineObjectProps: number; // prop={{}} anti-pattern
  inlineFunctionProps: number; // prop={() => {}} anti-pattern

  // State complexity
  useStateCount: number;
  useReducerCount: number;
  useContextCount: number;
  customHookCalls: string[];
}

function analyzeReactPatterns(component: Node): ReactAnalysis {
  const analysis: ReactAnalysis = {
    hooks: [],
    jsxDepth: 0,
    jsxElementCount: 0,
    conditionalRenderCount: 0,
    listRenderCount: 0,
    propsSpreadCount: 0,
    inlineObjectProps: 0,
    inlineFunctionProps: 0,
    useStateCount: 0,
    useReducerCount: 0,
    useContextCount: 0,
    customHookCalls: [],
  };

  component.forEachDescendant(node => {
    // Hook detection
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr)) {
        const name = expr.getText();

        if (name.startsWith('use')) {
          const hookInfo: ReactAnalysis['hooks'][0] = {
            name,
            line: node.getStartLineNumber(),
          };

          if (name === 'useEffect' || name === 'useLayoutEffect') {
            const args = node.getArguments();
            // Check for cleanup function
            if (args[0] && Node.isArrowFunction(args[0])) {
              const body = args[0].getBody();
              hookInfo.hasCleanup = body.getText().includes('return');
            }
            // Check deps array
            if (args[1] && Node.isArrayLiteralExpression(args[1])) {
              hookInfo.dependencyCount = args[1].getElements().length;
            }
          }

          analysis.hooks.push(hookInfo);

          // Categorize
          if (name === 'useState') analysis.useStateCount++;
          else if (name === 'useReducer') analysis.useReducerCount++;
          else if (name === 'useContext') analysis.useContextCount++;
          else if (
            !['useEffect', 'useLayoutEffect', 'useCallback', 'useMemo', 'useRef'].includes(name)
          ) {
            analysis.customHookCalls.push(name);
          }
        }
      }
    }

    // JSX analysis
    if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
      analysis.jsxElementCount++;

      // Calculate depth
      let depth = 0;
      let parent = node.getParent();
      while (parent) {
        if (Node.isJsxElement(parent)) depth++;
        parent = parent.getParent();
      }
      analysis.jsxDepth = Math.max(analysis.jsxDepth, depth);

      // Check for inline patterns (anti-patterns)
      const attributes = Node.isJsxElement(node)
        ? node.getOpeningElement().getAttributes()
        : node.getAttributes();

      attributes.forEach(attr => {
        if (Node.isJsxAttribute(attr)) {
          const init = attr.getInitializer();
          if (Node.isJsxExpression(init)) {
            const expr = init.getExpression();
            if (expr) {
              if (Node.isObjectLiteralExpression(expr)) {
                analysis.inlineObjectProps++;
              }
              if (Node.isArrowFunction(expr) || Node.isFunctionExpression(expr)) {
                analysis.inlineFunctionProps++;
              }
            }
          }
        }
        if (Node.isJsxSpreadAttribute(attr)) {
          analysis.propsSpreadCount++;
        }
      });
    }

    // Conditional rendering detection
    if (Node.isJsxExpression(node)) {
      const expr = node.getExpression();
      if (expr && (Node.isConditionalExpression(expr) || Node.isBinaryExpression(expr))) {
        analysis.conditionalRenderCount++;
      }
    }

    // List rendering detection (.map in JSX)
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isPropertyAccessExpression(expr) && expr.getName() === 'map') {
        // Check if inside JSX
        let parent = node.getParent();
        while (parent) {
          if (Node.isJsxExpression(parent)) {
            analysis.listRenderCount++;
            break;
          }
          parent = parent.getParent();
        }
      }
    }
  });

  return analysis;
}
```

### 5. Incremental Analysis Support

Structure the analyzer to support both CLI bulk analysis and VSCode single-file re-analysis:

```typescript
import { Project, SourceFile } from 'ts-morph';
import * as fs from 'fs';
import * as path from 'path';

interface AnalysisResult {
  filePath: string;
  analyzedAt: string;
  components: ComponentAnalysis[];
}

interface ComponentAnalysis {
  name: string;
  line: number;
  kind: 'function' | 'arrow' | 'expression';
  metrics: {
    cyclomaticComplexity: number;
    cognitiveComplexity: number;
    linesOfCode: number;
    parameterCount: number;
  };
  typeMetrics: TypeAwareMetrics;
  reactAnalysis: ReactAnalysis;
  debtScore: number;
  debtItems: { rule: string; severity: string; message: string; line: number }[];
}

class ReactAnalyzer {
  private project: Project;
  private cache: Map<string, AnalysisResult> = new Map();

  constructor(tsConfigPath?: string) {
    this.project = tsConfigPath
      ? new Project({ tsConfigFilePath: tsConfigPath })
      : new Project({
          compilerOptions: {
            jsx: 2, // JsxEmit.React
            strict: true,
            esModuleInterop: true,
          },
        });
  }

  // CLI: Analyze multiple files
  analyzeFiles(filePaths: string[]): AnalysisResult[] {
    const results: AnalysisResult[] = [];

    for (const filePath of filePaths) {
      const sourceFile = this.project.addSourceFileAtPath(filePath);
      results.push(this.analyzeSourceFile(sourceFile));
    }

    return results;
  }

  // CLI: Analyze entire project from tsconfig
  analyzeProject(): AnalysisResult[] {
    const results: AnalysisResult[] = [];

    for (const sourceFile of this.project.getSourceFiles()) {
      if (this.isReactFile(sourceFile)) {
        const result = this.analyzeSourceFile(sourceFile);
        results.push(result);
        this.cache.set(sourceFile.getFilePath(), result);
      }
    }

    return results;
  }

  // VSCode Extension: Single file re-analysis after changes
  reanalyzeFile(filePath: string): AnalysisResult {
    let sourceFile = this.project.getSourceFile(filePath);

    if (sourceFile) {
      // File exists in project, refresh from disk
      sourceFile.refreshFromFileSystemSync();
    } else {
      // New file, add to project
      sourceFile = this.project.addSourceFileAtPath(filePath);
    }

    const result = this.analyzeSourceFile(sourceFile);
    this.cache.set(filePath, result);
    return result;
  }

  // VSCode Extension: Analyze from in-memory content (unsaved changes)
  analyzeContent(filePath: string, content: string): AnalysisResult {
    let sourceFile = this.project.getSourceFile(filePath);

    if (sourceFile) {
      sourceFile.replaceWithText(content);
    } else {
      sourceFile = this.project.createSourceFile(filePath, content, { overwrite: true });
    }

    return this.analyzeSourceFile(sourceFile);
  }

  // Index persistence for VSCode extension fast startup
  saveIndex(outputPath: string): void {
    const index = {
      version: '1.0.0',
      generatedAt: new Date().toISOString(),
      files: Object.fromEntries(this.cache),
    };
    fs.writeFileSync(outputPath, JSON.stringify(index, null, 2));
  }

  loadIndex(indexPath: string): void {
    if (fs.existsSync(indexPath)) {
      const data = JSON.parse(fs.readFileSync(indexPath, 'utf8'));
      this.cache = new Map(Object.entries(data.files));
    }
  }

  getCache(): Map<string, AnalysisResult> {
    return this.cache;
  }

  private analyzeSourceFile(sourceFile: SourceFile): AnalysisResult {
    const components = findReactComponents(sourceFile);

    return {
      filePath: sourceFile.getFilePath(),
      analyzedAt: new Date().toISOString(),
      components: components.map(comp => {
        const metrics = {
          cyclomaticComplexity: calculateCyclomaticComplexity(comp.node),
          cognitiveComplexity: calculateCognitiveComplexity(comp.node),
          linesOfCode: getLOC(comp.node),
          parameterCount: comp.node.getParameters().length,
        };

        const typeMetrics = analyzeTypeMetrics(comp.node);
        const reactAnalysis = analyzeReactPatterns(comp.node);
        const { score, items } = calculateDebtScore(metrics, typeMetrics, reactAnalysis);

        return {
          name: comp.name,
          line: comp.node.getStartLineNumber(),
          kind: comp.kind,
          metrics,
          typeMetrics,
          reactAnalysis,
          debtScore: score,
          debtItems: items,
        };
      }),
    };
  }

  private isReactFile(sourceFile: SourceFile): boolean {
    const filePath = sourceFile.getFilePath();
    if (!filePath.match(/\.(tsx|jsx)$/)) return false;

    // Quick check for React imports or JSX
    const text = sourceFile.getText();
    return text.includes('react') || text.includes('React') || text.includes('<');
  }
}

// Export for CLI
export { ReactAnalyzer };
```

### 6. CLI Interface

```typescript
#!/usr/bin/env node
import { Command } from 'commander';
import { ReactAnalyzer } from './analyzer';
import * as fs from 'fs';

const program = new Command();

program.name('vipr').description('Analyze React components for technical debt').version('1.0.0');

program
  .command('analyze')
  .description('Analyze React files')
  .argument('<files...>', 'Files or glob patterns to analyze')
  .option('-o, --output <path>', 'Output JSON file path')
  .option('-c, --config <path>', 'Path to tsconfig.json')
  .option('--format <type>', 'Output format: json, summary', 'json')
  .action((files, options) => {
    const analyzer = new ReactAnalyzer(options.config);
    const results = analyzer.analyzeFiles(files);

    if (options.format === 'summary') {
      printSummary(results);
    } else {
      const output = JSON.stringify(results, null, 2);
      if (options.output) {
        fs.writeFileSync(options.output, output);
        console.log(`Results written to ${options.output}`);
      } else {
        console.log(output);
      }
    }
  });

program
  .command('index')
  .description('Index entire project for VSCode extension')
  .option('-c, --config <path>', 'Path to tsconfig.json', './tsconfig.json')
  .option('-o, --output <path>', 'Output index path', './.vipr/index.json')
  .action(options => {
    const analyzer = new ReactAnalyzer(options.config);
    const results = analyzer.analyzeProject();

    // Ensure directory exists
    const dir = require('path').dirname(options.output);
    if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });

    analyzer.saveIndex(options.output);
    console.log(`Indexed ${results.length} files to ${options.output}`);
  });

program.parse();

function printSummary(results: AnalysisResult[]) {
  const totalComponents = results.reduce((sum, r) => sum + r.components.length, 0);
  const avgComplexity =
    results.reduce(
      (sum, r) => sum + r.components.reduce((s, c) => s + c.metrics.cyclomaticComplexity, 0),
      0
    ) / totalComponents;
  const totalDebt = results.reduce(
    (sum, r) => sum + r.components.reduce((s, c) => s + c.debtScore, 0),
    0
  );

  console.log(`\nReact Analysis Summary`);
  console.log(`======================`);
  console.log(`Files analyzed: ${results.length}`);
  console.log(`Components found: ${totalComponents}`);
  console.log(`Average complexity: ${avgComplexity.toFixed(2)}`);
  console.log(`Total debt score: ${totalDebt}`);

  // Top 5 hotspots
  const hotspots = results
    .flatMap(r => r.components.map(c => ({ ...c, file: r.filePath })))
    .sort((a, b) => b.debtScore - a.debtScore)
    .slice(0, 5);

  console.log(`\nTop 5 Hotspots:`);
  hotspots.forEach((h, i) => {
    console.log(`  ${i + 1}. ${h.name} (${h.file}:${h.line}) - Debt: ${h.debtScore}`);
  });
}
```

### 7. Hybrid Approach for Large Codebases (Optional Optimization)

For codebases with 1000+ files, use SWC for fast pre-filtering before ts-morph deep analysis:

```typescript
import { parseSync } from '@swc/core';
import { Project } from 'ts-morph';
import * as fs from 'fs';

class HybridAnalyzer {
  private project: Project;

  constructor(tsConfigPath?: string) {
    this.project = new Project({
      tsConfigFilePath: tsConfigPath,
      skipAddingFilesFromTsConfig: true, // We'll add files selectively
    });
  }

  // Fast pass: identify React files using SWC (~10x faster than ts-morph for parsing)
  identifyReactFiles(filePaths: string[]): string[] {
    return filePaths.filter(filePath => {
      try {
        const code = fs.readFileSync(filePath, 'utf8');

        // Super fast parse with SWC
        const ast = parseSync(code, {
          syntax: 'typescript',
          tsx: true,
        });

        // Quick heuristic: contains JSX
        const astString = JSON.stringify(ast);
        return astString.includes('"JSXElement"') || astString.includes('"JSXFragment"');
      } catch {
        return false;
      }
    });
  }

  // Full analysis: only on identified React files
  analyzeWithFiltering(allFiles: string[]): AnalysisResult[] {
    console.time('SWC filter pass');
    const reactFiles = this.identifyReactFiles(allFiles);
    console.timeEnd('SWC filter pass');
    console.log(`Filtered ${allFiles.length} files down to ${reactFiles.length} React files`);

    console.time('ts-morph deep analysis');
    // Now add only React files to ts-morph project
    reactFiles.forEach(f => this.project.addSourceFileAtPath(f));

    const results: AnalysisResult[] = [];
    for (const sf of this.project.getSourceFiles()) {
      results.push(this.analyzeSourceFile(sf));
    }
    console.timeEnd('ts-morph deep analysis');

    return results;
  }

  private analyzeSourceFile(sourceFile: SourceFile): AnalysisResult {
    // ... same as ReactAnalyzer.analyzeSourceFile
  }
}

// Usage example:
// const analyzer = new HybridAnalyzer('./tsconfig.json');
// const allTsxFiles = glob.sync('src/**/*.tsx');
// const results = analyzer.analyzeWithFiltering(allTsxFiles);
```

**Performance expectations:**

- 10,000 files with SWC pre-filter: ~3-5 seconds
- ts-morph on ~1,000 actual React files: ~30-60 seconds
- vs ts-morph on all 10,000 files: ~5-10 minutes

Use hybrid approach when: `totalFiles > 500 && expectedReactFileRatio < 0.3`

### Update the CLI output

- Update `cli.ts` to include the new metrics, keeping the format of the visual output largely the same (the design is sufficient, just add or adjust metrics)

---

## Migration Checklist

- [ ] Replace `@babel/parser` import with `ts-morph` Project
- [ ] Replace `@babel/traverse` with `node.forEachDescendant()` or specific getters
- [ ] Update all visitor patterns to ts-morph syntax kind checks
- [ ] Migrate metric calculation functions (formulas unchanged, API different)
- [ ] Add new type-aware metrics (anyCount, propsTypeCoverage, etc.)
- [ ] Add/Update/Improve React-specific analysis (hooks, JSX patterns)
- [ ] Implement `reanalyzeFile()` for incremental updates
- [ ] Add `analyzeContent()` for unsaved file analysis
- [ ] Add index save/load for VSCode extension persistence
- [ ] Update CLI to use new analyzer class
- [ ] Add tests for ts-morph-based analysis
- [ ] (Optional) Implement hybrid SWC pre-filter for large codebases

## Dependencies Update

```json
{
  "dependencies": {
    "ts-morph": "^22.0.0",
    "commander": "^12.0.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0"
  },
  "optionalDependencies": {
    "@swc/core": "^1.4.0"
  }
}
```

Remove:

```json
{
  "@babel/parser": "...",
  "@babel/traverse": "...",
  "@babel/types": "..."
}
```
