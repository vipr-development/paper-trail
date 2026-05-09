# Phase 8: Property-Based Testing

**Priority:** P2 (Important)
**Estimated Effort:** 12 hours
**Dependencies:** Phases 1-3
**Status:** Planning

## Overview

Implement property-based testing using fast-check to validate invariants across randomly generated code samples. This catches edge cases that manual tests might miss.

## Agent Assignments

### @vitest-engineer

- Install and configure fast-check
- Write property-based tests for Halstead invariants
- Write property-based tests for complexity invariants
- Write property-based tests for maintainability index
- **Minimum:** 10 property tests

### @typescript-engineer

- Review any failing properties
- Fix violations of invariants
- Optimize if performance issues found

## Key Test Areas

### Halstead Properties

```typescript
import { fc } from 'fast-check';

describe('Halstead Metrics - Property-Based Tests', () => {
  it(
    'vocabularySize = uniqOperators + uniqOperands',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.vocabularySize).toBe(result.data.uniqOperators + result.data.uniqOperands);
    })
  );

  it(
    'programLength = totalOperators + totalOperands',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.programLength).toBe(
        result.data.totalOperators + result.data.totalOperands
      );
    })
  );

  it(
    'volume = programLength * log2(vocabularySize)',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      const expected = result.data.programLength * Math.log2(result.data.vocabularySize || 1);
      expect(result.data.volume).toBeCloseTo(expected, 2);
    })
  );

  it('effort = difficulty * volume');
  it('difficulty >= 0');
  it('volume >= 0');
  it('bugs >= 0');
});
```

### Complexity Properties

```typescript
describe('Cyclomatic Complexity - Property-Based Tests', () => {
  it(
    'complexity >= 1 (always at least base complexity)',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.complexity).toBeGreaterThanOrEqual(1);
    })
  );

  it(
    'complexity = decisionPoints + 1',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.complexity).toBe(result.data.decisionPoints + 1);
    })
  );

  it('adding decision point increases complexity by 1');
});
```

### Maintainability Index Properties

```typescript
describe('Maintainability Index - Property-Based Tests', () => {
  it(
    'MI is between 0 and 100',
    fc.property(fc.string().filter(isValidCode), code => {
      const result = analysis.execute(createSourceFile(code));
      expect(result.data.maintainabilityIndex).toBeGreaterThanOrEqual(0);
      expect(result.data.maintainabilityIndex).toBeLessThanOrEqual(100);
    })
  );

  it('MI decreases with increasing complexity');
  it('MI decreases with increasing volume');
  it('MI decreases with increasing LOC');
});
```

### Code Generators

```typescript
// Generate valid TypeScript code
const validCodeArbitrary = fc.oneof(
  fc.constant('const x = 1;'),
  fc.constant('function f() {}'),
  fc.constant('if (true) {}'),
  fc.string().map(s => `const x = "${s}";`)
  // Add more generators for complex code
);

function isValidCode(code: string): boolean {
  try {
    createSourceFile(code);
    return true;
  } catch {
    return false;
  }
}
```

## Acceptance Criteria

- [ ] fast-check installed and configured
- [ ] 10+ property-based tests
- [ ] Halstead invariants validated
- [ ] Complexity invariants validated
- [ ] Maintainability Index invariants validated
- [ ] All properties pass with 100+ generated examples
- [ ] No counterexamples found
- [ ] All existing tests pass

## Testing Commands

```bash
cd analyzers/core
pnpm add -D fast-check
pnpm test -t "Property-Based"
pnpm test
```

## CLI Integration

No CLI changes required.

## Dependencies

**Requires Phases 1-3** to be complete (all metrics implemented).

## Risks & Mitigations

| Risk                    | Impact | Mitigation                             |
| ----------------------- | ------ | -------------------------------------- |
| Slow test execution     | Medium | Limit number of examples, use filters  |
| Hard to debug failures  | Medium | Use fc.sample() to reproduce failures  |
| Invalid code generation | Low    | Use robust code generators and filters |

## Definition of Done

- [ ] fast-check configured
- [ ] 10+ property tests passing
- [ ] Invariants validated across 100+ examples
- [ ] No counterexamples found
- [ ] All existing tests pass
- [ ] Phase approved by user

## Next Steps After Completion

All phases complete! Ready for production deployment.
