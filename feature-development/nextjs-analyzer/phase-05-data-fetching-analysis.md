# Phase 5: Data Fetching Analysis

## Status

Not Started

## Goals

Implement the Data Fetching analysis module that detects data fetching anti-patterns in both Pages Router and App Router, including deprecated methods, missing configurations, and inefficient patterns.

## Files Created

### Implementation Files

- `analyses/data-fetching-analysis.ts` - DataFetchingAnalysis class implementing IAnalysis
- `analyses/data-fetching-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Pages Router Fixtures

- `testing/fixtures/data-fetching/pages-router/get-initial-props.tsx` - getInitialProps usage
- `testing/fixtures/data-fetching/pages-router/static-without-paths.tsx` - getStaticProps without getStaticPaths
- `testing/fixtures/data-fetching/pages-router/static-with-paths.tsx` - Valid static generation
- `testing/fixtures/data-fetching/pages-router/wrong-export-method.tsx` - Page.getStaticProps pattern
- `testing/fixtures/data-fetching/pages-router/client-side-fetch.tsx` - useEffect fetch in SSR page
- `testing/fixtures/data-fetching/pages-router/valid-ssr.tsx` - Valid getServerSideProps
- `testing/fixtures/data-fetching/pages-router/valid-isr.tsx` - Valid ISR with revalidate

#### App Router Fixtures

- `testing/fixtures/data-fetching/app-router/fetch-own-api.tsx` - Fetching own API route
- `testing/fixtures/data-fetching/app-router/fetch-no-cache.tsx` - fetch without cache options
- `testing/fixtures/data-fetching/app-router/server-action-no-revalidate.tsx` - Mutation without revalidation
- `testing/fixtures/data-fetching/app-router/valid-fetch-cached.tsx` - fetch with cache options
- `testing/fixtures/data-fetching/app-router/valid-server-action.tsx` - Server Action with revalidation
- `testing/fixtures/data-fetching/app-router/parallel-routes.tsx` - Parallel data fetching

## Implementation Details

### DataFetchingAnalysis Class

Implements `IAnalysis<unknown, DataFetchingComplexity>` with the following structure:

```typescript
export class DataFetchingAnalysis implements IAnalysis<unknown, DataFetchingComplexity> {
  readonly id = 'nextjs-data-fetching';
  readonly name = 'Next.js Data Fetching Analysis';
  readonly category = 'performance' as const;
  readonly description = 'Analyzes data fetching patterns in Pages Router and App Router';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<DataFetchingComplexity>;
}
```

### Detection Patterns

#### Router Type Detection

First determine which router paradigm the file uses:

```typescript
private execute(sourceFile: SourceFile): AnalysisResult<DataFetchingComplexity> {
  const routerType = detectRouterType(sourceFile);
  const issues: DataFetchingIssue[] = [];

  if (routerType === 'pages') {
    this.analyzePagesRouter(sourceFile, issues);
  } else if (routerType === 'app') {
    this.analyzeAppRouter(sourceFile, issues);
  }

  // Router type 'unknown' is not analyzed
}
```

#### Pages Router Analysis

**getInitialProps Detection:**

```typescript
private detectGetInitialProps(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  const text = sourceFile.getFullText();

  // Pattern 1: Page.getInitialProps = ...
  if (/\w+\.getInitialProps\s*=/.test(text)) {
    issues.push({
      type: 'get-initial-props',
      routerType: 'pages',
      severity: 'warning',
      message: 'getInitialProps blocks Automatic Static Optimization, use getServerSideProps or getStaticProps',
      recommendation: 'Replace with getServerSideProps for server-side rendering or getStaticProps for static generation',
    });
  }

  // Pattern 2: export async function getInitialProps
  if (/export\s+async\s+function\s+getInitialProps/.test(text)) {
    issues.push({
      type: 'get-initial-props',
      routerType: 'pages',
      severity: 'warning',
      message: 'getInitialProps blocks Automatic Static Optimization',
      recommendation: 'Replace with getServerSideProps or getStaticProps',
    });
  }
}
```

**Missing getStaticPaths Detection:**

```typescript
private detectMissingStaticPaths(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  const filePath = sourceFile.getFilePath();
  const hasGetStaticProps = hasPagesRouterDataFetching(sourceFile, 'getStaticProps');
  const hasGetStaticPaths = hasPagesRouterDataFetching(sourceFile, 'getStaticPaths');

  // Check if file is a dynamic route
  const isDynamicRoute = /\/\[[\w]+\](?:\/|\.tsx?$)/.test(filePath);

  if (isDynamicRoute && hasGetStaticProps && !hasGetStaticPaths) {
    issues.push({
      type: 'missing-static-paths',
      routerType: 'pages',
      severity: 'error',
      message: 'Dynamic route with getStaticProps must export getStaticPaths',
      recommendation: 'Add getStaticPaths to define which dynamic routes to pre-render',
      filePath,
    });
  }
}
```

**Wrong Export Method Detection:**

```typescript
private detectWrongExportMethod(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  const text = sourceFile.getFullText();

  // Pattern: Page.getStaticProps = () => ...
  const wrongExportPattern = /(\w+)\.(getStaticProps|getServerSideProps|getStaticPaths)\s*=/;
  const match = text.match(wrongExportPattern);

  if (match) {
    const method = match[2];
    issues.push({
      type: 'wrong-export-method',
      routerType: 'pages',
      severity: 'error',
      message: `${method} must be exported as a named export, not as a property`,
      recommendation: `Use: export async function ${method}() { ... }`,
    });
  }
}
```

**Client-Side Fetch Detection:**

```typescript
private detectClientSideFetch(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  const hasSSRMethod = hasPagesRouterDataFetching(sourceFile, 'getServerSideProps');
  const hasStaticMethod = hasPagesRouterDataFetching(sourceFile, 'getStaticProps');

  // If page has no data fetching method, check for useEffect + fetch
  if (!hasSSRMethod && !hasStaticMethod) {
    let hasUseEffect = false;
    let hasFetch = false;

    sourceFile.forEachDescendant(node => {
      if (Node.isCallExpression(node)) {
        const expr = node.getExpression();
        if (Node.isIdentifier(expr)) {
          if (expr.getText() === 'useEffect') hasUseEffect = true;
          if (expr.getText() === 'fetch') hasFetch = true;
        }
      }
    });

    if (hasUseEffect && hasFetch) {
      issues.push({
        type: 'client-side-fetch',
        routerType: 'pages',
        severity: 'warning',
        message: 'Client-side data fetching detected, consider using getServerSideProps or getStaticProps',
        recommendation: 'Move data fetching to getServerSideProps for SSR or getStaticProps for SSG',
      });
    }
  }
}
```

#### App Router Analysis

**Fetch Own API Route Detection:**

```typescript
private detectFetchOwnAPI(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  // Only check Server Components (no 'use client')
  if (hasUseClientDirective(sourceFile)) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === 'fetch') {
        const args = node.getArguments();
        if (args.length > 0) {
          const urlArg = args[0];
          const urlText = this.getStringValue(urlArg);

          if (urlText) {
            // Check for localhost or relative API routes
            const isOwnAPI =
              urlText.includes('localhost') ||
              urlText.startsWith('/api/') ||
              urlText.startsWith('http://localhost');

            if (isOwnAPI) {
              issues.push({
                type: 'fetch-own-api',
                routerType: 'app',
                severity: 'warning',
                message: 'Fetching own API route from Server Component is inefficient',
                recommendation: 'Import and call the server function directly instead of making HTTP request',
                url: urlText,
              });
            }
          }
        }
      }
    }
  });
}
```

**Missing Cache Options Detection:**

```typescript
private detectMissingCacheOptions(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  // Only in Server Components
  if (hasUseClientDirective(sourceFile)) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === 'fetch') {
        const args = node.getArguments();

        // Check if second argument (options) exists
        if (args.length < 2) {
          issues.push({
            type: 'fetch-no-cache',
            routerType: 'app',
            severity: 'warning',
            message: 'fetch() call without cache options (Next.js 15+ defaults to no-cache)',
            recommendation: 'Add cache option: { cache: "force-cache" } or { next: { revalidate: 3600 } }',
          });
          return;
        }

        const options = args[1];
        const optionsText = options.getText();

        // Check if cache or next.revalidate is specified
        const hasCacheOption =
          optionsText.includes('cache:') ||
          optionsText.includes('next:') && optionsText.includes('revalidate');

        if (!hasCacheOption) {
          issues.push({
            type: 'fetch-no-cache',
            routerType: 'app',
            severity: 'warning',
            message: 'fetch() call without explicit cache configuration',
            recommendation: 'Specify cache behavior: { cache: "force-cache" | "no-store" } or { next: { revalidate: seconds } }',
          });
        }
      }
    }
  });
}
```

**Missing Revalidation Detection:**

```typescript
private detectMissingRevalidation(sourceFile: SourceFile, issues: DataFetchingIssue[]): void {
  // Only check Server Actions
  if (!hasUseServerDirective(sourceFile)) return;

  // Detect database mutations
  const hasMutation = this.detectDatabaseMutation(sourceFile);

  if (hasMutation) {
    // Check for revalidatePath or revalidateTag calls
    const hasRevalidate = this.hasRevalidateCalls(sourceFile);

    if (!hasRevalidate) {
      issues.push({
        type: 'no-revalidation',
        routerType: 'app',
        severity: 'warning',
        message: 'Server Action with database mutation missing revalidation',
        recommendation: 'Add revalidatePath() or revalidateTag() to update cached data',
      });
    }
  }
}

