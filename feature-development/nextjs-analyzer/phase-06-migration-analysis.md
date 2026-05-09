# Phase 6: Migration Analysis

## Status

Not Started

## Goals

Implement the Migration analysis module that detects version-specific breaking changes, deprecated APIs, and migration issues across Next.js versions 12-15, with emphasis on Next.js 15 async Request APIs and Next.js 13 App Router migration.

## Files Created

### Implementation Files

- `analyses/migration-analysis.ts` - MigrationAnalysis class implementing IAnalysis
- `analyses/migration-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Next.js 15 Fixtures

- `testing/fixtures/migration/nextjs15/sync-cookies.tsx` - Synchronous cookies() usage
- `testing/fixtures/migration/nextjs15/sync-headers.tsx` - Synchronous headers() usage
- `testing/fixtures/migration/nextjs15/sync-draft-mode.tsx` - Synchronous draftMode() usage
- `testing/fixtures/migration/nextjs15/sync-params.tsx` - Synchronous params access
- `testing/fixtures/migration/nextjs15/sync-searchparams.tsx` - Synchronous searchParams access
- `testing/fixtures/migration/nextjs15/async-params-valid.tsx` - Valid async params
- `testing/fixtures/migration/nextjs15/use-form-state.tsx` - useFormState usage
- `testing/fixtures/migration/nextjs15/middleware-geo-ip.tsx` - request.geo/ip in middleware
- `testing/fixtures/migration/nextjs15/deprecated-config.js` - Deprecated config options
- `testing/fixtures/migration/nextjs15/get-route-no-cache.ts` - GET route handler

#### Next.js 13 Migration Fixtures

- `testing/fixtures/migration/nextjs13/legacy-image.tsx` - next/legacy/image import
- `testing/fixtures/migration/nextjs13/deprecated-image-props.tsx` - layout/objectFit props
- `testing/fixtures/migration/nextjs13/nested-anchor.tsx` - Link with nested anchor
- `testing/fixtures/migration/nextjs13/legacy-behavior.tsx` - legacyBehavior prop
- `testing/fixtures/migration/nextjs13/deprecated-font.tsx` - @next/font import
- `testing/fixtures/migration/nextjs13/wrong-router.tsx` - next/router in app/
- `testing/fixtures/migration/nextjs13/valid-modern.tsx` - Modern Next.js 13+ patterns

## Implementation Details

### MigrationAnalysis Class

Implements `IAnalysis<unknown, MigrationComplexity>` with the following structure:

```typescript
export class MigrationAnalysis implements IAnalysis<unknown, MigrationComplexity> {
  readonly id = 'nextjs-migration';
  readonly name = 'Next.js Migration Analysis';
  readonly category = 'migration' as const;
  readonly description = 'Detects version-specific breaking changes and deprecated patterns';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<MigrationComplexity>;
}
```

### Detection Patterns

#### Next.js 15 Breaking Changes

**Async Request APIs Detection:**

```typescript
private detectAsyncRequestAPIs(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const imports = getNextJsImports(sourceFile);

  // Check for cookies, headers, draftMode imports from next/headers
  const requestAPIs = ['cookies', 'headers', 'draftMode'];
  const importedAPIs = requestAPIs.filter(api =>
    imports.has(api) && imports.get(api) === 'next/headers'
  );

  if (importedAPIs.length === 0) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr)) {
        const apiName = expr.getText();

        if (importedAPIs.includes(apiName)) {
          // Check if result is awaited
          const parent = node.getParent();
          const isAwaited = Node.isAwaitExpression(parent);

          if (!isAwaited) {
            issues.push({
              type: 'async-request-api',
              version: 15,
              severity: 'error',
              api: apiName,
              message: `${apiName}() must be awaited in Next.js 15+`,
              recommendation: `Change to: const ${apiName}Store = await ${apiName}()`,
              breaking: true,
            });
          }
        }
      }
    }
  });
}
```

**Async Params Detection:**

```typescript
private detectAsyncParams(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  // Check page components and route handlers
  const isPage = isAppRouterFile(sourceFile) && !isRouteHandler(sourceFile);
  const isRoute = isRouteHandler(sourceFile);

  if (!isPage && !isRoute) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isFunctionDeclaration(node) || Node.isArrowFunction(node)) {
      const params = node.getParameters();

      for (const param of params) {
        const name = param.getName();

        // Check for params or searchParams parameter
        if (name === 'params' || name === 'searchParams') {
          // Check if function is async
          const isAsync = Node.isFunctionDeclaration(node)
            ? node.isAsync()
            : this.isAsyncArrowFunction(node);

          if (!isAsync) {
            issues.push({
              type: 'sync-params',
              version: 15,
              severity: 'error',
              param: name,
              message: `${name} must be awaited in Next.js 15+ (function must be async)`,
              recommendation: `Make function async and await ${name}`,
              breaking: true,
            });
            continue;
          }

          // Check if param is destructured without await
          const initializer = param.getInitializer();
          if (initializer && Node.isObjectBindingPattern(param.getNameNode())) {
            // Check if destructuring is inside await
            const funcBody = node.getBody();
            if (Node.isBlock(funcBody)) {
              const hasAwaitedAccess = this.hasAwaitedParamAccess(funcBody, name);

              if (!hasAwaitedAccess) {
                issues.push({
                  type: 'sync-params',
                  version: 15,
                  severity: 'error',
                  param: name,
                  message: `${name} must be awaited before destructuring`,
                  recommendation: `Change to: const ${name}Data = await ${name}`,
                  breaking: true,
                });
              }
            }
          }
        }
      }
    }
  });
}
```

**useFormState → useActionState Migration:**

```typescript
private detectUseFormState(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const imports = getNextJsImports(sourceFile);

  if (imports.has('useFormState')) {
    const importModule = imports.get('useFormState');

    if (importModule === 'react-dom' || importModule === 'react') {
      issues.push({
        type: 'use-form-state',
        version: 15,
        severity: 'warning',
        message: 'useFormState is deprecated in React 19, use useActionState instead',
        recommendation: 'Replace useFormState with useActionState',
        breaking: false,
      });
    }
  }
}
```

**Middleware request.geo and request.ip:**

```typescript
private detectMiddlewareGeoIP(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const filePath = sourceFile.getFilePath();

  // Check if file is middleware
  if (!filePath.includes('middleware')) return;

  const text = sourceFile.getFullText();

  // Check for request.geo or request.ip usage
  if (/request\.geo/.test(text)) {
    issues.push({
      type: 'middleware-geo',
      version: 15,
      severity: 'error',
      message: 'request.geo removed in Next.js 15',
      recommendation: 'Use geolocation in Route Handlers or Server Components instead',
      breaking: true,
    });
  }

  if (/request\.ip/.test(text)) {
    issues.push({
      type: 'middleware-ip',
      version: 15,
      severity: 'error',
      message: 'request.ip removed in Next.js 15',
      recommendation: 'Use request headers or geolocation service instead',
      breaking: true,
    });
  }
}
```

**Deprecated Config Options:**

```typescript
private detectDeprecatedConfig(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const filePath = sourceFile.getFilePath();

  // Check if file is next.config.*
  if (!/next\.config\.(js|ts|mjs|cjs)$/.test(filePath)) return;

  const text = sourceFile.getFullText();

  // Next.js 15 deprecated options
  const deprecatedOptions = DEPRECATED_CONFIG_OPTIONS[15];

  for (const [option, replacement] of Object.entries(deprecatedOptions)) {
    const pattern = new RegExp(`${option}\\s*:`);

    if (pattern.test(text)) {
      issues.push({
        type: 'deprecated-config',
        version: 15,
        severity: 'warning',
        option,
        message: `Config option '${option}' is deprecated in Next.js 15`,
        recommendation: replacement
          ? `Use '${replacement}' instead`
          : 'Remove this option',
        breaking: false,
      });
    }
  }

  // Check for removed options
  if (/swcMinify\s*:/.test(text)) {
    issues.push({
      type: 'removed-config',
      version: 15,
      severity: 'error',
      option: 'swcMinify',
      message: 'swcMinify removed in Next.js 15 (always enabled)',
      recommendation: 'Remove this option',
      breaking: true,
    });
  }
}
```

**GET Route Handler Caching:**

```typescript
private detectGETRouteCaching(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  if (!isRouteHandler(sourceFile)) return;

  const hasGETExport = hasHTTPMethodExports(sourceFile, ['GET']);
  if (!hasGETExport) return;

  // Check if export dynamic is set
  const text = sourceFile.getFullText();
  const hasDynamicExport = /export\s+const\s+dynamic\s*=/.test(text);

  if (!hasDynamicExport) {
    issues.push({
      type: 'get-route-no-cache',
      version: 15,
      severity: 'info',
      message: 'GET Route Handlers are no longer cached by default in Next.js 15',
      recommendation: "Add 'export const dynamic = \"force-static\"' if you want caching",
      breaking: false,
    });
  }
}
```

#### Next.js 13 Migration Patterns

**Legacy Image Import:**

```typescript
private detectLegacyImage(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const imports = getNextJsImports(sourceFile);

  if (imports.has('Image') && imports.get('Image') === 'next/legacy/image') {
    issues.push({
      type: 'legacy-image',
      version: 13,
      severity: 'warning',
      message: 'Using next/legacy/image instead of next/image',
      recommendation: 'Migrate to next/image with modern API',
      breaking: false,
    });
  }
}
```

**Deprecated Image Props:**

```typescript
private detectDeprecatedImageProps(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = image.getAttributes();

    for (const attr of attributes) {
      if (!Node.isJsxAttribute(attr)) continue;

      const attrName = attr.getName();

      // Check for deprecated props
      if (DEPRECATED_IMAGE_PROPS[13].includes(attrName)) {
        issues.push({
          type: 'deprecated-image-prop',
          version: 13,
          severity: 'warning',
          prop: attrName,
          message: `Image prop '${attrName}' is deprecated in Next.js 13+`,
          recommendation: this.getImagePropReplacement(attrName),
          breaking: false,
        });
      }
    }
  }
}

