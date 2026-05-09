# Phase 4: Server/Client Component Analysis

## Status

Not Started

## Goals

Implement the Server/Client Component analysis module that detects directive placement issues, unnecessary client components, hook usage violations, and non-serializable prop passing across component boundaries.

## Files Created

### Implementation Files

- `analyses/server-client-analysis.ts` - ServerClientAnalysis class implementing IAnalysis
- `analyses/server-client-analysis.test.ts` - Comprehensive test suite

### Test Fixtures

- `testing/fixtures/server-client/directive-placement-invalid.tsx` - Invalid directive placement
- `testing/fixtures/server-client/directive-placement-valid.tsx` - Valid directive placement
- `testing/fixtures/server-client/both-directives.tsx` - Both directives in same file
- `testing/fixtures/server-client/unnecessary-client.tsx` - Client directive without hooks
- `testing/fixtures/server-client/missing-client.tsx` - Hooks without client directive
- `testing/fixtures/server-client/server-action-in-client.tsx` - Server action in client file
- `testing/fixtures/server-client/non-serializable-props.tsx` - Non-serializable props
- `testing/fixtures/server-client/valid-server-component.tsx` - Valid server component
- `testing/fixtures/server-client/valid-client-component.tsx` - Valid client component

## Implementation Details

### ServerClientAnalysis Class

Implements `IAnalysis<unknown, ServerClientComplexity>` with the following structure:

```typescript
export class ServerClientAnalysis implements IAnalysis<unknown, ServerClientComplexity> {
  readonly id = 'nextjs-server-client';
  readonly name = 'Next.js Server/Client Component Analysis';
  readonly category = 'correctness' as const;
  readonly description =
    'Analyzes Server and Client Component usage, directive placement, and boundary violations';
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<ServerClientComplexity>;
}
```

### Detection Patterns

#### Directive Placement Detection

**Directive Not At Top:**

- Uses `getDirectivePlacement()` utility
- Detects directives after imports
- Detects directives after comments (non-critical)
- Creates `DirectivePlacement.AFTER_IMPORTS` or `DirectivePlacement.AFTER_COMMENTS` issue

```typescript
private detectDirectivePlacement(
  sourceFile: SourceFile,
  issues: ServerClientIssue[]
): void {
  const placement = getDirectivePlacement(sourceFile);

  if (placement.useClient && placement.useClient !== 'top') {
    issues.push({
      type: 'directive-not-at-top',
      directive: 'use client',
      placement: placement.useClient,
      line: placement.useClientLine,
      severity: 'error',
      message: `'use client' directive must be at the top of the file`,
    });
  }

  // Same for 'use server'
}
```

**Both Directives:**

- Checks for both `'use client'` and `'use server'` in same file
- Creates critical error

```typescript
if (hasUseClientDirective(sourceFile) && hasUseServerDirective(sourceFile)) {
  issues.push({
    type: 'both-directives',
    severity: 'error',
    message: 'File contains both "use client" and "use server" directives',
  });
}
```

**Server Action in Client File:**

- Detects `'use server'` inside function when file has `'use client'`
- Looks for string literals within function bodies
- Creates error indicating Server Actions won't work in Client Components

#### Unnecessary Client Component Detection

Detects files with `'use client'` that don't need it:

```typescript
private detectUnnecessaryClient(
  sourceFile: SourceFile,
  issues: ServerClientIssue[]
): void {
  if (!hasUseClientDirective(sourceFile)) return;

  const hasClientFeatures = needsUseClientDirective(sourceFile);

  if (!hasClientFeatures) {
    const hooks = getUsedHooks(sourceFile);
    issues.push({
      type: 'unnecessary-client',
      severity: 'warning',
      message: 'Component marked as Client Component but does not use hooks or event handlers',
      hooks: hooks.length > 0 ? hooks : undefined,
    });
  }
}
```

**Detection criteria:**

- No React hooks (`useState`, `useEffect`, `useContext`, etc.)
- No event handlers (`onClick`, `onChange`, `onSubmit`, etc.)
- No browser APIs (`window`, `document`, `localStorage`, etc.)
- No client-only hooks (`useRouter` from next/navigation, `useSearchParams`, etc.)

