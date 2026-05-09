# Core Analyzer Test Coverage Review

**Date:** 2026-01-15
**Analyzer:** `@analyzers/core`
**Location:** `/Users/jamesleebaker/Codespace/vipr/analyzers/core`

## Executive Summary

This review examines the test coverage for the Core Analyzer, which implements fundamental software metrics including Cyclomatic Complexity (McCabe) and Halstead Complexity Measures. The analyzer has **304 passing tests** across 8 test files, demonstrating solid foundational coverage. However, critical gaps exist in algorithm validation, edge case handling, and maintainability index calculations.

### Test Suite Overview

| Component             | Test File                                | Tests | Coverage Assessment                        |
| --------------------- | ---------------------------------------- | ----- | ------------------------------------------ |
| Cyclomatic Complexity | `cyclomatic-complexity-analysis.test.ts` | 34    | **Good** - Comprehensive operator coverage |
| Halstead Metrics      | `halstead-metrics-analysis.test.ts`      | 44    | **Good** - Formula validation present      |
| AST Helpers           | `ast-helpers.test.ts`                    | 52    | **Good** - Utility function coverage       |
| Traditional Metrics   | `traditional-metrics.test.ts`            | 40    | **Good** - Combined metric testing         |
| Scoring               | `scoring.test.ts`                        | 70    | **Excellent** - Extensive validation       |
| Engine                | `analysis-engine.integration.test.ts`    | 7     | **Fair** - Basic integration only          |
| Base Analyzer         | `base-analyzer.test.ts`                  | 25    | **Good** - Interface coverage              |
| Plugin                | `plugin.test.ts`                         | 32    | **Good** - Plugin architecture             |

---

## 1. Algorithm Accuracy Analysis

### 1.1 Cyclomatic Complexity (McCabe)

**Location:** `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts`

#### Implementation Review

The implementation follows McCabe's definition:

```typescript
// Base complexity = 1
// Decision points counted:
- IfStatement
- ConditionalExpression (ternary)
- ForStatement, ForInStatement, ForOfStatement
- WhileStatement, DoStatement
- CaseClause (but not DefaultClause)
- CatchClause
- Logical operators: &&, ||, ??
- Optional chaining: ?.
```

#### Test Coverage Strengths

1. **Decision Point Coverage** - Tests validate all standard control structures:
   - If statements (line 282-294)
   - Ternary expressions (line 297-309)
   - Loops: for, while, do-while, for-in, for-of (line 253-336)
   - Switch cases (line 230-251)
   - Catch clauses (line 337-353)

2. **Modern JavaScript Operators** - Comprehensive coverage:
   - Nullish coalescing `??` (line 74-104)
   - Optional chaining `?.` (line 106-151)
   - Logical operators `&&`, `||` (line 153-210)

3. **Nested Structures** - Tests handle nested functions and complex scenarios (line 370-410)

#### Critical Gaps

**GAP-CC-1: Missing McCabe Formula Validation**

The tests don't explicitly validate the fundamental McCabe formula:

```
V(G) = E - N + 2P
```

Where:

- E = number of edges in control flow graph
- N = number of nodes
- P = number of connected components

**Recommendation:** Add tests that construct known control flow graphs and verify the complexity matches theoretical calculations.

```typescript
describe('McCabe formula validation', () => {
  it('should match theoretical complexity for known control flow', () => {
    // Example: Diamond-shaped control flow
    // Entry -> Decision -> Path1/Path2 -> Merge -> Exit
    // E=5, N=5, P=1 => V(G) = 5-5+2(1) = 2
    const code = `
      function diamond(x: number): number {
        if (x > 0) {
          return x;  // Path 1
        } else {
          return -x; // Path 2
        }
      }
    `;
    expect(complexity).toBe(2); // 1 decision point + 1 base
  });
});
```

**GAP-CC-2: No Validation Against Industry Benchmarks**

Tests don't validate against published complexity values for standard algorithms.

**Recommendation:** Add benchmark tests:

```typescript
describe('complexity benchmarks', () => {
  it('should match published complexity for bubble sort', () => {
    const bubbleSort = `
      function bubbleSort(arr: number[]): number[] {
        for (let i = 0; i < arr.length; i++) {
          for (let j = 0; j < arr.length - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
              const temp = arr[j];
              arr[j] = arr[j + 1];
              arr[j + 1] = temp;
            }
          }
        }
        return arr;
      }
    `;
    // Expected: 2 for loops + 1 if = 4 (base 1 + 3 decisions)
    expect(result.data.complexity).toBe(4);
  });

  it('should match published complexity for binary search', () => {
    const binarySearch =
      `
      function binarySearch(arr: number[], target: number): number {
        let left = 0;
        let right = arr.length - 1;

        while (left ` <=
      ` right) {
          const mid = Math.floor((left + right) / 2);
          if (arr[mid] === target) return mid;
          if (arr[mid] < target) {
            left = mid + 1;
          } else {
            right = mid - 1;
          }
        }
        return -1;
      }
    `;
    // Expected: 1 while + 3 if = 5 (base 1 + 4 decisions)
    expect(result.data.complexity).toBe(5);
  });
});
```

