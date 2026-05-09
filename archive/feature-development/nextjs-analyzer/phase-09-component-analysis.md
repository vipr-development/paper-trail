# Phase 9: Component Anti-Patterns Analysis

## Status

Not Started

## Goals

Implement the Component Anti-Patterns analysis module that detects improper usage of Next.js optimized components including next/image, next/link, and next/script, focusing on deprecated props, missing required attributes, and patterns that degrade performance or accessibility.

## Files Created

### Implementation Files

- `analyses/component-analysis.ts` - ComponentAnalysis class implementing IAnalysis
- `analyses/component-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

#### next/image Fixtures

- `testing/fixtures/component/image/missing-dimensions.tsx` - Image without width/height
- `testing/fixtures/component/image/deprecated-layout.tsx` - layout prop usage
- `testing/fixtures/component/image/fill-no-position.tsx` - fill without positioned parent
- `testing/fixtures/component/image/hero-no-priority.tsx` - Large image without priority
- `testing/fixtures/component/image/missing-alt.tsx` - Missing alt attribute
- `testing/fixtures/component/image/valid-sized.tsx` - Valid with dimensions
- `testing/fixtures/component/image/valid-fill.tsx` - Valid fill with positioned parent
- `testing/fixtures/component/image/valid-priority.tsx` - Valid priority usage

#### next/link Fixtures

- `testing/fixtures/component/link/nested-anchor-13.tsx` - Nested anchor in Next.js 13+
- `testing/fixtures/component/link/legacy-behavior.tsx` - legacyBehavior prop usage
- `testing/fixtures/component/link/plain-anchor-internal.tsx` - `&lt;a&gt;` for internal route
- `testing/fixtures/component/link/valid-modern.tsx` - Modern Link without anchor
- `testing/fixtures/component/link/valid-external.tsx` - `&lt;a&gt;` for external links

#### next/script Fixtures

- `testing/fixtures/component/script/script-in-head.tsx` - Script inside next/head
- `testing/fixtures/component/script/inline-no-id.tsx` - Inline script without id
- `testing/fixtures/component/script/before-interactive-page.tsx` - beforeInteractive outside \_document
- `testing/fixtures/component/script/valid-after-interactive.tsx` - Valid afterInteractive
- `testing/fixtures/component/script/valid-lazy-onload.tsx` - Valid lazyOnload

## Implementation Details

### ComponentAnalysis Class

Implements `IAnalysis<unknown, ComponentComplexity>` with the following structure:

```typescript
export class ComponentAnalysis implements IAnalysis<unknown, ComponentComplexity> {
  readonly id = 'nextjs-component';
  readonly name = 'Next.js Component Anti-Patterns Analysis';
  readonly category = 'performance' as const;
  readonly description = 'Detects improper usage of next/image, next/link, and next/script';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<ComponentComplexity>;
}
```

### Detection Patterns

#### next/image Analysis

**Missing Dimensions Detection:**

```typescript
private detectMissingDimensions(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    const hasFill = attributes.has('fill');
    const hasWidth = attributes.has('width');
    const hasHeight = attributes.has('height');

    // If not using fill, must have width and height
    if (!hasFill && (!hasWidth || !hasHeight)) {
      const missing = [];
      if (!hasWidth) missing.push('width');
      if (!hasHeight) missing.push('height');

      issues.push({
        type: 'image-missing-dimensions',
        component: 'next/image',
        severity: 'error',
        message: `Image missing required ${missing.join(' and ')} ${missing.length > 1 ? 'attributes' : 'attribute'}`,
        recommendation: hasFill
          ? 'Add fill prop or specify width and height'
          : `Add ${missing.join(' and ')} props`,
        line: image.getStartLineNumber(),
      });
    }
  }
}
```

**Deprecated Layout Prop Detection:**

```typescript
private detectDeprecatedLayout(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    if (attributes.has('layout')) {
      const layoutValue = this.getAttributeValue(attributes.get('layout'));

      issues.push({
        type: 'image-deprecated-layout',
        component: 'next/image',
        severity: 'warning',
        message: 'layout prop is deprecated in Next.js 13+',
        recommendation: this.getLayoutReplacement(layoutValue),
        deprecatedProp: 'layout',
        line: image.getStartLineNumber(),
      });
    }

    // Check for other deprecated props
    const deprecatedProps = ['objectFit', 'objectPosition', 'lazyBoundary', 'lazyRoot'];
    for (const prop of deprecatedProps) {
      if (attributes.has(prop)) {
        issues.push({
          type: 'image-deprecated-prop',
          component: 'next/image',
          severity: 'warning',
          message: `${prop} prop is deprecated in Next.js 13+`,
          recommendation: this.getImagePropReplacement(prop),
          deprecatedProp: prop,
          line: image.getStartLineNumber(),
        });
      }
    }
  }
}

