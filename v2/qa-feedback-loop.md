# QA Feedback Loop: Analysis Accuracy Review System

## Overview

This document provides copy-paste prompt templates for rigorously QA testing analyzer accuracy across all fixture files. The workflow enables an LLM to independently review each fixture, identify discrepancies between expected and actual analyzer output, trace issues to source code, and propose concrete fixes.

## Four-Phase Workflow

```
Phase 1: Per-Fixture Review  →  Phase 2: Aggregation  →  Phase 3: Fix  →  Phase 4: Verify
(one prompt per file)           (one prompt, all data)     (manual)        (one prompt per fix)
```

### Phase Overview

- **Phase 1**: Run individual fixture reviews using category-specific templates (7-step structured analysis)
- **Phase 2**: Aggregate all findings, calculate priority scores, generate action plan
- **Phase 3**: Implement fixes based on prioritized action plan (manual development)
- **Phase 4**: Verify fixes work correctly and don't introduce regressions

---

## Template A: Core/JS/TS Fixtures

### When to Use

Use this template for fixtures in `packages/fixtures/src/core/`:

- `simple-function.ts`
- `moderate-complexity.ts`
- `high-complexity.ts`
- `very-complex.ts`

### Ground Truth Reference

Expected complexity ranges defined in `packages/fixtures/src/core/index.ts` (`FIXTURE_METADATA`):

- **simple-function**: Cyclomatic 1-2, Maintainability A (85-100), Low Halstead
- **moderate-complexity**: Cyclomatic 5-10, Maintainability B-C (65-84), Moderate Halstead
- **high-complexity**: Cyclomatic 10-20, Maintainability D (40-64), High Halstead
- **very-complex**: Cyclomatic 20+, Maintainability F (0-39), Very High Halstead

### Prompt Template

````markdown
# Core Analyzer Accuracy Review

## Task

Rigorously review the core analyzer output for a fixture file and identify all discrepancies between expected and actual results.

## Input Variables

- **File**: `packages/fixtures/src/core/{{FILENAME}}`
- **Intent**: {{INTENT}} (e.g., "good-example - should have low complexity scores")
- **Expected Complexity**: {{EXPECTED_COMPLEXITY}} (from FIXTURE_METADATA)
- **Expected Cyclomatic**: {{EXPECTED_CYCLOMATIC}}
- **Expected Maintainability**: {{EXPECTED_MAINTAINABILITY}}

## 7-Step Process

### Step 1: Run CLI Analysis

Run: `node clients/cli/dist/index.js analyze packages/fixtures/src/core/{{FILENAME}} -f json-full -q`

### Step 2: Read Source Code

Read the complete fixture file: `packages/fixtures/src/core/{{FILENAME}}`

### Step 3: Expert Review

As a skilled developer, independently identify:

- Cyclomatic complexity of each function (count decision points: if, else, while, for, case, &&, ||, ?:, catch)
- Halstead metrics assessment (operators vs operands, vocabulary, volume)
- Overall maintainability characteristics
- Any structural issues (deep nesting, long functions, repeated logic)

### Step 4: Compare Against Actual Output

Match your expert findings against the CLI json-full output:

- Do cyclomatic complexity scores match your manual count?
- Does maintainability grade (A-F) align with expected range?
- Are Halstead metrics reasonable for the code complexity?
- Are score interpretations (Low/Moderate/High/Very High) accurate?

### Step 5: Classify Discrepancies

For each discrepancy found, classify as:

- **false-positive**: Analyzer flags an issue that doesn't exist
- **false-negative**: Real issue the analyzer missed
- **severity-mismatch**: Issue detected but wrong severity/grade
- **score-accuracy**: Numeric score is incorrect (e.g., cyclomatic count off by 2+)
- **missing-context**: Finding lacks actionable detail or explanation
- **over-eager-detection**: Technically correct but not practically useful

### Step 6: Trace to Source

For each discrepancy:

1. Identify which analysis is responsible (cyclomatic-complexity, halstead-metrics, or maintainability-index)
2. Read the source file: `analyzers/core/src/analyses/{{ANALYSIS_NAME}}-analysis.ts`
3. Locate the exact function/line causing the issue
4. Explain the root cause

### Step 7: Propose Fix

For each discrepancy, provide:

- **Concrete code change**: Exact lines to modify with before/after snippets
- **Files to modify**: List all files that need changes
- **Expected behavior**: What the output should look like after the fix
- **Test strategy**: How to verify the fix works

## Output Format (JSON)

```json
{
  "fixture": "{{FILENAME}}",
  "intent": "{{INTENT}}",
  "reviewDate": "2026-01-28",
  "discrepancies": [
    {
      "type": "score-accuracy",
      "analysis": "cyclomatic-complexity-analysis",
      "description": "Cyclomatic complexity reported as 5, should be 3",
      "evidence": "Manual count: if (1) + for (1) + || (1) = 3 decision points",
      "sourceFile": "analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts",
      "sourceLocation": "Line 42-56 in calculateComplexity()",
      "rootCause": "Counting logical operators twice when they appear in if conditions",
      "proposedFix": {
        "files": ["analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts"],
        "change": "Modify complexity counter to skip logical operators already counted in parent if statement",
        "codeSnippet": "// Before:\nif (node.kind === SyntaxKind.BinaryExpression && isLogicalOperator(node)) complexity++;\n\n// After:\nif (node.kind === SyntaxKind.BinaryExpression && isLogicalOperator(node) && !isWithinConditional(node)) complexity++;",
        "expectedBehavior": "Cyclomatic complexity should be 3, matching manual count"
      },
      "severity": "high",
      "impactScope": "All files with logical operators in conditionals"
    }
  ],
  "summary": {
    "totalDiscrepancies": 1,
    "byType": { "score-accuracy": 1 },
    "overallAccuracy": "moderate",
    "recommendedPriority": "high"
  }
}
```
````

