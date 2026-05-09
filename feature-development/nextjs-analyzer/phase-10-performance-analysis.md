# Phase 10: Performance Analysis

## Status

Not Started

## Goals

Implement the Performance analysis module that detects bundle size bloat patterns, hydration mismatch causes, and caching strategy issues that negatively impact Core Web Vitals and application performance in both Pages Router and App Router.

## Files Created

### Implementation Files

- `analyses/performance-analysis.ts` - PerformanceAnalysis class implementing IAnalysis
- `analyses/performance-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Bundle Size Fixtures

- `testing/fixtures/performance/bundle/lodash-full-import.ts` - Full lodash import
- `testing/fixtures/performance/bundle/lodash-tree-shakeable.ts` - Tree-shakeable lodash import
- `testing/fixtures/performance/bundle/barrel-file-import.tsx` - Barrel file import
- `testing/fixtures/performance/bundle/direct-import.tsx` - Direct file import
- `testing/fixtures/performance/bundle/import-star.ts` - import \* pattern
- `testing/fixtures/performance/bundle/valid-named-imports.ts` - Valid named imports

#### Hydration Mismatch Fixtures

- `testing/fixtures/performance/hydration/server-client-date.tsx` - Date() causing mismatch
- `testing/fixtures/performance/hydration/server-client-random.tsx` - Math.random() causing mismatch
- `testing/fixtures/performance/hydration/browser-api-ssr.tsx` - window/localStorage during SSR
- `testing/fixtures/performance/hydration/conditional-typeof-window.tsx` - typeof window conditional
- `testing/fixtures/performance/hydration/valid-use-effect.tsx` - Valid useEffect for client-only
- `testing/fixtures/performance/hydration/valid-dynamic-import.tsx` - Valid dynamic import with ssr: false

#### Caching Strategy Fixtures

- `testing/fixtures/performance/caching/ssr-should-be-ssg.tsx` - getServerSideProps for static data
- `testing/fixtures/performance/caching/valid-isr.tsx` - Valid ISR with revalidate
- `testing/fixtures/performance/caching/force-dynamic-unnecessary.tsx` - force-dynamic for static content
- `testing/fixtures/performance/caching/valid-dynamic.tsx` - Valid dynamic route
- `testing/fixtures/performance/caching/db-query-no-cache.ts` - Database query without cache
- `testing/fixtures/performance/caching/db-query-with-cache.ts` - Valid unstable_cache usage

## Implementation Details

### PerformanceAnalysis Class

Implements `IAnalysis<unknown, PerformanceComplexity>` with the following structure:

```typescript
export class PerformanceAnalysis implements IAnalysis<unknown, PerformanceComplexity> {
  readonly id = 'nextjs-performance';
  readonly name = 'Next.js Performance Analysis';
  readonly category = 'performance' as const;
  readonly description = 'Detects bundle bloat, hydration issues, and caching problems';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 4 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<PerformanceComplexity>;
}
```

### Detection Patterns

#### Bundle Size Bloat Detection

**Full Lodash Import:**

```typescript
private detectFullLodashImport(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    if (moduleSpec === 'lodash') {
      const defaultImport = imp.getDefaultImport();
      const namespaceImport = imp.getNamespaceImport();

      if (defaultImport || namespaceImport) {
        issues.push({
          type: 'bundle-lodash-full',
          severity: 'warning',
          category: 'bundle-size',
          message: 'Importing entire lodash library (~70KB) instead of specific functions',
          recommendation: 'Use tree-shakeable imports: import get from "lodash/get"',
          impact: 'high',
          estimatedKB: 70,
          line: imp.getStartLineNumber(),
        });
      }
    }

    // Also check for lodash-es namespace import
    if (moduleSpec === 'lodash-es') {
      const namespaceImport = imp.getNamespaceImport();
      if (namespaceImport) {
        issues.push({
          type: 'bundle-lodash-full',
          severity: 'info',
          category: 'bundle-size',
          message: 'Namespace import from lodash-es may include unused functions',
          recommendation: 'Prefer named imports for better tree-shaking',
          impact: 'medium',
          line: imp.getStartLineNumber(),
        });
      }
    }
  }
}
```

**Barrel File Import Detection:**

```typescript
private detectBarrelFileImports(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    // Check for relative imports to directories (barrel files)
    if (moduleSpec.startsWith('./') || moduleSpec.startsWith('../') || moduleSpec.startsWith('@/')) {
      // If module spec doesn't include file extension or specific file name
      // and ends with a directory-like pattern
      const isLikelyBarrel =
        !moduleSpec.match(/\.(tsx?|jsx?)$/) &&
        (moduleSpec.split('/').length >= 2 || moduleSpec.includes('@/components'));

      if (isLikelyBarrel) {
        const namedImports = imp.getNamedImports();

        if (namedImports.length > 0) {
          issues.push({
            type: 'bundle-barrel-import',
            severity: 'info',
            category: 'bundle-size',
            message: 'Importing from barrel file (index) may include unused code',
            recommendation: `Import directly: ${this.getDirectImportSuggestion(moduleSpec, namedImports)}`,
            impact: 'medium',
            moduleSpec,
            line: imp.getStartLineNumber(),
          });
        }
      }
    }
  }
}

