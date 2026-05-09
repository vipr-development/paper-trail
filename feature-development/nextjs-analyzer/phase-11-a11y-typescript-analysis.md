# Phase 11: Accessibility and TypeScript Analysis

## Status

Not Started

## Goals

Implement the Accessibility and TypeScript analysis module that detects accessibility violations specific to Next.js and TypeScript type safety issues including Next.js 15 breaking changes, untyped data fetching functions, and Server Action type problems.

## Files Created

### Implementation Files

- `analyses/a11y-typescript-analysis.ts` - A11yTypeScriptAnalysis class implementing IAnalysis
- `analyses/a11y-typescript-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### Accessibility Fixtures

- `testing/fixtures/a11y-typescript/a11y/document-no-lang.tsx` - \_document without lang attribute
- `testing/fixtures/a11y-typescript/a11y/document-with-lang.tsx` - Valid \_document with lang
- `testing/fixtures/a11y-typescript/a11y/image-missing-alt.tsx` - Image without alt
- `testing/fixtures/a11y-typescript/a11y/image-redundant-alt.tsx` - Image with redundant alt text
- `testing/fixtures/a11y-typescript/a11y/image-decorative.tsx` - Decorative image with empty alt
- `testing/fixtures/a11y-typescript/a11y/image-valid-alt.tsx` - Image with descriptive alt
- `testing/fixtures/a11y-typescript/a11y/i18n-config.js` - Valid i18n config

#### TypeScript Fixtures

- `testing/fixtures/a11y-typescript/typescript/untyped-gssp.tsx` - Untyped getServerSideProps
- `testing/fixtures/a11y-typescript/typescript/typed-gssp.tsx` - Typed getServerSideProps
- `testing/fixtures/a11y-typescript/typescript/untyped-gsp.tsx` - Untyped getStaticProps
- `testing/fixtures/a11y-typescript/typescript/typed-gsp.tsx` - Typed getStaticProps
- `testing/fixtures/a11y-typescript/typescript/sync-params.tsx` - Synchronous params type (Next.js 15)
- `testing/fixtures/a11y-typescript/typescript/async-params.tsx` - Async params type (Next.js 15)
- `testing/fixtures/a11y-typescript/typescript/route-handler-untyped.ts` - Untyped route handler
- `testing/fixtures/a11y-typescript/typescript/route-handler-typed.ts` - Typed route handler
- `testing/fixtures/a11y-typescript/typescript/server-action-untyped.ts` - Untyped Server Action
- `testing/fixtures/a11y-typescript/typescript/server-action-typed.ts` - Typed Server Action

## Implementation Details

### A11yTypeScriptAnalysis Class

Implements `IAnalysis<unknown, A11yTypeScriptComplexity>` with the following structure:

```typescript
export class A11yTypeScriptAnalysis implements IAnalysis<unknown, A11yTypeScriptComplexity> {
  readonly id = 'nextjs-a11y-typescript';
  readonly name = 'Next.js Accessibility and TypeScript Analysis';
  readonly category = 'correctness' as const;
  readonly description = 'Detects accessibility issues and TypeScript type safety problems';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<A11yTypeScriptComplexity>;
}
```

### Detection Patterns

#### Accessibility Analysis

**Missing lang Attribute in \_document:**

```typescript
private detectMissingLang(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  const filePath = sourceFile.getFilePath();

  // Only check _document files
  if (!/_document\.(tsx?|jsx?)$/.test(filePath)) return;

  // Look for <Html> component from next/document
  let hasHtmlTag = false;
  let hasLangAttribute = false;

  sourceFile.forEachDescendant(node => {
    if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
      const tagName = Node.isJsxElement(node)
        ? node.getOpeningElement().getTagNameNode().getText()
        : node.getTagNameNode().getText();

      if (tagName === 'Html') {
        hasHtmlTag = true;

        const attributes = Node.isJsxElement(node)
          ? node.getOpeningElement().getAttributes()
          : node.getAttributes();

        for (const attr of attributes) {
          if (Node.isJsxAttribute(attr) && attr.getName() === 'lang') {
            hasLangAttribute = true;
            break;
          }
        }
      }
    }
  });

  if (hasHtmlTag && !hasLangAttribute) {
    issues.push({
      type: 'a11y-missing-lang',
      severity: 'warning',
      category: 'accessibility',
      message: 'Html component missing lang attribute in _document',
      recommendation: 'Add lang="en" attribute or configure i18n in next.config.js',
      wcagCriterion: '3.1.1',
      line: this.findHtmlTagLine(sourceFile),
    });
  }
}