**GAP-CC-3: No Cross-Validation with Other Tools**

Tests don't compare results with established tools like ESLint's complexity rule or other static analysis tools.

**Recommendation:** Add comparative tests:

```typescript
describe('cross-tool validation', () => {
  it('should match ESLint complexity calculation', () => {
    // Test against code samples with known ESLint complexity values
    // Document: ESLint uses McCabe, so should match exactly
  });
});
```

**GAP-CC-4: Short-Circuit Evaluation Not Tested**

While `&&` and `||` are tested, the tests don't validate that short-circuit evaluation is properly counted.

```typescript
describe('short-circuit evaluation', () => {
  it('should count each logical operator in chain', () => {
    const code = `
      function check(a, b, c, d) {
        return a && b && c && d; // 4 short-circuit points
      }
    `;
    expect(result.data.complexity).toBe(4); // 3 && operators + base
  });
});
```

### 1.2 Halstead Complexity Measures

**Location:** `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/analyses/halstead-metrics-analysis.ts`

#### Implementation Review

The implementation correctly follows Halstead's 1977 definitions:

```typescript
// Operators (n1, N1): Unique/total operators
// Operands (n2, N2): Unique/total operands

vocabularySize = n1 + n2;
programLength = N1 + N2;
volume = programLength * log2(vocabularySize);
difficulty = (n1 / 2) * (N2 / n2);
effort = difficulty * volume;
time = effort / 18; // Stroud number
bugs = volume / 3000; // Empirical constant
```

#### Test Coverage Strengths

1. **Formula Validation** - Excellent coverage of Halstead formulas (line 460-569):
   - Program length calculation (line 461-474)
   - Vocabulary size calculation (line 476-489)
   - Volume calculation (line 491-504)
   - Difficulty calculation (line 506-526)
   - Effort calculation (line 528-540)
   - Time calculation (line 542-554)
   - Bugs calculation (line 556-568)

2. **Operator Counting** - Comprehensive operator tests (line 289-345):
   - Binary operators
   - Unary operators
   - Call expressions
   - Property access
   - Arrow functions

3. **Operand Counting** - Good coverage (line 347-396):
   - Identifiers
   - Numeric literals
   - String literals

4. **JSX Support** - Modern framework support (line 93-168):
   - JSX elements as distinct operators
   - Self-closing elements
   - Custom components

#### Critical Gaps

**GAP-HM-1: Halstead Constants Not Empirically Validated**

The implementation uses hardcoded constants:

```typescript
timeMultiplier: 18,      // Stroud number
bugsThreshold: 3000,     // Bugs per volume
```

**Issue:** These constants are from Halstead's 1977 work on FORTRAN/PL-I. No tests validate they're appropriate for modern JavaScript/TypeScript.

**Recommendation:** Add tests that validate against empirical data:

```typescript
describe('Halstead constant validation', () => {
  it('should document rationale for timeMultiplier = 18', () => {
    // Test with known programs and actual debugging times
    // Document: Stroud number may need adjustment for modern languages
  });

  it('should validate bugsThreshold against real-world data', () => {
    // Compare predicted bugs vs actual bugs in test corpus
    // Modern studies suggest 3000 may be too conservative
  });
});
```

**GAP-HM-2: No Validation Against Known Programs**

Tests don't include well-known programs with published Halstead metrics.

**Recommendation:**

```typescript
describe('published Halstead values', () => {
  it('should match metrics for Kernighan hello world', () => {
    const code = `
      function main() {
        console.log("Hello, World!");
      }
    `;
    // Compare against published values from academic literature
  });
});
```

**GAP-HM-3: Operator/Operand Classification Edge Cases**

Some edge cases aren't tested:

```typescript
describe('operator classification edge cases', () => {
  it('should classify destructuring correctly', () => {
    const code = `const { a, b } = obj;`;
    // Is {} an operator? Each , ? The = ?
  });

  it('should classify template literals correctly', () => {
    const code = '`Hello ${name}!`';
    // How to count template interpolation?
  });

  it('should handle computed property names', () => {
    const code = `const obj = { [key]: value };`;
    // [] as operator? How to count?
  });
});
```

**GAP-HM-4: String Hashing for Long Strings Not Validated**

The code hashes strings > 200 chars (line 168-176), but tests don't verify:

1. Hash collisions are rare
2. Hash function is deterministic
3. Performance is acceptable

```typescript
describe('string operand hashing', () => {
  it('should produce deterministic hashes', () => {
    const longString = 'a'.repeat(250);
    const code1 = `const s = "${longString}";`;
    const code2 = `const s = "${longString}";`;
    const result1 = analysis.execute(createSourceFile(code1));
    const result2 = analysis.execute(createSourceFile(code2));
    expect(result1.data.uniqOperands).toBe(result2.data.uniqOperands);
  });

  it('should distinguish similar long strings', () => {
    const string1 = 'a'.repeat(250);
    const string2 = 'b'.repeat(250);
    const code = `const s1 = "${string1}"; const s2 = "${string2}";`;
    const result = analysis.execute(createSourceFile(code));
    expect(result.data.uniqOperands).toBeGreaterThanOrEqual(3); // s1, s2, two hashes
  });
});
```