private getDirectImportSuggestion(moduleSpec: string, namedImports: ImportSpecifier[]): string {
  if (namedImports.length === 0) return moduleSpec;

  const firstImport = namedImports[0].getName();
  return `${moduleSpec}/${firstImport}`;
}
```

**Import Star Pattern:**

```typescript
private detectImportStar(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const namespaceImport = imp.getNamespaceImport();
    const moduleSpec = imp.getModuleSpecifierValue();

    if (namespaceImport) {
      // Exclude certain libraries where namespace import is expected
      const excludedModules = ['react', 'next', 'next/router', 'next/navigation'];
      if (excludedModules.includes(moduleSpec)) continue;

      issues.push({
        type: 'bundle-import-star',
        severity: 'info',
        category: 'bundle-size',
        message: `Namespace import (import * as ${namespaceImport.getText()}) may hinder tree-shaking`,
        recommendation: 'Use named imports for better bundle optimization',
        impact: 'medium',
        moduleSpec,
        line: imp.getStartLineNumber(),
      });
    }
  }
}
```

#### Hydration Mismatch Detection

**Server-Client Value Differences:**

```typescript
private detectServerClientDifferences(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  // Look for Date usage in render
  sourceFile.forEachDescendant(node => {
    if (Node.isNewExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === 'Date') {
        // Check if used in JSX or render context
        if (this.isInRenderContext(node)) {
          issues.push({
            type: 'hydration-date',
            severity: 'error',
            category: 'hydration',
            message: 'new Date() in render causes hydration mismatch (server vs client timestamps)',
            recommendation: 'Use useEffect for client-only rendering or pass timestamp as prop',
            impact: 'high',
            line: node.getStartLineNumber(),
          });
        }
      }
    }

    // Check for Math.random()
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isPropertyAccessExpression(expr)) {
        const obj = expr.getExpression();
        const prop = expr.getName();

        if (Node.isIdentifier(obj) && obj.getText() === 'Math' && prop === 'random') {
          if (this.isInRenderContext(node)) {
            issues.push({
              type: 'hydration-random',
              severity: 'error',
              category: 'hydration',
              message: 'Math.random() in render causes hydration mismatch',
              recommendation: 'Use useEffect to set random values after hydration',
              impact: 'high',
              line: node.getStartLineNumber(),
            });
          }
        }
      }
    }
  });
}

