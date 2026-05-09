# Phase 7: Edge Cases - Unicode & Modern JavaScript

**Priority:** P2 (Important)
**Estimated Effort:** 10 hours
**Dependencies:** Phases 1-3
**Status:** Planning

## Overview

Test analyzer handling of Unicode identifiers, emojis, and modern JavaScript features (async generators, private fields, top-level await, decorators).

## Agent Assignments

### @vitest-engineer

- Write Unicode handling tests
- Write modern JavaScript feature tests
- Test edge cases in operator/operand counting
- **Minimum:** 15 new tests

### @typescript-engineer

- Fix Unicode handling issues if found
- Add support for missing modern features
- Update ts-morph version if needed

## Key Test Areas

### Unicode Tests

```typescript
describe('Unicode Handling', () => {
  it('should handle unicode identifiers', () => {
    const code = `const 日本語 = '値';`;
    const result = analysis.execute(createSourceFile(code));
    expect(result.data.uniqOperands).toBeGreaterThan(0);
  });

  it('should handle emojis in strings');
  it('should handle right-to-left text');
  it('should handle zero-width characters');
  it('should handle surrogate pairs correctly');
});
```

### Modern JavaScript Tests

```typescript
describe('Modern JavaScript Features', () => {
  it('should handle async generators', () => {
    const code = `
      async function* gen() {
        yield await Promise.resolve(1);
      }
    `;
    // Should analyze without error
  });

  it('should handle private fields', () => {
    const code = `
      class Example {
        #privateField = 0;
        get value() { return this.#privateField; }
      }
    `;
    // Should count # as operator
  });

  it('should handle top-level await');
  it('should handle import assertions');
  it('should handle decorators (experimental)');
  it('should handle nullish assignment (??=)');
  it('should handle logical assignment (&&=, ||=)');
});
```

### Operator Classification Edge Cases

```typescript
describe('Edge Case Operators', () => {
  it('should classify BigInt correctly', () => {
    const code = `const big = 100n;`;
    // n suffix is part of literal or operator?
  });

  it('should classify numeric separators', () => {
    const code = `const num = 1_000_000;`;
    // _ is ignored in literals
  });

  it('should handle optional catch binding', () => {
    const code = `try {} catch {}`;
    // Catch without parameter
  });
});
```

## Acceptance Criteria

- [ ] 15+ Unicode and modern feature tests
- [ ] Unicode identifiers handled correctly
- [ ] All modern JavaScript features supported
- [ ] Edge case operators classified correctly
- [ ] All existing tests pass

## Testing Commands

```bash
cd analyzers/core
pnpm test -t "Unicode"
pnpm test -t "Modern JavaScript"
pnpm test
```

## CLI Integration

Test CLI with Unicode and modern features:

```bash
cd clients/cli
echo "const 日本語 = async function* gen() { yield 1; };" > /tmp/test.ts
./dist/index.js analyze /tmp/test.ts
```

## Definition of Done

- [ ] Tests implemented and passing
- [ ] Unicode handled correctly
- [ ] Modern features supported
- [ ] Edge cases covered
- [ ] Phase approved by user