private findHtmlTagLine(sourceFile: SourceFile): number {
  let line = 1;

  sourceFile.forEachDescendant(node => {
    if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
      const tagName = Node.isJsxElement(node)
        ? node.getOpeningElement().getTagNameNode().getText()
        : node.getTagNameNode().getText();

      if (tagName === 'Html') {
        line = node.getStartLineNumber();
      }
    }
  });

  return line;
}
```

**Missing and Redundant Alt Text:**

```typescript
private detectAltTextIssues(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    if (!attributes.has('alt')) {
      issues.push({
        type: 'a11y-missing-alt',
        severity: 'error',
        category: 'accessibility',
        message: 'Image missing alt attribute',
        recommendation: 'Add alt prop with descriptive text or empty string for decorative images',
        wcagCriterion: '1.1.1',
        line: image.getStartLineNumber(),
      });
      continue;
    }

    // Check for redundant alt text
    const altValue = this.getAttributeValue(attributes.get('alt'));
    if (altValue) {
      const redundantPatterns = [
        /^image\s+of/i,
        /^picture\s+of/i,
        /^photo\s+of/i,
        /^graphic\s+of/i,
        /^icon\s+of/i,
      ];

      for (const pattern of redundantPatterns) {
        if (pattern.test(altValue)) {
          issues.push({
            type: 'a11y-redundant-alt',
            severity: 'warning',
            category: 'accessibility',
            message: `Alt text contains redundant phrase: "${altValue}"`,
            recommendation: 'Remove "image of", "picture of", etc. Screen readers already announce it as an image',
            wcagCriterion: '1.1.1',
            altText: altValue,
            line: image.getStartLineNumber(),
          });
          break;
        }
      }
    }
  }
}
```

**Decorative Images Without Empty Alt:**

```typescript
private detectDecorativeImages(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);
    const src = this.getAttributeValue(attributes.get('src'));
    const alt = this.getAttributeValue(attributes.get('alt'));

    if (!src) continue;

    // Heuristic for decorative images
    const decorativePatterns = [
      /decorative/i,
      /background/i,
      /divider/i,
      /separator/i,
      /spacer/i,
      /border/i,
    ];

    const isLikelyDecorative = decorativePatterns.some(pattern => pattern.test(src));

    if (isLikelyDecorative && alt !== '') {
      issues.push({
        type: 'a11y-decorative-alt',
        severity: 'info',
        category: 'accessibility',
        message: 'Decorative image should have empty alt attribute (alt="")',
        recommendation: 'Set alt="" for decorative images to hide from screen readers',
        wcagCriterion: '1.1.1',
        src,
        line: image.getStartLineNumber(),
      });
    }
  }
}
```

#### TypeScript Analysis

**Untyped Pages Router Data Fetching:**

```typescript
private detectUntypedPagesRouter(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  const routerType = detectRouterType(sourceFile);
  if (routerType !== 'pages') return;

  // Check getServerSideProps
  const gsspFunc = this.findDataFetchingFunction(sourceFile, 'getServerSideProps');
  if (gsspFunc) {
    const hasType = this.hasTypeAnnotation(gsspFunc, 'GetServerSideProps');
    if (!hasType) {
      issues.push({
        type: 'typescript-untyped-gssp',
        severity: 'warning',
        category: 'type-safety',
        message: 'getServerSideProps lacks type annotation',
        recommendation: 'Add type: export const getServerSideProps: GetServerSideProps<Props> = async (context) => { ... }',
        line: gsspFunc.getStartLineNumber(),
      });
    }
  }

  // Check getStaticProps
  const gspFunc = this.findDataFetchingFunction(sourceFile, 'getStaticProps');
  if (gspFunc) {
    const hasType = this.hasTypeAnnotation(gspFunc, 'GetStaticProps');
    if (!hasType) {
      issues.push({
        type: 'typescript-untyped-gsp',
        severity: 'warning',
        category: 'type-safety',
        message: 'getStaticProps lacks type annotation',
        recommendation: 'Add type: export const getStaticProps: GetStaticProps<Props> = async (context) => { ... }',
        line: gspFunc.getStartLineNumber(),
      });
    }
  }

  // Check getStaticPaths
  const gspathsFunc = this.findDataFetchingFunction(sourceFile, 'getStaticPaths');
  if (gspathsFunc) {
    const hasType = this.hasTypeAnnotation(gspathsFunc, 'GetStaticPaths');
    if (!hasType) {
      issues.push({
        type: 'typescript-untyped-gsp',
        severity: 'warning',
        category: 'type-safety',
        message: 'getStaticPaths lacks type annotation',
        recommendation: 'Add type: export const getStaticPaths: GetStaticPaths = async () => { ... }',
        line: gspathsFunc.getStartLineNumber(),
      });
    }
  }
}

