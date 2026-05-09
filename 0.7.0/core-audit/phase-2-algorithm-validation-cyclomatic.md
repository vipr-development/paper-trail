# Phase 2: Algorithm Validation - Cyclomatic Complexity

**Priority:** P1 (Critical)
**Estimated Effort:** 16 hours
**Dependencies:** None
**Status:** Planning

## Overview

Validate the Cyclomatic Complexity implementation against academic standards, published algorithms, and industry tools (ESLint, SonarQube). The current implementation has good test coverage but lacks validation against known benchmarks and theoretical calculations.

## Background

### McCabe's Cyclomatic Complexity

**Formula:**

```
V(G) = E - N + 2P
```

Where:

- `E` = number of edges in the control flow graph
- `N` = number of nodes in the control flow graph
- `P` = number of connected components (typically 1)

**Simplified for structured programming:**

```
V(G) = number of decision points + 1
```

### Academic References

- McCabe, T. J. (1976). "A Complexity Measure". IEEE Transactions on Software Engineering.
- Watson, A. H., & McCabe, T. J. (1996). "Structured Testing: A Testing Methodology Using the Cyclomatic Complexity Metric"
- ESLint Complexity Rule: https://eslint.org/docs/rules/complexity
- SonarQube Cognitive Complexity: https://www.sonarsource.com/resources/cognitive-complexity/

## Current Implementation Status

**Location:** `analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts`

**Decision Points Counted:**

- IfStatement
- ConditionalExpression (ternary `? :`)
- ForStatement, ForInStatement, ForOfStatement
- WhileStatement, DoStatement
- CaseClause (not DefaultClause)
- CatchClause
- Logical operators: `&&`, `||`, `??`
- Optional chaining: `?.`

**Current Test Coverage:** 34 tests

- ✅ Good coverage of operators
- ✅ Nested structures tested
- ❌ No validation against theoretical CFGs
- ❌ No benchmark algorithms tested
- ❌ No cross-tool comparison

## Agent Assignments

### @code-complexity-analyzer

**Responsibilities:**

1. Research McCabe's original work and validate our interpretation
2. Create control flow graph examples with known complexity
3. Identify standard algorithms (bubble sort, binary search, quicksort, etc.) with published complexity values
4. Compare our implementation with ESLint's complexity rule
5. Document any differences in counting methodology
6. Provide benchmark test cases with:
   - Code samples
   - Expected complexity
   - Justification/reference

**Deliverables:**

- `docs/0.7.0/core-audit/phase-2-research-cyclomatic-complexity.md`
  - McCabe formula validation
  - Control flow graph examples
  - Benchmark algorithms with expected CC values
  - ESLint comparison analysis
  - Edge case identification

### @vitest-engineer

**Responsibilities:**

1. Write benchmark test suite based on complexity-analyzer research
2. Create tests for theoretical CFG validation
3. Add tests for edge cases identified by complexity-analyzer
4. Ensure all magic numbers in tests are documented
5. Add regression tests for any bugs found during validation

**Deliverables:**

- `analyzers/core/src/analyses/__benchmarks__/cyclomatic-complexity-benchmarks.test.ts`
- Enhanced `analyzers/core/src/analyses/cyclomatic-complexity-analysis.test.ts` with:
  - CFG validation tests
  - Benchmark algorithm tests
  - Edge case tests
  - Documented expected values

**Minimum:** 25 new tests

### @typescript-engineer

**Responsibilities:**

1. Review implementation for correctness based on complexity-analyzer findings
2. Fix any discrepancies found during validation
3. Document the implementation with formula and references
4. Ensure code comments explain non-obvious complexity calculations
5. Optimize if performance issues discovered during benchmark testing

**Deliverables:**

- Updated `analyzers/core/src/analyses/cyclomatic-complexity-analysis.ts` (if changes needed)
- Enhanced documentation comments
- Performance improvements (if needed)

## Implementation Details

### Benchmark Test Suite Structure

