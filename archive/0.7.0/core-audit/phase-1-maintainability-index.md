# Phase 1: Maintainability Index Implementation

**Priority:** P1 (Critical)
**Estimated Effort:** 12 hours
**Dependencies:** None
**Status:** Planning

## Overview

Implement the Maintainability Index (MI), an industry-standard metric that combines Cyclomatic Complexity, Halstead Volume, and Lines of Code into a single 0-100 maintainability score. This metric is currently **missing** from the Core Analyzer despite being widely used in tools like Visual Studio, SonarQube, and Code Climate.

## Background

### The Maintainability Index Formula

**Original Formula (Oman & Hagemeister, 1992):**

```
MI = 171 - 5.2 * ln(V) - 0.23 * CC - 16.2 * ln(LOC)
```

Where:

- `V` = Halstead Volume
- `CC` = Cyclomatic Complexity
- `LOC` = Lines of Code (executable lines, excluding comments/blanks)

**Scaled to 0-100:**

```
MI_scaled = max(0, (MI / 171) * 100)
```

### Industry Standards

| Range  | Rating | Maintainability                 |
| ------ | ------ | ------------------------------- |
| 85-100 | A      | Excellent maintainability       |
| 65-85  | B      | Good maintainability            |
| 40-65  | C      | Moderate maintainability        |
| 10-40  | D      | Difficult to maintain           |
| 0-10   | F      | Extremely difficult to maintain |

### Academic References

- Oman, P., & Hagemeister, J. (1992). "Metrics for assessing a software system's maintainability"
- Coleman, D., et al. (1994). "Using metrics to evaluate software system maintainability"
- Microsoft Visual Studio Code Metrics: https://docs.microsoft.com/en-us/visualstudio/code-quality/code-metrics-values

## Agent Assignments

### @code-complexity-analyzer

**Responsibilities:**

1. Research and validate the MI formula against academic literature
2. Confirm the formula coefficients (5.2, 0.23, 16.2) are appropriate
3. Document how different tools (VS, SonarQube, Code Climate) implement MI
4. Provide benchmark test cases with known MI values
5. Validate the 0-100 scaling approach

**Deliverables:**

- `docs/0.7.0/core-audit/phase-1-research-mi-standards.md`
  - Academic references
  - Cross-tool comparison
  - Formula validation
  - Benchmark test cases with expected values

### @typescript-engineer

**Responsibilities:**

1. Implement `MaintainabilityIndexAnalysis` class
2. Follow existing patterns from `cyclomatic-complexity-analysis.ts` and `halstead-metrics-analysis.ts`
3. Use existing types from `@packages/common/src/types`
4. Ensure SOLID principles and DRY code
5. Create appropriate TypeScript interfaces for MI data
6. Register the new analysis in the Core Plugin

**Deliverables:**

- `analyzers/core/src/analyses/maintainability-index-analysis.ts`
- Updated `analyzers/core/src/analyses/index.ts`
- Updated `analyzers/core/src/plugin.ts` (register new analysis)
- Type definitions in `@packages/common/src/types/core` (if needed)

**Architecture Requirements:**

- Reuse existing utility functions: `calculateCyclomaticComplexity`, `calculateHalsteadMetrics`
- Follow the `IAnalysis<unknown, TMaintainabilityIndexData>` interface
- Generate insights based on MI thresholds
- Include component breakdown in results (volume, complexity, LOC contributions)

### @vitest-engineer

**Responsibilities:**

1. Write comprehensive test suite for `MaintainabilityIndexAnalysis`
2. Include formula validation tests
3. Test MI scaling to 0-100 range
4. Test maintainability level boundaries
5. Test with benchmark code samples provided by complexity-analyzer
6. Follow project conventions (peer tests, no **tests** folders)

**Deliverables:**

- `analyzers/core/src/analyses/maintainability-index-analysis.test.ts`
- Minimum 30 tests covering:
  - Formula correctness
  - Boundary conditions
  - Simple code (high MI)
  - Complex code (low MI)
  - Edge cases (empty files, etc.)

## Implementation Details

### File Structure

```
analyzers/core/src/analyses/
├── maintainability-index-analysis.ts       (NEW)
├── maintainability-index-analysis.test.ts  (NEW)
├── cyclomatic-complexity-analysis.ts       (existing)
├── halstead-metrics-analysis.ts            (existing)
└── index.ts                                (update exports)

analyzers/core/src/
└── plugin.ts                               (register new analysis)

packages/common/src/types/core/
└── (add MI types if needed)
```

### Type Definition

```typescript
// In @packages/common/src/types/core or in the analysis file
export interface MaintainabilityIndexData {
  /** Scaled maintainability index (0-100) */
  maintainabilityIndex: number;

  /** Raw MI value before scaling */
  rawIndex: number;

  /** Component breakdown */
  components: {
    /** Volume component: 5.2 * ln(V) */
    volumeComponent: number;
    /** Complexity component: 0.23 * CC */
    complexityComponent: number;
    /** LOC component: 16.2 * ln(LOC) */
    locComponent: number;
  };

  /** Maintainability rating */
  rating: 'A' | 'B' | 'C' | 'D' | 'F';
}
```

