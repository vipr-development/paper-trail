# CLI-Parity in VSCode extension UX

The following subagents should work together to create an HTML and CSS parity with the CLI output and the VSCode extension. They should feel like one brand. The VScode extension sidebar should have a similar approach to per-file metric reporting, except with sections from individual analyses shown either via a
dropdown or in-pane tabs. If there is a more VSCode way of showing these different sections, let's discuss that in your plan proposal and give me options.

## Feature requirements

- Keep the radial "Overall Score" chart with the score inside of it. That looks great.
- Let's use Tabler.io icons for all icons via `npm install @tabler/icons --save`. We'll need to consider tree-shaking and bundling now or down the road. I'm also okay with storing the SVGs directly for savings
- It may be more cost-effective to build our own single-bar horizontal bar chart to compliment the CLI. The bar and the data values associated with each bar should vary in color, depending upon the value of the metric. For example, a Security value of 0% would have an empty but darkly-shaded red bar with a `0%` red text to the right of it.
- The report should be ripe with info icon hover/click information overlays. I would like to see a custom modal that shows up over the side pane "dashboard" view, with a blurry background. This modal is height adaptive and centered. If the information is brief, the explanation only takes up the real estate needed visually. This modal sets us up to provide rich context and education into what these metrics mean. For example, we should be educating engineers on why Cyclomatic Complexity in a temporal space is cause for concern. These markdown hint should live with the respective analyses in the analyzers and their types updated to include this information. Presently, we have descriptions but we should have separate markdown files that live in their own directory for these metrics to educate users. These can be stringified and passed along to the respective clients (e.g., CLI, VSCode Extension, Desktop application).
- The logo for light modes is in `packages/common/src/images/logo-on-light-trimmed.png` and for darker modes, `packages/common/src/images/loog-on-dark-trimmed.png`. You can put that in the top-left of the pane in place of the text, about the same height as the "Overall Score" section.
- Make sure your charts and overal theme continue to blend within the color theme provided by the user in VSCode.

@typescript-refactoring-expert
@vscode-extension-optimizer
@vitest-engineer
@vscode-lit-ux-engineer

> vipr@0.8.0 analyze /Users/jamesleebaker/Codespace/vipr
> node clients/cli/dist/index.js analyzers/react/src/fixtures/InsecureComponent.tsx -f markdown