private isInRenderContext(node: Node): boolean {
  let parent = node.getParent();

  while (parent) {
    // Check if inside JSX
    if (Node.isJsxElement(parent) || Node.isJsxExpression(parent) || Node.isJsxSelfClosingElement(parent)) {
      return true;
    }

    // Check if inside return statement of component
    if (Node.isReturnStatement(parent)) {
      const func = this.findParentFunction(parent);
      if (func && this.isComponentFunction(func)) {
        return true;
      }
    }

    parent = parent.getParent();
  }

  return false;
}
```

**Browser API During SSR:**

```typescript
private detectBrowserAPIsDuringSSR(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  // Skip Client Components
  if (hasUseClientDirective(sourceFile)) return;

  const browserAPIs = [
    'window',
    'document',
    'localStorage',
    'sessionStorage',
    'navigator',
    'location',
  ];

  sourceFile.forEachDescendant(node => {
    if (Node.isIdentifier(node)) {
      const name = node.getText();

      if (browserAPIs.includes(name)) {
        // Check if not in typeof guard or useEffect
        if (!this.isInTypeofGuard(node) && !this.isInUseEffect(node)) {
          issues.push({
            type: 'hydration-browser-api',
            severity: 'error',
            category: 'hydration',
            message: `${name} is not available during SSR, will crash or cause hydration mismatch`,
            recommendation: `Use useEffect for ${name} access or wrap in typeof ${name} !== 'undefined' check`,
            impact: 'high',
            api: name,
            line: node.getStartLineNumber(),
          });
        }
      }
    }
  });
}

private isInTypeofGuard(node: Node): boolean {
  let parent = node.getParent();

  // Check if inside typeof expression
  if (parent && Node.isTypeOfExpression(parent)) {
    return true;
  }

  // Check for typeof x !== 'undefined' pattern
  while (parent) {
    if (Node.isBinaryExpression(parent)) {
      const left = parent.getLeft();
      const operator = parent.getOperatorToken().getText();

      if ((operator === '!==' || operator === '!=') && Node.isTypeOfExpression(left)) {
        const typeofExpr = left.getExpression();
        if (typeofExpr.getText() === node.getText()) {
          return true;
        }
      }
    }

    parent = parent.getParent();
  }

  return false;
}