```typescript
// analyzers/core/src/analyses/__benchmarks__/cyclomatic-complexity-benchmarks.test.ts

import { describe, it, expect } from 'vitest';
import { CyclomaticComplexityAnalysis } from '../cyclomatic-complexity-analysis';
import { createSourceFile } from '../../testing/test-utils';

describe('Cyclomatic Complexity - Algorithm Benchmarks', () => {
  const analysis = new CyclomaticComplexityAnalysis();

  describe('Control Flow Graph Validation', () => {
    it('should match CFG formula: V(G) = E - N + 2P', () => {
      // Example: Diamond-shaped control flow
      // Entry -> Decision -> PathA/PathB -> Merge -> Exit
      // Nodes: 5 (Entry, Decision, PathA, PathB, Exit)
      // Edges: 5 (Entry->Decision, Decision->PathA, Decision->PathB, PathA->Exit, PathB->Exit)
      // Components: 1
      // V(G) = 5 - 5 + 2(1) = 2
      const code = `
        function diamond(x: number): number {
          if (x > 0) {
            return x;  // PathA
          } else {
            return -x; // PathB
          }
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: 1 base + 1 if statement = 2
      expect(result.data.complexity).toBe(2);
      expect(result.data.decisionPoints).toBe(1);
    });

    it('should match CFG for sequential decisions', () => {
      // Two independent if statements
      // Creates two diamond patterns
      // V(G) = 1 base + 2 decisions = 3
      const code = `
        function sequential(a: number, b: number): number {
          if (a > 0) {
            console.log(a);
          }
          if (b > 0) {
            console.log(b);
          }
          return a + b;
        }
      `;
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.complexity).toBe(3);
    });

    it('should match CFG for nested decisions', () => {
      // Nested if creates more paths
      const code = `
        function nested(a: number, b: number): number {
          if (a > 0) {
            if (b > 0) {
              return a + b;
            }
            return a;
          }
          return 0;
        }
      `;
      const result = analysis.execute(createSourceFile(code));
      // Expected: 1 base + 2 if statements = 3
      expect(result.data.complexity).toBe(3);
    });
  });

  describe('Published Algorithm Benchmarks', () => {
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
      const result = analysis.execute(createSourceFile(bubbleSort));

      // Expected complexity calculation:
      // Base: 1
      // Outer for loop: +1
      // Inner for loop: +1
      // If statement: +1
      // Total: 4
      expect(result.data.complexity).toBe(4);
      expect(result.data.decisionPoints).toBe(3);
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
            if (arr[mid] === target) {
              return mid;
            }
            if (arr[mid] < target) {
              left = mid + 1;
            } else {
              right = mid - 1;
            }
          }
          return -1;
        }
      `;
      const result = analysis.execute(createSourceFile(binarySearch));

      // Expected complexity calculation:
      // Base: 1
      // while loop: +1
      // if (arr[mid] === target): +1
      // if (arr[mid] < target): +1
      // else path (implicit): counted via if statement
      // Total: 4
      expect(result.data.complexity).toBe(4);
      expect(result.data.decisionPoints).toBe(3);
    });

    it('should match published complexity for quicksort', () => {
      const quicksort =
        `
        function quicksort(arr: number[]): number[] {
          if (arr.length ` <=
        ` 1) {
            return arr;
          }
          const pivot = arr[0];
          const left = arr.slice(1).filter(x => x < pivot);
          const right = arr.slice(1).filter(x => x >= pivot);
          return [...quicksort(left), pivot, ...quicksort(right)];
        }
      `;
      const result = analysis.execute(createSourceFile(quicksort));

      // Expected complexity calculation:
      // Base: 1
      // if (arr.length `<=` 1): +1
      // filter predicate (x => x < pivot): +0 (no decision in predicate)
      // filter predicate (x => x >= pivot): +0
      // Total: 2
      expect(result.data.complexity).toBe(2);
    });

    it('should match complexity for fibonacci (recursive)', () => {
      const fibonacci =
        `
        function fibonacci(n: number): number {
          if (n ` <=
        ` 1) {
            return n;
          }
          return fibonacci(n - 1) + fibonacci(n - 2);
        }
      `;
      const result = analysis.execute(createSourceFile(fibonacci));

      // Expected complexity calculation:
      // Base: 1
      // if (n `<=` 1): +1
      // Total: 2
      expect(result.data.complexity).toBe(2);
    });

    it('should match complexity for merge sort', () => {
      const mergeSort =
        `
        function mergeSort(arr: number[]): number[] {
          if (arr.length ` <=
        ` 1) return arr;

          const mid = Math.floor(arr.length / 2);
          const left = mergeSort(arr.slice(0, mid));
          const right = mergeSort(arr.slice(mid));

          return merge(left, right);
        }

        function merge(left: number[], right: number[]): number[] {
          const result: number[] = [];
          let i = 0, j = 0;

          while (i < left.length && j < right.length) {
            if (left[i] ` <=
        ` right[j]) {
              result.push(left[i++]);
            } else {
              result.push(right[j++]);
            }
          }

          return result.concat(left.slice(i)).concat(right.slice(j));
        }
      `;
      const result = analysis.execute(createSourceFile(mergeSort));

      // mergeSort function:
      // Base: 1, if: +1 = CC: 2
      // merge function:
      // Base: 1, while: +1, &&: +1, if: +1 = CC: 4
      // Total file complexity should aggregate both functions
    });
  });

  describe('Short-Circuit Evaluation', () => {
    it('should count each && in chain', () => {
      const code = `
        function check(a, b, c, d) {
          return a && b && c && d;
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + three && operators = 4
      expect(result.data.complexity).toBe(4);
      expect(result.data.decisionPoints).toBe(3);
    });

    it('should count each || in chain', () => {
      const code = `
        function check(a, b, c) {
          return a || b || c;
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + two || operators = 3
      expect(result.data.complexity).toBe(3);
    });

    it('should count mixed logical operators', () => {
      const code = `
        function check(a, b, c, d) {
          return (a && b) || (c && d);
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + 2 && + 1 || = 4
      expect(result.data.complexity).toBe(4);
    });
  });

  describe('Edge Cases from Research', () => {
    it('should handle switch with multiple cases', () => {
      const code = `
        function handleCode(code: number): string {
          switch (code) {
            case 200:
              return 'OK';
            case 404:
              return 'Not Found';
            case 500:
              return 'Server Error';
            default:
              return 'Unknown';
          }
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + 3 case clauses (not default) = 4
      expect(result.data.complexity).toBe(4);
    });

    it('should count optional chaining in chain', () => {
      const code = `
        function getValue(obj: any): any {
          return obj?.prop1?.prop2?.prop3;
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + 3 ?. operators = 4
      expect(result.data.complexity).toBe(4);
    });

    it('should count nullish coalescing in chain', () => {
      const code = `
        function getValue(a, b, c): any {
          return a ?? b ?? c ?? 'default';
        }
      `;
      const result = analysis.execute(createSourceFile(code));

      // Expected: Base 1 + 3 ?? operators = 4
      expect(result.data.complexity).toBe(4);
    });
  });

  describe('McCabe Original Examples', () => {
    it('should match McCabe 1976 paper example', () => {
      // Include examples from McCabe's original paper if available
      // TODO: complexity-analyzer to provide these
    });
  });
});
```

### Enhanced Main Test File

Add to existing `cyclomatic-complexity-analysis.test.ts`:

```typescript
describe('CyclomaticComplexityAnalysis - Formula Validation', () => {
  it('should document that we count decision points + 1', () => {
    // Document our implementation approach
    const code = `function simple(x) { if (x) return x; }`;
    const result = analysis.execute(createSourceFile(code));

    // Complexity = 1 (base) + 1 (if) = 2
    expect(result.data.complexity).toBe(result.data.decisionPoints + 1);
  });

  it('should explain magic numbers in expected values', () => {
    const code = `
      function complex(a, b, c) {
        if (a) {
          if (b) {
            return c;
          }
        }
        return 0;
      }
    `;
    const result = analysis.execute(createSourceFile(code));

    // Expected complexity calculation:
    // - Base complexity: 1
    // - First if statement: +1
    // - Second if statement: +1
    // Total: 3
    expect(result.data.complexity).toBe(3);
    expect(result.data.decisionPoints).toBe(2);
  });
});
```

## Acceptance Criteria

### Must Have

- [ ] Research document completed with:
  - [ ] McCabe formula validation
  - [ ] 10+ benchmark algorithms with expected values
  - [ ] ESLint comparison findings
  - [ ] Academic references
- [ ] 25+ new benchmark tests passing
- [ ] All benchmark tests include:
  - [ ] Code sample
  - [ ] Expected complexity value
  - [ ] Comment explaining calculation
  - [ ] Academic/tool reference
- [ ] All existing tests still pass
- [ ] All magic numbers in tests documented
- [ ] Implementation validated against research
- [ ] Any bugs found are fixed

### Should Have

- [ ] Cross-validation with ESLint complexity rule
- [ ] Documentation comments updated with references
- [ ] Performance benchmarks (time to analyze 10k LOC file)

### Nice to Have

- [ ] Comparison test that runs ESLint and compares results
- [ ] Visual control flow graph examples in documentation

## Testing Strategy

### Research Validation (code-complexity-analyzer)

1. **McCabe Formula Validation**
   - Create CFG examples
   - Calculate E, N, P manually
   - Verify V(G) = E - N + 2P matches our implementation

2. **Benchmark Algorithms**
   - Identify 10+ standard algorithms
   - Find published complexity values (or calculate from CFG)
   - Document expected values with justification

3. **ESLint Comparison**
   - Run ESLint complexity rule on test files
   - Compare results with our implementation
   - Document any differences and reasons

### Test Implementation (vitest-engineer)

1. **CFG Tests**
   - Simple diamond (if/else)
   - Sequential decisions
   - Nested decisions
   - Loops with conditions

2. **Benchmark Tests**
   - Bubble sort
   - Binary search
   - Quicksort
   - Merge sort
   - Fibonacci
   - Other standard algorithms

3. **Edge Case Tests**
   - Short-circuit evaluation chains
   - Switch statements with many cases
   - Optional chaining chains
   - Nullish coalescing chains
   - Mixed logical operators

### Code Review (typescript-engineer)

1. **Validation Review**
   - Compare implementation with McCabe definition
   - Verify decision point counting is correct
   - Check for edge cases not handled

2. **Documentation**
   - Add formula to code comments
   - Reference academic papers
   - Explain non-obvious decisions

## Testing Commands

```bash
# Run benchmark tests only
cd analyzers/core
pnpm test cyclomatic-complexity-benchmarks.test.ts

