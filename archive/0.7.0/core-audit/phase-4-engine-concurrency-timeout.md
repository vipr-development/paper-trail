# Phase 4: Analysis Engine - Concurrency & Timeout

**Priority:** P1 (Critical)
**Estimated Effort:** 12 hours
**Dependencies:** None
**Status:** Planning

## Overview

Comprehensively test the Analysis Engine's concurrency limiting and timeout handling. The engine has these features implemented but they are not tested, making them unreliable in production.

## Background

### Analysis Engine Critical Features

**Location:** `analyzers/core/src/engine/analysis-engine.ts`

**Implemented but Untested:**

1. **Concurrency Limiting** (line ~324)
   - Limits number of analyses running simultaneously
   - Prevents resource exhaustion
   - Configuration: `maxConcurrentAnalyses`

2. **Timeout Handling** (line ~361)
   - Prevents analyses from running indefinitely
   - Configuration: `analysisTimeout`
   - Should gracefully handle timeout and continue with other analyses

**Current Test Coverage:** Only 7 integration tests

- ✅ Basic plugin execution
- ❌ No concurrency tests
- ❌ No timeout tests
- ❌ No resource exhaustion tests
- ❌ No stress tests

## Agent Assignments

### @vitest-engineer

**Responsibilities:**

1. Write comprehensive concurrency tests
2. Write timeout handling tests
3. Test error conditions and edge cases
4. Ensure tests are deterministic (not flaky)
5. Follow project conventions (peer tests)

**Deliverables:**

- Enhanced `analyzers/core/src/engine/analysis-engine.integration.test.ts` with:
  - Concurrency limiting tests (10+ tests)
  - Timeout handling tests (10+ tests)
  - Edge case tests (5+ tests)

**Minimum:** 25 new tests

### @typescript-engineer

**Responsibilities:**

1. Review concurrency implementation for correctness
2. Review timeout implementation for correctness
3. Fix any race conditions or bugs found
4. Ensure proper cleanup on timeout
5. Add documentation comments
6. Consider performance optimizations if needed

**Deliverables:**

- Updated `analyzers/core/src/engine/analysis-engine.ts` (if bugs found)
- Enhanced documentation comments
- Fix for any race conditions

## Implementation Details

### Test Suite Structure