---

## Template B: React Fixtures

### When to Use

Use this template for fixtures in `packages/fixtures/src/react/`:

- Main fixtures (14 files): AntiPatternShowcase, CouplingComponent, DataFlowComponent, etc.
- Migration fixtures (7 files): SimpleButton, ModernComponent, ClassComponentLegacy, etc.

### Analysis Coverage

React analyzer includes 14 analyses:

1. **structural-analysis**: Component structure, composition patterns
2. **hook-analysis**: Hook usage, custom hooks, dependency arrays
3. **temporal-analysis**: Lifecycle patterns, effect timing
4. **coupling-analysis**: Component dependencies, prop drilling
5. **identity-analysis**: Key props, ref patterns, memoization
6. **dataflow-analysis**: State management, data flow patterns
7. **anti-pattern-analysis**: Common React anti-patterns
8. **security-analysis**: XSS risks, dangerous props, injection vulnerabilities
9. **accessibility-analysis**: ARIA, semantic HTML, keyboard navigation
10. **performance-analysis**: Re-render optimization, expensive operations
11. **reliability-analysis**: Error boundaries, null checks, fallbacks
12. **migration-analysis**: Legacy patterns (class components, PropTypes, old Context API)
13. **technical-debt-analysis**: Code quality, complexity, maintainability
14. **types-analysis**: TypeScript usage, type safety

### Prompt Template

````markdown
# React Analyzer Accuracy Review

## Task

Rigorously review the React analyzer output for a fixture file and identify all discrepancies between expected and actual results.

## Input Variables

- **File**: `packages/fixtures/src/react/{{FILENAME}}`
- **Intent**: {{INTENT}} (e.g., "anti-pattern - should detect accessibility violations" or "good-example - should have no major issues")

## 7-Step Process

### Step 1: Run CLI Analysis

Run: `node clients/cli/dist/index.js analyze packages/fixtures/src/react/{{FILENAME}} -f json-full -q`

### Step 2: Read Source Code

Read the complete fixture file: `packages/fixtures/src/react/{{FILENAME}}`

### Step 3: Expert Review

As a skilled React developer, independently identify:

- **Structural issues**: Component organization, composition anti-patterns
- **Hook issues**: Invalid hook calls, missing dependencies, unnecessary dependencies
- **Performance issues**: Unnecessary re-renders, missing memoization, expensive inline operations
- **Security issues**: XSS vulnerabilities, dangerouslySetInnerHTML, eval usage
- **Accessibility issues**: Missing ARIA labels, invalid semantics, keyboard support gaps
- **Reliability issues**: Missing error boundaries, unhandled edge cases, null pointer risks
- **Anti-patterns**: Props spreading to DOM, index as key, inline function definitions in JSX
- **Migration issues**: Class components, PropTypes, legacy Context API, string refs
- **Type issues**: Missing types, any usage, weak type definitions

### Step 4: Compare Against Actual Output

Match your expert findings against the CLI json-full output across all 14 analyses:

- Are all real issues detected? (check for false-negatives)
- Are flagged issues actually problematic? (check for false-positives)
- Do severity ratings (critical/high/medium/low) match issue impact?
- Are issue descriptions clear and actionable?
- Do recommendations provide concrete fix guidance?

### Step 5: Classify Discrepancies

For each discrepancy found, classify as:

- **false-positive**: Analyzer flags an issue that doesn't exist or is acceptable
- **false-negative**: Real issue the analyzer missed
- **severity-mismatch**: Issue detected but severity is wrong (e.g., critical vs medium)
- **score-accuracy**: Numeric metric is incorrect
- **missing-context**: Finding lacks sufficient explanation or fix guidance
- **over-eager-detection**: Technically correct but not practically useful (common pattern, accepted idiom)

### Step 6: Trace to Source

For each discrepancy:

1. Identify which of the 14 analyses is responsible
2. Read the source file: `analyzers/react/src/analyses/{{ANALYSIS_NAME}}-analysis.ts`
3. Locate the exact function/detector causing the issue
4. Explain the root cause (logic error, missing check, overly strict rule, etc.)

**React analysis source files:**

- `structural-analysis.ts`
- `hook-analysis.ts`
- `temporal-analysis.ts`
- `coupling-analysis.ts`
- `identity-analysis.ts`
- `dataflow-analysis.ts`
- `anti-pattern-analysis.ts`
- `security-analysis.ts`
- `accessibility-analysis.ts`
- `performance-analysis.ts`
- `reliability-analysis.ts`
- `migration-analysis.ts`
- `technical-debt-analysis.ts`
- `types-analysis.ts`

### Step 7: Propose Fix

For each discrepancy, provide:

- **Concrete code change**: Exact lines to modify with before/after snippets
- **Files to modify**: List all analysis files that need changes
- **Expected behavior**: What the output should look like after the fix
- **Test strategy**: Specific test cases to add/modify to prevent regression

