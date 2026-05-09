# Phase 8: Config Analysis

## Status

Not Started

## Goals

Implement the Config analysis module that detects configuration issues in next.config.js/ts files, route configuration problems, and invalid patterns that cause build or runtime failures.

## Files Created

### Implementation Files

- `analyses/config-analysis.ts` - ConfigAnalysis class implementing IAnalysis
- `analyses/config-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Config File Fixtures

- `testing/fixtures/config/next-config/deprecated-options.js` - Deprecated config options
- `testing/fixtures/config/next-config/removed-options.js` - Removed config options
- `testing/fixtures/config/next-config/invalid-image-domains.js` - Old domains config
- `testing/fixtures/config/next-config/valid-remote-patterns.js` - Modern remotePatterns
- `testing/fixtures/config/next-config/missing-type-annotation.js` - Missing NextConfig type
- `testing/fixtures/config/next-config/experimental-app-dir.js` - Unnecessary experimental.appDir
- `testing/fixtures/config/next-config/amp-config.js` - Removed AMP support
- `testing/fixtures/config/next-config/valid-modern.ts` - Valid TypeScript config
- `testing/fixtures/config/next-config/env-config.js` - Runtime config (deprecated)
- `testing/fixtures/config/next-config/swc-minify.js` - Removed swcMinify option

#### Route Config Fixtures

- `testing/fixtures/config/routes/invalid-matcher.ts` - Invalid matcher pattern
- `testing/fixtures/config/routes/invalid-http-method.ts` - Lowercase HTTP methods
- `testing/fixtures/config/routes/invalid-static-params.ts` - Wrong generateStaticParams return
- `testing/fixtures/config/routes/valid-matcher.ts` - Valid middleware matcher
- `testing/fixtures/config/routes/valid-http-methods.ts` - Correct HTTP methods
- `testing/fixtures/config/routes/valid-static-params.ts` - Correct generateStaticParams

## Implementation Details

### ConfigAnalysis Class

Implements `IAnalysis<unknown, ConfigComplexity>` with the following structure:

```typescript
export class ConfigAnalysis implements IAnalysis<unknown, ConfigComplexity> {
  readonly id = 'nextjs-config';
  readonly name = 'Next.js Configuration Analysis';
  readonly category = 'correctness' as const;
  readonly description = 'Analyzes Next.js configuration files and route configurations';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 2 as const; // Lower cost - simpler patterns

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<ConfigComplexity>;
}
```

### Detection Patterns

#### Config File Detection

```typescript
private execute(sourceFile: SourceFile): AnalysisResult<ConfigComplexity> {
  const filePath = sourceFile.getFilePath();
  const issues: ConfigIssue[] = [];
  const insights: ComplexityInsight[] = [];

  // Check if file is next.config.*
  const isConfigFile = /next\.config\.(js|ts|mjs|cjs)$/.test(filePath);

  if (isConfigFile) {
    this.analyzeConfigFile(sourceFile, issues);
  }

  // Check route configurations in all files
  this.analyzeRouteConfig(sourceFile, issues);

  const data = this.calculateMetrics(issues);
  this.addInsights(issues, insights);

  return {
    analysisId: this.id,
    category: this.category,
    data,
    insights,
    score: data.score,
    executionTimeMs: 0,
  };
}
```

#### Deprecated Config Options

```typescript
private detectDeprecatedOptions(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  const text = sourceFile.getFullText();

  // Check for deprecated options across all versions
  for (const [version, options] of Object.entries(DEPRECATED_CONFIG_OPTIONS)) {
    for (const [option, replacement] of Object.entries(options)) {
      const pattern = new RegExp(`${option}\\s*:`, 'g');

      if (pattern.test(text)) {
        issues.push({
          type: 'deprecated-option',
          severity: 'warning',
          option,
          version: parseInt(version),
          message: `Config option '${option}' is deprecated in Next.js ${version}`,
          recommendation: replacement
            ? `Use '${replacement}' instead`
            : 'Remove this option',
        });
      }
    }
  }
}
```

**DEPRECATED_CONFIG_OPTIONS constant:**

```typescript
// In constants/nextjs-versions.ts
export const DEPRECATED_CONFIG_OPTIONS: Record<number, Record<string, string | null>> = {
  14: {
    'images.domains': 'images.remotePatterns',
    'experimental.appDir': null, // No replacement - no longer needed
    'experimental.serverActions': null,
  },
  15: {
    'experimental.bundlePagesExternals': 'bundlePagesRouterDependencies',
    'experimental.serverComponentsExternalPackages': 'serverExternalPackages',
  },
};
```

#### Removed Config Options

```typescript
private detectRemovedOptions(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  const text = sourceFile.getFullText();

  const removedOptions = [
    { option: 'swcMinify', version: 15, reason: 'SWC is now always enabled' },
    { option: 'target', version: 12, reason: 'Serverless mode removed' },
    { option: 'amp', version: 16, reason: 'AMP support removed' },
    { option: 'serverRuntimeConfig', version: 16, reason: 'Use environment variables' },
    { option: 'publicRuntimeConfig', version: 16, reason: 'Use NEXT_PUBLIC_ env vars' },
    { option: 'eslint.dirs', version: 16, reason: 'Use ESLint CLI directly' },
  ];

  for (const { option, version, reason } of removedOptions) {
    const pattern = new RegExp(`${option}\\s*:`);

    if (pattern.test(text)) {
      issues.push({
        type: 'removed-option',
        severity: 'error',
        option,
        version,
        message: `Config option '${option}' was removed in Next.js ${version}`,
        recommendation: reason,
      });
    }
  }
}
```

#### Invalid Image Configuration

```typescript
private detectInvalidImageConfig(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  const text = sourceFile.getFullText();

  // Check for old 'domains' config
  if (/images\s*:\s*{[^}]*domains\s*:/.test(text)) {
    issues.push({
      type: 'invalid-image-config',
      severity: 'warning',
      option: 'images.domains',
      message: 'images.domains is deprecated, use images.remotePatterns',
      recommendation: 'Migrate to remotePatterns for better security and flexibility',
      example: this.getRemotePatternsExample(),
    });
  }

  // Check for invalid remotePatterns
  const remotePatternsMatch = text.match(/remotePatterns\s*:\s*\[([\s\S]*?)\]/);

  if (remotePatternsMatch) {
    const patternsContent = remotePatternsMatch[1];

    // Each pattern should have at least 'hostname'
    const patterns = this.extractPatternObjects(patternsContent);

    for (const pattern of patterns) {
      if (!pattern.includes('hostname')) {
        issues.push({
          type: 'invalid-image-config',
          severity: 'error',
          message: 'remotePatterns entry missing required "hostname" property',
          recommendation: 'Each pattern must have at least a hostname',
        });
      }
    }
  }
}