#### Missing Client Directive Detection

Detects Server Components using client-only features:

```typescript
private detectMissingClient(
  sourceFile: SourceFile,
  issues: ServerClientIssue[]
): void {
  if (hasUseClientDirective(sourceFile)) return;

  const needsClient = needsUseClientDirective(sourceFile);

  if (needsClient) {
    const hooks = getUsedHooks(sourceFile);
    issues.push({
      type: 'missing-client',
      severity: 'error',
      message: 'Component uses hooks or event handlers but is missing "use client" directive',
      hooks,
    });
  }
}
```

#### Non-Serializable Props Detection

Detects Server Components passing non-serializable data to Client Components:

```typescript
private detectNonSerializableProps(
  sourceFile: SourceFile,
  issues: ServerClientIssue[]
): void {
  // Only check in Server Components (no 'use client' directive)
  if (hasUseClientDirective(sourceFile)) return;

  sourceFile.forEachDescendant(node => {
    if (Node.isJsxElement(node) || Node.isJsxSelfClosingElement(node)) {
      const tagName = this.getJsxTagName(node);

      // Check if component is imported from a 'use client' file
      const isClientComponent = this.isImportedClientComponent(sourceFile, tagName);

      if (isClientComponent) {
        this.checkPropsForNonSerializable(node, issues);
      }
    }
  });
}
```

**Non-serializable types detected:**

- Inline functions (`onClick={() => ...}`)
- Function references from variables
- Class instances
- `Date` objects
- `Map`, `Set` objects
- `Symbol`, `RegExp` objects
- `undefined` (when explicit)

### Scoring Calculation

```typescript
private calculateMetrics(issues: ServerClientIssue[]): ServerClientComplexity {
  const stats = {
    totalIssues: issues.length,
    byType: this.groupIssuesByType(issues),
    bySeverity: this.groupIssuesBySeverity(issues),
  };

  let deductions = 0;

  // Apply severity deductions
  deductions += stats.bySeverity.error * SEVERITY_DEDUCTIONS.serverClient.error;
  deductions += stats.bySeverity.warning * SEVERITY_DEDUCTIONS.serverClient.warning;
  deductions += stats.bySeverity.info * SEVERITY_DEDUCTIONS.serverClient.info;

  // Apply issue type weights
  for (const [type, count] of Object.entries(stats.byType)) {
    const weight = ISSUE_TYPE_WEIGHTS.serverClient[type] || 0;
    deductions += count * weight;
  }

  const score = roundScore(Math.max(0, 100 - deductions));

  return {
    issues,
    stats,
    score,
  };
}
```

### Insights Generation

Creates actionable insights for each issue category:

```typescript
private addInsights(issues: ServerClientIssue[], insights: ComplexityInsight[]): void {
  const criticalIssues = issues.filter(i => i.severity === 'error');

  if (criticalIssues.length > 0) {
    insights.push({
      severity: 'error',
      category: 'correctness',
      message: `Found ${criticalIssues.length} critical Server/Client Component issues`,
      recommendation: 'Fix directive placement and hook usage violations',
    });
  }

  const unnecessaryClient = issues.filter(i => i.type === 'unnecessary-client');
  if (unnecessaryClient.length > 0) {
    insights.push({
      severity: 'warning',
      category: 'performance',
      message: `${unnecessaryClient.length} components unnecessarily marked as Client Components`,
      recommendation: 'Remove "use client" directive to reduce bundle size',
    });
  }
}
```

## Test Coverage

### Test Suite Structure (40 tests)

**Directive Placement Tests (12 tests):**

- Detects directive after imports
- Detects directive after comments
- Accepts directive at top (single quotes)
- Accepts directive at top (double quotes)
- Detects both directives in same file
- Detects server action in client file
- Accepts valid use client placement
- Accepts valid use server placement
- Handles files without directives
- Handles directive after multiple imports
- Handles directive after JSDoc comment
- Handles directive in middle of file

**Unnecessary Client Tests (8 tests):**