```typescript
// analyzers/core/src/engine/analysis-engine.integration.test.ts

import { describe, it, expect, vi } from 'vitest';
import { AnalysisEngine } from './analysis-engine';
import { IAnalysis, AnalysisResult } from '@vipr/common/types';

describe('AnalysisEngine - Concurrency Control', () => {
  describe('maxConcurrentAnalyses', () => {
    it('should respect maxConcurrentAnalyses limit', async () => {
      let currentConcurrent = 0;
      let maxConcurrentReached = 0;

      const slowAnalysis: IAnalysis = {
        id: 'slow-test',
        name: 'Slow Test',
        description: 'Test analysis',
        category: 'complexity',
        execute: async sourceFile => {
          currentConcurrent++;
          maxConcurrentReached = Math.max(maxConcurrentReached, currentConcurrent);

          // Simulate slow analysis
          await new Promise(resolve => setTimeout(resolve, 50));

          currentConcurrent--;

          return {
            analysisId: 'slow-test',
            category: 'complexity',
            data: {},
            insights: [],
            score: 50,
            executionTimeMs: 50,
          };
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: {
          maxConcurrentAnalyses: 3,
        },
      });

      // Create 10 slow analyses
      const analyses = Array(10).fill(slowAnalysis);

      // Execute all analyses
      await engine.executeAnalyses(analyses, createSourceFile(''), {});

      // Verify max concurrency was respected
      expect(maxConcurrentReached).toBeLessThanOrEqual(3);
      expect(maxConcurrentReached).toBeGreaterThan(0);
    });

    it('should handle maxConcurrentAnalyses = 1 (sequential)', async () => {
      let concurrentCount = 0;
      let maxConcurrent = 0;

      const analysis: IAnalysis = {
        id: 'test',
        name: 'Test',
        description: 'Test',
        category: 'complexity',
        execute: async () => {
          concurrentCount++;
          maxConcurrent = Math.max(maxConcurrent, concurrentCount);
          await new Promise(resolve => setTimeout(resolve, 10));
          concurrentCount--;
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { maxConcurrentAnalyses: 1 },
      });

      await engine.executeAnalyses(Array(5).fill(analysis), createSourceFile(''), {});

      // Should never exceed 1
      expect(maxConcurrent).toBe(1);
    });

    it('should handle maxConcurrentAnalyses = 0 (unlimited)', async () => {
      let maxConcurrent = 0;
      let currentConcurrent = 0;

      const analysis: IAnalysis = {
        id: 'test',
        name: 'Test',
        description: 'Test',
        category: 'complexity',
        execute: async () => {
          currentConcurrent++;
          maxConcurrent = Math.max(maxConcurrent, currentConcurrent);
          await new Promise(resolve => setTimeout(resolve, 20));
          currentConcurrent--;
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { maxConcurrentAnalyses: 0 }, // 0 = unlimited
      });

      await engine.executeAnalyses(Array(10).fill(analysis), createSourceFile(''), {});

      // Should run all concurrently
      expect(maxConcurrent).toBe(10);
    });

    it('should handle concurrency with failing analyses', async () => {
      let successCount = 0;
      let failureCount = 0;

      const mixedAnalyses = [
        createSuccessAnalysis(() => successCount++),
        createFailingAnalysis(() => failureCount++),
        createSuccessAnalysis(() => successCount++),
        createFailingAnalysis(() => failureCount++),
        createSuccessAnalysis(() => successCount++),
      ];

      const engine = new AnalysisEngine({
        analysisExecution: { maxConcurrentAnalyses: 2 },
      });

      const results = await engine.executeAnalyses(mixedAnalyses, createSourceFile(''), {});

      expect(successCount).toBe(3);
      expect(failureCount).toBe(2);
      expect(results.successful).toHaveLength(3);
      expect(results.failed).toHaveLength(2);
    });

    it('should maintain concurrency during mixed fast/slow analyses', async () => {
      const fastAnalysis: IAnalysis = {
        id: 'fast',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 1));
          return createMockResult();
        },
      };

      const slowAnalysis: IAnalysis = {
        id: 'slow',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 100));
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { maxConcurrentAnalyses: 3 },
      });

      const startTime = Date.now();

      // Mix of fast and slow
      const analyses = [slowAnalysis, fastAnalysis, fastAnalysis, slowAnalysis, fastAnalysis];

      await engine.executeAnalyses(analyses, createSourceFile(''), {});

      const duration = Date.now() - startTime;

      // Fast analyses should complete while slow ones are running
      // With concurrency=3, should complete faster than sequential
      expect(duration).toBeLessThan(200); // Sequential would be 300ms
    });
  });

  describe('Timeout Handling', () => {
    it('should timeout analyses that exceed analysisTimeout', async () => {
      const slowAnalysis: IAnalysis = {
        id: 'very-slow',
        name: 'Very Slow',
        description: 'Never completes',
        category: 'complexity',
        execute: async () => {
          // Simulate analysis that takes too long
          await new Promise(resolve => setTimeout(resolve, 10000)); // 10 seconds
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: {
          analysisTimeout: 100, // 100ms timeout
        },
      });

      const sourceFile = createSourceFile('const x = 1;');
      const context = {};

      const result = await engine.executeAnalysis(slowAnalysis, sourceFile, context);

      expect(result.error).toBeDefined();
      expect(result.error.error.message).toMatch(/timeout/i);
    });

    it('should continue with other analyses after timeout', async () => {
      const slowAnalysis: IAnalysis = {
        id: 'slow',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 10000));
          return createMockResult();
        },
      };

      const fastAnalysis: IAnalysis = {
        id: 'fast',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 10));
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { analysisTimeout: 100 },
      });

      const results = await engine.executeAnalyses(
        [slowAnalysis, fastAnalysis, slowAnalysis, fastAnalysis],
        createSourceFile(''),
        {}
      );

      expect(results.successful).toHaveLength(2); // Two fast analyses
      expect(results.failed).toHaveLength(2); // Two slow analyses timed out
    });

    it('should handle timeout = 0 (no timeout)', async () => {
      const analysis: IAnalysis = {
        id: 'normal',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 200));
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { analysisTimeout: 0 }, // No timeout
      });

      const result = await engine.executeAnalysis(analysis, createSourceFile(''), {});

      expect(result.success).toBeDefined();
      expect(result.error).toBeUndefined();
    });

    it('should clean up resources on timeout', async () => {
      let cleanupCalled = false;

      const analysisWithCleanup: IAnalysis = {
        id: 'cleanup-test',
        execute: async () => {
          try {
            await new Promise(resolve => setTimeout(resolve, 10000));
            return createMockResult();
          } finally {
            cleanupCalled = true;
          }
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { analysisTimeout: 50 },
      });

      await engine.executeAnalysis(analysisWithCleanup, createSourceFile(''), {});

      // Allow time for cleanup
      await new Promise(resolve => setTimeout(resolve, 100));

      // Cleanup should have been called
      // Note: This depends on implementation details
    });

    it('should include timeout duration in error message', async () => {
      const slowAnalysis: IAnalysis = {
        id: 'slow',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 10000));
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { analysisTimeout: 500 },
      });

      const result = await engine.executeAnalysis(slowAnalysis, createSourceFile(''), {});

      expect(result.error?.error.message).toMatch(/500/); // Timeout duration mentioned
    });

    it('should handle analyses that complete just before timeout', async () => {
      const justInTimeAnalysis: IAnalysis = {
        id: 'just-in-time',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 90)); // Just under 100ms
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: { analysisTimeout: 100 },
      });

      const result = await engine.executeAnalysis(justInTimeAnalysis, createSourceFile(''), {});

      expect(result.success).toBeDefined();
      expect(result.error).toBeUndefined();
    });
  });

  describe('Combined Concurrency & Timeout', () => {
    it('should handle concurrency with timeouts', async () => {
      const fastAnalysis: IAnalysis = {
        id: 'fast',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 10));
          return createMockResult();
        },
      };

      const slowAnalysis: IAnalysis = {
        id: 'slow',
        execute: async () => {
          await new Promise(resolve => setTimeout(resolve, 10000));
          return createMockResult();
        },
      };

      const engine = new AnalysisEngine({
        analysisExecution: {
          maxConcurrentAnalyses: 3,
          analysisTimeout: 100,
        },
      });

      const analyses = [fastAnalysis, slowAnalysis, fastAnalysis, slowAnalysis, fastAnalysis];

      const results = await engine.executeAnalyses(analyses, createSourceFile(''), {});

      expect(results.successful).toHaveLength(3); // Fast analyses
      expect(results.failed).toHaveLength(2); // Slow analyses timed out
    });

    it('should not exceed concurrency limit even with timeouts', async () => {
      let currentConcurrent = 0;
      let maxConcurrent = 0;

      const createTrackedAnalysis = (duration: number): IAnalysis => ({
        id: `tracked-${duration}`,
        execute: async () => {
          currentConcurrent++;
          maxConcurrent = Math.max(maxConcurrent, currentConcurrent);
          await new Promise(resolve => setTimeout(resolve, duration));
          currentConcurrent--;
          return createMockResult();
        },
      });

      const engine = new AnalysisEngine({
        analysisExecution: {
          maxConcurrentAnalyses: 2,
          analysisTimeout: 50,
        },
      });

      const analyses = [
        createTrackedAnalysis(10), // Fast
        createTrackedAnalysis(1000), // Will timeout
        createTrackedAnalysis(10), // Fast
        createTrackedAnalysis(1000), // Will timeout
        createTrackedAnalysis(10), // Fast
      ];

      await engine.executeAnalyses(analyses, createSourceFile(''), {});

      expect(maxConcurrent).toBeLessThanOrEqual(2);
    });
  });

  describe('Edge Cases', () => {
    it('should handle empty analyses array', async () => {
      const engine = new AnalysisEngine();
      const results = await engine.executeAnalyses([], createSourceFile(''), {});

      expect(results.successful).toHaveLength(0);
      expect(results.failed).toHaveLength(0);
    });

    it('should handle single analysis', async () => {
      const engine = new AnalysisEngine({
        analysisExecution: { maxConcurrentAnalyses: 10 },
      });

      const result = await engine.executeAnalyses(
        [createSuccessAnalysis()],
        createSourceFile(''),
        {}
      );

      expect(result.successful).toHaveLength(1);
    });

    it('should handle analyses that throw synchronously', async () => {
      const throwingAnalysis: IAnalysis = {
        id: 'throwing',
        execute: () => {
          throw new Error('Synchronous error');
        },
      };

      const engine = new AnalysisEngine();
      const result = await engine.executeAnalysis(throwingAnalysis, createSourceFile(''), {});

      expect(result.error).toBeDefined();
      expect(result.error.error.message).toContain('Synchronous error');
    });

    it('should handle analyses that reject promises', async () => {
      const rejectingAnalysis: IAnalysis = {
        id: 'rejecting',
        execute: async () => {
          throw new Error('Async rejection');
        },
      };

      const engine = new AnalysisEngine();
      const result = await engine.executeAnalysis(rejectingAnalysis, createSourceFile(''), {});

      expect(result.error).toBeDefined();
      expect(result.error.error.message).toContain('Async rejection');
    });
  });
});

// Helper functions
function createMockResult(): AnalysisResult<any> {
  return {
    analysisId: 'mock',
    category: 'complexity',
    data: {},
    insights: [],
    score: 50,
    executionTimeMs: 0,
  };
}

function createSuccessAnalysis(onExecute?: () => void): IAnalysis {
  return {
    id: 'success',
    name: 'Success',
    description: 'Always succeeds',
    category: 'complexity',
    execute: async () => {
      onExecute?.();
      await new Promise(resolve => setTimeout(resolve, 10));
      return createMockResult();
    },
  };
}

function createFailingAnalysis(onExecute?: () => void): IAnalysis {
  return {
    id: 'failure',
    name: 'Failure',
    description: 'Always fails',
    category: 'complexity',
    execute: async () => {
      onExecute?.();
      await new Promise(resolve => setTimeout(resolve, 10));
      throw new Error('Analysis failed');
    },
  };
}
```