private getRemotePatternsExample(): string {
  return `
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'example.com',
      pathname: '/images/**',
    },
  ],
}`;
}
```

#### Missing Type Annotation

```typescript
private detectMissingTypeAnnotation(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  const filePath = sourceFile.getFilePath();

  // Only check TypeScript config files
  if (!filePath.endsWith('.ts')) return;

  const text = sourceFile.getFullText();

  // Check for type annotation comment or import
  const hasTypeComment = /@type\s+{import\('next'\)\.NextConfig}/.test(text);
  const hasImport = /import.*NextConfig.*from\s+['"]next['"]/.test(text);

  if (!hasTypeComment && !hasImport) {
    issues.push({
      type: 'missing-type-annotation',
      severity: 'info',
      message: 'next.config.ts missing NextConfig type annotation',
      recommendation: 'Add /** @type {import("next").NextConfig} */ for better IDE support',
    });
  }
}
```

#### Experimental Options That Are Stable

```typescript
private detectUnnecessaryExperimental(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  const text = sourceFile.getFullText();

  const unnecessaryOptions = [
    { option: 'appDir', version: 14, reason: 'App Router is now stable' },
    { option: 'serverActions', version: 14, reason: 'Server Actions are now stable' },
    { option: 'serverComponentsExternalPackages', version: 15, reason: 'Now stable as serverExternalPackages' },
  ];

  for (const { option, version, reason } of unnecessaryOptions) {
    const pattern = new RegExp(`experimental\s*:\s*{[^}]*${option}\s*:`);

    if (pattern.test(text)) {
      issues.push({
        type: 'unnecessary-experimental',
        severity: 'info',
        option,
        version,
        message: `experimental.${option} is no longer needed in Next.js ${version}+`,
        recommendation: reason,
      });
    }
  }
}
```

#### Route Configuration Analysis

**Invalid Matcher Pattern:**

```typescript
private detectInvalidMatcher(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  // Check for middleware config export
  const text = sourceFile.getFullText();

  const matcherMatch = text.match(/matcher\s*:\s*['"]([^'"]+)['"]/g);

  if (matcherMatch) {
    for (const match of matcherMatch) {
      const pathMatch = match.match(/['"]([^'"]+)['"]/);

      if (pathMatch) {
        const path = pathMatch[1];

        // Matcher must start with /
        if (!path.startsWith('/')) {
          issues.push({
            type: 'invalid-matcher',
            severity: 'error',
            message: `Matcher pattern '${path}' must start with '/'`,
            recommendation: `Change to '/${path}'`,
            pattern: path,
          });
        }

        // Check for invalid regex patterns
        if (path.includes('*') && !this.isValidGlobPattern(path)) {
          issues.push({
            type: 'invalid-matcher',
            severity: 'error',
            message: `Invalid glob pattern in matcher: '${path}'`,
            recommendation: 'Use valid glob patterns like /api/* or /dashboard/:path*',
            pattern: path,
          });
        }
      }
    }
  }
}

private isValidGlobPattern(pattern: string): boolean {
  // Valid patterns:
  // /api/* - single level wildcard
  // /api/:path* - named wildcard
  // /api/(.*) - regex group (if supported)

  const validPatterns = [
    /^\/[\w/-]*\*$/, // /api/*
    /^\/[\w/-]*:\w+\*$/, // /api/:path*
    /^\/[\w/-]*\([^)]+\)/, // /api/(.*)
  ];

  return validPatterns.some(p => p.test(pattern));
}
```

**Invalid HTTP Method Exports:**

```typescript
private detectInvalidHTTPMethods(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  if (!isRouteHandler(sourceFile)) return;

  const lowercaseMethods = getLowercaseHTTPMethods(sourceFile);

  for (const method of lowercaseMethods) {
    issues.push({
      type: 'invalid-http-method',
      severity: 'error',
      method,
      message: `HTTP method export '${method}' must be uppercase`,
      recommendation: `Change to '${method.toUpperCase()}'`,
    });
  }
}
```

**Invalid generateStaticParams:**

```typescript
private detectInvalidStaticParams(sourceFile: SourceFile, issues: ConfigIssue[]): void {
  if (!isAppRouterFile(sourceFile)) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isFunctionDeclaration(node)) {
      const name = node.getName();

      if (name === 'generateStaticParams') {
        const returnType = this.inferReturnType(node);

        // Should return array of objects
        if (!this.returnsArrayOfObjects(node)) {
          issues.push({
            type: 'invalid-static-params',
            severity: 'error',
            message: 'generateStaticParams must return array of objects',
            recommendation: 'Return [{ slug: "post-1" }, { slug: "post-2" }] not ["post-1", "post-2"]',
          });
        }
      }
    }
  });
}