private getLayoutReplacement(layoutValue: string): string {
  const replacements: Record<string, string> = {
    fill: 'Use fill prop instead',
    responsive: 'Use width, height, and style={{ width: "100%", height: "auto" }}',
    intrinsic: 'Use width and height props (default behavior)',
    fixed: 'Use width and height props',
  };
  return replacements[layoutValue] || 'Use fill prop or width/height props';
}
```

**Fill Without Position Detection:**

```typescript
private detectFillWithoutPosition(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    if (!attributes.has('fill')) continue;

    // Check if parent element has position styling
    const parent = image.getParent();
    if (!parent) continue;

    const hasPositionedParent = this.hasPositionedParent(image);

    if (!hasPositionedParent) {
      issues.push({
        type: 'image-fill-no-position',
        component: 'next/image',
        severity: 'warning',
        message: 'Image with fill prop requires parent element with position: relative/absolute/fixed',
        recommendation: 'Wrap Image in a div with position: relative',
        line: image.getStartLineNumber(),
      });
    }
  }
}

private hasPositionedParent(image: JsxElement | JsxSelfClosingElement): boolean {
  let parent = image.getParent();

  // Check up to 3 levels up
  for (let i = 0; i < 3; i++) {
    if (!parent || !Node.isJsxElement(parent)) break;

    const attributes = this.getAttributeMap(parent);

    // Check for style prop with position
    if (attributes.has('style')) {
      const styleValue = this.getAttributeValue(attributes.get('style'));
      if (styleValue && /position\s*:\s*['"]?(relative|absolute|fixed)/.test(styleValue)) {
        return true;
      }
    }

    // Check for className that might have positioning
    if (attributes.has('className')) {
      // Note: This is a heuristic, can't check actual CSS
      return true; // Assume className may have positioning
    }

    parent = parent.getParent();
  }

  return false;
}
```

**Hero Image Without Priority Detection:**

```typescript
private detectHeroWithoutPriority(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    const hasPriority = attributes.has('priority');
    if (hasPriority) continue;

    // Check if this is a large image (likely hero/LCP)
    const width = this.getNumericAttributeValue(attributes.get('width'));
    const height = this.getNumericAttributeValue(attributes.get('height'));
    const hasFill = attributes.has('fill');

    const isLarge = hasFill || (width && width > 800) || (height && height > 400);

    // Check if image is likely above the fold
    const isAboveFold = this.isLikelyAboveFold(image, sourceFile);

    if (isLarge && isAboveFold) {
      issues.push({
        type: 'image-hero-no-priority',
        component: 'next/image',
        severity: 'warning',
        message: 'Large above-the-fold image without priority prop may delay LCP',
        recommendation: 'Add priority prop to prioritize loading',
        line: image.getStartLineNumber(),
      });
    }
  }
}