**GAP-HM-5: Property Access Counting Ambiguity**

Line 128-133 counts each property access as an operator occurrence but notes this is "correct per Halstead metrics." This needs validation.

```typescript
describe('property access counting', () => {
  it('should validate against Halstead original methodology', () => {
    const code = `obj.a.b.c`;
    // Halstead's original work: is each . a separate operator occurrence?
    // Or is the entire chain one operation?
    // Test against academic consensus
  });
});
```

### 1.3 Maintainability Index

**CRITICAL: No Implementation Found**

Maintainability Index is a widely-used metric defined as:

```
MI = 171 - 5.2 * ln(V) - 0.23 * CC - 16.2 * ln(LOC)
```

Where:

- V = Halstead Volume
- CC = Cyclomatic Complexity
- LOC = Lines of Code

**Scaled to 0-100:**

```
MI_scaled = max(0, (MI / 171) * 100)
```

**Industry Standards:**

- **85-100**: Excellent maintainability
- **65-85**: Good maintainability
- **40-65**: Moderate maintainability
- **10-40**: Difficult to maintain
- **0-10**: Extremely difficult to maintain

**Recommendation: PRIORITY 1 - Add Maintainability Index**

```typescript
// New file: src/analyses/maintainability-index-analysis.ts
export class MaintainabilityIndexAnalysis implements IAnalysis<unknown, MaintainabilityIndexData> {
  readonly id = 'core-maintainability';
  readonly name = 'Maintainability Index';

  execute(sourceFile: SourceFile): AnalysisResult<MaintainabilityIndexData> {
    const cyclomaticComplexity = calculateCyclomaticComplexity(sourceFile);
    const halstead = calculateHalsteadMetrics(sourceFile);
    const loc = getLOC(sourceFile);

    // Original formula
    const mi =
      171 - 5.2 * Math.log(halstead.volume) - 0.23 * cyclomaticComplexity - 16.2 * Math.log(loc);

    // Scale to 0-100
    const miScaled = Math.max(0, (mi / 171) * 100);

    return {
      analysisId: this.id,
      category: this.category,
      data: {
        maintainabilityIndex: Math.round(miScaled),
        rawIndex: Math.round(mi * 100) / 100,
        components: {
          volumeComponent: 5.2 * Math.log(halstead.volume),
          complexityComponent: 0.23 * cyclomaticComplexity,
          locComponent: 16.2 * Math.log(loc),
        },
      },
      insights: this.generateInsights(miScaled),
      score: 100 - miScaled, // Invert: lower MI = higher complexity score
      executionTimeMs: 0,
    };
  }
}
```

**Corresponding Tests:**

```typescript
describe('MaintainabilityIndexAnalysis', () => {
  describe('formula validation', () => {
    it('should calculate MI using standard formula', () => {
      // Test with known values
      const code = `function simple() { return 42; }`;
      const result = analysis.execute(createSourceFile(code));

      // Verify: MI = 171 - 5.2*ln(V) - 0.23*CC - 16.2*ln(LOC)
      // Calculate expected value manually
    });

    it('should scale MI to 0-100 range', () => {
      expect(result.data.maintainabilityIndex).toBeGreaterThanOrEqual(0);
      expect(result.data.maintainabilityIndex).toBeLessThanOrEqual(100);
    });
  });

  describe('maintainability levels', () => {
    it('should rate simple code as excellent (85-100)', () => {
      const code = `function add(a, b) { return a + b; }`;
      expect(result.data.maintainabilityIndex).toBeGreaterThanOrEqual(85);
    });

    it('should rate complex code as poor (10-40)', () => {
      const complexCode = /* deeply nested, high CC, high volume */;
      expect(result.data.maintainabilityIndex).toBeLessThan(40);
    });
  });
});
```

### 1.4 Cognitive Complexity

**RECOMMENDED: Consider Adding Cognitive Complexity**

Cognitive Complexity (Sonar 2017) is increasingly preferred over Cyclomatic Complexity as it better reflects human perception of complexity.

**Key Differences from Cyclomatic:**

1. Increments for **nesting depth** (each level adds to complexity)
2. Doesn't count simple sequences equally
3. Better correlates with perceived difficulty

**Example:**

```typescript
// Cyclomatic: 4 (3 if + base)
// Cognitive: 7 (3 if + 1 for outer nesting + 2 for deeper nesting + 3 for deepest)
function complex(x) {
  if (a) {
    // +1
    if (b) {
      // +2 (nested once)
      if (c) {
        // +3 (nested twice)
        return x;
      }
    }
  }
}
```

**Recommendation:**

