# Scoring Methodology

## Overview

Vipr uses a quality scoring system to help you understand code maintainability and quality. All scores range from 0-100, where higher scores indicate better quality.

## Quality Levels

| Score Range | Quality Level | Meaning                                                    |
| ----------- | ------------- | ---------------------------------------------------------- |
| 80-100      | Excellent     | Highly maintainable code with minimal issues               |
| 60-79       | Good          | Generally maintainable with some room for improvement      |
| 40-59       | Fair          | Moderate maintainability concerns that should be addressed |
| 0-39        | Poor          | Significant maintainability issues requiring attention     |

## How Scores are Calculated

### Individual Metrics

Each analyzer calculates specific metrics for your code, then converts them into quality scores.

#### Core Metrics

The Core analyzer measures fundamental code quality metrics:

**Cyclomatic Complexity**

- Measures decision points in code (if, while, for, switch, etc.)
- Lower complexity = higher quality score
- Complexity 0-5 → 90-100 (Excellent)
- Complexity 6-10 → 75-89 (Good)
- Complexity 11-20 → 55-74 (Fair)
- Complexity 21-30 → 35-54 (Poor)
- Complexity 31+ → 0-34 (Critical)

**Maintainability Index**

- Industry-standard metric combining complexity, volume, and code length
- Based on Oman & Hagemeister (1992) formula
- Already produces a quality score (0-100, higher = better)
- Formula: 171 - 5.2 × ln(Volume) - 0.23 × Complexity - 16.2 × ln(LOC)
- Scaled to 0-100 range

**Halstead Metrics**

- Measures code volume and difficulty based on operators and operands
- Volume: How much information is in the code
- Difficulty: How hard the code is to understand
- Both metrics converted to quality scores (lower volume/difficulty = higher quality)
- Volume < 500 → 90-100 (Excellent)
- Volume 500-1000 → 75-89 (Good)
- Volume 1000-2000 → 55-74 (Fair)
- Volume > 2000 → < 55 (Poor)

#### React Metrics

The React analyzer measures component-specific quality across multiple dimensions:

**Structural Complexity**

- Component structure and JSX nesting depth
- Branches, conditionals, loops, and logical operators
- Lower structural complexity = higher quality score

**Hooks Complexity**

- Number and patterns of React hooks usage
- Custom hooks identification
- Fewer hooks (when appropriate) = higher quality score

**Temporal Complexity**

- State changes and effect dependencies
- useEffect patterns and cleanup functions
- Better effect organization = higher quality score

**Coupling Complexity**

- Component interdependencies
- Props count, context consumers, callbacks
- Lower coupling = higher quality score

**Identity Complexity**

- Memoization patterns (useCallback, useMemo)
- Reference stability issues
- Appropriate memoization = higher quality score

**Dataflow Complexity**

- Props and state propagation
- Prop drilling depth and chains
- Derived state patterns
- Simpler data flow = higher quality score

**Anti-patterns**

- React anti-pattern violations
- Direct DOM manipulation, missing keys, etc.
- Fewer anti-patterns = higher quality score

**Security**

- Security vulnerabilities (XSS, injection risks)
- Compliance with security best practices
- Better security = higher quality score

**Performance**

- Render performance issues
- Memoization effectiveness
- Bundle impact and code splitting opportunities
- Better performance = higher quality score

**Reliability**

- Error handling and crash risk
- Null safety and async error handling
- Better reliability = higher quality score

**Technical Debt**

- Code health and maintainability metrics
- React-specific maintainability index
- Lower technical debt = higher quality score

**Type Complexity**

- TypeScript type system complexity
- Generic depth, conditional types, unions
- Simpler types = higher quality score

**Accessibility**

- WCAG compliance level
- Keyboard navigation and screen reader support
- Better accessibility = higher quality score

**Migration Readiness**

- React version migration readiness
- Deprecated patterns and blockers
- Higher readiness = higher quality score

### Plugin Scores

Each analyzer plugin aggregates its individual metrics into a single score.

#### Core Plugin Score

**Method: Simple average of analysis scores**

The Core plugin averages the quality scores from three analyses:

1. Cyclomatic Complexity score
2. Maintainability Index score
3. Halstead Metrics score

Example:

```
Cyclomatic: 85 (complexity of 3)
Maintainability: 72 (moderate MI)
Halstead: 80 (low volume/difficulty)

Core score = (85 + 72 + 80) / 3 = 79 (Good)
```

#### React Plugin Score

**Method: Weighted average of dimension scores**

The React plugin calculates a weighted average based on dimension importance:

- Structural complexity (weight: 1.2) - Most important
- Hooks complexity (weight: 1.0) - Very important
- Coupling complexity (weight: 1.0) - Very important
- Temporal complexity (weight: 0.8) - Important
- Dataflow complexity (weight: 0.7) - Important
- Identity complexity (weight: 0.6) - Moderately important
- Other dimensions (weights vary by importance)

Example:

```
Structural: 75 (weight 1.2) → 90 weighted points
Hooks: 80 (weight 1.0) → 80 weighted points
Temporal: 70 (weight 0.8) → 56 weighted points
Coupling: 85 (weight 1.0) → 85 weighted points
Dataflow: 65 (weight 0.7) → 45.5 weighted points
Identity: 78 (weight 0.6) → 46.8 weighted points

Total weights = 5.3
React score = 403.3 / 5.3 = 76 (Good)
```

#### Next.js Plugin Score

The Next.js plugin extends React analysis with framework-specific patterns:

- Server Components and Client Components
- Data fetching patterns (server actions, API routes)
- Routing patterns (App Router, Pages Router)
- Next.js-specific performance optimizations

Calculation method is similar to React, with additional Next.js-specific dimensions.

### Overall File Score

The overall file score is a weighted average of all active plugin scores.

**Method: Weighted average of plugin scores**

Each plugin can specify a weight (default: 1). The overall score combines all plugin scores:

```
Overall = (Core × weight₁ + React × weight₂ + NextJS × weight₃) / Total Weights
```

Example for a React component:

```
Core score: 79 (weight: 1)
React score: 76 (weight: 1)

Overall = (79 × 1 + 76 × 1) / 2 = 77.5 → 78 (Good)
```

Example for a Next.js server component:

```
Core score: 82 (weight: 1)
React score: 75 (weight: 1)
Next.js score: 88 (weight: 1)

Overall = (82 × 1 + 75 × 1 + 88 × 1) / 3 = 81.7 → 82 (Excellent)
```

## Score Transparency

The Vipr dashboard and CLI output include a "Score Calculation Details" section that breaks down exactly how your overall score is calculated:

- Which plugins contributed to the score
- Each plugin's individual score
- The weights applied to each plugin
- The final calculation formula

This transparency helps you understand:

- What factors are affecting your code quality score
- Which areas need the most improvement
- How changes to specific metrics impact the overall score

## Understanding Score Changes

When refactoring code, you can track quality improvements by monitoring how specific changes affect scores:

### Improving Core Scores

- **Reduce cyclomatic complexity**: Fewer nested ifs/loops → increases Core score
  - Extract complex conditions into well-named functions
  - Replace switch statements with polymorphism or lookup tables
  - Simplify boolean logic

- **Improve maintainability index**: Better structure → increases Core score
  - Break down long functions into smaller, focused ones
  - Reduce Halstead volume by extracting repeated patterns
  - Add meaningful comments to improve the comment ratio

- **Optimize Halstead metrics**: Less verbose code → increases Core score
  - Extract utility functions to reduce operator count
  - Simplify algorithms to reduce difficulty
  - Refactor to reduce code volume

### Improving React Scores

- **Simplify component structure**: Less nesting → increases Structural score
  - Extract nested JSX into separate components
  - Reduce conditional rendering complexity
  - Flatten component hierarchies

- **Optimize hooks usage**: Better patterns → increases Hooks score
  - Combine related useState calls into useReducer
  - Extract custom hooks for reusable logic
  - Avoid unnecessary hooks

- **Better effect organization**: Cleaner dependencies → increases Temporal score
  - Split effects with different dependencies
  - Add cleanup functions where needed
  - Avoid effects that run on every render

- **Reduce coupling**: Fewer dependencies → increases Coupling score
  - Use composition over prop drilling
  - Implement context for widely-used data
  - Minimize callback props

- **Appropriate memoization**: Better performance → increases Identity score
  - Add useCallback for callback props
  - Use useMemo for expensive calculations
  - Avoid over-memoization of cheap operations

- **Simplify data flow**: Clear patterns → increases Dataflow score
  - Lift state to common ancestors
  - Avoid prop drilling beyond 2-3 levels
  - Use derived state instead of duplicating

### Improving Other Scores

- **Fix anti-patterns**: Follow best practices → increases Anti-patterns score
- **Address security issues**: Remove vulnerabilities → increases Security score
- **Optimize performance**: Reduce render cost → increases Performance score
- **Add error handling**: Better resilience → increases Reliability score
- **Reduce technical debt**: Cleaner code → increases Technical Debt score
- **Simplify types**: Clearer types → increases Type Complexity score
- **Improve accessibility**: WCAG compliance → increases Accessibility score
- **Update deprecated code**: Modern patterns → increases Migration score

## Best Practices

### Prioritize Improvements

1. **Focus on Fair/Poor scores first**: Address files scoring below 60
   - These files have the highest risk of maintenance issues
   - Improvements here have the biggest impact

2. **Set improvement targets**: Aim to move files into the next quality level
   - Poor → Fair: Focus on critical issues (cyclomatic complexity, anti-patterns)
   - Fair → Good: Address moderate concerns (structure, coupling)
   - Good → Excellent: Polish and optimize (performance, accessibility)

3. **Track trends**: Use the history feature to see score changes over time
   - Monitor whether code quality is improving or degrading
   - Identify patterns in what changes improve scores
   - Celebrate quality improvements