### Implementation Skeleton

```typescript
import { SourceFile } from 'ts-morph';
import { IAnalysis, AnalysisResult, PluginInsight } from '@vipr/common/types';
import { calculateCyclomaticComplexity } from '../utils/ast-helpers';
import { calculateHalsteadMetrics } from './halstead-metrics-analysis';

export class MaintainabilityIndexAnalysis implements IAnalysis<unknown, MaintainabilityIndexData> {
  readonly id = 'core-maintainability';
  readonly name = 'Maintainability Index';
  readonly description =
    'Calculates the Maintainability Index (MI) combining complexity, volume, and LOC';
  readonly category = 'complexity';

  execute(sourceFile: SourceFile): AnalysisResult<MaintainabilityIndexData> {
    // 1. Calculate dependencies
    const cc = calculateCyclomaticComplexity(sourceFile);
    const halstead = calculateHalsteadMetrics(sourceFile);
    const loc = this.calculateExecutableLOC(sourceFile);

    // 2. Apply MI formula
    const rawMI = this.calculateRawMI(halstead.volume, cc, loc);

    // 3. Scale to 0-100
    const scaledMI = Math.max(0, (rawMI / 171) * 100);

    // 4. Generate data
    const data: MaintainabilityIndexData = {
      maintainabilityIndex: Math.round(scaledMI),
      rawIndex: Math.round(rawMI * 100) / 100,
      components: {
        volumeComponent: 5.2 * Math.log(halstead.volume || 1),
        complexityComponent: 0.23 * cc,
        locComponent: 16.2 * Math.log(loc || 1),
      },
      rating: this.getRating(scaledMI),
    };

    // 5. Generate insights
    const insights = this.generateInsights(data);

    return {
      analysisId: this.id,
      category: this.category,
      data,
      insights,
      score: 100 - scaledMI, // Invert: high MI = low complexity score
      executionTimeMs: 0,
    };
  }

  private calculateRawMI(volume: number, cc: number, loc: number): number {
    // Handle edge cases
    if (volume `<=` 0 || loc `<=` 0) return 171; // Maximum maintainability

    return 171 - 5.2 * Math.log(volume) - 0.23 * cc - 16.2 * Math.log(loc);
  }

  private calculateExecutableLOC(sourceFile: SourceFile): number {
    // Count non-comment, non-blank lines
    // Exclude import statements
    // Exclude type-only declarations
    // TODO: Implement
    return 1;
  }

  private getRating(mi: number): 'A' | 'B' | 'C' | 'D' | 'F' {
    if (mi >= 85) return 'A';
    if (mi >= 65) return 'B';
    if (mi >= 40) return 'C';
    if (mi >= 10) return 'D';
    return 'F';
  }

  private generateInsights(data: MaintainabilityIndexData): PluginInsight[] {
    const insights: PluginInsight[] = [];

    if (data.maintainabilityIndex < 40) {
      insights.push({
        severity: 'warning',
        message: `Low maintainability index (${data.maintainabilityIndex}). Consider refactoring.`,
        category: 'maintainability',
      });
    }

    // Add component-specific insights
    // TODO: Implement

    return insights;
  }
}
```

## Acceptance Criteria

### Must Have

- [ ] `MaintainabilityIndexAnalysis` class implemented
- [ ] MI formula correctly implements: `171 - 5.2*ln(V) - 0.23*CC - 16.2*ln(LOC)`
- [ ] MI scaled to 0-100 range
- [ ] Component breakdown included (volume, complexity, LOC)
- [ ] Rating system (A/B/C/D/F) implemented
- [ ] Insights generated based on MI thresholds
- [ ] Minimum 30 tests passing
- [ ] Tests validate formula correctness
- [ ] Tests include benchmark cases from research
- [ ] Registered in Core Plugin
- [ ] Exported from `analyzers/core/src/analyses/index.ts`
- [ ] All existing tests still pass

### Should Have

- [ ] Executable LOC calculation excludes comments, blanks, imports, type-only declarations
- [ ] Insights explain which component is driving low MI
- [ ] Performance is acceptable (`<50`ms for typical files)

### Nice to Have

- [ ] Documentation comments with academic references
- [ ] Comparison with other tools' MI calculations in tests

## Testing Strategy

### Unit Tests (vitest-engineer)