## Output Format (JSON)

```json
{
  "fixture": "{{FILENAME}}",
  "intent": "{{INTENT}}",
  "reviewDate": "2026-01-28",
  "discrepancies": [
    {
      "type": "false-negative",
      "analysis": "accessibility-analysis",
      "description": "Missing detection of button without accessible name",
      "evidence": "Line 23: <button onClick={handleClick}><Icon /></button> has no text or aria-label",
      "sourceFile": "analyzers/react/src/analyses/accessibility-analysis.ts",
      "sourceLocation": "checkButtonAccessibility() function",
      "rootCause": "Detector only checks for text children, doesn't handle icon-only buttons",
      "proposedFix": {
        "files": ["analyzers/react/src/analyses/accessibility-analysis.ts"],
        "change": "Add check for aria-label/aria-labelledby when button has no text children",
        "codeSnippet": "if (!hasTextContent(button) && !hasAriaLabel(button)) {\n  addFinding('missing-button-label', 'critical', node);\n}",
        "expectedBehavior": "Should detect missing accessible name on icon-only buttons"
      },
      "severity": "high",
      "impactScope": "All components with icon-only buttons"
    }
  ],
  "summary": {
    "totalDiscrepancies": 1,
    "byType": { "false-negative": 1 },
    "affectedAnalyses": ["accessibility-analysis"],
    "overallAccuracy": "good",
    "recommendedPriority": "high"
  }
}
```
````

````

---

## Template C: Next.js Fixtures

### When to Use

Use this template for fixtures in `packages/fixtures/src/nextjs/`:
- Anti-pattern fixtures (32 files): directive placement, data fetching, component usage, migration, performance, security, etc.
- Good example fixtures (4 files): 29-good-examples-server-component.tsx through 32-good-examples-next-components.tsx
- Helper fixtures: client-button.tsx

### Ground Truth Reference

`packages/fixtures/src/nextjs/index.ts` provides `EXPECTED_ISSUES` map with exact expected detections per file.

### Analysis Coverage

Next.js analyzer includes 7 analyses:
1. **server-client-analysis**: Server/Client Component boundaries, directive placement, serialization
2. **data-fetching-analysis**: getStaticProps, getServerSideProps, fetch patterns, server actions
3. **migration-analysis**: Next.js 13/14/15 breaking changes, deprecated APIs
4. **security-analysis**: Server action validation, environment variables, middleware edge runtime
5. **config-analysis**: next.config.js deprecated options, invalid patterns
6. **route-structure-analysis**: Route handlers, generateStaticParams, middleware matchers
7. **rendering-analysis**: SSR vs SSG vs ISR patterns, caching strategies

### Plugin Coordination Check