## Acceptance Criteria

### Must Have

- [ ] 25+ new tests for concurrency and timeout
- [ ] Concurrency limiting validated:
  - [ ] Respects maxConcurrentAnalyses
  - [ ] Handles unlimited (0)
  - [ ] Handles sequential (1)
  - [ ] Works with failures
  - [ ] Works with mixed fast/slow
- [ ] Timeout handling validated:
  - [ ] Times out long-running analyses
  - [ ] Continues with other analyses after timeout
  - [ ] Handles no timeout (0)
  - [ ] Cleans up resources
  - [ ] Includes timeout in error message
- [ ] Edge cases tested
- [ ] All existing tests still pass
- [ ] Tests are deterministic (not flaky)

### Should Have

- [ ] Performance benchmarks
- [ ] Stress tests (100+ analyses)
- [ ] Memory leak tests

### Nice to Have

- [ ] Visualization of concurrency in test output
- [ ] Detailed timing analysis

## Testing Commands

```bash
# Run engine tests
cd analyzers/core
pnpm test analysis-engine

# Run only new concurrency/timeout tests
pnpm test analysis-engine.integration.test.ts -t "Concurrency"
pnpm test analysis-engine.integration.test.ts -t "Timeout"

# Run all tests
pnpm test

# Coverage
pnpm test --coverage
```

## CLI Integration

No CLI changes required. This phase hardens existing engine functionality.

## Dependencies

**None** - Can run in parallel with Phases 1, 2, and 3.

## Risks & Mitigations

| Risk                              | Impact | Mitigation                                   |
| --------------------------------- | ------ | -------------------------------------------- |
| Flaky tests due to timing         | High   | Use sufficient delays, deterministic mocking |
| Race conditions in implementation | High   | Careful review by typescript-engineer        |
| Tests too slow                    | Medium | Use minimal delays, run concurrently         |
| Platform-specific timing issues   | Low    | Use relative timing, not absolute            |

## Definition of Done

- [ ] 25+ new tests passing
- [ ] Concurrency limiting validated
- [ ] Timeout handling validated
- [ ] Edge cases tested
- [ ] No flaky tests
- [ ] All existing tests pass
- [ ] Any bugs found are fixed
- [ ] Phase approved by user

## Next Steps After Completion

1. User reviews findings
2. Address any issues
3. Proceed to Phase 5: Caching & Error Recovery
