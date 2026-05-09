# Phase 7: Security Analysis

## Status

Not Started

## Goals

Implement the Security analysis module that detects Next.js-specific security vulnerabilities including Server Action authentication and validation issues, environment variable exposure, middleware security issues, and CVE-2025-29927 detection.

## Files Created

### Implementation Files

- `analyses/security-analysis.ts` - SecurityAnalysis class implementing IAnalysis
- `analyses/security-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Server Action Fixtures

- `testing/fixtures/security/server-action/no-auth.ts` - Server Action without authentication
- `testing/fixtures/security/server-action/no-validation.ts` - Server Action without input validation
- `testing/fixtures/security/server-action/non-async.ts` - Non-async Server Action
- `testing/fixtures/security/server-action/valid-with-auth.ts` - Server Action with auth
- `testing/fixtures/security/server-action/valid-with-validation.ts` - Server Action with validation
- `testing/fixtures/security/server-action/public-action.ts` - Intentionally public action

#### Environment Variable Fixtures

- `testing/fixtures/security/env/public-secret.env` - NEXT*PUBLIC* with sensitive data
- `testing/fixtures/security/env/dynamic-access.tsx` - Dynamic env var access
- `testing/fixtures/security/env/client-non-public.tsx` - Client accessing non-public var
- `testing/fixtures/security/env/valid-public.tsx` - Valid NEXT*PUBLIC* usage
- `testing/fixtures/security/env/valid-server.tsx` - Valid server-side env usage

#### Middleware Fixtures

- `testing/fixtures/security/middleware/no-matcher.ts` - Middleware without matcher
- `testing/fixtures/security/middleware/node-apis.ts` - Node.js APIs in Edge Runtime
- `testing/fixtures/security/middleware/heavy-operations.ts` - Heavy operations in middleware
- `testing/fixtures/security/middleware/valid-middleware.ts` - Valid middleware
- `testing/fixtures/security/middleware/cve-vulnerable.ts` - CVE-2025-29927 detection example

## Implementation Details

### SecurityAnalysis Class

Implements `IAnalysis<unknown, SecurityComplexity>` with the following structure:

```typescript
export class SecurityAnalysis implements IAnalysis<unknown, SecurityComplexity> {
  readonly id = 'nextjs-security';
  readonly name = 'Next.js Security Analysis';
  readonly category = 'security' as const;
  readonly description =
    'Detects security vulnerabilities including Server Actions, env vars, and middleware issues';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 4 as const; // Higher cost due to comprehensive checks

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<SecurityComplexity>;
}
```

### Detection Patterns

#### Server Action Security Detection

**Missing Authentication:**

```typescript
private detectServerActionAuth(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  // Only check files with 'use server' directive
  if (!hasUseServerDirective(sourceFile)) return;

  const hasAuthImport = this.hasAuthLibraryImport(sourceFile);

  if (!hasAuthImport) {
    // Find all exported functions (Server Actions)
    const exportedFunctions = this.getExportedFunctions(sourceFile);

    for (const func of exportedFunctions) {
      const hasAuthCheck = this.hasAuthenticationCheck(func);

      if (!hasAuthCheck) {
        const funcName = func.getName() || 'anonymous';

        issues.push({
          type: 'server-action-no-auth',
          severity: 'critical',
          riskLevel: 'critical',
          message: `Server Action '${funcName}' lacks authentication check`,
          recommendation: 'Add authentication check at the start of the function',
          function: funcName,
          cwe: 'CWE-306',
          owasp: 'A07:2021',
        });
      }
    }
  }
}

private hasAuthLibraryImport(sourceFile: SourceFile): boolean {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    // Check against known auth libraries
    for (const pattern of AUTH_LIBRARY_PATTERNS) {
      if (pattern.test(moduleSpec)) {
        return true;
      }
    }
  }

  return false;
}

