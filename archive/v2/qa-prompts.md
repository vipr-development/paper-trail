Phase 1: Per-Fixture Reviews (27 prompts)

Round 1: Good Examples First (10 fixtures)

Prompt 1 - Core Good Example:
Review core analyzer accuracy for packages/fixtures/src/core/simple-function.ts using Template A from qa-feedback-loop.md. Variables:
FILENAME=simple-function.ts, INTENT="good-example - should have low complexity scores", EXPECTED_COMPLEXITY="Low",
EXPECTED_CYCLOMATIC="1-2", EXPECTED_MAINTAINABILITY="A (85-100)". Follow all 7 steps and output JSON. Save output as
documentation/docs/qa/simple-function-review.json.

Prompt 2 - React Good Example:
Review React analyzer accuracy for packages/fixtures/src/react/SimpleComponent.tsx using Template B from qa-feedback-loop.md. Variables:
FILENAME=SimpleComponent.tsx, INTENT="good-example - clean React component". Follow all 7 steps and output JSON. Save output as
documentation/docs/qa/SimpleComponent-review.json.

Prompt 3 - React Migration Good Example:
Review React analyzer accuracy for packages/fixtures/src/react/migration/SimpleButton.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=migration/SimpleButton.tsx, INTENT="good-example - modern React patterns". Follow all 7 steps and output JSON. Save
output as documentation/docs/qa/SimpleButton-review.json.

Prompt 4 - React Migration Good Example:
Review React analyzer accuracy for packages/fixtures/src/react/migration/ModernComponent.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=migration/ModernComponent.tsx, INTENT="good-example - modern React patterns". Follow all 7 steps and output JSON. Save
output as documentation/docs/qa/ModernComponent-review.json.

Prompt 5 - React 19 Features:
Review React analyzer accuracy for packages/fixtures/src/react/migration/React19Features.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=migration/React19Features.tsx, INTENT="good-example - React 19 features". Follow all 7 steps and output JSON. Save
output as documentation/docs/qa/React19Features-review.json.

Prompt 6 - Next.js Server Component Good Example:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/29-good-examples-server-component.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=29-good-examples-server-component.tsx, INTENT="good-example - proper server component",
EXPECTED_ISSUES=0. Follow all 7 steps including plugin coordination check. Output JSON. Save as
documentation/docs/qa/29-good-examples-server-component-review.json.

Prompt 7 - Next.js Client Component Good Example:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/30-good-examples-client-component.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=30-good-examples-client-component.tsx, INTENT="good-example - proper client component",
EXPECTED_ISSUES=0. Follow all 7 steps including plugin coordination check. Output JSON. Save as
documentation/docs/qa/30-good-examples-client-component-review.json.

Prompt 8 - Next.js Server Actions Good Example:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/31-good-examples-server-actions.ts using Template C from
qa-feedback-loop.md. Variables: FILENAME=31-good-examples-server-actions.ts, INTENT="good-example - proper server actions",
EXPECTED_ISSUES=0. Follow all 7 steps including plugin coordination check. Output JSON. Save as documentation/docs/qa/31-good-examples-server-actions-review.json.

Prompt 9 - Next.js Components Good Example:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/32-good-examples-next-components.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=32-good-examples-next-components.tsx, INTENT="good-example - proper Next.js components",
EXPECTED_ISSUES=0. Follow all 7 steps including plugin coordination check. Output JSON. Save as
documentation/docs/qa/32-good-examples-next-components-review.json.

Prompt 10 - Next.js Integration Good Example:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/39-server-client-correct.tsx using Template C from qa-feedback-loop.md.
Variables: FILENAME=39-server-client-correct.tsx, INTENT="good-example - correct server/client usage", EXPECTED_ISSUES=0. Follow all 7 steps
including plugin coordination check. Output JSON. Save as documentation/docs/qa/39-server-client-correct-review.json.

Round 2: One Anti-Pattern Per Analysis Domain (17 fixtures)