private isInUseEffect(node: Node): boolean {
  let parent = node.getParent();

  while (parent) {
    if (Node.isCallExpression(parent)) {
      const expr = parent.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === 'useEffect') {
        return true;
      }
    }
    parent = parent.getParent();
  }

  return false;
}
```

**Conditional Rendering on Client State:**

```typescript
private detectConditionalClientState(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  sourceFile.forEachDescendant(node => {
    if (Node.isConditionalExpression(node) || Node.isIfStatement(node)) {
      const condition = Node.isConditionalExpression(node)
        ? node.getCondition()
        : node.getExpression();

      const conditionText = condition.getText();

      // Check for typeof window checks
      if (/typeof\s+window\s*!==\s*['"]undefined['"]/.test(conditionText)) {
        // Check if different JSX is rendered
        if (this.hasDifferentJSXBranches(node)) {
          issues.push({
            type: 'hydration-conditional-client',
            severity: 'warning',
            category: 'hydration',
            message: 'Conditional rendering based on typeof window causes hydration mismatch',
            recommendation: 'Use useEffect with state to handle client-only rendering, or use dynamic import with ssr: false',
            impact: 'medium',
            line: node.getStartLineNumber(),
          });
        }
      }
    }
  });
}

private hasDifferentJSXBranches(node: ConditionalExpression | IfStatement): boolean {
  if (Node.isConditionalExpression(node)) {
    const whenTrue = node.getWhenTrue().getText();
    const whenFalse = node.getWhenFalse().getText();
    return whenTrue !== whenFalse;
  }

  if (Node.isIfStatement(node)) {
    const elseStatement = node.getElseStatement();
    return elseStatement !== undefined;
  }

  return false;
}
```

#### Caching Strategy Issues

**SSR When SSG Would Work (Pages Router):**

```typescript
private detectSSRShouldBeSSG(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const routerType = detectRouterType(sourceFile);
  if (routerType !== 'pages') return;

  const hasSSR = hasPagesRouterDataFetching(sourceFile, 'getServerSideProps');
  if (!hasSSR) return;

  // Analyze getServerSideProps to see if it fetches static data
  const ssrFunction = this.findDataFetchingFunction(sourceFile, 'getServerSideProps');
  if (!ssrFunction) return;

  const funcBody = ssrFunction.getBody();
  if (!funcBody || !Node.isBlock(funcBody)) return;

  // Check if function uses request-specific data
  const usesRequestData = this.usesRequestSpecificData(funcBody);

  if (!usesRequestData) {
    issues.push({
      type: 'caching-ssr-should-be-ssg',
      severity: 'warning',
      category: 'caching',
      message: 'Using getServerSideProps for data that appears static (same for all users)',
      recommendation: 'Consider using getStaticProps with revalidate for Incremental Static Regeneration (ISR)',
      impact: 'high',
      line: ssrFunction.getStartLineNumber(),
    });
  }
}

private usesRequestSpecificData(block: Block): boolean {
  const text = block.getText();

  // Check for context.req, context.query, cookies, etc.
  const requestPatterns = [
    /context\.req/,
    /context\.query/,
    /context\.params/,
    /cookies\(\)/,
    /headers\(\)/,
    /req\.cookies/,
    /req\.headers/,
  ];

  return requestPatterns.some(pattern => pattern.test(text));
}
```

**Unnecessary force-dynamic (App Router):**

```typescript
private detectUnnecessaryForceDynamic(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const routerType = detectRouterType(sourceFile);
  if (routerType !== 'app') return;

  // Check for export const dynamic = 'force-dynamic'
  const dynamicExport = this.findDynamicExport(sourceFile);
  if (!dynamicExport || dynamicExport !== 'force-dynamic') return;

  // Analyze component to see if it actually needs dynamic rendering
  const needsDynamic = this.componentNeedsDynamic(sourceFile);

  if (!needsDynamic) {
    issues.push({
      type: 'caching-unnecessary-force-dynamic',
      severity: 'warning',
      category: 'caching',
      message: 'Component exports force-dynamic but appears to be static',
      recommendation: 'Remove dynamic export or change to "auto" to allow static optimization',
      impact: 'medium',
      line: this.findDynamicExportLine(sourceFile),
    });
  }
}

private findDynamicExport(sourceFile: SourceFile): string | null {
  const text = sourceFile.getFullText();
  const match = text.match(/export\s+const\s+dynamic\s*=\s*['"]([^'"]+)['"]/);
  return match ? match[1] : null;
}

private componentNeedsDynamic(sourceFile: SourceFile): boolean {
  const text = sourceFile.getFullText();

  // Check for dynamic APIs
  const dynamicAPIs = [
    /cookies\(\)/,
    /headers\(\)/,
    /searchParams/,
    /\.json\(\).*request/,
  ];

  return dynamicAPIs.some(pattern => pattern.test(text));
}
```

**Database Queries Without Caching:**

```typescript
private detectDatabaseQueriesWithoutCache(sourceFile: SourceFile, issues: PerformanceIssue[]): void {
  const routerType = detectRouterType(sourceFile);
  if (routerType !== 'app') return;

  // Skip Client Components
  if (hasUseClientDirective(sourceFile)) return;

  // Look for database queries
  const dbPatterns = [
    { pattern: /prisma\.\w+\.find/, orm: 'Prisma' },
    { pattern: /db\.\w+\.find/, orm: 'Database' },
    { pattern: /query\(/, orm: 'SQL' },
  ];

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const callText = node.getExpression().getText();

      for (const { pattern, orm } of dbPatterns) {
        if (pattern.test(callText)) {
          // Check if wrapped in unstable_cache
          if (!this.isWrappedInCache(node)) {
            issues.push({
              type: 'caching-db-no-cache',
              severity: 'warning',
              category: 'caching',
              message: `${orm} query without caching will run on every request`,
              recommendation: 'Wrap in unstable_cache from next/cache for better performance',
              impact: 'high',
              line: node.getStartLineNumber(),
            });
          }
        }
      }
    }
  });
}