```typescript
// New file: src/analyses/cognitive-complexity-analysis.ts
export class CognitiveComplexityAnalysis implements IAnalysis {
  // Implementation following SonarSource white paper
}
```

---

## 2. Test Quality Assessment

### 2.1 Positive Findings

#### Comprehensive Operator Coverage

The cyclomatic complexity tests cover all JavaScript/TypeScript decision operators:

```typescript
// Traditional
✓ if/else statements
✓ switch/case
✓ for/while/do-while loops
✓ try/catch
✓ ternary operators

// Modern JavaScript
✓ Optional chaining (?.)
✓ Nullish coalescing (??)
✓ Logical operators (&&, ||)
```

#### Formula Validation

Halstead tests include explicit formula validation:

```typescript
it('should verify volume = programLength * log2(vocabularySize)', () => {
  const expectedVolume = result.data.programLength * Math.log2(result.data.vocabularySize || 1);
  expect(result.data.volume).toBeCloseTo(expectedVolume, 2);
});
```

This is **excellent practice** and should be extended to other metrics.

#### Precision Testing

Tests verify numeric precision:

```typescript
it('should round all metrics to 2 decimal places', () => {
  expect(result.data.volume.toString().split('.')[1]?.length || 0).toBeLessThanOrEqual(2);
});
```

### 2.2 Critical Test Quality Issues

#### ISSUE-1: Magic Numbers Without Explanation

Many tests use hardcoded expected values without documenting why:

```typescript
it('should detect high complexity with nested if statements', () => {
  // 4 if statements = complexity 5
  expect(result.data.complexity).toBe(5); // WHY 5? Document: base 1 + 4 ifs
  expect(result.data.decisionPoints).toBe(4);
});
```

**Recommendation:** Add comments explaining all expected values:

```typescript
it('should detect high complexity with nested if statements', () => {
  const code = `/* ... 4 if statements ... */`;
  const result = analysis.execute(sourceFile);

  // Expected complexity calculation:
  // - Base complexity: 1
  // - If statements: 4
  // Total: 1 + 4 = 5
  expect(result.data.complexity).toBe(5);
  expect(result.data.decisionPoints).toBe(4);
});
```

#### ISSUE-2: Insufficient Boundary Testing

While scoring tests have good boundaries:

```typescript
it('should return A for scores 0-25', () => {
  expect(scoreToGrade(25)).toBe('A');
  expect(scoreToGrade(25.1)).toBe('B'); // Good: testing boundary
});
```

Complexity tests lack similar rigor:

```typescript
// Missing: Tests at complexity thresholds
describe('complexity thresholds', () => {
  it('should generate no insight at complexity 5', () => {
    expect(insights.length).toBe(0);
  });

  it('should generate info insight at complexity 6', () => {
    expect(insights[0].severity).toBe('info');
  });

  it('should generate warning insight at complexity 11', () => {
    expect(insights[0].severity).toBe('warning');
  });
});
```

#### ISSUE-3: Property-Based Testing Absent

No property-based tests to validate invariants:

```typescript
import { fc } from 'fast-check';

describe('Halstead properties', () => {
  it(
    'should maintain vocabulary = uniqOperators + uniqOperands',
    fc.property(fc.string(), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.vocabularySize).toBe(result.data.uniqOperators + result.data.uniqOperands);
    })
  );

  it(
    'should have difficulty >= 1',
    fc.property(fc.string(), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.difficulty).toBeGreaterThanOrEqual(1);
    })
  );
});
```

#### ISSUE-4: No Regression Tests

No tests document known bugs or regressions:

```typescript
describe('regression tests', () => {
  it('should fix: optional chaining counted twice (issue #123)', () => {
    const code = `obj?.prop`;
    expect(complexity).toBe(2); // Not 3
  });
});
```

---

## 3. Coverage Gaps by Component

### 3.1 AST Helpers (`ast-helpers.ts`)

**Current Coverage: Good (52 tests)**

#### Gaps:

**GAP-AST-1: Optional Chaining Detection**

Line 162-168 handles optional chaining for complexity:

```typescript
if (kind === SyntaxKind.PropertyAccessExpression) {
  const propAccess = descendant.asKind(SyntaxKind.PropertyAccessExpression);
  if (propAccess && propAccess.hasQuestionDotToken()) {
    complexity++;
  }
}
```

But there's no test for:

- Multiple chained optional accesses: `a?.b?.c?.d`
- Optional method calls: `obj?.method?.()`
- Optional element access: `arr?.[0]`

**Recommendation:**

```typescript
describe('optional chaining complexity', () => {
  it('should count each ?. in chain separately', () => {
    const code = `obj?.a?.b?.c`;
    expect(calculateCyclomaticComplexity(sourceFile)).toBe(4); // base + 3 ?.
  });

  it('should count optional method calls', () => {
    const code = `obj?.method?.()`;
    expect(calculateCyclomaticComplexity(sourceFile)).toBe(3); // base + 2 ?.
  });

  it('should count optional element access', () => {
    const code = `arr?.[index]`;
    expect(calculateCyclomaticComplexity(sourceFile)).toBe(2); // base + 1 ?.
  });
});
```