Prompt 11 - Core Moderate Complexity:
Review core analyzer accuracy for packages/fixtures/src/core/moderate-complexity.ts using Template A from qa-feedback-loop.md. Variables:
FILENAME=moderate-complexity.ts, INTENT="good-example - demonstrates moderate complexity", EXPECTED_COMPLEXITY="Moderate",
EXPECTED_CYCLOMATIC="5-10", EXPECTED_MAINTAINABILITY="B-C (65-84)". Follow all 7 steps and output JSON. Save as
documentation/docs/qa/moderate-complexity-review.json.

Prompt 12 - Core High Complexity:
Review core analyzer accuracy for packages/fixtures/src/core/high-complexity.ts using Template A from qa-feedback-loop.md. Variables:
FILENAME=high-complexity.ts, INTENT="anti-pattern - demonstrates high complexity", EXPECTED_COMPLEXITY="High", EXPECTED_CYCLOMATIC="10-20",
EXPECTED_MAINTAINABILITY="D (40-64)". Follow all 7 steps and output JSON. Save as documentation/docs/qa/high-complexity-review.json.

Prompt 13 - Core Very Complex:
Review core analyzer accuracy for packages/fixtures/src/core/very-complex.ts using Template A from qa-feedback-loop.md. Variables:
FILENAME=very-complex.ts, INTENT="anti-pattern - demonstrates very high complexity", EXPECTED_COMPLEXITY="Very High",
EXPECTED_CYCLOMATIC="20+", EXPECTED_MAINTAINABILITY="F (0-39)". Follow all 7 steps and output JSON. Save as documentation/docs/qa/very-complex-review.json.

Prompt 14 - React Accessibility:
Review React analyzer accuracy for packages/fixtures/src/react/InaccessibleComponent.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=InaccessibleComponent.tsx, INTENT="anti-pattern - accessibility violations". Follow all 7 steps focusing on
accessibility-analysis. Output JSON. Save as documentation/docs/qa/InaccessibleComponent-review.json.

Prompt 15 - React Security:
Review React analyzer accuracy for packages/fixtures/src/react/InsecureComponent.tsx using Template B from qa-feedback-loop.md. Variables:
FILENAME=InsecureComponent.tsx, INTENT="anti-pattern - security vulnerabilities". Follow all 7 steps focusing on security-analysis. Output
JSON. Save as documentation/docs/qa/InsecureComponent-review.json.

Prompt 16 - React Performance:
Review React analyzer accuracy for packages/fixtures/src/react/PerformanceIssuesComponent.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=PerformanceIssuesComponent.tsx, INTENT="anti-pattern - performance problems". Follow all 7 steps focusing on
performance-analysis. Output JSON. Save as documentation/docs/qa/PerformanceIssuesComponent-review.json.

Prompt 17 - React Anti-Patterns:
Review React analyzer accuracy for packages/fixtures/src/react/AntiPatternShowcase.tsx using Template B from qa-feedback-loop.md. Variables:
FILENAME=AntiPatternShowcase.tsx, INTENT="anti-pattern - multiple React anti-patterns". Follow all 7 steps focusing on
anti-pattern-analysis and structural-analysis. Output JSON. Save as documentation/docs/qa/AntiPatternShowcase-review.json.

Prompt 18 - React Hooks:
Review React analyzer accuracy for packages/fixtures/src/react/HookPatternComponent.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=HookPatternComponent.tsx, INTENT="anti-pattern - hook misuse patterns". Follow all 7 steps focusing on hook-analysis and
temporal-analysis. Output JSON. Save as documentation/docs/qa/HookPatternComponent-review.json.

Prompt 19 - React Reliability:
Review React analyzer accuracy for packages/fixtures/src/react/ReliabilityComponent.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=ReliabilityComponent.tsx, INTENT="anti-pattern - reliability issues". Follow all 7 steps focusing on
reliability-analysis and technical-debt-analysis. Output JSON. Save as documentation/docs/qa/ReliabilityComponent-review.json.