```typescript
describe('MaintainabilityIndexAnalysis', () => {
  describe('formula validation', () => {
    it('should calculate MI using standard formula', () => {
      // Given: Known values V=100, CC=5, LOC=50
      // Expected: MI = 171 - 5.2*ln(100) - 0.23*5 - 16.2*ln(50)
      //              = 171 - 23.93 - 1.15 - 63.36
      //              ≈ 82.56
      // Scaled: (82.56 / 171) * 100 ≈ 48.28
    });

    it('should scale MI to 0-100 range', () => {
      // Test scaling
    });

    it('should handle edge case: empty file', () => {
      // MI should be 100 (max maintainability)
    });
  });

  describe('maintainability levels', () => {
    it('should rate simple code as excellent (85-100)', () => {
      const code = `function add(a, b) { return a + b; }`;
      // Expected rating: 'A'
    });

    it('should rate complex code as poor (10-40)', () => {
      const complexCode = /* deeply nested, high CC, high volume */;
      // Expected rating: 'D' or 'F'
    });

    it('should rate at boundary: 85 should be A', () => {
      // Construct code that yields MI = 85
    });

    it('should rate at boundary: 84.9 should be B', () => {
      // Construct code that yields MI = 84.9
    });
  });

  describe('component breakdown', () => {
    it('should calculate volume component correctly', () => {
      // Verify: 5.2 * ln(volume)
    });

    it('should calculate complexity component correctly', () => {
      // Verify: 0.23 * CC
    });

    it('should calculate LOC component correctly', () => {
      // Verify: 16.2 * ln(LOC)
    });
  });

  describe('insights', () => {
    it('should generate warning for MI < 40', () => {
      // Low MI should trigger warning
    });

    it('should generate no insights for MI >= 85', () => {
      // Excellent MI should have no warnings
    });

    it('should explain which component is problematic', () => {
      // If volume is high, insight should mention it
    });
  });

  describe('benchmark cases', () => {
    it('should match expected MI for bubble sort', () => {
      // Use benchmark from complexity-analyzer research
    });

    it('should match expected MI for binary search', () => {
      // Use benchmark from complexity-analyzer research
    });

    it('should match expected MI for simple getter', () => {
      // Use benchmark from complexity-analyzer research
    });
  });
});
```

### Integration Testing

```bash
# Run MI tests
cd analyzers/core
pnpm test maintainability-index-analysis.test.ts

# Run all core tests (ensure no regressions)
pnpm test

# Verify plugin registration
pnpm test plugin.test.ts
```

## CLI Integration

### Updates Required

1. **CLI Formatter Updates**
   - Update `clients/cli/src/formatters/cli-formatter.ts` to display MI
   - Update `clients/cli/src/formatters/json-formatter.ts` to include MI
   - Update `clients/cli/src/formatters/markdown-formatter.ts` to include MI

2. **CLI Output Example**

```bash
$ vipr analyze src/example.ts

Maintainability Analysis:
  Maintainability Index: 68 (B - Good)
    Volume Component:     23.9
    Complexity Component:  1.2
    LOC Component:        63.4

  Cyclomatic Complexity: 5
  Halstead Volume:      100.2
  Lines of Code:        50
```

3. **JSON Output Example**

```json
{
  "analyses": [
    {
      "analysisId": "core-maintainability",
      "category": "complexity",
      "data": {
        "maintainabilityIndex": 68,
        "rawIndex": 82.56,
        "components": {
          "volumeComponent": 23.93,
          "complexityComponent": 1.15,
          "locComponent": 63.36
        },
        "rating": "B"
      },
      "insights": [],
      "score": 32
    }
  ]
}
```

### CLI Integration Testing

```bash
# Build CLI
cd clients/cli
pnpm build

# Test with simple file
echo "function add(a, b) { return a + b; }" > /tmp/test.ts
./dist/index.js analyze /tmp/test.ts --format json

# Verify MI is present in output
# Verify MI value is reasonable (should be high for simple code)

# Test with complex file
# Create file with high CC, high volume
./dist/index.js analyze /tmp/complex.ts --format cli

# Verify MI is low
# Verify rating is D or F
```

## Testing Commands

```bash
# Phase 1 specific tests
cd analyzers/core
pnpm test maintainability-index-analysis.test.ts

# Run all core tests
pnpm test

# Coverage report
pnpm test --coverage

# CLI integration
cd ../../clients/cli
pnpm build
pnpm test
./dist/index.js analyze <test-file>
```

## Dependencies

**None** - This phase can start immediately.

## Risks & Mitigations

| Risk                                  | Impact | Mitigation                                            |
| ------------------------------------- | ------ | ----------------------------------------------------- |
| Formula disagreement with other tools | Medium | Research phase validates against VS, SonarQube        |
| LOC calculation complexity            | Medium | Use ts-morph AST to accurately count executable lines |
| Performance impact                    | Low    | MI reuses existing CC and Halstead calculations       |
| Type system complexity                | Low    | Follow existing patterns from other analyses          |

## Definition of Done

- [ ] Research document completed with academic references
- [ ] Implementation complete and follows existing patterns
- [ ] 30+ tests passing
- [ ] All existing tests still pass
- [ ] Registered in Core Plugin
- [ ] CLI integration complete (formatters updated)
- [ ] CLI integration tested manually
- [ ] Code reviewed by user
- [ ] Documentation comments include formula and references
- [ ] Phase approved by user

## Next Steps After Completion

1. User reviews and approves implementation
2. Proceed to Phase 2: Algorithm Validation - Cyclomatic Complexity
3. Proceed to Phase 3: Algorithm Validation - Halstead Metrics
4. Consider Cognitive Complexity (optional future enhancement)