**GAP-AST-2: No Tests for Generator Functions**

```typescript
describe('generator functions', () => {
  it('should identify generator as function-like', () => {
    const code = `function* gen() { yield 1; }`;
    expect(isFunctionLike(node)).toBe(true);
  });

  it('should count yield as decision point', () => {
    // Should yield* count differently?
  });
});
```

**GAP-AST-3: Async Function Complexity**

```typescript
describe('async complexity', () => {
  it('should count await as decision point', () => {
    const code = `async function f() { await promise; }`;
    // Does await introduce a decision path?
    // Current implementation: No. Should it?
  });
});
```

### 3.2 Traditional Metrics (`traditional-metrics.ts`)

**Current Coverage: Good (40 tests)**

#### Gaps:

**GAP-TM-1: Performance Tests Missing**

The utility promises single-pass efficiency but lacks performance validation:

```typescript
describe('performance', () => {
  it('should be faster than separate calculations', () => {
    const code = /* large file */;

    const start1 = performance.now();
    const combined = calculateTraditionalMetrics(sourceFile);
    const time1 = performance.now() - start1;

    const start2 = performance.now();
    const cc = calculateCyclomaticComplexity(sourceFile);
    const halstead = calculateHalsteadMetrics(sourceFile);
    const time2 = performance.now() - start2;

    expect(time1).toBeLessThan(time2 * 0.7); // At least 30% faster
  });
});
```

**GAP-TM-2: Memory Tests Missing**

```typescript
describe('memory efficiency', () => {
  it('should not leak memory on large files', () => {
    const initialMemory = process.memoryUsage().heapUsed;

    for (let i = 0; i < 100; i++) {
      const code = generateLargeFile();
      calculateTraditionalMetrics(createSourceFile(code));
    }

    global.gc?.(); // Requires --expose-gc flag
    const finalMemory = process.memoryUsage().heapUsed;

    expect(finalMemory - initialMemory).toBeLessThan(10_000_000); // `<10`MB growth
  });
});
```

**GAP-TM-3: No Tests for Option Combinations**

```typescript
describe('option interactions', () => {
  it('should produce consistent results regardless of option order', () => {
    const code = `const x = a?.b ?? c;`;

    const result1 = calculateTraditionalMetrics(sourceFile, {
      includeOptionalChaining: true,
      includeNullishCoalescing: true,
    });

    const result2 = calculateTraditionalMetrics(sourceFile, {
      includeNullishCoalescing: true,
      includeOptionalChaining: true,
    });

    expect(result1.cyclomaticComplexity).toBe(result2.cyclomaticComplexity);
  });
});
```

### 3.3 Analysis Engine (`analysis-engine.ts`)

**Current Coverage: Fair (7 tests)**

This is the **most under-tested component** despite being critical.

#### Major Gaps:

**GAP-ENGINE-1: No Concurrency Tests**

Line 324 implements concurrency limiting but it's not tested:

```typescript
describe('analysis concurrency', () => {
  it('should respect maxConcurrentAnalyses limit', async () => {
    let concurrent = 0;
    let maxConcurrent = 0;

    const mockAnalysis: IAnalysis = {
      id: 'test',
      execute: async () => {
        concurrent++;
        maxConcurrent = Math.max(maxConcurrent, concurrent);
        await delay(10);
        concurrent--;
        return {
          /* ... */
        };
      },
    };

    const engine = new AnalysisEngine({
      analysisExecution: { maxConcurrentAnalyses: 3 },
    });

    // Run 10 analyses
    await engine.executeWithConcurrencyLimit(Array(10).fill(mockAnalysis), 3, a =>
      a.execute(sourceFile)
    );

    expect(maxConcurrent).toBeLessThanOrEqual(3);
  });
});
```

**GAP-ENGINE-2: No Timeout Tests**

Line 361 implements timeout but it's not tested:

```typescript
describe('analysis timeout', () => {
  it('should timeout long-running analyses', async () => {
    const slowAnalysis: IAnalysis = {
      id: 'slow',
      execute: async () => {
        await delay(60000); // 60 seconds
        return {
          /* ... */
        };
      },
    };

    const engine = new AnalysisEngine({
      analysisExecution: { analysisTimeout: 1000 }, // 1 second
    });

    const result = await engine.executeAnalysis(slowAnalysis, sourceFile, context);

    expect(result.error).toBeDefined();
    expect(result.error.error.message).toContain('Timeout');
  });
});
```

**GAP-ENGINE-3: No Cache Invalidation Tests**

Lines 549-575 implement caching logic but tests don't cover:

- Cache invalidation on file modification
- Cache TTL expiration
- Cache key collision handling
- Cache size limits