private isLikelyAboveFold(image: JsxElement | JsxSelfClosingElement, sourceFile: SourceFile): boolean {
  // Check if image is in first 100 lines of component
  const componentStart = this.findComponentStart(image);
  if (!componentStart) return false;

  const imageStart = image.getStartLineNumber();
  const componentStartLine = componentStart.getStartLineNumber();

  // If within first 20 lines of component JSX, likely above fold
  return (imageStart - componentStartLine) < 20;
}
```

**Missing Alt Text Detection:**

```typescript
private detectMissingAlt(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const imageUsages = getImageUsages(sourceFile);

  for (const image of imageUsages) {
    const attributes = this.getAttributeMap(image);

    if (!attributes.has('alt')) {
      issues.push({
        type: 'image-missing-alt',
        component: 'next/image',
        severity: 'error',
        message: 'Image missing alt attribute',
        recommendation: 'Add alt prop with descriptive text or empty string for decorative images',
        line: image.getStartLineNumber(),
      });
    }
  }
}
```

#### next/link Analysis

**Nested Anchor Detection:**

```typescript
private detectNestedAnchor(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const linkUsages = getLinkUsages(sourceFile);

  for (const link of linkUsages) {
    const children = link.getJsxChildren();

    for (const child of children) {
      if (Node.isJsxElement(child)) {
        const tagName = child.getOpeningElement().getTagNameNode().getText();

        if (tagName === 'a') {
          issues.push({
            type: 'link-nested-anchor',
            component: 'next/link',
            severity: 'warning',
            message: 'Link with nested `&lt;a&gt;` tag is deprecated in Next.js 13+',
            recommendation: 'Remove `&lt;a&gt;` tag, Link now renders as anchor directly',
            line: link.getStartLineNumber(),
          });
        }
      }
    }
  }
}
```

**legacyBehavior Detection:**

```typescript
private detectLegacyBehavior(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const linkUsages = getLinkUsages(sourceFile);

  for (const link of linkUsages) {
    const attributes = this.getAttributeMap(link);

    if (attributes.has('legacyBehavior')) {
      issues.push({
        type: 'link-legacy-behavior',
        component: 'next/link',
        severity: 'warning',
        message: 'legacyBehavior prop indicates incomplete migration',
        recommendation: 'Remove legacyBehavior and nested anchor tag',
        deprecatedProp: 'legacyBehavior',
        line: link.getStartLineNumber(),
      });
    }
  }
}
```

**Plain Anchor for Internal Routes:**

```typescript
private detectPlainAnchorForInternal(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  // Find all <a> tags (not inside Link components)
  const anchorElements: JsxElement[] = [];

  sourceFile.forEachDescendant(node => {
    if (Node.isJsxElement(node)) {
      const tagName = node.getOpeningElement().getTagNameNode().getText();

      if (tagName === 'a') {
        // Check if not inside a Link component
        if (!this.isInsideLinkComponent(node)) {
          anchorElements.push(node);
        }
      }
    }
  });

  for (const anchor of anchorElements) {
    const attributes = this.getAttributeMap(anchor);
    const href = this.getAttributeValue(attributes.get('href'));

    if (href && this.isInternalRoute(href)) {
      issues.push({
        type: 'link-plain-anchor-internal',
        component: 'next/link',
        severity: 'warning',
        message: 'Using `&lt;a&gt;` tag for internal navigation loses client-side routing',
        recommendation: 'Replace with `&lt;Link&gt;` from next/link',
        href,
        line: anchor.getStartLineNumber(),
      });
    }
  }
}

private isInsideLinkComponent(node: Node): boolean {
  let parent = node.getParent();

  while (parent) {
    if (Node.isJsxElement(parent) || Node.isJsxSelfClosingElement(parent)) {
      const tagName = Node.isJsxElement(parent)
        ? parent.getOpeningElement().getTagNameNode().getText()
        : parent.getTagNameNode().getText();

      if (tagName === 'Link') return true;
    }
    parent = parent.getParent();
  }

  return false;
}