private detectDatabaseMutation(sourceFile: SourceFile): boolean {
  const text = sourceFile.getFullText();

  // Common mutation patterns
  const mutationPatterns = [
    /\.create\(/,
    /\.update\(/,
    /\.delete\(/,
    /\.upsert\(/,
    /\.insert\(/,
    /\.remove\(/,
    /INSERT\s+INTO/i,
    /UPDATE\s+\w+\s+SET/i,
    /DELETE\s+FROM/i,
  ];

  return mutationPatterns.some(pattern => pattern.test(text));
}

private hasRevalidateCalls(sourceFile: SourceFile): boolean {
  let hasRevalidate = false;

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr)) {
        const name = expr.getText();
        if (name === 'revalidatePath' || name === 'revalidateTag') {
          hasRevalidate = true;
        }
      }
    }
  });

  return hasRevalidate;
}
```

### Statistics Collection

```typescript
private collectStats(issues: DataFetchingIssue[]): DataFetchingStats {
  const stats: DataFetchingStats = {
    pagesRouter: {
      getInitialProps: 0,
      getServerSideProps: 0,
      getStaticProps: 0,
      getStaticPaths: 0,
      clientSideFetch: 0,
    },
    appRouter: {
      fetchCalls: 0,
      fetchWithCache: 0,
      serverActions: 0,
      serverActionsWithRevalidation: 0,
    },
  };

  for (const issue of issues) {
    if (issue.type === 'get-initial-props') stats.pagesRouter.getInitialProps++;
    if (issue.type === 'client-side-fetch') stats.pagesRouter.clientSideFetch++;
    if (issue.type === 'fetch-own-api' || issue.type === 'fetch-no-cache') {
      stats.appRouter.fetchCalls++;
    }
    // etc.
  }

  return stats;
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: DataFetchingIssue[]): DataFetchingComplexity {
  const stats = this.collectStats(issues);

  let deductions = 0;

  // Apply severity deductions
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.dataFetching[issue.severity];
    deductions += severityDeduction;

    // Apply issue type weights
    const typeWeight = ISSUE_TYPE_WEIGHTS.dataFetching[issue.type] || 0;
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

### Test Suite Structure (45 tests)

**Pages Router Tests (25 tests):**

- Detects getInitialProps (property assignment)
- Detects getInitialProps (function export)
- Detects missing getStaticPaths for dynamic routes
- Accepts getStaticPaths with getStaticProps
- Detects wrong export method (Page.getStaticProps)
- Detects client-side fetch with useEffect
- Accepts getServerSideProps
- Accepts getStaticProps with revalidate (ISR)
- Handles static routes without getStaticPaths
- Handles catch-all routes correctly
- Handles optional catch-all routes
- Tests multiple dynamic segments
- Tests nested dynamic routes
- Tests getStaticProps with params typing
- Tests getServerSideProps with context typing

**App Router Tests (20 tests):**

- Detects fetch to localhost
- Detects fetch to relative /api/ routes
- Detects fetch without cache options
- Detects fetch with empty options object
- Accepts fetch with cache: 'force-cache'
- Accepts fetch with cache: 'no-store'
- Accepts fetch with next.revalidate
- Detects Server Action without revalidation
- Accepts Server Action with revalidatePath
- Accepts Server Action with revalidateTag
- Tests parallel data fetching pattern
- Tests streaming with Suspense
- Tests fetch in route handlers
- Tests fetch in middleware (allowed)
- Tests database queries without cache
- Tests unstable_cache usage
- Handles multiple fetch calls in one component
- Tests generateStaticParams

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 45 tests pass
- [ ] Detects all Pages Router anti-patterns (getInitialProps, missing paths)
- [ ] Detects all App Router anti-patterns (fetch issues, missing revalidation)
- [ ] Correctly distinguishes between Pages Router and App Router
- [ ] Provides router-specific recommendations
- [ ] Collects accurate statistics for both routers
- [ ] Score calculation uses correct weights
- [ ] No false positives on valid patterns
- [ ] Handles edge cases (catch-all routes, parallel fetching)

## Technical Notes

### Router Detection Strategy

Uses `detectRouterType()` utility which checks:

1. File path (`app/` vs `pages/`)
2. Function exports (getServerSideProps vs generateMetadata)
3. Directives ('use client', 'use server')
4. Import patterns (next/router vs next/navigation)

### Dynamic Route Detection

Pattern matching for dynamic routes:

- `[id]` - Single dynamic segment
- `[...slug]` - Catch-all segment
- `[[...slug]]` - Optional catch-all segment
- Multiple segments: `[category]/[id]`

Regex pattern:

```typescript
const isDynamicRoute = /\/\[[\w.]+\](?:\/|\.tsx?$)/.test(filePath);
const isCatchAll = /\/\[\.\.\.[\w]+\]/.test(filePath);
const isOptionalCatchAll = /\/\[\[\.\.\.[\w]+\]\]/.test(filePath);
```

### Database Mutation Detection

Detects mutations by pattern matching:

- ORM methods: `.create()`, `.update()`, `.delete()`, `.upsert()`
- Raw SQL: `INSERT INTO`, `UPDATE ... SET`, `DELETE FROM`
- Query builders: `.insert()`, `.remove()`

### String Value Extraction

Helper to extract string values from AST nodes:

```typescript
private getStringValue(node: Node): string | null {
  if (Node.isStringLiteral(node)) {
    return node.getLiteralText();
  }
  if (Node.isTemplateExpression(node)) {
    // Extract static parts only
    return node.getHead().getLiteralText();
  }
  return null;
}
```

### Edge Cases Handled

- Fetch in Client Components (ignored)
- Fetch in middleware (allowed)
- getStaticProps in static routes (paths optional)
- Server Actions without mutations (no revalidation needed)
- Fetch with partial cache config (still flagged)
- Multiple fetches in one component (each analyzed)

## Next Steps

Proceed to Phase 6: Migration Analysis

With data fetching patterns covered, the next phase will detect version-specific breaking changes and deprecated patterns across Next.js 12-15.