Prompt 20 - React Migration:
Review React analyzer accuracy for packages/fixtures/src/react/migration/ClassComponentLegacy.tsx using Template B from qa-feedback-loop.md.
Variables: FILENAME=migration/ClassComponentLegacy.tsx, INTENT="anti-pattern - legacy class component". Follow all 7 steps focusing on
migration-analysis and types-analysis. Output JSON. Save as documentation/docs/qa/ClassComponentLegacy-review.json.

Prompt 21 - Next.js Directive Placement:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/01-directive-placement-issues.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=01-directive-placement-issues.tsx, INTENT="anti-pattern - directive placement", EXPECTED_ISSUES=1.
Read expected issues from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as
documentation/docs/qa/01-directive-placement-issues-review.json.

Prompt 22 - Next.js Data Fetching:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/08-data-fetching-app-router.tsx using Template C from qa-feedback-loop.md.
Variables: FILENAME=08-data-fetching-app-router.tsx, INTENT="anti-pattern - App Router data fetching", EXPECTED_ISSUES=2. Read expected
issues from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as documentation/docs/qa/08-data-fetching-app-router-review.json.

Prompt 23 - Next.js Security:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/21-security-server-actions.ts using Template C from qa-feedback-loop.md.
Variables: FILENAME=21-security-server-actions.ts, INTENT="anti-pattern - server action security", EXPECTED_ISSUES=3. Read expected issues
from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as documentation/docs/qa/21-security-server-actions-review.json.

Prompt 24 - Next.js Migration:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/13-nextjs-15-async-request-apis.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=13-nextjs-15-async-request-apis.tsx, INTENT="anti-pattern - Next.js 15 breaking changes",
EXPECTED_ISSUES=1. Read expected issues from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as
documentation/docs/qa/13-nextjs-15-async-request-apis-review.json.

Prompt 25 - Next.js Config:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/27-config-deprecated-options.js using Template C from qa-feedback-loop.md.
Variables: FILENAME=27-config-deprecated-options.js, INTENT="anti-pattern - next.config.js issues", EXPECTED_ISSUES=1. Read expected issues
from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as documentation/docs/qa/27-config-deprecated-options-review.json.

Prompt 26 - Next.js Route Structure:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/28-config-route-issues.tsx using Template C from qa-feedback-loop.md.
Variables: FILENAME=28-config-route-issues.tsx, INTENT="anti-pattern - route configuration issues", EXPECTED_ISSUES=3. Read expected issues
from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as documentation/docs/qa/28-config-route-issues-review.json.

Prompt 27 - Next.js Performance/Rendering:
Review Next.js analyzer accuracy for packages/fixtures/src/nextjs/20-performance-caching-issues.tsx using Template C from
qa-feedback-loop.md. Variables: FILENAME=20-performance-caching-issues.tsx, INTENT="anti-pattern - caching anti-patterns",
EXPECTED_ISSUES=3. Read expected issues from packages/fixtures/src/nextjs/index.ts. Follow all 7 steps. Output JSON. Save as
documentation/docs/qa/20-performance-caching-issues-review.json.

---

Phase 2: Aggregation (1 prompt)

Prompt 28 - Aggregate All Findings:
Run Template D (Aggregation Prompt) from qa-feedback-loop.md. Collect all 27 JSON review outputs from Phase 1. Follow the aggregation
process: group by source file, calculate priority scores, identify patterns, generate action plan. Output prioritized markdown action plan.
Save as qa-aggregation-action-plan.md.

---

Phase 3: Implementation (Manual)

Work through the prioritized action plan from Prompt 28. For each fix:

1. Read the analysis file identified
2. Make the code changes
3. Add/update tests
4. Run Phase 4 verification prompt (below)

---

Phase 4: Verification (1 prompt per fix)

Template for each fix verification:
Run Template E (Verification Prompt) from qa-feedback-loop.md for the fix to `{{ANALYSIS_FILE_PATH}}`. Variables: `FIXED_FILE={{path}}`,
`RELATED_FIXTURES={{list}}`, `EXPECTED_BEHAVIOR={{description}}`. Follow all 5 verification steps: rebuild, unit tests, affected fixtures,
regression check, integration test. Output markdown verification report. Save as verification-`{{analysis-name}}`.md.
