# Phase 6: Edge Cases - Large Files & Malformed Code

**Priority:** P2 (Important)
**Estimated Effort:** 10 hours
**Dependencies:** Phases 1-3 (Algorithm implementations)
**Status:** Planning

## Overview

Test analyzer robustness with large files (10k+ lines) and malformed code. Current tests use mostly small, valid code samples.

## Agent Assignments

### @vitest-engineer

- Write large file tests (performance)
- Write malformed code tests (graceful failure)
- Write incomplete code tests
- Write memory leak tests
- **Minimum:** 15 new tests

### @typescript-engineer

- Optimize for large files if needed
- Ensure graceful handling of syntax errors
- Add error boundaries where needed
- Fix any crashes found

## Key Test Areas

### Large File Tests

```typescript
describe('Large File Handling', () => {
  it('should handle file with 10,000 lines', () => {
    const code = generateCode(10000);
    const result = analysis.execute(createSourceFile(code));
    // Should complete in < 5 seconds
    // Metrics should be accurate
  });

  it('should handle file with deeply nested code (50 levels)');
  it('should handle file with 1000+ functions');
  it('should not leak memory on repeated large file analysis');
});
```

### Malformed Code Tests

```typescript
describe('Malformed Code', () => {
  it('should handle syntax errors gracefully', () => {
    const code = `function broken() { if (x { }`;
    expect(() => analysis.execute(createSourceFile(code))).not.toThrow();
  });

  it('should handle incomplete expressions');
  it('should handle mismatched braces');
  it('should handle invalid operators');
  it('should return reasonable metrics for invalid code');
});
```

### Performance Tests

```typescript
describe('Performance', () => {
  it('should analyze 1000 LOC in < 100ms');
  it('should analyze 10000 LOC in < 1s');
  it('should maintain O(n) complexity');
});
```

## Acceptance Criteria

- [ ] 15+ edge case tests
- [ ] Large file performance validated (10k lines < 5s)
- [ ] Malformed code handled gracefully (no crashes)
- [ ] Memory usage stays reasonable
- [ ] Performance benchmarks established
- [ ] All existing tests pass

## Testing Commands

```bash
cd analyzers/core
pnpm test -t "Large File"
pnpm test -t "Malformed"
pnpm test -t "Performance"
pnpm test --coverage
```

## CLI Integration

Test CLI with large files:

```bash
cd clients/cli
pnpm build
./dist/index.js analyze <large-file.ts> --format json
```

## Definition of Done

- [ ] Tests implemented and passing
- [ ] Large files handled efficiently
- [ ] Malformed code handled gracefully
- [ ] Performance benchmarks met
- [ ] Phase approved by user