```typescript
describe('analysis caching', () => {
  it('should invalidate cache when file modified', async () => {
    const filePath = createTempFile('const x = 1;', 'cache-test.ts');

    const result1 = await engine.analyzeFile(filePath);

    // Modify file
    writeFileSync(filePath, 'const x = 2;', 'utf-8');

    const result2 = await engine.analyzeFile(filePath);

    expect(result1.data).not.toEqual(result2.data);
  });

  it('should invalidate cache after TTL', async () => {
    vi.useFakeTimers();

    const engine = new AnalysisEngine({
      analysisExecution: { analysisCacheTTL: 5000 },
    });

    const result1 = await engine.analyzeFile(filePath);

    vi.advanceTimersByTime(6000); // Exceed TTL

    const result2 = await engine.analyzeFile(filePath);

    // Should re-execute, not use cache
  });

  it('should handle hash collisions gracefully', () => {
    // Test with strings that produce same hash
  });
});
```

**GAP-ENGINE-4: No Error Recovery Tests**

```typescript
describe('error handling', () => {
  it('should continue after single analysis failure', async () => {
    const goodAnalysis = createMockAnalysis('good', () => ({ score: 50 }));
    const badAnalysis = createMockAnalysis('bad', () => {
      throw new Error('fail');
    });

    const plugin = {
      getAnalyses: () => [goodAnalysis, badAnalysis],
      // ...
    };

    const result = await engine.executePluginAnalyses(plugin, sourceFile, context);

    expect(result.successful).toHaveLength(1);
    expect(result.failed).toHaveLength(1);
  });

  it('should retry failed analyses when enabled', async () => {
    let attempts = 0;
    const flakyAnalysis = createMockAnalysis('flaky', () => {
      attempts++;
      if (attempts === 1) throw new Error('fail');
      return { score: 50 };
    });

    const engine = new AnalysisEngine({
      analysisExecution: { retryFailedAnalyses: true, maxRetries: 1 },
    });

    await engine.executeAnalysis(flakyAnalysis, sourceFile, context);

    expect(attempts).toBe(2);
  });
});
```

**GAP-ENGINE-5: No Integration with Multiple Plugins**

Current test only uses CoreAnalyzerPlugin. Need tests with multiple plugins:

```typescript
describe('multi-plugin scenarios', () => {
  it('should aggregate scores from multiple plugins', async () => {
    engine.registerPlugin(corePlugin);
    engine.registerPlugin(reactPlugin);
    engine.registerPlugin(securityPlugin);

    const result = await engine.analyzeFile('Component.tsx');

    expect(result.pluginResults.size).toBe(3);
    expect(result.overallScore).toBeCloseTo(
      weightedAverage([
        corePlugin.result.score,
        reactPlugin.result.score,
        securityPlugin.result.score,
      ])
    );
  });

  it('should handle plugin priority correctly', async () => {
    engine.registerPlugin(lowPriorityPlugin, { priority: 1 });
    engine.registerPlugin(highPriorityPlugin, { priority: 10 });

    const plugins = engine.getPlugins();
    expect(plugins[0].id).toBe('high-priority');
  });
});
```

### 3.4 Scoring System

**Current Coverage: Excellent (70 tests)**

The scoring system has the **best test coverage** in the codebase. This serves as a model for other components.

#### Strengths:

- Comprehensive boundary testing
- Input validation tests
- Edge case coverage
- Integration scenarios

#### Minor Gap:

**GAP-SCORE-1: No Tests for Grade Calibration**

```typescript
describe('grade calibration', () => {
  it('should align with industry standards', () => {
    // Compare grade boundaries with:
    // - SonarQube ratings
    // - Code Climate grades
    // - Microsoft maintainability index

    expect(scoreToGrade(25)).toBe('A'); // Excellent
    expect(scoreToGrade(45)).toBe('B'); // Good
    // etc.
  });
});
```

---

## 4. Edge Cases and Boundary Conditions

### 4.1 Well-Tested Edge Cases

✅ **Empty files**

```typescript
it('should handle empty file', () => {
  expect(complexity).toBe(1); // Base complexity
  expect(halstead.volume).toBe(0);
});
```

✅ **Files with only comments**

```typescript
it('should handle file with only comments', () => {
  expect(complexity).toBe(1);
  expect(halstead.uniqOperators).toBe(0);
});
```

✅ **Deeply nested structures**

```typescript
it('should handle deeply nested code', () => {
  const code = /* 4 levels of if nesting */;
  expect(complexity).toBe(5); // 4 ifs + base
});
```

### 4.2 Missing Edge Cases

**EDGE-1: Very Large Files**

No tests for files with thousands of lines:

```typescript
describe('large file handling', () => {
  it('should handle file with 10,000 lines', () => {
    const code = generateCode(10000);
    const start = performance.now();
    const result = analysis.execute(createSourceFile(code));
    const duration = performance.now() - start;

    expect(duration).toBeLessThan(5000); // `<5` seconds
    expect(result.data.complexity).toBeGreaterThan(0);
  });
});
```

**EDGE-2: Malformed Code**

Tests use mostly valid code. Need tests for:

- Syntax errors
- Incomplete code
- Invalid AST structures

```typescript
describe('malformed code', () => {
  it('should handle syntax errors gracefully', () => {
    const code = `function broken() { if (x { }`;
    expect(() => analysis.execute(createSourceFile(code))).not.toThrow();
  });

  it('should handle incomplete expressions', () => {
    const code = `const x = `;
    // Should not crash
  });
});
```

**EDGE-3: Unicode and Special Characters**

```typescript
describe('unicode handling', () => {
  it('should handle unicode identifiers', () => {
    const code = `const 日本語 = '値';`;
    expect(result.data.uniqOperands).toBeGreaterThan(0);
  });

  it('should handle emojis in strings', () => {
    const code = `const emoji = '😀🎉';`;
    expect(() => analysis.execute(createSourceFile(code))).not.toThrow();
  });
});
```

**EDGE-4: Circular Dependencies**

```typescript
describe('circular structures', () => {
  it('should handle recursive function calls', () => {
    const code =
      `
      function factorial(n) {
        if (n ` <=
      ` 1) return 1;
        return n * factorial(n - 1);
      }
    `;
    // Should count recursion correctly
  });
});
```

**EDGE-5: Modern JavaScript Features**

Missing tests for:

- Async generators
- Private class fields
- Top-level await
- Import assertions
- Decorators (experimental)

```typescript
describe('modern JavaScript', () => {
  it('should handle async generators', () => {
    const code = `
      async function* gen() {
        yield await Promise.resolve(1);
      }
    `;
  });

  it('should handle private fields', () => {
    const code = `
      class Example {
        #privateField = 0;
        get value() { return this.#privateField; }
      }
    `;
  });

  it('should handle top-level await', () => {
    const code = `
      const data = await fetch('/api');
    `;
  });
});
```

---

## 5. Recommendations by Priority

### Priority 1: Critical (Implement Immediately)

1. **Add Maintainability Index Calculation**
   - File: `src/analyses/maintainability-index-analysis.ts`
   - Tests: `src/analyses/maintainability-index-analysis.test.ts`
   - Effort: 8 hours
   - Impact: HIGH - Industry-standard metric currently missing

2. **Validate Algorithms Against Known Values**
   - Add benchmark tests with published complexity values
   - Cross-validate with ESLint, SonarQube
   - Effort: 16 hours
   - Impact: HIGH - Ensures correctness

3. **Comprehensive Engine Testing**
   - Concurrency limits
   - Timeouts
   - Cache invalidation
   - Error recovery
   - Effort: 12 hours
   - Impact: HIGH - Critical component under-tested

### Priority 2: Important (Next Sprint)

4. **Property-Based Testing**
   - Use fast-check for invariant testing
   - Effort: 8 hours
   - Impact: MEDIUM - Catches subtle bugs

5. **Edge Case Coverage**
   - Large files
   - Malformed code
   - Unicode handling
   - Modern JavaScript features
   - Effort: 12 hours
   - Impact: MEDIUM - Improves robustness

6. **Formula Validation Documentation**
   - Document all magic numbers
   - Add references to academic sources
   - Effort: 4 hours
   - Impact: MEDIUM - Improves maintainability

### Priority 3: Nice to Have

7. **Cognitive Complexity**
   - Implement as alternative to McCabe
   - Effort: 16 hours
   - Impact: LOW - Modern alternative metric

8. **Performance Benchmarks**
   - Add performance regression tests
   - Memory leak detection
   - Effort: 8 hours
   - Impact: LOW - Prevents future performance issues

9. **Comparative Analysis**
   - Compare with other tools (ESLint, SonarQube, CodeClimate)
   - Effort: 8 hours
   - Impact: LOW - Builds confidence in accuracy

---

## 6. Test Coverage Metrics

### Current Test Metrics

```
Test Files:  8 passed (8)
Tests:       304 passed (304)
Duration:    622ms
```

### Estimated Coverage by Component

| Component             | Line Coverage | Branch Coverage | Function Coverage | Assessment     |
| --------------------- | ------------- | --------------- | ----------------- | -------------- |
| Cyclomatic Complexity | ~95%          | ~90%            | 100%              | Excellent      |
| Halstead Metrics      | ~90%          | ~85%            | 100%              | Good           |
| AST Helpers           | ~85%          | ~80%            | ~95%              | Good           |
| Traditional Metrics   | ~90%          | ~85%            | 100%              | Good           |
| Scoring               | ~98%          | ~95%            | 100%              | Excellent      |
| Engine                | ~60%          | ~50%            | ~70%              | **Needs Work** |
| Base Analyzer         | ~80%          | ~75%            | 100%              | Good           |
| Plugin                | ~85%          | ~80%            | ~95%              | Good           |

**Overall Assessment:** Good foundational coverage (304 tests) but critical gaps in:

1. Algorithm validation against standards
2. Engine concurrency and caching
3. Edge case handling
4. Missing Maintainability Index metric

---

## 7. Validation Checklist

### Algorithm Correctness

- [ ] McCabe complexity validated against control flow graphs
- [ ] Halstead constants validated for modern languages
- [ ] Formulas cross-checked with academic literature
- [ ] Results compared with ESLint complexity rule
- [ ] Results compared with SonarQube complexity
- [ ] Maintainability Index implemented
- [ ] Cognitive Complexity considered

### Test Quality

- [ ] All magic numbers documented
- [ ] Boundary conditions tested for all thresholds
- [ ] Property-based tests for invariants
- [ ] Regression tests for known bugs
- [ ] Performance tests for large files
- [ ] Memory leak tests

### Edge Cases

- [ ] Empty files
- [ ] Comment-only files
- [ ] Malformed code
- [ ] Very large files (>10k lines)
- [ ] Unicode identifiers
- [ ] Modern JavaScript features
- [ ] Generator functions
- [ ] Async/await patterns
- [ ] Private class fields
- [ ] Top-level await

### Integration

- [ ] Multiple plugin coordination
- [ ] Plugin priority handling
- [ ] Cache invalidation on file changes
- [ ] Cache TTL expiration
- [ ] Timeout handling
- [ ] Concurrency limits
- [ ] Error recovery and retries

---

## 8. Recommended Test Plan

### Phase 1: Algorithm Validation (Week 1-2)

```typescript
// New file: src/analyses/__benchmarks__/complexity-benchmarks.test.ts
describe('Algorithm Validation', () => {
  describe('McCabe Complexity Benchmarks', () => {
    // Test against published algorithms
  });

  describe('Halstead Validation', () => {
    // Test against academic examples
  });

  describe('Cross-Tool Validation', () => {
    // Compare with ESLint, SonarQube
  });
});
```

### Phase 2: Missing Metrics (Week 2-3)

```typescript
// New file: src/analyses/maintainability-index-analysis.ts
// New file: src/analyses/maintainability-index-analysis.test.ts
// New file: src/analyses/cognitive-complexity-analysis.ts (optional)
```

### Phase 3: Engine Hardening (Week 3-4)

```typescript
// Enhance: src/engine/analysis-engine.integration.test.ts
describe('Analysis Engine - Comprehensive', () => {
  describe('Concurrency', () => {
    /* ... */
  });
  describe('Timeouts', () => {
    /* ... */
  });
  describe('Caching', () => {
    /* ... */
  });
  describe('Error Recovery', () => {
    /* ... */
  });
  describe('Multi-Plugin', () => {
    /* ... */
  });
});
```

### Phase 4: Edge Cases (Week 4-5)

```typescript
// New file: src/analyses/__tests__/edge-cases.test.ts
describe('Edge Case Handling', () => {
  describe('Large Files', () => {
    /* ... */
  });
  describe('Malformed Code', () => {
    /* ... */
  });
  describe('Unicode', () => {
    /* ... */
  });
  describe('Modern Features', () => {
    /* ... */
  });
});
```

---

## 9. Success Criteria

After implementing recommendations, the Core Analyzer should meet:

### Coverage Targets

- Line Coverage: >90% (currently ~85%)
- Branch Coverage: >85% (currently ~75%)
- Function Coverage: >95% (currently ~90%)

### Quality Targets

- All algorithms validated against published standards ✅
- All edge cases tested ✅
- All critical paths tested under failure conditions ✅
- Performance benchmarks established ✅
- No magic numbers without documentation ✅

### Metric Completeness

- [x] Cyclomatic Complexity (McCabe)
- [x] Halstead Metrics
- [ ] Maintainability Index ⚠️ **MISSING**
- [ ] Cognitive Complexity (optional)
- [x] Lines of Code

---

## 10. Conclusion

The Core Analyzer has **solid foundational test coverage** with 304 passing tests, demonstrating good testing practices in scoring, formula validation, and operator/operand counting. However, critical gaps exist that undermine confidence in production use:

### Strengths

1. **Comprehensive operator coverage** for modern JavaScript
2. **Formula validation** for Halstead metrics
3. **Excellent scoring system tests** (70 tests)
4. **Good edge case handling** for empty files and comments

### Critical Weaknesses

1. **Missing Maintainability Index** - Industry-standard metric absent
2. **No algorithm validation** against published standards
3. **Under-tested engine** - Concurrency, caching, timeouts not validated
4. **Limited edge case coverage** - Large files, malformed code, Unicode
5. **No cross-tool validation** - Results not compared with ESLint, SonarQube

### Immediate Actions Required

1. **Implement Maintainability Index** (8 hours) - P1
2. **Add algorithm benchmark tests** (16 hours) - P1
3. **Comprehensive engine testing** (12 hours) - P1
4. **Document all magic numbers** (4 hours) - P2

With these improvements, the Core Analyzer will be production-ready with confidence in correctness, robustness, and maintainability.

---

**Reviewed by:** Claude Code (Vitest TypeScript Testing Agent)
**Review Date:** 2026-01-15
**Next Review:** After P1 recommendations implemented
