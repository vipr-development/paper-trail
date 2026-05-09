# Test Coverage Summary: React Analyzer

Quick reference guide for test coverage status and priorities.

## Coverage Status

| Component Type | Files  | Tested | Untested | Coverage |
| -------------- | ------ | ------ | -------- | -------- |
| Analyses       | 14     | 14     | 0        | 100%     |
| Utilities      | 2      | 2      | 0        | 100%     |
| Presenters     | 10     | 0      | 10       | 0%       |
| Constants      | 4      | 0      | 2        | 0%       |
| Plugin Core    | 1      | 1      | 0        | 100%     |
| Types          | 19     | 0      | 19       | N/A      |
| **Total**      | **50** | **17** | **31**   | **34%**  |

## Critical Findings

### Strengths

- All analysis modules have comprehensive tests
- All utility modules have comprehensive tests
- Tests follow project conventions (co-located, AAA pattern)
- Good fixture coverage for testing

### Critical Gaps

- **Zero presenter test coverage** (~2,000 lines untested)
- **No constants validation** (weights, thresholds untested)
- **Limited integration scenarios** (10+ patterns missing)
- **Missing edge case coverage** across all analyses

## Priority Action Items

### Phase 1: Critical (2 weeks)

1. Add presenter tests (8 files) - **16-20 hours**
2. Add constants validation (2 files) - **4-6 hours**

### Phase 2: High (2 weeks)

3. Expand integration tests - **8-12 hours**
4. Add edge case coverage - **12-16 hours**

### Phase 3: Medium (2 weeks)

5. Performance regression tests - **8-10 hours**
6. Type guard tests (optional) - **4-6 hours**

**Total Estimated Effort:** 52-70 hours

## Untested Files List

### Presenters (Priority: Critical)

- src/presenters/overview-presenter.ts
- src/presenters/performance-presenter.ts
- src/presenters/accessibility-presenter.ts
- src/presenters/migration-presenter.ts
- src/presenters/reliability-presenter.ts
- src/presenters/security-presenter.ts
- src/presenters/dataflow-presenter.ts
- src/presenters/anti-pattern-presenter.ts
- src/presenters/base-presenter.ts

### Constants (Priority: High)

- src/constants/weights.ts
- src/constants/thresholds.ts

### Integration Gaps (Priority: High)

Missing test scenarios:

- Class components
- Error boundaries
- Server components
- Suspense patterns
- Context providers
- Custom hooks
- HOC patterns
- Render props
- Compound components
- Portal usage
- Ref forwarding edge cases

## Quick Start Guide

### Adding Presenter Tests

```typescript
// src/presenters/[name]-presenter.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { [Name]Presenter } from './[name]-presenter';

describe('[Name]Presenter', () => {
  let presenter: [Name]Presenter;

  beforeEach(() => {
    presenter = new [Name]Presenter();
  });

  describe('metadata', () => {
    it('should have correct metadata', () => {
      expect(presenter.reportType).toBe('[report-type]');
      expect(presenter.pluginId).toBe('react');
    });
  });

  describe('canPresent', () => {
    it('should accept valid React plugin results', () => {
      // Test acceptance logic
    });

    it('should reject invalid results', () => {
      // Test rejection logic
    });
  });

  describe('present', () => {
    it('should create valid presentation', () => {
      // Test transformation logic
    });

    it('should handle missing data gracefully', () => {
      // Test error handling
    });
  });
});
```

### Adding Constants Tests

```typescript
// src/constants/weights.test.ts
import { describe, it, expect } from 'vitest';
import { COMPLEXITY_WEIGHTS } from './weights';

describe('COMPLEXITY_WEIGHTS', () => {
  it('should sum to 100%', () => {
    const sum = Object.values(COMPLEXITY_WEIGHTS).reduce((a, b) => a + b, 0);
    expect(sum).toBeCloseTo(1.0, 2);
  });

  it('should have non-negative weights', () => {
    Object.values(COMPLEXITY_WEIGHTS).forEach(weight => {
      expect(weight).toBeGreaterThanOrEqual(0);
    });
  });
});
```

### Adding Integration Tests

```typescript
// testing/react.integration.ts (expand existing file)
describe('React Analyzer - [Feature] Workflows', () => {
  it('should handle [scenario]', async () => {
    const code = `/* test code */`;
    const sourceFile = TestUtils.createSourceFile(code);
    const result = await plugin.analyze(sourceFile);

    expect(result).toBeDefined();
    // Add specific assertions
  });
});
```

## Testing Standards

### Test File Naming

- Co-locate tests: `file-name.ts` → `file-name.test.ts`
- Integration tests: `feature.integration.ts`
- Benchmark tests: `feature.benchmark.ts`

### Test Structure

- Use AAA pattern (Arrange-Act-Assert)
- Group with `describe` blocks
- Descriptive test names: "should [expected behavior]"
- One logical assertion per test

### Test Performance

- Target: < 100ms per test
- Total suite: < 30 seconds
- Use `beforeEach` for setup
- Clean up in `afterEach`

## Running Tests

```bash
# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Run specific test file
npm test -- performance-presenter.test.ts

# Run with coverage
npm test -- --coverage
```

## Success Criteria

- [ ] All presenters have test coverage
- [ ] All constants are validated
- [ ] Integration tests cover 15+ scenarios
- [ ] Edge cases tested in all analyses
- [ ] Performance benchmarks established
- [ ] Test suite runs in < 30s
- [ ] Zero flaky tests

## Next Steps

1. Review detailed analysis: `test-coverage-analysis-vipr.md`
2. Start with Phase 1 presenter tests
3. Add constants validation
4. Expand integration scenarios
5. Add edge case coverage
6. Establish performance baselines

## Resources

- Project conventions: `/CLAUDE.md`
- Testing utilities: `@vipr/testing`
- Fixtures directory: `src/fixtures/`
- Existing test examples: `src/analyses/*.test.ts`