private isInternalRoute(href: string): boolean {
  // Internal routes start with / and are not protocol links
  return href.startsWith('/') && !href.startsWith('//');
}
```

#### next/script Analysis

**Script in next/head Detection:**

```typescript
private detectScriptInHead(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  // Check if next/head is imported
  const hasHeadImport = sourceFile.getImportDeclarations().some(imp =>
    imp.getModuleSpecifierValue() === 'next/head'
  );

  if (!hasHeadImport) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isJsxElement(node)) {
      const tagName = node.getOpeningElement().getTagNameNode().getText();

      if (tagName === 'Head') {
        // Check for <script> or <Script> children
        const children = node.getJsxChildren();

        for (const child of children) {
          if (Node.isJsxElement(child) || Node.isJsxSelfClosingElement(child)) {
            const childTag = Node.isJsxElement(child)
              ? child.getOpeningElement().getTagNameNode().getText()
              : child.getTagNameNode().getText();

            if (childTag === 'script' || childTag === 'Script') {
              issues.push({
                type: 'script-in-head',
                component: 'next/script',
                severity: 'error',
                message: 'Script components should not be used inside next/head',
                recommendation: 'Move Script outside of Head component or use regular <script> tag',
                line: child.getStartLineNumber(),
              });
            }
          }
        }
      }
    }
  });
}
```

**Inline Script Without ID Detection:**

```typescript
private detectInlineScriptNoId(sourceFile: SourceFile, issues: ComponentIssue[]): void {
  const scriptUsages = getScriptUsages(sourceFile);

  for (const script of scriptUsages) {
    const attributes = this.getAttributeMap(script);

    // Check if script has children (inline script)
    const hasChildren = Node.isJsxElement(script) &&
      script.getJsxChildren().some(child => Node.isJsxText(child) || Node.isJsxExpression(child));

    if (hasChildren && !attributes.has('id')) {
      issues.push({
        type: 'script-inline-no-id',
        component: 'next/script',
        severity: 'error',
        message: 'Inline Script component requires id prop',
        recommendation: 'Add unique id prop to inline Script',
        line: script.getStartLineNumber(),
      });
    }
  }
}
```

**beforeInteractive Outside \_document Detection:**

```typescript
private detectBeforeInteractiveOutsideDocument(
  sourceFile: SourceFile,
  issues: ComponentIssue[]
): void {
  const filePath = sourceFile.getFilePath();
  const isDocument = /_document\.(tsx?|jsx?)$/.test(filePath);
  const isPagesRouter = /\/pages\//.test(filePath);

  // Only check Pages Router files
  if (!isPagesRouter) return;

  if (isDocument) return; // Valid usage

  const scriptUsages = getScriptUsages(sourceFile);

  for (const script of scriptUsages) {
    const attributes = this.getAttributeMap(script);
    const strategy = this.getAttributeValue(attributes.get('strategy'));

    if (strategy === 'beforeInteractive') {
      issues.push({
        type: 'script-before-interactive-outside-document',
        component: 'next/script',
        severity: 'error',
        message: 'beforeInteractive strategy only works in pages/_document',
        recommendation: 'Move Script to _document or use afterInteractive/lazyOnload strategy',
        line: script.getStartLineNumber(),
      });
    }
  }
}
```

### Statistics Collection

```typescript
private collectStats(issues: ComponentIssue[]): ComponentStats {
  const stats: ComponentStats = {
    imageIssues: {
      missingDimensions: 0,
      deprecatedProps: 0,
      fillWithoutPosition: 0,
      heroWithoutPriority: 0,
      missingAlt: 0,
    },
    linkIssues: {
      nestedAnchor: 0,
      legacyBehavior: 0,
      plainAnchorInternal: 0,
    },
    scriptIssues: {
      inHead: 0,
      inlineNoId: 0,
      beforeInteractiveOutsideDocument: 0,
    },
  };

  for (const issue of issues) {
    switch (issue.type) {
      case 'image-missing-dimensions':
        stats.imageIssues.missingDimensions++;
        break;
      case 'image-deprecated-layout':
      case 'image-deprecated-prop':
        stats.imageIssues.deprecatedProps++;
        break;
      case 'image-fill-no-position':
        stats.imageIssues.fillWithoutPosition++;
        break;
      case 'image-hero-no-priority':
        stats.imageIssues.heroWithoutPriority++;
        break;
      case 'image-missing-alt':
        stats.imageIssues.missingAlt++;
        break;
      case 'link-nested-anchor':
        stats.linkIssues.nestedAnchor++;
        break;
      case 'link-legacy-behavior':
        stats.linkIssues.legacyBehavior++;
        break;
      case 'link-plain-anchor-internal':
        stats.linkIssues.plainAnchorInternal++;
        break;
      case 'script-in-head':
        stats.scriptIssues.inHead++;
        break;
      case 'script-inline-no-id':
        stats.scriptIssues.inlineNoId++;
        break;
      case 'script-before-interactive-outside-document':
        stats.scriptIssues.beforeInteractiveOutsideDocument++;
        break;
    }
  }

  return stats;
}
```

### Scoring Calculation

```typescript
private calculateMetrics(issues: ComponentIssue[]): ComponentComplexity {
  const stats = this.collectStats(issues);

  let deductions = 0;

  // Apply severity deductions
  for (const issue of issues) {
    const severityDeduction = SEVERITY_DEDUCTIONS.component[issue.severity];
    deductions += severityDeduction;

    // Apply issue type weights
    const typeWeight = ISSUE_TYPE_WEIGHTS.component[issue.type] || 0;
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

### Test Suite Structure (55 tests)

**next/image Tests (25 tests):**

- Detects missing width attribute
- Detects missing height attribute
- Detects missing both width and height
- Accepts fill without dimensions
- Accepts valid width and height
- Detects deprecated layout="fill"
- Detects deprecated layout="responsive"
- Detects deprecated objectFit prop
- Detects deprecated objectPosition prop
- Accepts modern fill prop
- Detects fill without positioned parent
- Accepts fill with positioned parent (style)
- Accepts fill with positioned parent (className assumption)
- Detects hero image without priority
- Accepts large image with priority
- Accepts small image without priority
- Detects missing alt attribute
- Accepts valid alt text
- Accepts empty alt for decorative images
- Tests multiple dimensions checks in one file
- Tests complex image scenarios
- Tests responsive image patterns
- Tests fill with parent div
- Tests image in loop
- Tests conditional image rendering

**next/link Tests (15 tests):**

- Detects Link with nested `&lt;a&gt;`
- Accepts modern Link without anchor
- Detects legacyBehavior prop
- Accepts Link without legacyBehavior
- Detects plain `&lt;a&gt;` for internal route /about
- Detects plain `&lt;a&gt;` for internal route /products/123
- Accepts `&lt;a&gt;` for external https:// link
- Accepts `&lt;a&gt;` for external // protocol link
- Accepts `&lt;a&gt;` for mailto: link
- Accepts `&lt;a&gt;` for tel: link
- Tests multiple Link components
- Tests nested route structures
- Tests Link with dynamic href
- Tests Link inside conditional
- Tests Link with query parameters

**next/script Tests (15 tests):**

- Detects Script inside Head component
- Accepts Script outside Head
- Detects inline Script without id
- Accepts inline Script with id
- Accepts external Script without id
- Detects beforeInteractive in pages/index
- Accepts beforeInteractive in \_document
- Accepts afterInteractive anywhere
- Accepts lazyOnload anywhere
- Tests Script with strategy combinations
- Tests Script in App Router (beforeInteractive allowed)
- Tests Script with onLoad callback
- Tests Script with data attributes
- Tests multiple Script components
- Tests Script with JSX expression content

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 55 tests pass
- [ ] Detects all next/image issues (dimensions, deprecated props, fill, priority, alt)
- [ ] Detects all next/link issues (nested anchor, legacyBehavior, plain anchor)
- [ ] Detects all next/script issues (in head, inline without id, beforeInteractive)
- [ ] Provides component-specific recommendations
- [ ] Collects accurate statistics by component type
- [ ] Score calculation uses correct weights
- [ ] No false positives on valid usage patterns
- [ ] Handles edge cases (conditional rendering, loops, nested structures)

## Technical Notes

### JSX Attribute Extraction

Helper to extract attributes into a map:

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
```

### Attribute Value Extraction

Handles different attribute value types:

```typescript
private getAttributeValue(attr: JsxAttribute | undefined): string | null {
  if (!attr) return null;

  const initializer = attr.getInitializer();
  if (!initializer) return null;

  if (Node.isStringLiteral(initializer)) {
    return initializer.getLiteralText();
  }

  if (Node.isJsxExpression(initializer)) {
    const expr = initializer.getExpression();
    if (expr) {
      if (Node.isStringLiteral(expr)) {
        return expr.getLiteralText();
      }
      // Return text representation for analysis
      return expr.getText();
    }
  }

  return null;
}

private getNumericAttributeValue(attr: JsxAttribute | undefined): number | null {
  const value = this.getAttributeValue(attr);
  if (!value) return null;

  const num = parseInt(value, 10);
  return isNaN(num) ? null : num;
}
```

### Component Import Detection

Uses utility functions from `utilities/nextjs-detection.ts`:

- `getImageUsages(sourceFile)` - Finds all Image components
- `getLinkUsages(sourceFile)` - Finds all Link components
- `getScriptUsages(sourceFile)` - Finds all Script components

These functions handle both default and named imports.

### Edge Cases Handled

**Image component:**

- fill prop overrides dimension requirement
- Large images above fold detection heuristic
- Positioned parent detection limited to 3 levels
- Empty alt for decorative images is valid

**Link component:**

- External links with `&lt;a&gt;` are valid
- Protocol-relative URLs treated as external
- mailto: and tel: links excluded
- Link inside conditional rendering

**Script component:**

- App Router allows beforeInteractive anywhere
- External scripts don't need id prop
- Script with strategy prop variations
- Script in different file types

### Component Start Detection

```typescript
private findComponentStart(node: Node): FunctionDeclaration | ArrowFunction | undefined {
  let parent = node.getParent();

  while (parent) {
    if (Node.isFunctionDeclaration(parent) || Node.isArrowFunction(parent)) {
      return parent;
    }
    parent = parent.getParent();
  }

  return undefined;
}
```

## Next Steps

Proceed to Phase 10: Performance Analysis

With component anti-patterns covered, the next phase will detect performance issues including bundle size bloat, hydration mismatches, and caching strategy problems.