```bash

╔════════════════════════════════════════════════════════════════════╗
║ Code Analysis Report                               Overall: 45/100 ║
║ InsecureComponent.tsx                                              ║
║ [Component]  [42 issues]                                           ║
║ ────────────────────────────────────────────────────────────────── ║
║ Security       ░░░░░░░░░░░░░░░░░░░░ 0%                             ║
║ Accessibility  █████████████████░░░ 85%                            ║
║ Performance    █████████░░░░░░░░░░░ 43%                            ║
║ Reliability    █████████░░░░░░░░░░░ 47%                            ║
║ Migration      ████████████████████ 100%                           ║
║ Dataflow       ░░░░░░░░░░░░░░░░░░░░ 1%                             ║
║ Anti-patterns  ████████████░░░░░░░░ 60%                            ║
╚════════════════════════════════════════════════════════════════════╝

Maintainability
  ├─ Maintainability Index  [███░░░░░░░] 29.0  (D)
  ├─ Volume Impact          [████░░░░░░] 41.41  (5.2 * ln(V))
  ├─ Complexity Impact      [░░░░░░░░░░] 2.07  (0.23 * CC)
  └─ LOC Impact             [████████░░] 78.22  (16.2 * ln(LOC))

Cyclomatic Complexity
  ├─ Cyclomatic Complexity  [██░░░░░░░░] 9  (should be < 30)
  └─ Decision Points        [██░░░░░░░░] 8  (branches)

Halstead Complexity Measures
  ├─ Volume                [██████░░░░] 2875.11  (should be < 5000)
  ├─ Difficulty            [██░░░░░░░░] 8.88  (should be < 50)
  ├─ Effort                [█░░░░░░░░░] 25529.87  (should be < 200000)
  ├─ Est. Bugs             [█████░░░░░] 0.96  (should be < 2)
  └─ Program Length (LOC)  [████░░░░░░] 420  (operators + operands)

╔═══════════════════════════╗
║ REACT COMPLEXITY OVERVIEW ║
╚═══════════════════════════╝
Provides a high-level summary of React-specific complexity metrics including hook usage, effect dependencies, and component structure. This report helps you understand the overall complexity of your React components and identify areas that may need refactoring. Use the dimensional breakdown to understand how structural, temporal, and coupling complexity contribute to your overall score.

React Component Complexity
Score: ████████░░░░░░░░░░░░ 41

Constructs Complexity
  ├─ Structural  [█░░░░░░░░░] 6.2  (branches: 3, loops: 0)
  ├─ Hooks       [██░░░░░░░░] 18.0  (9 hooks total)
  ├─ Temporal    [█░░░░░░░░░] 12.0  (3 effects, 0 risky)
  ├─ Coupling    [░░░░░░░░░░] 0.6  (2 props, 0 contexts)
  └─ Identity    [░░░░░░░░░░] 1.8  (6 unstable refs)

Hook Breakdown
  ├─ useEffect (3x)  12
  └─ useState (6x)   6

Top 10 Insights (out of 69)
  ● Hardcoded Private key detected
  │  Location: Line 34, Column 9
  │  → Use environment variables or secure secret management

  ● Hardcoded Token detected
  │  Location: Line 36, Column 9
  │  → Use environment variables or secure secret management

  ● Hardcoded Token detected
  │  Location: Line 37, Column 9
  │  → Use environment variables or secure secret management

  ● Sensitive data stored in localStorage
  │  Location: Line 53, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● Sensitive data stored in localStorage
  │  Location: Line 54, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● Sensitive data stored in localStorage
  │  Location: Line 55, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● Sensitive data stored in localStorage
  │  Location: Line 56, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● Sensitive data stored in localStorage
  │  Location: Line 57, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● Sensitive data stored in localStorage
  │  Location: Line 58, Column 5
  │  → Use secure storage or avoid storing sensitive data client-side

  ● eval() usage detected
  │  Location: Line 72, Column 22
  │  → Use JSON.parse() or other safe parsing methods

╔═══════════════════╗
║ SECURITY ANALYSIS ║
╚═══════════════════╝
Identifies security vulnerabilities and risks in your React components, including XSS vectors, unsafe prop usage, and OWASP Top 10 violations. Use this report to prioritize security fixes and ensure your components handle user input safely. Critical and high-severity findings should be addressed immediately to prevent potential exploits.

Summary
  ├─ Security Score    [░░░░░░░░░░] 0.0
  ├─ OWASP Compliance  [░░░░░░░░░░] 0.0
  └─ Vulnerabilities   42

Vulnerabilities by Severity
  ├─ Critical  5
  ├─ High      20
  └─ Medium    17

Vulnerabilities by Type
  ├─ Xss             6
  ├─ Injection       5
  ├─ Sensitive Data  25
  └─ Cryptography    6

Detected Vulnerabilities
  ● eval() Usage
  │  Location: Line 72, Column 22
  │  Use of eval() can execute arbitrary code
  │  Impact: Arbitrary code execution can lead to XSS or remote code execution
  │  CWE-94 | A03:2021
  │  → Use JSON.parse() or other safe parsing methods instead

  ● Function() Constructor Usage
  │  Location: Line 82, Column 12
  │  Use of Function() constructor can execute arbitrary code
  │  Impact: Arbitrary code execution can lead to XSS or remote code execution
  │  CWE-94 | A03:2021
  │  → Use safer alternatives or validate input thoroughly

  ● Dynamic URL Injection
  │  Location: Line 131, Column 10
  │  Dynamic href attribute can contain malicious URLs
  │  Impact: javascript: protocol can execute arbitrary code
  │  CWE-79 | A03:2021
  │  → Validate and sanitize URLs before use, whitelist allowed protocols (http/https)

  ● Dangerous URL Protocol
  │  Location: Line 146, Column 10
  │  URL with dangerous protocol in href attribute
  │  Impact: javascript:, data:, or vbscript: protocol can execute arbitrary code
  │  CWE-79 | A03:2021
  │  → Remove dangerous protocol or use safe alternatives

  ● Dynamic URL Injection
  │  Location: Line 147, Column 10
  │  Dynamic href attribute can contain malicious URLs
  │  Impact: javascript: protocol can execute arbitrary code
  │  CWE-79 | A03:2021
  │  → Validate and sanitize URLs before use, whitelist allowed protocols (http/https)

  ● Sensitive Data Exposure
  │  Location: Line 34, Column 9
  │  Hardcoded Private key detected
  │  Impact: Hardcoded secrets can be exposed in source code or bundles
  │  CWE-798 | A02:2021
  │  → Use environment variables or secure secret management

  ● Sensitive Data Exposure
  │  Location: Line 36, Column 9
  │  Hardcoded Token detected
  │  Impact: Hardcoded secrets can be exposed in source code or bundles
  │  CWE-798 | A02:2021
  │  → Use environment variables or secure secret management

  ● Sensitive Data Exposure
  │  Location: Line 37, Column 9
  │  Hardcoded Token detected
  │  Impact: Hardcoded secrets can be exposed in source code or bundles
  │  CWE-798 | A02:2021
  │  → Use environment variables or secure secret management

  ● Sensitive Data Storage
  │  Location: Line 53, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 54, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 55, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 56, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 57, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 58, Column 5
  │  Sensitive data stored in localStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Insecure Random Number Generation
  │  Location: Line 87, Column 22
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Insecure Random Number Generation
  │  Location: Line 88, Column 19
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Insecure Random Number Generation
  │  Location: Line 89, Column 19
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Insecure Random Number Generation
  │  Location: Line 90, Column 18
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Insecure Random Number Generation
  │  Location: Line 92, Column 7
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Insecure Random Number Generation
  │  Location: Line 94, Column 7
  │  Math.random() used in security-sensitive context
  │  Impact: Math.random() is predictable and not cryptographically secure
  │  CWE-330 | A02:2021
  │  → Use crypto.getRandomValues() or a cryptographically secure random generator

  ● Dynamic URL Injection
  │  Location: Line 134, Column 12
  │  Dynamic src attribute can contain malicious URLs
  │  Impact: Malicious URLs can redirect users or load harmful content
  │  CWE-79 | A03:2021
  │  → Validate and sanitize URLs before use, whitelist allowed protocols (http/https)

  ● Dynamic URL Injection
  │  Location: Line 137, Column 13
  │  Dynamic action attribute can contain malicious URLs
  │  Impact: Malicious URLs can redirect users or load harmful content
  │  CWE-79 | A03:2021
  │  → Validate and sanitize URLs before use, whitelist allowed protocols (http/https)

  ● Dangerous HTML Usage
  │  Location: Line 143, Column 12
  │  Use of dangerouslySetInnerHTML can lead to XSS attacks
  │  Impact: User-controlled or unsanitized HTML can execute malicious scripts
  │  CWE-79 | A03:2021
  │  → Sanitize HTML input or use safer alternatives like React components

  ● Sensitive Data Storage
  │  Location: Line 169, Column 11
  │  Sensitive data stored in sessionStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Storage
  │  Location: Line 170, Column 11
  │  Sensitive data stored in sessionStorage
  │  Impact: Sensitive data in browser storage is accessible to scripts and can be stolen
  │  CWE-922 | A02:2021
  │  → Use secure storage mechanisms or avoid storing sensitive data client-side

  ● Sensitive Data Logging
  │  Location: Line 41, Column 5
  │  Sensitive data logged to console: API key
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 42, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 42, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 43, Column 5
  │  Sensitive data logged to console: Password
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 43, Column 5
  │  Sensitive data logged to console: Password
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 44, Column 5
  │  Sensitive data logged to console: Secret
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 44, Column 5
  │  Sensitive data logged to console: Secret
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 45, Column 5
  │  Sensitive data logged to console: Private key
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 46, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 46, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 47, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 47, Column 5
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Unsafe Regex Pattern
  │  Location: Line 103, Column 30
  │  Regex pattern may cause catastrophic backtracking (ReDoS)
  │  Impact: Nested quantifiers can cause exponential time complexity, freezing the browser
  │  CWE-1333 | A03:2021
  │  → Simplify the regex pattern, avoid nested quantifiers like (a+)+

  ● Unsafe Regex Pattern
  │  Location: Line 104, Column 31
  │  Regex pattern may cause catastrophic backtracking (ReDoS)
  │  Impact: Nested quantifiers can cause exponential time complexity, freezing the browser
  │  CWE-1333 | A03:2021
  │  → Simplify the regex pattern, avoid nested quantifiers like (a+)+

  ● Unsafe Regex Pattern
  │  Location: Line 105, Column 24
  │  Regex pattern may cause catastrophic backtracking (ReDoS)
  │  Impact: Nested quantifiers can cause exponential time complexity, freezing the browser
  │  CWE-1333 | A03:2021
  │  → Simplify the regex pattern, avoid nested quantifiers like (a+)+

  ● Sensitive Data Logging
  │  Location: Line 158, Column 11
  │  Sensitive data logged to console: Session ID
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

  ● Sensitive Data Logging
  │  Location: Line 160, Column 11
  │  Sensitive data logged to console: Token
  │  Impact: Sensitive data in console logs can be exposed in production
  │  CWE-532 | A09:2021
  │  → Remove sensitive data from logs or use secure logging

╔════════════════════════╗
║ ACCESSIBILITY ANALYSIS ║
╚════════════════════════╝
Evaluates your React components against WCAG 2.2 guidelines to ensure they are usable by people with disabilities. This report highlights missing ARIA attributes, keyboard navigation issues, and screen reader compatibility problems. Addressing these findings improves your application for all users and helps meet legal compliance requirements in many jurisdictions.

Summary
  ├─ Keyboard Navigation  [██████████] 100.0
  ├─ Screen Reader        [█████████░] 85.0
  ├─ WCAG Level           A
  ├─ Violations           1
  └─ Warnings             6

Issues by Severity
  └─ Serious  1

WCAG Violations
  ● Form Label
  │  Location: Line 138, Column 0
  │  Form input missing associated label
  │  Impact: Users cannot understand the purpose of form fields
  │  WCAG 1.3.1
  │  → Add a <label> element with htmlFor attribute, or aria-label

Warnings
  ● Math.random() for IDs
  │  Location: Line 87, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

  ● Math.random() for IDs
  │  Location: Line 88, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

  ● Math.random() for IDs
  │  Location: Line 89, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

  ● Math.random() for IDs
  │  Location: Line 90, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

  ● Math.random() for IDs
  │  Location: Line 92, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

  ● Math.random() for IDs
  │  Location: Line 94, Column 0
  │  Math.random() for IDs causes SSR mismatch
  │  → Use React.useId() for server-safe IDs

Best Practices
  ● Landmark Regions
  │  Use semantic HTML5 landmarks (main, nav, header, footer) to define page regions

  ● Skip Navigation Link
  │  Provide a skip link to allow keyboard users to bypass navigation

  ● Focus Management
  │  Implement programmatic focus management for dynamic content

  ● Live Regions
  │  Use aria-live regions to announce dynamic content changes

  ● Reduced Motion Support
  │  Respect user preferences for reduced motion

╔══════════════════════╗
║ PERFORMANCE ANALYSIS ║
╚══════════════════════╝
Analyzes render performance, memoization opportunities, and bundle size impact to identify bottlenecks in your React components. This report helps you optimize re-renders, reduce unnecessary computations, and improve time-to-interactive metrics. Focus on high-impact optimizations first, such as missing memoization on expensive computations or unnecessary re-renders of child components.

Summary
  ├─ Performance Score           [████░░░░░░] 42.5
  └─ Optimization Opportunities  15

Render Performance
  ├─ Render Score                    [░░░░░░░░░░] 0.0
  ├─ Unnecessary Re-render Risk      [██████████] 100.0
  ├─ Memoization Effectiveness       [███░░░░░░░] 25.0%
  └─ Expensive Operations in Render  2

Memoization Analysis
  ├─ useCallback  0
  ├─ useMemo      0
  └─ React.memo   0

  ● Missing useCallback
  │  Location: Line 150, Column 15
  │  Inline callback in "onClick" prop should be memoized
  │  → Add useCallback to optimize performance

  ● Missing useCallback
  │  Location: Line 151, Column 15
  │  Inline callback in "onClick" prop should be memoized
  │  → Add useCallback to optimize performance

  ● Missing useCallback
  │  Location: Line 153, Column 15
  │  Inline callback in "onClick" prop should be memoized
  │  → Add useCallback to optimize performance

  ● Missing useCallback
  │  Location: Line 157, Column 9
  │  Inline callback in "onClick" prop should be memoized
  │  → Add useCallback to optimize performance

  ● Missing useCallback
  │  Location: Line 168, Column 9
  │  Inline callback in "onClick" prop should be memoized
  │  → Add useCallback to optimize performance

Bundle Impact
  ├─ Estimated Size Impact  0 B
  ├─ Import Count           1
  └─ Tree-shaking Score     [██████████] 100.0%

Optimization Opportunities
  ● add-useMemo
  │  Location: Line 58, Column 41
  │  Expensive computation in render without memoization
  │  → Wrap in useMemo to prevent recalculation on every render
  │  Example: const result = useMemo(() => expensiveOperation(data), [data])

  ● add-useMemo
  │  Location: Line 143, Column 12
  │  Inline object in "dangerouslySetInnerHTML" prop
  │  → Extract to useMemo or constant outside component

  ● add-useCallback
  │  Location: Line 150, Column 15
  │  Inline callback in "onClick" prop
  │  → Extract to useCallback: const handleClick = useCallback(() => {...}, [deps])

  ● add-useCallback
  │  Location: Line 151, Column 15
  │  Inline callback in "onClick" prop
  │  → Extract to useCallback: const handleClick = useCallback(() => {...}, [deps])

  ● add-useCallback
  │  Location: Line 153, Column 15
  │  Inline callback in "onClick" prop
  │  → Extract to useCallback: const handleClick = useCallback(() => {...}, [deps])

  ● add-useCallback
  │  Location: Line 157, Column 9
  │  Inline callback in "onClick" prop
  │  → Extract to useCallback: const handleClick = useCallback(() => {...}, [deps])

  ● add-useCallback
  │  Location: Line 168, Column 9
  │  Inline callback in "onClick" prop
  │  → Extract to useCallback: const handleClick = useCallback(() => {...}, [deps])

  ● add-useMemo
  │  Location: Line 171, Column 50
  │  Expensive computation in render without memoization
  │  → Wrap in useMemo to prevent recalculation on every render
  │  Example: const result = useMemo(() => expensiveOperation(data), [data])

  ● add-memo
  │  Location: Line 23, Column 1
  │  Component "InsecureComponent" receives props but is not wrapped in React.memo
  │  → Consider wrapping with React.memo if parent re-renders frequently with unchanged props
  │  Example: const InsecureComponent = memo(function InsecureComponent(props) { ... })

╔══════════════════════╗
║ RELIABILITY ANALYSIS ║
╚══════════════════════╝
Assesses error handling, null safety, and edge case coverage to identify potential runtime failures in your React components. This report helps you build more resilient components by highlighting missing error boundaries, unsafe null/undefined access, and unhandled edge cases. Prioritize fixing critical failure modes that could cause application crashes or data loss.

Summary
  ├─ Reliability Score       [█████░░░░░] 47.0
  └─ Failure Modes Detected  1

Risk Metrics
  ├─ Crash Risk               [█░░░░░░░░░] 10.0
  ├─ Memory Leak Risk         [██████████] 100.0
  └─ Error Boundary Coverage  [░░░░░░░░░░] 0.0

Error Handling
  ├─ Async Error Handling      [██████████] 100.0
  ├─ Try-Catch Blocks          1
  ├─ Error Boundaries          0
  └─ Promise Handling Quality  Good

Null Safety
  ├─ Null Safety Score              [░░░░░░░░░░] 0.0
  ├─ Optional Chaining (?.) Usage   0
  ├─ Nullish Coalescing (??) Usage  0
  └─ Explicit Null Checks           0

Detected Failure Modes
  ● crash
  │  Location: Line 64, Column 7
  │  Potential null/undefined access
  │  → Use optional chaining: obj?.prop?.subprop

╔═════════════════════╗
║ MIGRATION READINESS ║
╚═════════════════════╝
Evaluates your codebase readiness for upgrading to React 19.0.0, identifying deprecated APIs, breaking changes, and migration blockers. This report provides a clear roadmap for your upgrade path, including estimated effort and automated codemod opportunities. Use the blockers section to prioritize critical issues that must be resolved before migration, and leverage suggested codemods to automate repetitive changes.

Summary
  ├─ Readiness Score  [██████████] 100.0
  ├─ Target Version   19.0.0
  ├─ Blockers         0
  └─ Warnings         0

Estimated Effort
  └─ Complexity  Trivial

Component Analysis
  ├─ Class Components       0
  └─ Functional Components  1

╔═══════════════════╗
║ DATAFLOW ANALYSIS ║
╚═══════════════════╝
Maps data flow patterns through your React component tree, identifying prop drilling, state management complexity, and derived value calculations. This report helps you refactor components to reduce coupling and improve maintainability by highlighting opportunities to extract state management or use context providers. Pay attention to deep prop chains and complex derived values that could benefit from memoization or state management libraries.

Summary
  ├─ Dataflow Score           [░░░░░░░░░░] 1.3
  ├─ State Update Paths       2
  ├─ Max Prop Drilling Depth  0
  └─ Max Transform Chain      0

State Management
  ├─ State Update Complexity      1
  ├─ Shared Mutable State (refs)  0
  └─ State Variables              2

  ● sessionId
  │  State type: state, Updated 1 times

  ● htmlContent
  │  State type: state, Updated 1 times

Prop Drilling
  └─ No prop drilling chains detected  0.0

╔═══════════════════════╗
║ ANTI-PATTERN ANALYSIS ║
╚═══════════════════════╝
Detects common React anti-patterns and code anti-patterns that reduce maintainability and increase technical debt. This report identifies issues like missing keys in lists, improper hook usage, and component design violations that can lead to bugs or performance problems. Use these findings to refactor components following React best practices and improve overall code quality.

Summary
  ├─ Quality Score        [██████░░░░] 60.0
  ├─ Impact Score         [████░░░░░░] 40.0
  ├─ Total Anti-Patterns  7
  └─ Auto-Fixable Issues  0

Issues by Category
  ├─ Performance       6
  └─ State Management  1

Issues by Severity
  ├─ Medium  1
  └─ Low     6

Most Common Issues
  ├─ Inline Function Prop  5
  ├─ Excessive State       1
  └─ Inline Object Prop    1

Detected Anti-Patterns
  ● Excessive State Variables
  │  Location: Line 24, Column 20
  │  Component has 6 useState calls
  │  Impact: Too many state variables can make components hard to understand and maintain. Consider using useReducer or combining related state.
  │  Refs: react.dev, react.dev
  │  → Consider using useReducer for complex state or combining related state into objects

  ● Inline Object Prop
  │  Location: Line 143, Column 12
  │  Inline object in "dangerouslySetInnerHTML" prop
  │  Impact: Inline objects create new object instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract object to component scope or use useMemo

  ● Inline Function Prop
  │  Location: Line 150, Column 15
  │  Inline function in "onClick" prop
  │  Impact: Inline functions create new function instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract function to component scope or wrap in useCallback

  ● Inline Function Prop
  │  Location: Line 151, Column 15
  │  Inline function in "onClick" prop
  │  Impact: Inline functions create new function instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract function to component scope or wrap in useCallback

  ● Inline Function Prop
  │  Location: Line 153, Column 15
  │  Inline function in "onClick" prop
  │  Impact: Inline functions create new function instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract function to component scope or wrap in useCallback

  ● Inline Function Prop
  │  Location: Line 157, Column 9
  │  Inline function in "onClick" prop
  │  Impact: Inline functions create new function instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract function to component scope or wrap in useCallback

  ● Inline Function Prop
  │  Location: Line 168, Column 9
  │  Inline function in "onClick" prop
  │  Impact: Inline functions create new function instances on every render, causing child components to re-render unnecessarily
  │  Refs: react.dev, react.dev
  │  → Extract function to component scope or wrap in useCallback

```
