# Cognitive Complexity

Cognitive Complexity measures how difficult code is to understand, based on SonarSource's published methodology. Unlike Cyclomatic Complexity, which counts execution paths, Cognitive Complexity penalizes deeply nested structures that increase cognitive load for developers.

## Formula

The score is the sum of increments for each control flow break, with a nesting penalty applied:

```text
CC_cognitive = sum(i = 1..n) (1 + nesting_depth_i)
```

Where each qualifying control structure adds `1 + current nesting depth` to the total score.

### Key Differences from Cyclomatic Complexity

| Feature            | Cyclomatic           | Cognitive                 |
| ------------------ | -------------------- | ------------------------- |
| Nesting penalty    | None                 | `+1` per nesting level    |
| `switch` statement | `+1` per `case`      | `+1` total                |
| `else if` chains   | `+1` per branch      | Only initial `if` counts  |
| Logical operators  | `+1` per `&&`/`\|\|` | Only on operator switches |
| Recursion          | No special handling  | `+1` for recursive calls  |

### Scoring Rules

1. **Structural increment (+1)**: `if`, `else if`, `else`, `switch`, `for`, `while`, `do-while`, `catch`, `throw`, ternary `?:`
2. **Nesting penalty (+depth)**: For each structural element nested inside another, add the current nesting depth
3. **Logical operator breaks (+1)**: When switching between `&&` and `||` in a boolean sequence (e.g., `a && b || c` scores +1 for the switch)

## Thresholds

| Range | Rating    | Meaning                                        |
| ----- | --------- | ---------------------------------------------- |
| 0-5   | Excellent | Simple, easy to understand                     |
| 6-15  | Good      | Moderate complexity, acceptable                |
| 16-30 | Fair      | Complex, consider refactoring                  |
| 31+   | Poor      | Very complex, refactoring strongly recommended |

## Max Nesting Depth

Alongside cognitive complexity, the analysis reports the maximum nesting depth found in the file. Deep nesting is a primary driver of cognitive load.

| Depth | Rating    | Meaning                               |
| ----- | --------- | ------------------------------------- |
| 0-2   | Excellent | Flat, easy to follow                  |
| 3-4   | Good      | Moderate nesting, acceptable          |
| 5-6   | Fair      | Deep nesting, consider extraction     |
| 7+    | Poor      | Very deep nesting, refactoring needed |

## How to Improve

- **Flatten nested conditionals** using guard clauses and early returns
- **Extract nested blocks** into named helper functions with descriptive names
- **Simplify boolean expressions** by breaking complex sequences into intermediate variables
- **Use polymorphism** instead of deeply nested type-checking conditionals
- **Replace callback pyramids** with async/await patterns
- **Apply single level of abstraction** principle per function

## Example

```typescript
// High cognitive complexity (score: 13)
function processOrder(order: Order) {
  if (order.items.length > 0) {
    // +1
    for (const item of order.items) {
      // +1 (nesting: +1)
      if (item.inStock) {
        // +1 (nesting: +2)
        if (item.quantity > 0) {
          // +1 (nesting: +3)
          // process
        } else {
          // +1 (nesting: +3)
          // handle
        }
      }
    }
  }
}

// Refactored (score: 4)
function processOrder(order: Order) {
  if (order.items.length === 0) return; // +1

  for (const item of order.items) {
    // +1
    processItem(item); // extracted
  }
}

function processItem(item: Item) {
  if (!item.inStock || item.quantity <= 0) return; // +1 (+ +1 for ||)
  // process
}
```

## Reference

- [Cognitive Complexity: A New Way of Measuring Understandability (SonarSource Whitepaper)](https://www.sonarsource.com/docs/CognitiveComplexity.pdf)
- [Cognitive Complexity: Because Testability != Understandability](https://www.sonarsource.com/blog/cognitive-complexity-because-testability-understandability/)