For Next.js fixtures, verify:
- Next.js plugin successfully handles the file (check fileType in output)
- React plugin correctly defers (doesn't analyze Next.js files)
- File technology detection is accurate (should show both Next.js and React in hierarchy)

### Prompt Template

```markdown
# Next.js Analyzer Accuracy Review

## Task
Rigorously review the Next.js analyzer output for a fixture file and identify all discrepancies between expected and actual results.

## Input Variables
- **File**: `packages/fixtures/src/nextjs/{{FILENAME}}`
- **Intent**: {{INTENT}} (e.g., "anti-pattern - directive placement issues" or "good-example - best practices")
- **Expected Issues**: {{EXPECTED_ISSUES}} (from EXPECTED_ISSUES map in packages/fixtures/src/nextjs/index.ts)

## 7-Step Process

### Step 1: Run CLI Analysis
Run: `node clients/cli/dist/index.js analyze packages/fixtures/src/nextjs/{{FILENAME}} -f json-full -q`

**Important**: Verify plugin coordination in output:
- Check `fileType` field shows Next.js detection
- Verify Next.js plugin handled the file (not React plugin)
- Confirm technology hierarchy includes both Next.js and React

### Step 2: Read Source Code
Read the complete fixture file: `packages/fixtures/src/nextjs/{{FILENAME}}`

### Step 3: Expert Review
As a skilled Next.js developer, independently identify:
- **Directive issues**: `'use client'`/`'use server'` placement, conflicts, unnecessary usage
- **Serialization issues**: Non-serializable props passed to client components (functions, class instances)
- **Data fetching issues**: Anti-patterns in getStaticProps, getServerSideProps, server actions, fetch patterns
- **Migration issues**: Next.js 13/14/15 breaking changes (async cookies/headers/params, deprecated APIs)
- **Component issues**: next/image, next/link, next/script misuse
- **Security issues**: Server actions without validation/auth, NEXT_PUBLIC misuse, middleware risks
- **Performance issues**: Bundle size (full lodash), hydration mismatches, caching anti-patterns
- **Configuration issues**: Deprecated next.config.js options, invalid route patterns

**Note**: `'use client'` and `'use server'` directives alone do NOT indicate Next.js - these are React Server Component directives. Only flag as Next.js-specific if Next.js imports (next/*, app/, pages/) are present.

### Step 4: Compare Against Actual Output
Match your expert findings against:
1. **Expected issues from ground truth**: All items in EXPECTED_ISSUES array for this file
2. **CLI json-full output**: Actual detections from all 7 Next.js analyses

Check for:
- **Missing detections**: Expected issues not found in output (false-negatives)
- **Extra detections**: Output contains issues not in expected list (could be false-positives OR missed ground truth)
- **Severity alignment**: Critical issues (security, data loss risks) vs medium/low
- **Clarity**: Are issue descriptions clear about what's wrong and how to fix it?

### Step 5: Classify Discrepancies
For each discrepancy found, classify as:
- **false-positive**: Analyzer flags an issue that doesn't exist or is acceptable Next.js pattern
- **false-negative**: Expected issue not detected (check EXPECTED_ISSUES)
- **severity-mismatch**: Issue detected but severity is wrong
- **score-accuracy**: Numeric metric is incorrect
- **missing-context**: Finding lacks sufficient explanation or Next.js-specific fix guidance
- **over-eager-detection**: Technically correct but not practically useful
- **ground-truth-gap**: Real issue detected but missing from EXPECTED_ISSUES (update ground truth)

### Step 6: Trace to Source
For each discrepancy:
1. Identify which of the 7 analyses is responsible
2. Read the source file: `analyzers/nextjs/src/analyses/{{ANALYSIS_NAME}}-analysis.ts`
3. Locate the exact detector/function causing the issue
4. Explain the root cause

**Next.js analysis source files:**
- `server-client-analysis.ts`
- `data-fetching-analysis.ts`
- `migration-analysis.ts`
- `security-analysis.ts`
- `config-analysis.ts`
- `route-structure-analysis.ts`
- `rendering-analysis.ts`

**Plugin coordination files** (if fileType detection is wrong):
- `analyzers/nextjs/src/plugin.ts` - NextJsAnalyzerPlugin
- `analyzers/react/src/plugin.ts` - ReactAnalyzerPlugin (should defer to Next.js)

### Step 7: Propose Fix
For each discrepancy, provide:
- **Concrete code change**: Exact lines to modify with before/after snippets
- **Files to modify**: Analysis files, plugin coordination files, or ground truth (EXPECTED_ISSUES)
- **Expected behavior**: What the output should look like after the fix
- **Test strategy**: Add test cases to prevent regression
- **Ground truth update**: If real issue detected but not in EXPECTED_ISSUES, propose update to ground truth

## Output Format (JSON)

```json
{
  "fixture": "{{FILENAME}}",
  "intent": "{{INTENT}}",
  "expectedIssues": {{EXPECTED_ISSUES}},
  "reviewDate": "2026-01-28",
  "pluginCoordination": {
    "nextjsPluginHandled": true,
    "reactPluginDeferred": true,
    "fileTypeDetection": "correct",
    "technologyHierarchy": ["Next.js", "React"]
  },
  "discrepancies": [
    {
      "type": "false-negative",
      "analysis": "security-analysis",
      "description": "Server action lacks input validation",
      "evidence": "Line 15: async function updateUser(userId, data) { await db.update(...) } - no zod schema validation",
      "expectedInGroundTruth": true,
      "sourceFile": "analyzers/nextjs/src/analyses/security-analysis.ts",
      "sourceLocation": "checkServerActionValidation() function",
      "rootCause": "Detector only checks for zod imports at file level, doesn't handle inline validation",
      "proposedFix": {
        "files": ["analyzers/nextjs/src/analyses/security-analysis.ts"],
        "change": "Expand validation check to look for runtime type checks, not just zod imports",
        "codeSnippet": "const hasValidation = hasZodImport || hasRuntimeTypeCheck || hasInputSanitization;",
        "expectedBehavior": "Should detect missing input validation on server actions",
        "groundTruthUpdate": "Verify EXPECTED_ISSUES['21-security-server-actions.ts'] includes this"
      },
      "severity": "critical",
      "impactScope": "All Next.js server actions"
    }
  ],
  "summary": {
    "totalDiscrepancies": 1,
    "byType": {"false-negative": 1},
    "affectedAnalyses": ["security-analysis"],
    "overallAccuracy": "good",
    "recommendedPriority": "critical"
  }
}
````

````

---

## Fixture Inventory Table

Use this table to quickly find the correct template and input variables for each fixture.

### Core Fixtures

| File | Template | Intent | Expected Complexity | Expected Cyclomatic | Expected Maintainability |
|------|----------|--------|---------------------|---------------------|--------------------------|
| `simple-function.ts` | A | good-example - should have low complexity | Low | 1-2 | A (85-100) |
| `moderate-complexity.ts` | A | good-example - demonstrates moderate complexity | Moderate | 5-10 | B-C (65-84) |
| `high-complexity.ts` | A | anti-pattern - demonstrates high complexity | High | 10-20 | D (40-64) |
| `very-complex.ts` | A | anti-pattern - demonstrates very high complexity | Very High | 20+ | F (0-39) |

### React Fixtures

| File | Template | Intent | Category |
|------|----------|--------|----------|
| `SimpleComponent.tsx` | B | good-example - clean React component | main |
| `AntiPatternShowcase.tsx` | B | anti-pattern - multiple React anti-patterns | main |
| `InaccessibleComponent.tsx` | B | anti-pattern - accessibility violations | main |
| `InsecureComponent.tsx` | B | anti-pattern - security vulnerabilities | main |
| `PerformanceIssuesComponent.tsx` | B | anti-pattern - performance problems | main |
| `ReliabilityComponent.tsx` | B | anti-pattern - reliability issues | main |
| `HookPatternComponent.tsx` | B | anti-pattern - hook misuse patterns | main |
| `DataFlowComponent.tsx` | B | anti-pattern - data flow issues | main |
| `CouplingComponent.tsx` | B | anti-pattern - tight coupling | main |
| `IdentityPatternsComponent.tsx` | B | anti-pattern - identity/key issues | main |
| `TypeComplexityComponent.tsx` | B | anti-pattern - type complexity issues | main |
| `ProblematicComponent.tsx` | B | anti-pattern - general problematic patterns | main |
| `DataTable.tsx` | B | mixed - complex component with some issues | main |
| `SearchInput.tsx` | B | mixed - input component with accessibility/performance considerations | main |
| `migration/SimpleButton.tsx` | B | good-example - modern React patterns | migration |
| `migration/ModernComponent.tsx` | B | good-example - modern React patterns | migration |
| `migration/ClassComponentLegacy.tsx` | B | anti-pattern - legacy class component | migration |
| `migration/LegacyContextExample.tsx` | B | anti-pattern - legacy context API | migration |
| `migration/PropTypesExample.tsx` | B | anti-pattern - PropTypes usage | migration |
| `migration/StringRefsExample.tsx` | B | anti-pattern - string refs | migration |
| `migration/React19Features.tsx` | B | good-example - React 19 features | migration |

### Next.js Fixtures

| File | Template | Intent | Expected Issues (Count) | Category |
|------|----------|--------|-------------------------|----------|
| `01-directive-placement-issues.tsx` | C | anti-pattern - directive placement | 1 | serverClientComponents |
| `02-conflicting-directives.tsx` | C | anti-pattern - directive conflicts | 2 | serverClientComponents |
| `03-unnecessary-client-component.tsx` | C | anti-pattern - unnecessary client directive | 1 | serverClientComponents |
| `04-missing-use-client-directive.tsx` | C | anti-pattern - missing client directive | 2 | serverClientComponents |
| `05-non-serializable-props.tsx` | C | anti-pattern - serialization issues | 1 | serverClientComponents |
| `06-data-fetching-pages-router.tsx` | C | anti-pattern - Pages Router data fetching | 3 | dataFetching |
| `07-missing-static-paths.tsx` | C | anti-pattern - missing getStaticPaths | 1 | dataFetching |
| `08-data-fetching-app-router.tsx` | C | anti-pattern - App Router data fetching | 2 | dataFetching |
| `09-server-actions-no-revalidation.ts` | C | anti-pattern - server action without revalidation | 1 | dataFetching |
| `10-next-image-issues.tsx` | C | anti-pattern - next/image misuse | 7 | nextjsComponents |
| `11-next-link-issues.tsx` | C | anti-pattern - next/link misuse | 3 | nextjsComponents |
| `12-next-script-issues.tsx` | C | anti-pattern - next/script misuse | 3 | nextjsComponents |
| `13-nextjs-15-async-request-apis.tsx` | C | anti-pattern - Next.js 15 breaking changes | 1 | migration |
| `14-nextjs-15-sync-params.tsx` | C | anti-pattern - Next.js 15 params changes | 1 | migration |
| `15-nextjs-15-other-breaking-changes.tsx` | C | anti-pattern - Next.js 15 various changes | 4 | migration |
| `16-nextjs-13-migration.tsx` | C | anti-pattern - Next.js 13 deprecated patterns | 4 | migration |
| `17-wrong-router-import.tsx` | C | anti-pattern - wrong router for context | 1 | migration |
| `18-performance-bundle-size.tsx` | C | anti-pattern - bundle size issues | 3 | performance |
| `19-performance-hydration-mismatch.tsx` | C | anti-pattern - hydration mismatches | 3 | performance |
| `20-performance-caching-issues.tsx` | C | anti-pattern - caching anti-patterns | 3 | performance |
| `21-security-server-actions.ts` | C | anti-pattern - server action security | 3 | security |
| `22-security-env-variables.tsx` | C | anti-pattern - environment variable misuse | 3 | security |
| `23-security-middleware.ts` | C | anti-pattern - middleware security | 3 | security |
| `24-accessibility-missing-lang.tsx` | C | anti-pattern - accessibility in Next.js | 1 | accessibility |
| `25-typescript-pages-router.tsx` | C | anti-pattern - TypeScript Pages Router | 1 | typescript |
| `26-typescript-app-router.tsx` | C | anti-pattern - TypeScript App Router | 1 | typescript |
| `27-config-deprecated-options.js` | C | anti-pattern - next.config.js issues | 1 | configuration |
| `28-config-route-issues.tsx` | C | anti-pattern - route configuration issues | 3 | configuration |
| `29-good-examples-server-component.tsx` | C | good-example - proper server component | 0 | goodExamples |
| `30-good-examples-client-component.tsx` | C | good-example - proper client component | 0 | goodExamples |
| `31-good-examples-server-actions.ts` | C | good-example - proper server actions | 0 | goodExamples |
| `32-good-examples-next-components.tsx` | C | good-example - proper Next.js components | 0 | goodExamples |
| `39-server-client-correct.tsx` | C | good-example - correct server/client usage | 0 | integration |
| `client-button.tsx` | C | helper - reusable client component | 0 | helpers |

---

## Template D: Aggregation Prompt

Run this prompt ONCE after completing all individual fixture reviews in Phase 1.

### Input Preparation

Collect all JSON outputs from Phase 1 reviews into a single file or concatenated format.

### Prompt Template

```markdown
# Aggregate QA Review: Priority Action Plan

## Task
Synthesize all individual fixture review findings into a prioritized action plan for fixing analyzer accuracy issues.

## Input
All JSON outputs from Phase 1 reviews (paste below):

````

[Paste all JSON review outputs here]

```

## Aggregation Process

### Step 1: Group by Source File
Organize all discrepancies by the source analysis file they originate from.

### Step 2: Calculate Priority Scores
For each source file with issues, calculate priority score:

```

priority_score = (false_negatives × 3) + (false_positives × 2) + (severity_mismatches × 1.5) + (score_accuracy × 1) + (breadth_factor × 0.5)

````

Where:
- **false_negatives**: Count of missed real issues (highest impact)
- **false_positives**: Count of incorrect flags (noise, user trust)
- **severity_mismatches**: Count of wrong severity ratings
- **score_accuracy**: Count of incorrect numeric metrics
- **breadth_factor**: Number of different fixture files affected

**Safety multiplier**: Apply 1.5× weight to security-analysis, accessibility-analysis, reliability-analysis

### Step 3: Identify Patterns
Look for:
- Repeated root causes across multiple fixtures
- Systematic gaps (e.g., all icon-only button accessibility issues)
- Overly strict detectors causing many false-positives
- Missing entire categories of checks

### Step 4: Generate Action Plan
For each source file, provide:
1. **File**: Path to analysis file needing changes
2. **Priority**: Critical/High/Medium/Low (based on score)
3. **Issues**: Count and types of discrepancies
4. **Affected Fixtures**: List of fixtures with issues from this analysis
5. **Root Causes**: Common patterns across discrepancies
6. **Recommended Changes**: High-level fix strategy
7. **Estimated Scope**: Small (1 function) / Medium (multiple functions) / Large (architectural)

## Output Format (Markdown)

```markdown
# QA Aggregation Report
Generated: 2026-01-28

## Summary Statistics
- Total fixtures reviewed: X
- Total discrepancies found: Y
- Analysis files needing fixes: Z
- Critical priority: N files
- High priority: M files

## Discrepancy Breakdown
| Type | Count | % of Total |
|------|-------|------------|
| False Negatives | X | Y% |
| False Positives | X | Y% |
| Severity Mismatches | X | Y% |
| Score Accuracy | X | Y% |
| Missing Context | X | Y% |
| Over-Eager Detection | X | Y% |

## Priority Action Plan

### Critical Priority

#### 1. `analyzers/nextjs/src/analyses/security-analysis.ts` (Score: 45)
- **Issues**: 8 false-negatives, 2 severity-mismatches
- **Affected Fixtures**: 21-security-server-actions.ts, 22-security-env-variables.tsx, 23-security-middleware.ts
- **Root Causes**:
  - Server action validation only checks for zod imports
  - NEXT_PUBLIC detection misses dynamic access patterns
  - Middleware edge runtime checks too strict
- **Recommended Changes**:
  - Expand validation detection to runtime checks, sanitization
  - Add AST traversal for dynamic env variable access
  - Relax edge runtime checks for safe Node.js APIs
- **Estimated Scope**: Medium (3 functions need modification)

[Continue for all files requiring changes...]

### High Priority
[...]

### Medium Priority
[...]

### Low Priority
[...]

## Patterns Observed

### Systematic Gaps
1. **Accessibility icon-only buttons**: Multiple fixtures show missing detection of buttons without accessible names when only icons are present
2. **Dynamic access patterns**: Static analysis misses computed property access (e.g., `process.env[key]`)
3. **Next.js 15 async APIs**: Detection misses await on cookies()/headers() but catches them when assigned to variables

### Over-Eager Detection
1. **Logical operators in conditionals**: Core analyzer counts them twice (once in if, once as operator)
2. **Accepted React patterns**: Some idiomatic patterns flagged as anti-patterns (e.g., index as key for static lists)

### Ground Truth Updates Needed
- `EXPECTED_ISSUES` map missing 3 legitimate detections in good-example fixtures
- Recommend adding to ground truth: [list specifics]

## Recommended Implementation Order
1. Security-critical fixes first (security-analysis, accessibility-analysis)
2. High-false-negative issues (missing real problems)
3. High-false-positive issues (noise reduction)
4. Score accuracy improvements
5. Missing context / better messaging
````

````

---

## Template E: Verification Prompt

Run this prompt ONCE per fix after implementing changes from the action plan.

### Prompt Template

```markdown
# Fix Verification Review

## Task
Verify that a fix to an analysis file works correctly and doesn't introduce regressions.

## Input Variables
- **Fixed File**: `{{ANALYSIS_FILE_PATH}}` (e.g., `analyzers/nextjs/src/analyses/security-analysis.ts`)
- **Related Fixtures**: {{FIXTURE_LIST}} (fixtures that should be affected by the fix)
- **Expected Behavior**: {{EXPECTED_BEHAVIOR}} (what should change in output)

## Verification Process

### Step 1: Rebuild Project
```bash
pnpm build
````

Verify build succeeds with no errors.

### Step 2: Run Unit Tests

```bash
pnpm test analyzers/{{PLUGIN}}/src/analyses/{{ANALYSIS_NAME}}-analysis.test.ts
```

Verify all tests pass. If tests fail, analyze whether:

- Tests need updating to match new behavior (expected)
- Fix introduced a regression (unexpected - needs correction)

### Step 3: Re-Run Affected Fixtures

For each fixture in the Related Fixtures list:

```bash
node clients/cli/dist/index.js analyze packages/fixtures/src/{{CATEGORY}}/{{FIXTURE}} -f json-full -q
```

Compare output to expected behavior:

- Did the fix resolve the discrepancy?
- Are outputs now matching expected ground truth?
- Any new unexpected issues introduced?

### Step 4: Regression Check

Run CLI analysis on ALL fixtures in the same category:

**Core**: All 4 fixtures in `packages/fixtures/src/core/`
**React**: All 21 fixtures in `packages/fixtures/src/react/`
**Next.js**: All 32+ fixtures in `packages/fixtures/src/nextjs/`

Check for:

- New false-positives on previously clean fixtures
- Changed severity ratings on unrelated issues
- Performance degradation (significantly slower analysis)

### Step 5: Integration Test

```bash
pnpm test
```

Run full test suite. All tests should pass.

## Output Format (Markdown)

```markdown
# Verification Report: {{ANALYSIS_FILE_PATH}}

Date: 2026-01-28

## Build Status

- [ ] Build succeeded
- [ ] No new TypeScript errors

## Unit Test Status

- [ ] All existing tests pass
- [ ] New tests added: {{COUNT}}
- [ ] Tests updated to match new behavior: {{COUNT}}

## Affected Fixture Verification

### Fixture: {{FIXTURE_1}}

- **Expected Behavior**: {{EXPECTED}}
- **Actual Behavior**: {{ACTUAL}}
- **Status**: PASS / FAIL
- **Notes**: [Any observations]

[Repeat for each affected fixture...]

## Regression Check Results

### Category: {{CATEGORY}} ({{COUNT}} fixtures tested)

- **New false-positives**: {{COUNT}}
- **Changed severities**: {{COUNT}}
- **Performance impact**: {{PERCENTAGE}} (acceptable if &lt;10%)
- **Unexpected changes**: [List any]

## Integration Test Status

- [ ] Full test suite passes
- [ ] No new test failures

## Overall Verdict

**PASS** / **PASS-WITH-NOTES** / **FAIL**

### Notes

[Explain any concerns, unexpected behaviors, or recommendations for follow-up]

### Recommendation

- [ ] Ready to merge
- [ ] Needs additional adjustments: [describe]
- [ ] Needs follow-up fixes: [describe]
```

````

---

## Execution Guide

### Recommended Review Order

Follow this order to maximize efficiency and catch issues systematically.

#### Round 1: Good Examples First (Establish False-Positive Baseline)

Review these fixtures to establish what SHOULD NOT be flagged:

**Core (1 fixture)**
1. `simple-function.ts`

**React (4 fixtures)**
2. `SimpleComponent.tsx`
3. `migration/SimpleButton.tsx`
4. `migration/ModernComponent.tsx`
5. `migration/React19Features.tsx`

**Next.js (5 fixtures)**
6. `29-good-examples-server-component.tsx`
7. `30-good-examples-client-component.tsx`
8. `31-good-examples-server-actions.ts`
9. `32-good-examples-next-components.tsx`
10. `39-server-client-correct.tsx`

**Rationale**: Any issues detected in good-example fixtures are likely false-positives. Fix these first to reduce noise.

#### Round 2: One Anti-Pattern Per Analysis Domain (Breadth Coverage)

Review one fixture for each major analysis area to identify systematic gaps:

**Core (3 fixtures)**
11. `moderate-complexity.ts`
12. `high-complexity.ts`
13. `very-complex.ts`

**React (7 fixtures - covering all 14 analyses)**
14. `InaccessibleComponent.tsx` (accessibility)
15. `InsecureComponent.tsx` (security)
16. `PerformanceIssuesComponent.tsx` (performance)
17. `AntiPatternShowcase.tsx` (anti-patterns, structural)
18. `HookPatternComponent.tsx` (hooks, temporal)
19. `ReliabilityComponent.tsx` (reliability, technical-debt)
20. `migration/ClassComponentLegacy.tsx` (migration, types)

**Next.js (7 fixtures - covering all 7 analyses)**
21. `01-directive-placement-issues.tsx` (server-client)
22. `08-data-fetching-app-router.tsx` (data-fetching)
23. `21-security-server-actions.ts` (security)
24. `13-nextjs-15-async-request-apis.tsx` (migration)
25. `27-config-deprecated-options.js` (config)
26. `28-config-route-issues.tsx` (route-structure)
27. `20-performance-caching-issues.tsx` (rendering)

**Rationale**: Covers all analysis domains with one representative fixture each. Findings here indicate systematic issues across entire analysis categories.

#### Round 3: Remaining Fixtures (Complete Coverage)

Review all remaining fixtures to catch edge cases and less common patterns:

**React (10 remaining fixtures)**
- CouplingComponent.tsx
- DataFlowComponent.tsx
- DataTable.tsx
- IdentityPatternsComponent.tsx
- ProblematicComponent.tsx
- SearchInput.tsx
- TypeComplexityComponent.tsx
- migration/LegacyContextExample.tsx
- migration/PropTypesExample.tsx
- migration/StringRefsExample.tsx

**Next.js (20 remaining anti-pattern fixtures)**
- 02-conflicting-directives.tsx through 07-missing-static-paths.tsx
- 09-server-actions-no-revalidation.ts through 12-next-script-issues.tsx
- 14-nextjs-15-sync-params.tsx through 19-performance-hydration-mismatch.tsx
- 22-security-env-variables.tsx through 26-typescript-app-router.tsx

**Rationale**: Now that systematic issues are identified from Rounds 1-2, these reviews will primarily catch edge cases and validate fixes.

#### Round 4: Aggregation (Phase 2)

Run Template D (Aggregation Prompt) once with all collected JSON review outputs.

**Output**: Prioritized action plan for implementing fixes.

### Implementation Phase (Phase 3)

Work through the action plan from Round 4 in priority order:
1. Critical priority fixes
2. High priority fixes
3. Medium priority fixes
4. Low priority fixes

For each fix:
1. Make code changes
2. Add/update tests
3. Run Template E (Verification Prompt)
4. Only proceed to next fix if verification passes

### Time Estimates

- **Phase 1 (Per-Fixture Review)**: 5-10 minutes per fixture (7-10 hours total for all 74 fixtures)
- **Phase 2 (Aggregation)**: 15-30 minutes
- **Phase 3 (Implementation)**: Varies by fix complexity (2-40 hours total estimated)
- **Phase 4 (Verification)**: 10-15 minutes per fix

### Best Practices

1. **Batch reviews by category**: Do all Core, then all React, then all Next.js to maintain context
2. **Save JSON outputs**: Keep all Phase 1 JSON outputs in a dedicated folder for easy aggregation
3. **Track progress**: Use a checklist to mark completed fixtures
4. **Document surprises**: If a fixture behaves unexpectedly, note it for follow-up
5. **Update ground truth**: If legitimate issues are detected but not in EXPECTED_ISSUES, update the ground truth
6. **Run verification after each fix**: Don't batch fixes - verify each one individually to isolate regressions

---

## Reference Files

### Key Source Files

| File | Purpose |
|------|---------|
| `packages/fixtures/src/nextjs/index.ts` | Ground truth: EXPECTED_ISSUES map for Next.js fixtures |
| `packages/fixtures/src/core/index.ts` | Ground truth: FIXTURE_METADATA with expected complexity ranges |
| `packages/fixtures/src/react/index.ts` | React fixture inventory and categorization |
| `analyzers/core/src/analyses/*.ts` | 3 core analysis implementations |
| `analyzers/react/src/analyses/*.ts` | 14 React analysis implementations |
| `analyzers/nextjs/src/analyses/*.ts` | 7 Next.js analysis implementations |
| `clients/cli/src/formatters/full-json-formatter.ts` | Defines json-full output structure |
| `packages/common/src/types/output/full.ts` | TypeScript types for full JSON output |
| `analyzers/nextjs/src/plugin.ts` | Next.js plugin (handles Next.js files) |
| `analyzers/react/src/plugin.ts` | React plugin (defers to Next.js when appropriate) |

### CLI Commands

**Run analysis with full JSON output:**
```bash
node clients/cli/dist/index.js analyze <file-path> -f json-full -q
````

**Run analysis with summary JSON output:**

```bash
node clients/cli/dist/index.js analyze <file-path> -f json -q
```

**Build project:**

```bash
pnpm build
```

**Run tests:**

```bash
pnpm test
```

**Run specific test file:**

```bash
pnpm test <path-to-test-file>
```

---

## Appendix: Discrepancy Type Definitions

### false-positive

The analyzer flags an issue that doesn't exist or is an acceptable pattern in the framework/library context.

**Example**: Flagging `index` as a key in a static list that never reorders.

**Impact**: Noise, reduced user trust, wasted time investigating non-issues.

**Priority**: High for frequent occurrences, Medium otherwise.

### false-negative

The analyzer misses a real issue that should be detected based on the fixture intent or ground truth.

**Example**: Missing detection of a server action without input validation.

**Impact**: Security risks, bugs not caught, reduced tool value.

**Priority**: Critical for security/accessibility, High otherwise.

### severity-mismatch

The analyzer detects the issue but assigns the wrong severity level.

**Example**: Marking a critical security vulnerability as "medium" severity.

**Impact**: Users may ignore critical issues or over-prioritize minor ones.

**Priority**: High for safety-critical analyses (security, accessibility, reliability), Medium otherwise.

### score-accuracy

Numeric metrics are incorrect (e.g., cyclomatic complexity off by 2+, maintainability index in wrong grade).

**Example**: Reporting cyclomatic complexity of 8 when the actual count is 3.

**Impact**: Misleading metrics, incorrect complexity classification.

**Priority**: Medium (affects trust in metrics but not actionability).

### missing-context

The finding is technically correct but lacks sufficient explanation, fix guidance, or contextual information.

**Example**: "Component has performance issues" without explaining what's slow or how to fix it.

**Impact**: Reduced actionability, users don't know how to fix the issue.

**Priority**: Low to Medium (annoying but not blocking).

### over-eager-detection

The analyzer is technically correct but flags accepted idioms or patterns that are not practically problematic.

**Example**: Flagging every inline function in JSX even when memoization would add no benefit.

**Impact**: Noise, reduced signal-to-noise ratio, user fatigue.

**Priority**: Medium (important for tool usability).

---

## Document Metadata

- **Created**: 2026-01-28
- **Purpose**: QA feedback loop for analyzer accuracy
- **Audience**: Developers running systematic QA reviews of analyzer output
- **Maintenance**: Update fixture inventory when new fixtures are added; update analysis lists when new analyses are implemented