private getImagePropReplacement(prop: string): string {
  const replacements: Record<string, string> = {
    layout: 'Use fill prop or width/height',
    objectFit: 'Use style={{ objectFit: "..." }}',
    objectPosition: 'Use style={{ objectPosition: "..." }}',
    lazyBoundary: 'No longer needed',
    lazyRoot: 'No longer needed',
  };

  return replacements[prop] || 'Remove this prop';
}
```

**Link with Nested Anchor:**

```typescript
private detectNestedAnchor(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const linkUsages = getLinkUsages(sourceFile);

  for (const link of linkUsages) {
    const children = link.getJsxChildren();

    for (const child of children) {
      if (Node.isJsxElement(child)) {
        const tagName = child.getOpeningElement().getTagNameNode().getText();

        if (tagName === 'a') {
          issues.push({
            type: 'nested-anchor',
            version: 13,
            severity: 'warning',
            message: 'Link with nested <a> tag is deprecated in Next.js 13+',
            recommendation: 'Remove <a> tag, Link now renders as anchor directly',
            breaking: false,
          });
        }
      }
    }
  }
}
```

**legacyBehavior Prop:**

```typescript
private detectLegacyBehavior(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const linkUsages = getLinkUsages(sourceFile);

  for (const link of linkUsages) {
    const attrs = link.getAttributes();

    for (const attr of attrs) {
      if (Node.isJsxAttribute(attr)) {
        const attrName = attr.getName();

        if (attrName === 'legacyBehavior') {
          issues.push({
            type: 'legacy-behavior',
            version: 13,
            severity: 'warning',
            message: 'legacyBehavior prop indicates incomplete migration',
            recommendation: 'Remove legacyBehavior and nested anchor tag',
            breaking: false,
          });
        }
      }
    }
  }
}
```

**Deprecated Font Import:**

```typescript
private detectDeprecatedFont(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    if (moduleSpec.startsWith('@next/font/')) {
      issues.push({
        type: 'deprecated-font',
        version: 13,
        severity: 'warning',
        module: moduleSpec,
        message: '@next/font is deprecated, use next/font',
        recommendation: `Change import to ${moduleSpec.replace('@next/font', 'next/font')}`,
        breaking: false,
      });
    }
  }
}
```

**Wrong Router Import:**

```typescript
private detectWrongRouter(sourceFile: SourceFile, issues: MigrationIssue[]): void {
  const isAppRouter = isAppRouterFile(sourceFile);
  const wrongImport = hasWrongRouterImport(sourceFile);

  if (isAppRouter && wrongImport) {
    const correctImport = getCorrectRouterImport('app');

    issues.push({
      type: 'wrong-router',
      version: 13,
      severity: 'error',
      message: 'Using next/router in App Router (app/ directory)',
      recommendation: `Use ${correctImport} instead`,
      breaking: true,
    });
  }
}
```

### Migration Readiness Assessment

```typescript
private assessMigrationReadiness(issues: MigrationIssue[]): MigrationReadiness {
  const breakingChanges = issues.filter(i => i.breaking).length;
  const deprecations = issues.filter(i => !i.breaking).length;

  const byVersion: Record<number, number> = {};
  for (const issue of issues) {
    byVersion[issue.version] = (byVersion[issue.version] || 0) + 1;
  }

  let readiness: 'ready' | 'needs-work' | 'blocked';
  let blockers: string[] = [];

  if (breakingChanges === 0) {
    readiness = deprecations === 0 ? 'ready' : 'needs-work';
  } else {
    readiness = 'blocked';
    blockers = issues
      .filter(i => i.breaking)
      .map(i => i.message);
  }

  return {
    readiness,
    breakingChanges,
    deprecations,
    byVersion,
    blockers,
  };
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: MigrationIssue[]): MigrationComplexity {
  const readiness = this.assessMigrationReadiness(issues);

  let deductions = 0;

  // Breaking changes are heavily penalized
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.migration[issue.severity];
    deductions += severityDeduction;

    if (issue.breaking) {
      deductions += 10; // Additional penalty for breaking changes
    }

    const typeWeight = ISSUE_TYPE_WEIGHTS.migration[issue.type] || 0;
    deductions += typeWeight;
  }

  const score = roundScore(Math.max(0, 100 - deductions));

  return {
    issues,
    readiness,
    score,
  };
}
```

## Test Coverage

### Test Suite Structure (50 tests)

**Next.js 15 Tests (30 tests):**

- Detects sync cookies() usage
- Detects sync headers() usage
- Detects sync draftMode() usage
- Accepts awaited cookies()
- Accepts awaited headers()
- Detects sync params access
- Detects sync searchParams access
- Accepts async params with await
- Detects params destructuring without await
- Detects useFormState usage
- Detects request.geo in middleware
- Detects request.ip in middleware
- Detects swcMinify config (removed)
- Detects bundlePagesExternals config
- Detects GET route without dynamic export
- Tests fetch caching in routes
- Tests multiple async APIs in one file
- Tests nested await patterns
- Tests edge cases with params typing

**Next.js 13 Tests (20 tests):**

- Detects next/legacy/image import
- Detects layout prop on Image
- Detects objectFit prop on Image
- Detects Link with nested anchor
- Detects legacyBehavior prop
- Detects @next/font import
- Detects next/router in app/ directory
- Accepts next/navigation in app/
- Accepts next/router in pages/
- Tests modern Image API
- Tests modern Link API
- Tests modern font API
- Validates migration recommendations
- Tests multiple migration issues in one file

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 50 tests pass
- [ ] Detects all Next.js 15 breaking changes
- [ ] Detects all Next.js 13 migration issues
- [ ] Correctly identifies breaking vs non-breaking changes
- [ ] Provides accurate migration readiness assessment
- [ ] Includes version-specific recommendations
- [ ] Score reflects migration complexity accurately
- [ ] No false positives on modern patterns
- [ ] Handles edge cases (nested awaits, complex patterns)

## Technical Notes

### Async Detection Strategy

Multiple patterns for detecting async/await:

1. `AwaitExpression` parent of call expression
2. Function `isAsync()` method
3. Arrow function with async keyword detection
4. Variable assignment with await

### Version Constants Usage

Uses constants from `constants/nextjs-versions.ts`:

- `DEPRECATED_IMAGE_PROPS` - Props by version
- `DEPRECATED_CONFIG_OPTIONS` - Config by version
- `BREAKING_CHANGES` - Breaking changes with patterns
- `ROUTER_API_DIFFERENCES` - API differences

### Migration Readiness Levels

- `ready` - No breaking changes, no deprecations
- `needs-work` - Deprecations but no breaking changes
- `blocked` - Has breaking changes that must be fixed

### Edge Cases Handled

- Params with default values
- Destructured params in function signature
- Awaited params in variable declaration
- Multiple request APIs in one file
- Nested async functions
- Conditional await expressions
- Type annotations on params
- Spread params

### AST Patterns for Async Detection

**Await expression detection:**

```typescript
const parent = node.getParent();
const isAwaited = Node.isAwaitExpression(parent);
```

**Async function detection:**

```typescript
const isAsync = Node.isFunctionDeclaration(node)
  ? node.isAsync()
  : node.getText().startsWith('async');
```

**Param access with await:**

```typescript
private hasAwaitedParamAccess(block: Block, paramName: string): boolean {
  let hasAwait = false;

  block.forEachDescendant(node => {
    if (Node.isAwaitExpression(node)) {
      const expr = node.getExpression();
      if (Node.isIdentifier(expr) && expr.getText() === paramName) {
        hasAwait = true;
      }
    }
  });

  return hasAwait;
}
```

## Next Steps

Proceed to Phase 7: Security Analysis

With migration patterns covered, the next phase will implement security vulnerability detection including Server Actions, environment variables, and middleware security.
