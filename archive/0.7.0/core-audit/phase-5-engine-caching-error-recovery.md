# Phase 5: Analysis Engine - Caching & Error Recovery

**Priority:** P1 (Critical)
**Estimated Effort:** 12 hours
**Dependencies:** Phase 4 (Concurrency & Timeout)
**Status:** Planning

## Overview

Test the Analysis Engine's caching logic and error recovery mechanisms. The engine implements file-based caching with TTL and invalidation, but these features are not tested.

## Agent Assignments

### @vitest-engineer

- Write cache invalidation tests (file modification, TTL expiration)
- Write cache key collision tests
- Write error recovery and retry tests
- Test cache statistics and monitoring
- **Minimum:** 20 new tests

### @typescript-engineer

- Review caching implementation for race conditions
- Review error recovery logic
- Fix any bugs found
- Ensure proper cache cleanup
- Add documentation

## Key Test Areas

### Cache Invalidation Tests

```typescript
describe('Cache Invalidation', () => {
  it('should invalidate cache when file modified');
  it('should invalidate cache after TTL');
  it('should invalidate cache on manual clear');
  it('should handle concurrent cache access');
  it('should detect file changes via mtime');
});
```

### Cache Key Tests

```typescript
describe('Cache Keys', () => {
  it('should generate deterministic keys');
  it('should handle hash collisions gracefully');
  it('should differentiate files with same content');
  it('should include analysis config in key');
});
```

### Error Recovery Tests

```typescript
describe('Error Recovery', () => {
  it('should continue after single analysis failure');
  it('should retry failed analyses when enabled');
  it('should respect maxRetries limit');
  it('should aggregate errors correctly');
  it('should not retry timeout errors');
});
```

## Acceptance Criteria

- [ ] 20+ cache and error recovery tests
- [ ] Cache invalidation on file modification validated
- [ ] Cache TTL expiration tested
- [ ] Hash collision handling tested
- [ ] Retry logic validated (max retries, selective retry)
- [ ] Error aggregation tested
- [ ] All existing tests pass
- [ ] No race conditions found

## Testing Commands

```bash
cd analyzers/core
pnpm test analysis-engine -t "Cache"
pnpm test analysis-engine -t "Error Recovery"
pnpm test
```

## CLI Integration

No CLI changes required.

## Definition of Done

- [ ] Tests implemented and passing
- [ ] Caching logic validated
- [ ] Error recovery validated
- [ ] Any bugs fixed
- [ ] Documentation updated
- [ ] Phase approved by user