private returnsArrayOfObjects(func: FunctionDeclaration): boolean {
  const body = func.getBody();
  if (!Node.isBlock(body)) return false;

  // Look for return statements
  const returnStatements = body.getDescendantsOfKind(SyntaxKind.ReturnStatement);

  for (const ret of returnStatements) {
    const expr = ret.getExpression();

    if (expr && Node.isArrayLiteralExpression(expr)) {
      const elements = expr.getElements();

      // Check if all elements are objects
      const allObjects = elements.every(el =>
        Node.isObjectLiteralExpression(el)
      );

      if (!allObjects) return false;
    }
  }

  return true;
}
```

### Statistics Collection

```typescript
private collectStats(issues: ConfigIssue[]): ConfigStats {
  return {
    totalIssues: issues.length,
    byType: this.groupIssuesByType(issues),
    bySeverity: this.groupIssuesBySeverity(issues),
    deprecatedOptions: issues.filter(i => i.type === 'deprecated-option').length,
    removedOptions: issues.filter(i => i.type === 'removed-option').length,
    routeIssues: issues.filter(i =>
      ['invalid-matcher', 'invalid-http-method', 'invalid-static-params'].includes(i.type)
    ).length,
  };
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: ConfigIssue[]): ConfigComplexity {
  const stats = this.collectStats(issues);

  let deductions = 0;

  // Apply severity deductions
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.config[issue.severity];
    deductions += severityDeduction;

    // Apply issue type weights
    const typeWeight = ISSUE_TYPE_WEIGHTS.config[issue.type] || 0;
    deductions += typeWeight;
  }

  // Removed options are critical
  const removedPenalty = stats.removedOptions * 10;
  deductions += removedPenalty;

  const score = roundScore(Math.max(0, 100 - deductions));

  return {
    issues,
    stats,
    score,
  };
}
```

### Insights Generation

```typescript
private addInsights(issues: ConfigIssue[], insights: ComplexityInsight[]): void {
  const removedOptions = issues.filter(i => i.type === 'removed-option');

  if (removedOptions.length > 0) {
    insights.push({
      severity: 'error',
      category: 'correctness',
      message: `${removedOptions.length} removed config options will cause build failures`,
      recommendation: 'Remove or replace these options before upgrading Next.js',
    });
  }

  const deprecatedOptions = issues.filter(i => i.type === 'deprecated-option');

  if (deprecatedOptions.length > 0) {
    insights.push({
      severity: 'warning',
      category: 'migration',
      message: `${deprecatedOptions.length} deprecated config options should be updated`,
      recommendation: 'Update to modern alternatives to ensure future compatibility',
    });
  }

  const routeIssues = issues.filter(i =>
    ['invalid-matcher', 'invalid-http-method'].includes(i.type)
  );

  if (routeIssues.length > 0) {
    insights.push({
      severity: 'error',
      category: 'correctness',
      message: `${routeIssues.length} route configuration issues found`,
      recommendation: 'Fix matcher patterns and HTTP method exports',
    });
  }
}
```

## Test Coverage

### Test Suite Structure (40 tests)

**Config File Tests (25 tests):**

- Detects images.domains (deprecated)
- Detects experimental.appDir (unnecessary)
- Detects swcMinify (removed)
- Detects target (removed)
- Detects amp config (removed)
- Detects serverRuntimeConfig (removed)
- Detects publicRuntimeConfig (removed)
- Detects bundlePagesExternals (deprecated)
- Accepts modern remotePatterns
- Detects invalid remotePatterns (missing hostname)
- Detects missing type annotation in .ts config
- Accepts type annotation comment
- Accepts NextConfig import
- Tests multiple deprecated options in one file
- Tests mixed valid and invalid config
- Tests ESM config (.mjs)
- Tests CommonJS config (.cjs)

**Route Configuration Tests (15 tests):**

- Detects matcher without leading /
- Detects invalid glob pattern
- Accepts valid matcher patterns
- Detects lowercase HTTP method (get)
- Detects lowercase HTTP method (post)
- Accepts uppercase HTTP methods
- Detects invalid generateStaticParams (string array)
- Accepts valid generateStaticParams (object array)
- Tests multiple matchers
- Tests matcher with regex
- Tests matcher with named wildcards
- Tests HTTP methods in .ts route
- Tests generateStaticParams with TypeScript
- Tests empty matchers
- Tests catch-all matcher (warning)

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 40 tests pass
- [ ] Detects all deprecated config options
- [ ] Detects all removed config options
- [ ] Detects invalid image configuration
- [ ] Detects invalid matcher patterns
- [ ] Detects lowercase HTTP method exports
- [ ] Detects invalid generateStaticParams returns
- [ ] Provides accurate upgrade recommendations
- [ ] Score reflects config quality
- [ ] No false positives on valid modern configs
- [ ] Handles all config file extensions (.js, .ts, .mjs, .cjs)

## Technical Notes

### Config File Parsing Strategy

Uses text-based pattern matching instead of full AST parsing:

- Faster for simple config files
- Works across JS/TS/MJS/CJS variants
- Handles dynamic configs (computed values)
- Less brittle than full AST analysis

### Pattern Extraction

Helper to extract config values:

```typescript
private extractConfigValue(text: string, key: string): string | null {
  const pattern = new RegExp(`${key}\\s*:\\s*([^,}]+)`);
  const match = text.match(pattern);

  return match ? match[1].trim() : null;
}
```

### Array Pattern Extraction

For remotePatterns and other arrays:

```typescript
private extractPatternObjects(arrayContent: string): string[] {
  const objects: string[] = [];
  let braceDepth = 0;
  let current = '';

  for (const char of arrayContent) {
    if (char === '{') braceDepth++;
    if (char === '}') braceDepth--;

    current += char;

    if (braceDepth === 0 && char === '}') {
      objects.push(current.trim());
      current = '';
    }
  }

  return objects;
}
```

### Matcher Validation

Valid matcher patterns:

- `/api/*` - Single-level wildcard
- `/api/:path*` - Named wildcard
- `/dashboard/:path*` - Multiple segments
- `/((?!api|_next/static|_next/image).*)` - Regex negation
- Array of patterns: `matcher: ['/api/*', '/dashboard/*']`

Invalid patterns:

- `api/*` - Missing leading /
- `/api/**` - Invalid glob (use :path\*)
- `/api/[id]` - Wrong syntax (use :id)

### Edge Cases Handled

- Dynamic config values (variables, function calls)
- Comments within config objects
- Multiline config values
- TypeScript type annotations
- ESM vs CommonJS exports
- Computed property names
- Spread operators in config

### Config Validation Levels

- **Error**: Will cause build/runtime failures
  - Removed options
  - Invalid matcher patterns
  - Invalid HTTP methods
  - Invalid generateStaticParams return

- **Warning**: Deprecated but still works
  - Deprecated options with replacements
  - Old image domains config
  - Unnecessary experimental flags

- **Info**: Best practices
  - Missing type annotations
  - Suboptimal configurations

## Next Steps

Proceed to Phase 9: Component Analysis (optional)

With all core analysis modules complete (Server/Client, Data Fetching, Migration, Security, Config), the next optional phase would implement Next.js component-specific analysis for next/image, next/link, and next/script usage patterns.

Alternatively, proceed directly to Phase 10: Report Presenters if component analysis is not required for the initial release.