private hasTypeAnnotation(
  func: FunctionDeclaration | ArrowFunction,
  expectedType: string
): boolean {
  // Check function declaration
  if (Node.isFunctionDeclaration(func)) {
    const typeNode = func.getReturnTypeNode();
    if (typeNode) {
      const typeText = typeNode.getText();
      return typeText.includes(expectedType);
    }
  }

  // Check variable declaration with arrow function
  const parent = func.getParent();
  if (Node.isVariableDeclaration(parent)) {
    const typeNode = parent.getTypeNode();
    if (typeNode) {
      const typeText = typeNode.getText();
      return typeText.includes(expectedType);
    }
  }

  return false;
}
```

**Synchronous Params Type (Next.js 15 Breaking Change):**

```typescript
private detectSyncParamsType(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  const routerType = detectRouterType(sourceFile);
  if (routerType !== 'app') return;

  // Find page component or route handler
  sourceFile.forEachDescendant(node => {
    if (Node.isFunctionDeclaration(node) || Node.isArrowFunction(node)) {
      const params = node.getParameters();

      for (const param of params) {
        const paramName = param.getName();

        if (paramName === 'params' || paramName === 'searchParams') {
          const typeNode = param.getTypeNode();
          if (!typeNode) continue;

          const typeText = typeNode.getText();

          // Check if type is not wrapped in Promise
          if (!typeText.includes('Promise<')) {
            issues.push({
              type: 'typescript-sync-params-type',
              severity: 'error',
              category: 'type-safety',
              message: `${paramName} type must be Promise in Next.js 15+ (breaking change)`,
              recommendation: `Change type to: ${paramName}: Promise<{ ... }>`,
              breaking: true,
              line: param.getStartLineNumber(),
            });
          }
        }
      }
    }
  });
}
```

**Missing NextRequest/NextResponse Types:**

```typescript
private detectMissingRouteHandlerTypes(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  if (!isRouteHandler(sourceFile)) return;

  const httpMethods = ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'HEAD', 'OPTIONS'];

  for (const method of httpMethods) {
    const func = this.findExportedFunction(sourceFile, method);
    if (!func) continue;

    // Check parameters
    const params = func.getParameters();
    if (params.length === 0) continue;

    const requestParam = params[0];
    const requestType = requestParam.getTypeNode();

    if (!requestType) {
      issues.push({
        type: 'typescript-route-handler-untyped',
        severity: 'warning',
        category: 'type-safety',
        message: `Route handler ${method} missing request parameter type`,
        recommendation: 'Add type: export async function GET(request: NextRequest) { ... }',
        method,
        line: func.getStartLineNumber(),
      });
      continue;
    }

    const typeText = requestType.getText();
    if (!typeText.includes('NextRequest')) {
      issues.push({
        type: 'typescript-route-handler-wrong-type',
        severity: 'warning',
        category: 'type-safety',
        message: `Route handler ${method} should use NextRequest type`,
        recommendation: 'Import NextRequest from next/server',
        method,
        actualType: typeText,
        line: func.getStartLineNumber(),
      });
    }

    // Check return type
    const returnType = func.getReturnType();
    const returnTypeText = returnType.getText();

    if (!returnTypeText.includes('Response') && !returnTypeText.includes('NextResponse')) {
      issues.push({
        type: 'typescript-route-handler-return-type',
        severity: 'info',
        category: 'type-safety',
        message: `Route handler ${method} should return Response or NextResponse`,
        recommendation: 'Add return type: Promise<Response> or Promise<NextResponse>',
        method,
        line: func.getStartLineNumber(),
      });
    }
  }
}