# Run all cyclomatic complexity tests
pnpm test cyclomatic-complexity

# Run all tests
pnpm test

# Coverage
pnpm test --coverage

# Compare with ESLint (manual)
npx eslint --rule 'complexity: ["error", 10]' <test-file>
```

## CLI Integration

No CLI changes required for this phase. This phase validates existing functionality.

### Validation Command

```bash
# Test that CLI output is accurate
cd clients/cli
pnpm build

# Test with bubble sort
cat > /tmp/bubble-sort.ts << 'EOF'
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
EOF

./dist/index.js analyze /tmp/bubble-sort.ts --format json

# Verify cyclomatic complexity is 4
# Verify decision points is 3
```

## Dependencies

**None** - This phase can start immediately in parallel with Phase 1.

## Risks & Mitigations

| Risk                                    | Impact | Mitigation                                     |
| --------------------------------------- | ------ | ---------------------------------------------- |
| Implementation differs from McCabe      | High   | Thorough research phase with CFG validation    |
| ESLint uses different methodology       | Medium | Document differences, justify our approach     |
| Performance degradation from new tests  | Low    | Benchmark tests are unit tests, minimal impact |
| Finding bugs in existing implementation | Medium | Fix bugs, add regression tests                 |

## Definition of Done

- [ ] Research document completed and reviewed
- [ ] 25+ benchmark tests implemented and passing
- [ ] All existing tests still pass
- [ ] All test magic numbers documented
- [ ] Implementation validated against McCabe formula
- [ ] ESLint comparison completed
- [ ] Any bugs found are fixed with regression tests
- [ ] Code comments updated with academic references
- [ ] Phase approved by user

## Next Steps After Completion

1. User reviews validation findings
2. Address any discrepancies found
3. Proceed to Phase 3: Algorithm Validation - Halstead Metrics
4. Use validated CC in Phase 1 (Maintainability Index)