private isWrappedInCache(node: Node): boolean {
  let parent = node.getParent();

  while (parent) {
    if (Node.isCallExpression(parent)) {
      const expr = parent.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === 'unstable_cache') {
        return true;
      }
    }
    parent = parent.getParent();
  }

  return false;
}
```

### Statistics Collection

```typescript
private collectStats(issues: PerformanceIssue[]): PerformanceStats {
  const stats: PerformanceStats = {
    bundleIssues: {
      lodashFullImport: 0,
      barrelImports: 0,
      importStar: 0,
      estimatedWasteKB: 0,
    },
    hydrationIssues: {
      dateUsage: 0,
      randomUsage: 0,
      browserAPIs: 0,
      conditionalClient: 0,
    },
    cachingIssues: {
      ssrShouldBeSSG: 0,
      unnecessaryForceDynamic: 0,
      dbQueriesNoCache: 0,
    },
  };

  for (const issue of issues) {
    switch (issue.type) {
      case 'bundle-lodash-full':
        stats.bundleIssues.lodashFullImport++;
        if (issue.estimatedKB) stats.bundleIssues.estimatedWasteKB += issue.estimatedKB;
        break;
      case 'bundle-barrel-import':
        stats.bundleIssues.barrelImports++;
        break;
      case 'bundle-import-star':
        stats.bundleIssues.importStar++;
        break;
      case 'hydration-date':
        stats.hydrationIssues.dateUsage++;
        break;
      case 'hydration-random':
        stats.hydrationIssues.randomUsage++;
        break;
      case 'hydration-browser-api':
        stats.hydrationIssues.browserAPIs++;
        break;
      case 'hydration-conditional-client':
        stats.hydrationIssues.conditionalClient++;
        break;
      case 'caching-ssr-should-be-ssg':
        stats.cachingIssues.ssrShouldBeSSG++;
        break;
      case 'caching-unnecessary-force-dynamic':
        stats.cachingIssues.unnecessaryForceDynamic++;
        break;
      case 'caching-db-no-cache':
        stats.cachingIssues.dbQueriesNoCache++;
        break;
    }
  }

  return stats;
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: PerformanceIssue[]): PerformanceComplexity {
  const stats = this.collectStats(issues);

  let deductions = 0;

  // Apply severity deductions
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.performance[issue.severity];
    deductions += severityDeduction;

    // Apply impact multiplier
    const impactMultiplier = IMPACT_MULTIPLIERS[issue.impact];
    deductions += severityDeduction * impactMultiplier;

    // Apply issue type weights
    const typeWeight = ISSUE_TYPE_WEIGHTS.performance[issue.type] || 0;
    deductions += typeWeight;
  }

  const score = roundScore(Math.max(0, 100 - deductions));

  return {
    issues,
    stats,
    score,
  };
}
```

## Test Coverage

### Test Suite Structure (50 tests)

**Bundle Size Tests (18 tests):**

- Detects full lodash import (default)
- Detects lodash namespace import
- Accepts tree-shakeable lodash import
- Accepts lodash-es named imports
- Detects barrel file import from @/components
- Detects barrel file import from relative path
- Accepts direct file imports
- Detects import \* pattern
- Accepts import \* from react
- Accepts import \* from next
- Tests multiple barrel imports
- Tests nested barrel imports
- Tests component library imports
- Calculates estimated KB waste
- Tests lodash methods usage
- Tests lodash/fp imports
- Tests date-fns imports
- Tests moment.js full import

**Hydration Tests (20 tests):**

- Detects new Date() in JSX
- Detects new Date() in return statement
- Accepts Date in useEffect
- Detects Math.random() in render
- Accepts Math.random() in useEffect
- Detects window usage without guard
- Detects localStorage without guard
- Detects document usage without guard
- Accepts window in typeof guard
- Accepts window in useEffect
- Detects conditional render on typeof window
- Accepts dynamic import with ssr: false
- Tests navigator API
- Tests sessionStorage API
- Tests location API
- Tests multiple browser APIs
- Tests nested typeof guards
- Tests complex conditional patterns
- Tests hydration in Server Components
- Tests Client Component exemption

**Caching Tests (12 tests):**

- Detects getServerSideProps for static data
- Accepts getServerSideProps with req.cookies
- Accepts getServerSideProps with context.query
- Accepts getStaticProps with revalidate
- Detects force-dynamic without dynamic APIs
- Accepts force-dynamic with cookies()
- Accepts force-dynamic with headers()
- Detects Prisma query without cache
- Detects database query without cache
- Accepts query wrapped in unstable_cache
- Tests multiple db queries
- Tests SQL queries without cache

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 50 tests pass
- [ ] Detects all bundle size issues (lodash, barrel files, import star)
- [ ] Detects all hydration issues (Date, random, browser APIs, conditionals)
- [ ] Detects all caching issues (SSR vs SSG, force-dynamic, db queries)
- [ ] Provides performance impact assessment
- [ ] Calculates estimated KB waste for bundle issues
- [ ] Collects accurate statistics by category
- [ ] Score calculation uses impact multipliers
- [ ] No false positives on valid optimization patterns
- [ ] Handles edge cases (typeof guards, useEffect usage, cache wrappers)

## Technical Notes

### Impact Assessment

Performance issues are categorized by impact:

- `high` - Directly affects Core Web Vitals (LCP, CLS, INP)
- `medium` - Affects initial load or interactivity
- `low` - Minor optimization opportunity

### Bundle Size Estimation

Estimated KB waste calculation:

```typescript
const LIBRARY_SIZES: Record<string, number> = {
  lodash: 70,
  moment: 72,
  'date-fns': 78, // full import
  rxjs: 98,
};
```

### Render Context Detection

Multiple strategies to detect render context:

1. JSX element/expression parent
2. Return statement in component function
3. Variable declaration used in JSX
4. Props object passed to component

### Function Analysis

Helper to find and analyze functions:

```typescript
private findDataFetchingFunction(
  sourceFile: SourceFile,
  name: string
): FunctionDeclaration | ArrowFunction | undefined {
  let targetFunc: FunctionDeclaration | ArrowFunction | undefined;

  sourceFile.forEachDescendant(node => {
    if (Node.isFunctionDeclaration(node)) {
      if (node.getName() === name) {
        targetFunc = node;
      }
    }

    if (Node.isVariableDeclaration(node)) {
      if (node.getName() === name) {
        const initializer = node.getInitializer();
        if (initializer && Node.isArrowFunction(initializer)) {
          targetFunc = initializer;
        }
      }
    }
  });

  return targetFunc;
}
```

### Edge Cases Handled

**Bundle size:**

- Namespace imports from core libraries (react, next) excluded
- Named imports from lodash-es treated differently
- Component libraries with barrel exports (allowed)
- Type-only imports (excluded from analysis)

**Hydration:**

- typeof guards prevent false positives
- useEffect wrapping excludes issues
- Client Components exempt from SSR checks
- Dynamic imports with ssr: false are valid

**Caching:**

- Request-specific data detection for SSR
- Dynamic APIs detection for force-dynamic
- Cache wrapper detection for db queries
- Route handlers vs page components

### Constants

```typescript
// From constants/performance-thresholds.ts
export const SEVERITY_DEDUCTIONS = {
  performance: {
    error: 15,
    warning: 8,
    info: 3,
  },
};

export const IMPACT_MULTIPLIERS = {
  high: 1.5,
  medium: 1.0,
  low: 0.5,
};

export const ISSUE_TYPE_WEIGHTS = {
  performance: {
    'bundle-lodash-full': 10,
    'bundle-barrel-import': 5,
    'hydration-date': 12,
    'hydration-browser-api': 15,
    'caching-ssr-should-be-ssg': 10,
    'caching-db-no-cache': 12,
  },
};
```

## Next Steps

Proceed to Phase 11: Accessibility and TypeScript Analysis

With performance patterns covered, the final analysis phase will detect accessibility issues and TypeScript type safety problems specific to Next.js.