private findExportedFunction(sourceFile: SourceFile, name: string): FunctionDeclaration | undefined {
  return sourceFile.getFunctions().find(func =>
    func.getName() === name && func.isExported()
  );
}
```

**Untyped Server Actions:**

```typescript
private detectUntypedServerActions(sourceFile: SourceFile, issues: A11yTypeScriptIssue[]): void {
  if (!hasUseServerDirective(sourceFile)) return;

  // Find all exported async functions (Server Actions)
  const functions = sourceFile.getFunctions().filter(func => func.isExported() && func.isAsync());

  for (const func of functions) {
    const params = func.getParameters();

    // Check prevState parameter (if using useActionState)
    if (params.length >= 2) {
      const prevStateParam = params[0];
      const prevStateType = prevStateParam.getTypeNode();

      if (!prevStateType) {
        issues.push({
          type: 'typescript-server-action-untyped',
          severity: 'warning',
          category: 'type-safety',
          message: `Server Action ${func.getName()} missing prevState parameter type`,
          recommendation: 'Add type: async function action(prevState: State | null, formData: FormData) { ... }',
          functionName: func.getName() || 'anonymous',
          line: func.getStartLineNumber(),
        });
      }
    }

    // Check FormData parameter
    const formDataParam = params.find(p => {
      const typeNode = p.getTypeNode();
      return typeNode && typeNode.getText() === 'FormData';
    });

    if (params.length > 0 && !formDataParam) {
      const lastParam = params[params.length - 1];
      const lastParamType = lastParam.getTypeNode();

      if (!lastParamType || !lastParamType.getText().includes('FormData')) {
        issues.push({
          type: 'typescript-server-action-no-formdata',
          severity: 'info',
          category: 'type-safety',
          message: `Server Action ${func.getName()} should accept FormData parameter`,
          recommendation: 'Add FormData parameter for form submissions',
          functionName: func.getName() || 'anonymous',
          line: func.getStartLineNumber(),
        });
      }
    }

    // Check return type
    const returnType = func.getReturnType();
    const returnTypeText = returnType.getText();

    if (returnTypeText === 'Promise<void>') {
      issues.push({
        type: 'typescript-server-action-void-return',
        severity: 'info',
        category: 'type-safety',
        message: `Server Action ${func.getName()} returns void, consider returning state`,
        recommendation: 'Return state object for useActionState: Promise<{ success: boolean; error?: string }>',
        functionName: func.getName() || 'anonymous',
        line: func.getStartLineNumber(),
      });
    }
  }
}
```

### Statistics Collection

```typescript
private collectStats(issues: A11yTypeScriptIssue[]): A11yTypeScriptStats {
  const stats: A11yTypeScriptStats = {
    accessibilityIssues: {
      missingLang: 0,
      missingAlt: 0,
      redundantAlt: 0,
      decorativeAlt: 0,
    },
    typeScriptIssues: {
      untypedPagesRouter: 0,
      syncParamsType: 0,
      routeHandlerTypes: 0,
      serverActionTypes: 0,
    },
  };

  for (const issue of issues) {
    switch (issue.type) {
      case 'a11y-missing-lang':
        stats.accessibilityIssues.missingLang++;
        break;
      case 'a11y-missing-alt':
        stats.accessibilityIssues.missingAlt++;
        break;
      case 'a11y-redundant-alt':
        stats.accessibilityIssues.redundantAlt++;
        break;
      case 'a11y-decorative-alt':
        stats.accessibilityIssues.decorativeAlt++;
        break;
      case 'typescript-untyped-gssp':
      case 'typescript-untyped-gsp':
        stats.typeScriptIssues.untypedPagesRouter++;
        break;
      case 'typescript-sync-params-type':
        stats.typeScriptIssues.syncParamsType++;
        break;
      case 'typescript-route-handler-untyped':
      case 'typescript-route-handler-wrong-type':
      case 'typescript-route-handler-return-type':
        stats.typeScriptIssues.routeHandlerTypes++;
        break;
      case 'typescript-server-action-untyped':
      case 'typescript-server-action-no-formdata':
      case 'typescript-server-action-void-return':
        stats.typeScriptIssues.serverActionTypes++;
        break;
    }
  }

  return stats;
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: A11yTypeScriptIssue[]): A11yTypeScriptComplexity {
  const stats = this.collectStats(issues);

  let deductions = 0;

  // Apply severity deductions
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.a11yTypeScript[issue.severity];
    deductions += severityDeduction;

    // Breaking changes heavily penalized
    if (issue.breaking) {
      deductions += 15;
    }

    // Apply issue type weights
    const typeWeight = ISSUE_TYPE_WEIGHTS.a11yTypeScript[issue.type] || 0;
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

**Accessibility Tests (18 tests):**

- Detects missing lang in \_document
- Accepts lang="en" in \_document
- Accepts lang with region code (en-US)
- Skips non-document files
- Detects missing alt on Image
- Accepts descriptive alt text
- Accepts empty alt for decorative images
- Detects redundant "image of" in alt
- Detects redundant "picture of" in alt
- Detects redundant "photo of" in alt
- Accepts valid descriptive alt without redundancy
- Detects decorative image with non-empty alt
- Accepts decorative image with empty alt
- Tests multiple images with mixed alt issues
- Tests i18n config as alternative to lang attribute
- Tests Html component variations
- Tests image in different contexts
- Tests WCAG criterion mapping

**TypeScript Tests (27 tests):**

- Detects untyped getServerSideProps
- Accepts typed GetServerSideProps
- Detects untyped getStaticProps
- Accepts typed GetStaticProps
- Detects untyped getStaticPaths
- Accepts typed GetStaticPaths
- Tests InferGetServerSidePropsType usage
- Tests InferGetStaticPropsType usage
- Detects synchronous params type in page
- Detects synchronous searchParams type
- Accepts `Promise&lt;params&gt;` type
- Accepts `Promise&lt;searchParams&gt;` type
- Detects params type in route handlers
- Tests dynamic route with params
- Detects untyped GET route handler
- Detects wrong request type (Request vs NextRequest)
- Accepts NextRequest type
- Detects missing return type annotation
- Accepts NextResponse return type
- Detects untyped Server Action
- Detects Server Action without FormData
- Accepts typed Server Action with FormData
- Detects void return in Server Action
- Accepts state return type in Server Action
- Tests prevState parameter typing
- Tests multiple Server Actions
- Tests async Server Action requirement

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 45 tests pass
- [ ] Detects all accessibility issues (lang, alt text problems)
- [ ] Detects all TypeScript issues (untyped functions, breaking changes)
- [ ] Provides WCAG criterion references for a11y issues
- [ ] Flags Next.js 15 breaking changes with high severity
- [ ] Collects accurate statistics by category
- [ ] Score calculation penalizes breaking changes
- [ ] No false positives on properly typed code
- [ ] Handles edge cases (i18n config, decorative images, complex types)

## Technical Notes

### WCAG Criterion Mapping

Each accessibility issue includes WCAG reference:

- `3.1.1` - Language of Page
- `1.1.1` - Non-text Content (alt text)

### Type Annotation Detection

Multiple strategies for detecting types:

1. Function return type node
2. Variable declaration type node
3. Parameter type nodes
4. Type inference from type checker

### Helper Functions

```typescript
private getAttributeMap(element: JsxElement | JsxSelfClosingElement): Map<string, JsxAttribute> {
  const map = new Map<string, JsxAttribute>();

  const attrs = Node.isJsxElement(element)
    ? element.getOpeningElement().getAttributes()
    : element.getAttributes();

  for (const attr of attrs) {
    if (Node.isJsxAttribute(attr)) {
      map.set(attr.getName(), attr);
    }
  }

  return map;
}

private getAttributeValue(attr: JsxAttribute | undefined): string | null {
  if (!attr) return null;

  const initializer = attr.getInitializer();
  if (!initializer) return null;

  if (Node.isStringLiteral(initializer)) {
    return initializer.getLiteralText();
  }

  if (Node.isJsxExpression(initializer)) {
    const expr = initializer.getExpression();
    if (expr && Node.isStringLiteral(expr)) {
      return expr.getLiteralText();
    }
  }

  return null;
}
```

### Edge Cases Handled

**Accessibility:**

- i18n config as alternative to lang attribute
- Empty alt for decorative images is valid
- Redundant patterns case-insensitive
- Multiple images in one component
- Html component from next/document vs html tag

**TypeScript:**

- Arrow function vs function declaration
- Variable declaration with type annotation
- Inferred types from context
- Generic type parameters
- Union types for prevState (State | null)
- Optional parameters
- Rest parameters in Server Actions

### Breaking Changes Detection

Next.js 15 breaking changes flagged with `breaking: true`:

- Synchronous params/searchParams types
- Must be Promise types in Next.js 15+

### Type Checker Usage

When available, uses ts-morph's type checker:

```typescript
private getTypeText(node: Node): string {
  const type = node.getType();
  return type.getText();
}
```

### Constants

```typescript
// From constants/a11y-typescript-thresholds.ts
export const SEVERITY_DEDUCTIONS = {
  a11yTypeScript: {
    error: 15,
    warning: 8,
    info: 3,
  },
};

export const ISSUE_TYPE_WEIGHTS = {
  a11yTypeScript: {
    'a11y-missing-alt': 10,
    'a11y-missing-lang': 8,
    'typescript-sync-params-type': 15, // Breaking change
    'typescript-untyped-gssp': 5,
    'typescript-server-action-untyped': 7,
  },
};
```

## Next Steps

This completes the core analysis modules for the Next.js analyzer. The next phases will focus on:

- Phase 12: Presenter Implementation - Create report presenters for all analyses
- Phase 13: Plugin Integration - Integrate with plugin system and registry
- Phase 14: Testing and Refinement - End-to-end testing and optimization
- Phase 15: Documentation - User guide and API documentation