- Detects client directive with no hooks
- Detects client directive with no event handlers
- Detects client directive with only props usage
- Accepts client directive with useState
- Accepts client directive with useEffect
- Accepts client directive with onClick
- Accepts client directive with browser API
- Handles static components correctly

**Missing Client Directive Tests (10 tests):**

- Detects useState without client directive
- Detects useEffect without client directive
- Detects useContext without client directive
- Detects onClick without client directive
- Detects onChange without client directive
- Detects window usage without client directive
- Detects localStorage usage without client directive
- Detects useRouter (next/navigation) without client directive
- Accepts server component without hooks
- Handles multiple hooks correctly

**Non-Serializable Props Tests (10 tests):**

- Detects inline function props
- Detects function reference props
- Detects Date object props
- Detects Map object props
- Detects Set object props
- Detects class instance props
- Accepts serializable props (strings, numbers, objects)
- Accepts Server Actions (functions with 'use server')
- Handles client-to-client prop passing
- Handles deeply nested non-serializable props

### Test Fixtures

Each fixture is a complete TypeScript/TSX file demonstrating a specific pattern:

**directive-placement-invalid.tsx:**

```typescript
import { useState } from 'react';
'use client'; // WRONG: After import

export function Component() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}
```

**both-directives.tsx:**

```typescript
'use client';
'use server';

export function Component() {
  return <div>Invalid</div>;
}
```

**unnecessary-client.tsx:**

```typescript
'use client';

export function StaticCard({ title }: { title: string }) {
  return <h2>{title}</h2>; // No hooks, no interactivity
}
```

**missing-client.tsx:**

```typescript
// Missing 'use client' directive
import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0); // Will crash
  return <div>{count}</div>;
}
```

**non-serializable-props.tsx:**

```typescript
// Server Component
export default function Page() {
  const handleClick = () => console.log('clicked');
  return <ClientButton onClick={handleClick} />; // Can't serialize function
}
```

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 40 tests pass
- [ ] Detects directive placement issues (after imports, both directives)
- [ ] Detects unnecessary client components (no hooks/events)
- [ ] Detects missing client directive (hooks without directive)
- [ ] Detects non-serializable props passed to client components
- [ ] Detects server actions in client files
- [ ] Score calculation uses correct weights and deductions
- [ ] Insights provide actionable recommendations
- [ ] No false positives on valid patterns

## Technical Notes

### AST Pattern Detection

**Directive Detection:**

- Uses text-based regex for initial detection
- Validates position using ts-morph node analysis
- Handles both single and double quote variants

**Hook Detection:**

- Searches for CallExpression nodes
- Matches identifier names against React hooks list
- Includes both built-in hooks and Next.js client hooks

**Event Handler Detection:**

- Searches for JsxAttribute nodes
- Matches attribute names against event handler patterns
- Checks for function expressions in attribute values

**Non-Serializable Detection:**

- Analyzes JsxAttribute initializers
- Detects ArrowFunctionExpression, FunctionExpression
- Detects NewExpression for Date, Map, Set, etc.
- Uses type checker when available for advanced detection

### Client Component Import Detection

To detect if a component is a Client Component:

1. Find ImportDeclaration for the component
2. Resolve the imported file path
3. Read the imported file
4. Check for `'use client'` directive at the top

```typescript
private isImportedClientComponent(sourceFile: SourceFile, tagName: string): boolean {
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    const namedImports = imp.getNamedImports();
    const defaultImport = imp.getDefaultImport();

    if (defaultImport?.getText() === tagName ||
        namedImports.some(ni => ni.getName() === tagName)) {
      const moduleFile = imp.getModuleSpecifierSourceFile();
      if (moduleFile) {
        return hasUseClientDirective(moduleFile);
      }
    }
  }

  return false;
}
```

### Edge Cases Handled

- Comments before directives (acceptable)
- Multiple imports before directive (error)
- Directive in template literal (not detected)
- Re-exported components (follow import chain)
- Dynamic imports (not analyzed for serialization)
- Server Actions passed as props (allowed)
- Conditional hooks (still require directive)

## Next Steps

Proceed to Phase 5: Data Fetching Analysis

With Server/Client Component analysis complete, the next phase will detect data fetching anti-patterns across both Pages Router and App Router paradigms.