4. **Balance effort**: Not all code needs to be Excellent
   - Excellent (80-100): Critical business logic, shared libraries
   - Good (60-79): Most application code - this is often sufficient
   - Fair (40-59): Acceptable for prototype or experimental code
   - Poor (0-39): Technical debt that needs addressing

### Interpreting Scores in Context

Different file types have different expectations:

- **Complex business logic**: Aim for Good (60-79) or higher
- **UI components**: Aim for Good (60-79) or higher
- **Utility functions**: Aim for Excellent (80-100)
- **Configuration files**: Core metrics less relevant
- **Test files**: Different quality standards apply

### Using Scores for Code Review

Scores can guide code review priorities:

- **New files scoring Poor**: Likely needs refactoring before merge
- **Existing files degrading**: Regression in code quality
- **Consistent Excellent scores**: Well-maintained codebase
- **Improving trends**: Effective refactoring efforts

### Avoiding Score Manipulation

Scores should reflect actual quality, not be artificially inflated:

- Don't split files just to reduce LOC if it hurts cohesion
- Don't add unnecessary abstraction to reduce cyclomatic complexity
- Don't over-memoize to boost performance scores
- Don't suppress warnings without fixing underlying issues

Focus on making code genuinely more maintainable, and scores will naturally improve.

## Technical Details

### Score Calculation Algorithm

The calculation happens in three stages:

1. **Analysis Level**: Each analysis calculates a quality score for its specific metric
   - Raw metric value → Quality score conversion
   - Example: Cyclomatic complexity 15 → Quality score 62

2. **Plugin Level**: Each plugin aggregates its analysis scores
   - Weighted or simple average depending on plugin
   - Core: Simple average of cyclomatic, maintainability, halstead
   - React: Weighted average of structural, hooks, temporal, etc.

3. **File Level**: Engine aggregates plugin scores
   - Weighted average of all plugin scores
   - Default weight: 1 for each plugin
   - Formula: Σ(plugin_score × weight) / Σ(weights)

### Score Consistency

All scores in Vipr follow these principles:

- **0-100 scale**: All scores normalized to 0-100 range
- **Higher is better**: 100 is perfect quality, 0 is critical issues
- **Rounded to integers**: No decimal places in displayed scores
- **Bounded**: Scores clamped to 0-100 range (no negative, no > 100)

### Calculation Performance

Score calculations are optimized for performance:

- **Analysis caching**: Results cached based on file content hash
- **Parallel execution**: Analyses run concurrently when possible
- **Incremental updates**: Only recalculate changed files
- **Cost-based scheduling**: Expensive analyses scheduled efficiently

## References

### Academic Sources

- Oman, P., & Hagemeister, J. (1992). "Metrics for assessing a software system's maintainability"
- Coleman, D., et al. (1994). "Using metrics to evaluate software system maintainability"
- McCabe, T. (1976). "A Complexity Measure" - Original cyclomatic complexity paper
- Halstead, M. (1977). "Elements of Software Science" - Halstead metrics

### Industry Standards

- Microsoft Visual Studio Code Metrics: [Code Metrics Values](https://docs.microsoft.com/en-us/visualstudio/code-quality/code-metrics-values)
- ESLint Complexity Rules: [complexity](https://eslint.org/docs/latest/rules/complexity)
- React Documentation: [Hooks Rules](https://react.dev/reference/rules)
- WCAG Guidelines: [Web Content Accessibility Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

## Frequently Asked Questions

### Why is my score different from other tools?

Vipr uses a comprehensive quality scoring system that considers multiple dimensions:

- Other tools may measure only one aspect (e.g., just cyclomatic complexity)
- Vipr combines multiple metrics into a holistic quality score
- React-specific dimensions (hooks, temporal, coupling) are unique to Vipr

### Can I customize score weights?

Currently, weights are fixed based on empirical importance. Future versions may support custom weights through configuration.

### Why did my score go down after adding comments?

Comments affect the Maintainability Index calculation:

- More comments can improve maintainability slightly
- However, if you added comments without reducing complexity, the overall impact may be minimal
- Focus on reducing actual complexity, not just adding comments

### Should I aim for 100 on every file?

No. Here is a practical guide:

- 60-79 (Good) is sufficient for most code
- 80-100 (Excellent) is ideal for critical paths and shared libraries
- 40-59 (Fair) is acceptable for rapid prototypes (but needs eventual improvement)
- 0-39 (Poor) always needs attention

Perfect scores are not always achievable or necessary. Focus on consistent quality across the codebase.

### How often are scores recalculated?

- **CLI**: On every analysis run
- **VSCode Extension**: When you save a file or manually trigger analysis
- **CI/CD**: On each build/commit as configured

Scores are cached based on file content, so unchanged files use cached scores for performance.

### What if my complex algorithm gets a low score?

Some algorithms are inherently complex. Consider:

- Can it be broken into smaller functions?
- Can you extract helper functions to reduce cyclomatic complexity?
- Is the complexity necessary, or are there simpler approaches?
- Add comprehensive tests to mitigate the maintainability risk

Sometimes a Fair score (40-59) is acceptable for genuinely complex logic, as long as it's well-tested and documented.