private hasAuthenticationCheck(func: FunctionDeclaration | ArrowFunction): boolean {
  const body = func.getBody();
  if (!Node.isBlock(body)) return false;

  const text = body.getText();

  // Look for common auth patterns
  const authPatterns = [
    /auth\(/,
    /getSession\(/,
    /verifyToken\(/,
    /checkAuth\(/,
    /requireAuth\(/,
    /session\s*=/,
    /user\s*=.*auth/i,
    /if\s*\(\s*!.*session/,
    /if\s*\(\s*!.*user/,
    /throw.*[Uu]nauthorized/,
    /throw.*[Ff]orbidden/,
  ];

  return authPatterns.some(pattern => pattern.test(text));
}
```

**Missing Input Validation:**

```typescript
private detectServerActionValidation(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  if (!hasUseServerDirective(sourceFile)) return;

  const hasValidationImport = this.hasValidationLibraryImport(sourceFile);

  if (!hasValidationImport) {
    const exportedFunctions = this.getExportedFunctions(sourceFile);

    for (const func of exportedFunctions) {
      const params = func.getParameters();

      if (params.length > 0) {
        const hasValidation = this.hasInputValidation(func);

        if (!hasValidation) {
          const funcName = func.getName() || 'anonymous';

          issues.push({
            type: 'server-action-no-validation',
            severity: 'high',
            riskLevel: 'high',
            message: `Server Action '${funcName}' lacks input validation`,
            recommendation: 'Add input validation using Zod, Valibot, or similar library',
            function: funcName,
            cwe: 'CWE-20',
            owasp: 'A03:2021',
          });
        }
      }
    }
  }
}

private hasValidationLibraryImport(sourceFile: SourceFile): boolean {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    for (const pattern of VALIDATION_LIBRARY_PATTERNS) {
      if (pattern.test(moduleSpec)) {
        return true;
      }
    }
  }

  return false;
}

private hasInputValidation(func: FunctionDeclaration | ArrowFunction): boolean {
  const body = func.getBody();
  if (!Node.isBlock(body)) return false;

  const text = body.getText();

  // Look for validation patterns
  const validationPatterns = [
    /\.parse\(/,
    /\.safeParse\(/,
    /\.validate\(/,
    /\.check\(/,
    /schema\./,
    /validator\./,
    /if\s*\(!.*\.test\(/,
    /throw.*[Vv]alidation/,
  ];

  return validationPatterns.some(pattern => pattern.test(text));
}
```

**Non-Async Server Action:**

```typescript
private detectNonAsyncServerAction(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  if (!hasUseServerDirective(sourceFile)) return;

  const exportedFunctions = this.getExportedFunctions(sourceFile);

  for (const func of exportedFunctions) {
    const isAsync = Node.isFunctionDeclaration(func)
      ? func.isAsync()
      : this.isAsyncArrowFunction(func);

    if (!isAsync) {
      const funcName = func.getName() || 'anonymous';

      issues.push({
        type: 'non-async-server-action',
        severity: 'error',
        riskLevel: 'medium',
        message: `Server Action '${funcName}' is not async`,
        recommendation: 'Server Actions must be async functions',
        function: funcName,
      });
    }
  }
}
```

#### Environment Variable Security

**NEXT*PUBLIC* Misuse:**

```typescript
private detectPublicEnvMisuse(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  const text = sourceFile.getFullText();

  // Check for NEXT_PUBLIC_ with sensitive names
  for (const pattern of SENSITIVE_ENV_PATTERNS) {
    const regex = new RegExp(`NEXT_PUBLIC_${pattern.source}`, 'gi');
    const matches = text.match(regex);

    if (matches) {
      for (const match of matches) {
        issues.push({
          type: 'env-public-sensitive',
          severity: 'critical',
          riskLevel: 'critical',
          message: `Sensitive data '${match}' exposed with NEXT_PUBLIC_ prefix`,
          recommendation: 'Remove NEXT_PUBLIC_ prefix - this variable is exposed to the browser',
          variable: match,
          cwe: 'CWE-798',
          owasp: 'A02:2021',
        });
      }
    }
  }
}

// In constants/nextjs-versions.ts
const SENSITIVE_ENV_PATTERNS = [
  /API[_-]?KEY/i,
  /SECRET/i,
  /PASSWORD/i,
  /PRIVATE[_-]?KEY/i,
  /TOKEN/i,
  /DATABASE[_-]?URL/i,
  /DB[_-]?PASSWORD/i,
  /ENCRYPTION[_-]?KEY/i,
];
```

**Dynamic Env Access:**

```typescript
private detectDynamicEnvAccess(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  sourceFile.forEachDescendant(node => {
    if (Node.isPropertyAccessExpression(node)) {
      const expr = node.getExpression();

      // Check for process.env[variable]
      if (
        Node.isPropertyAccessExpression(expr) &&
        expr.getExpression().getText() === 'process' &&
        expr.getName() === 'env'
      ) {
        const parent = node.getParent();

        if (Node.isElementAccessExpression(parent)) {
          const arg = parent.getArgumentExpression();

          // Dynamic access: process.env[varName]
          if (arg && !Node.isStringLiteral(arg)) {
            issues.push({
              type: 'env-dynamic-access',
              severity: 'high',
              riskLevel: 'high',
              message: 'Dynamic environment variable access detected',
              recommendation: 'NEXT_PUBLIC_ variables require static access at build time',
            });
          }
        }
      }
    }
  });
}
```

**Client Component Env Access:**

```typescript
private detectClientEnvAccess(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  if (!hasUseClientDirective(sourceFile)) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isPropertyAccessExpression(node)) {
      const text = node.getText();

      // Match process.env.VAR_NAME
      const match = text.match(/process\.env\.(\w+)/);

      if (match) {
        const varName = match[1];

        // If it doesn't start with NEXT_PUBLIC_, it won't work
        if (!varName.startsWith('NEXT_PUBLIC_')) {
          issues.push({
            type: 'env-client-non-public',
            severity: 'error',
            riskLevel: 'medium',
            message: `Client Component accessing non-public env var '${varName}'`,
            recommendation: 'Only NEXT_PUBLIC_ variables are available in Client Components',
            variable: varName,
          });
        }
      }
    }
  });
}
```

#### Middleware Security

**CVE-2025-29927 Detection:**

```typescript
private detectMiddlewareCVE(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  const filePath = sourceFile.getFilePath();

  if (!filePath.includes('middleware')) return;

  // Check if file handles authentication
  const text = sourceFile.getFullText();
  const hasAuth =
    /auth/.test(text) ||
    /session/.test(text) ||
    /token/.test(text) ||
    /protected/.test(text);

  if (hasAuth) {
    issues.push({
      type: 'middleware-cve-2025-29927',
      severity: 'critical',
      riskLevel: 'critical',
      message: 'Middleware authentication vulnerable to bypass (CVE-2025-29927)',
      recommendation: 'Upgrade to Next.js 15.2.3+, 14.2.25+, 13.5.9+, or 12.3.5+',
      cve: 'CVE-2025-29927',
      affectedVersions: '11.1.4 - 15.2.2',
    });
  }
}
```

**Missing Matcher:**

```typescript
private detectMissingMatcher(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  const filePath = sourceFile.getFilePath();

  if (!filePath.includes('middleware')) return;

  const text = sourceFile.getFullText();

  // Check for config export with matcher
  const hasMatcherConfig = /export\s+const\s+config\s*=[\s\S]*matcher/.test(text);

  if (!hasMatcherConfig) {
    issues.push({
      type: 'middleware-no-matcher',
      severity: 'high',
      riskLevel: 'high',
      message: 'Middleware without matcher runs on all routes including static assets',
      recommendation: 'Add matcher config to specify which routes to run middleware on',
    });
  }
}
```

**Node.js APIs in Edge Runtime:**

```typescript
private detectNodeAPIsInEdge(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  const filePath = sourceFile.getFilePath();

  if (!filePath.includes('middleware')) return;

  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const moduleSpec = imp.getModuleSpecifierValue();

    // Check against Node.js built-in modules not available in Edge
    for (const nodeAPI of NODE_APIS_NOT_IN_EDGE) {
      if (moduleSpec === nodeAPI) {
        issues.push({
          type: 'middleware-node-api',
          severity: 'error',
          riskLevel: 'high',
          message: `Node.js module '${nodeAPI}' not available in Edge Runtime`,
          recommendation: 'Use Edge-compatible alternatives or move logic to Route Handler',
          module: nodeAPI,
        });
      }
    }
  }
}

// In constants/nextjs-versions.ts
const NODE_APIS_NOT_IN_EDGE = [
  'fs',
  'path',
  'crypto', // Node crypto, not Web Crypto API
  'os',
  'process',
  'child_process',
  'stream',
  'http',
  'https',
  '@prisma/client',
];
```

**Heavy Operations:**

```typescript
private detectHeavyOperations(
  sourceFile: SourceFile,
  issues: SecurityIssue[]
): void {
  const filePath = sourceFile.getFilePath();

  if (!filePath.includes('middleware')) return;

  const text = sourceFile.getFullText();

  // Look for database queries
  const hasDBQuery =
    /await\s+.*\.findMany\(/.test(text) ||
    /await\s+.*\.findUnique\(/.test(text) ||
    /await\s+.*query\(/.test(text) ||
    /await\s+.*execute\(/.test(text);

  if (hasDBQuery) {
    issues.push({
      type: 'middleware-heavy-operation',
      severity: 'high',
      riskLevel: 'high',
      message: 'Heavy database operations in middleware affect all requests',
      recommendation: 'Move database queries to Route Handlers or Server Components',
    });
  }

  // Look for complex computations
  const hasHeavyCompute =
    /for\s*\([^)]{50,}\)/.test(text) || // Long for loops
    /\.map\(.*\.filter\(/.test(text) || // Chained operations
    /JSON\.parse\(.*\.substring\(/.test(text); // Complex string operations

  if (hasHeavyCompute) {
    issues.push({
      type: 'middleware-heavy-operation',
      severity: 'medium',
      riskLevel: 'medium',
      message: 'Complex computations in middleware may impact performance',
      recommendation: 'Keep middleware logic lightweight and fast',
    });
  }
}
```

### Risk Level Assessment

```typescript
private calculateRiskLevel(issues: SecurityIssue[]): RiskLevel {
  const criticalCount = issues.filter(i => i.riskLevel === 'critical').length;
  const highCount = issues.filter(i => i.riskLevel === 'high').length;

  if (criticalCount > 0) return 'critical';
  if (highCount >= 3) return 'critical';
  if (highCount > 0) return 'high';
  if (issues.length > 5) return 'medium';
  if (issues.length > 0) return 'low';
  return 'none';
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: SecurityIssue[]): SecurityComplexity {
  const stats = {
    totalIssues: issues.length,
    byType: this.groupIssuesByType(issues),
    byRiskLevel: this.groupIssuesByRiskLevel(issues),
  };

  let deductions = 0;

  // Apply risk-based deductions
  deductions += stats.byRiskLevel.critical * SECURITY_SEVERITY_DEDUCTIONS.critical;
  deductions += stats.byRiskLevel.high * SECURITY_SEVERITY_DEDUCTIONS.high;
  deductions += stats.byRiskLevel.medium * SECURITY_SEVERITY_DEDUCTIONS.medium;
  deductions += stats.byRiskLevel.low * SECURITY_SEVERITY_DEDUCTIONS.low;

  // Apply issue type weights
  for (const [type, count] of Object.entries(stats.byType)) {
    const weight = ISSUE_TYPE_WEIGHTS.security[type] || 0;
    deductions += count * weight;
  }

  const score = roundScore(Math.max(0, 100 - deductions));
  const overallRisk = this.calculateRiskLevel(issues);

  return {
    issues,
    stats,
    score,
    overallRisk,
  };
}
```

## Test Coverage

### Test Suite Structure (55 tests)

**Server Action Tests (20 tests):**

- Detects Server Action without auth import
- Detects Server Action without auth check
- Accepts Server Action with auth() call
- Accepts Server Action with getSession()
- Accepts Server Action with session check
- Detects Server Action without validation import
- Detects Server Action without validation logic
- Accepts Server Action with Zod parse
- Accepts Server Action with safeParse
- Accepts Server Action with custom validation
- Detects non-async Server Action
- Accepts async Server Action
- Tests public Server Actions (intentional)
- Tests Server Action with both auth and validation
- Tests multiple Server Actions in one file
- Tests arrow function Server Actions
- Tests conditional auth checks
- Tests early return auth patterns

**Environment Variable Tests (15 tests):**

- Detects NEXT_PUBLIC_API_KEY
- Detects NEXT_PUBLIC_SECRET
- Detects NEXT_PUBLIC_DATABASE_URL
- Detects NEXT_PUBLIC_PRIVATE_KEY
- Accepts NEXT_PUBLIC_ANALYTICS_ID
- Detects dynamic env access
- Detects template literal env access
- Detects client non-public env access
- Accepts NEXT*PUBLIC* in client
- Accepts server env in server component
- Tests case variations (api_key, apiKey)
- Tests .env file parsing

**Middleware Tests (20 tests):**

- Detects middleware with auth (CVE flag)
- Detects middleware without matcher
- Accepts middleware with matcher
- Detects fs import in middleware
- Detects path import in middleware
- Detects crypto import in middleware
- Detects Prisma in middleware
- Accepts Web Crypto API usage
- Detects database queries in middleware
- Detects heavy computations
- Accepts lightweight middleware
- Tests matcher patterns
- Tests Edge Runtime compatibility
- Tests subrequest header vulnerability

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 55 tests pass
- [ ] Detects Server Actions without authentication
- [ ] Detects Server Actions without validation
- [ ] Detects sensitive data in NEXT*PUBLIC* variables
- [ ] Detects CVE-2025-29927 vulnerability
- [ ] Detects middleware without matcher
- [ ] Detects Node.js APIs in Edge Runtime
- [ ] Provides accurate risk level assessment
- [ ] Includes CWE and OWASP references
- [ ] No false positives on secure patterns
- [ ] Handles edge cases (conditional auth, public actions)

## Technical Notes

### Authentication Detection Strategy

Multiple patterns to avoid false positives:

1. Auth library import detection (next-auth, clerk, etc.)
2. Function call pattern matching (auth(), getSession())
3. Variable assignment patterns (session =, user =)
4. Conditional check patterns (if (!session))
5. Exception patterns (throw Unauthorized)

### Validation Library Detection

Supports major validation libraries:

- Zod (.parse, .safeParse)
- Valibot (.parse, .validate)
- Joi (.validate)
- Yup (.validate)
- Custom validators (schema., validator.)

### CVE-2025-29927 Details

Critical middleware authentication bypass:

- Affects Next.js 11.1.4 through 15.2.2
- Attack vector: x-middleware-subrequest header
- Impact: Complete auth bypass
- Fix: Upgrade to patched versions

### Environment Variable Security

Best practices enforced:

- Never use NEXT*PUBLIC* for secrets
- Only static access works at build time
- Client components can't access non-public vars
- Server-side env vars are safe

### Edge Runtime Restrictions

Node.js APIs not available in Edge:

- File system (fs, path)
- Child processes
- Node crypto (use Web Crypto instead)
- Database drivers requiring Node.js

### False Positive Prevention

Intentionally public actions are allowed:

```typescript
// Marker comment to indicate intentional public access
// @public-action - no auth required
export async function subscribeToNewsletter(email: string) {
  // No auth check - this is intentional
}
```

Detection checks for `@public-action` comment to avoid flagging.

## Next Steps

Proceed to Phase 8: Config Analysis

With security analysis complete, the final core analysis module will detect configuration issues in next.config.js files and route configuration problems.
