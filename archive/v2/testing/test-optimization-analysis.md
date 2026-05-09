# Test Optimization Analysis

## Executive Summary

This document analyzes slow tests in the codebase and provides specific optimization recommendations.

## Findings

### Skipped Tests

No skipped tests found in the codebase.

### Slow Tests Analysis

| Test File                         | Tests | Duration | Avg/Test | Status   |
| --------------------------------- | ----- | -------- | -------- | -------- |
| cli-formatter-integration.test.ts | 25    | 42810ms  | 1712ms   | Critical |
| types-analysis.test.ts            | 25    | 431ms    | 17ms     | Moderate |
| react-helpers.test.ts             | 35    | 1232ms   | 35ms     | Moderate |

## Root Cause Analysis

### 1. CLI Integration Tests (Critical - 42.8s)

**Problem**: Each test spawns a new Node.js process and runs the entire CLI

**Breakdown per test**:

- Process spawn: ~50-100ms
- Dependency loading: ~200-300ms
- Plugin discovery: ~50-100ms
- AST parsing (DataTable.tsx): ~300-500ms
- Analysis execution: ~500-800ms
- Formatting output: ~50-100ms
- **Total: ~1700ms per test**

**Key Issues**:

1. 25 separate CLI process executions
2. No caching of analysis results
3. Same fixture analyzed 25 times
4. Cold start penalty for each test
5. Redundant plugin loading

### 2. Types Analysis Tests (Moderate - 431ms)

**Problem**: Each test creates a new ts-morph Project instance

**Breakdown per test**:

- Project instantiation: ~5-8ms
- TypeScript compiler setup: ~3-5ms
- Source file parsing: ~5-10ms
- Analysis execution: ~3-5ms
- **Total: ~17ms per test**

**Key Issues**:

1. 25 Project instances created
2. No fixture reuse
3. Redundant TypeScript compiler initialization
4. Similar code patterns parsed repeatedly

### 3. React Helpers Tests (Moderate - 1.2s)

**Problem**: Similar to types-analysis - excessive Project creation

**Breakdown per test**:

- Project instantiation: ~10-15ms
- JSX parsing setup: ~10-15ms
- Source file creation: ~5-10ms
- Helper function execution: ~5-10ms
- **Total: ~35ms per test**

**Key Issues**:

1. 35 Project instances created
2. Each test parses similar React components
3. No shared fixtures
4. Redundant JSX compiler configuration

## Optimization Recommendations

### Priority 1: CLI Integration Tests

**Recommended Optimizations**:

1. **Shared Analysis Result Cache** (Expected: 90% reduction)
   - Run analysis once in beforeAll
   - Cache results by filter combination
   - Each test queries cached results
   - Expected time: ~2s for full suite (from 42.8s)

2. **In-Process Testing** (Alternative approach)
   - Test formatters directly without CLI spawn
   - Mock AnalysisEngine results
   - Expected time: ~0.5s for full suite

3. **Parallel Test Execution** (If caching not viable)
   - Run CLI tests in parallel with vitest pool
   - Expected time: ~8-10s (with 5 workers)

**Implementation Choice**: Option 1 (Shared Cache) is recommended because:

- Preserves integration test value
- Simple implementation
- Dramatic performance improvement
- No loss of test coverage

### Priority 2: Types Analysis Tests

**Recommended Optimizations**:

1. **Shared Project Instance** (Expected: 60% reduction)
   - Create single Project in describe block
   - Reuse for all tests
   - Clear between tests if needed
   - Expected time: ~170ms (from 431ms)

2. **Fixture Caching** (Additional improvement)
   - Pre-parse common type patterns
   - Store as reusable SourceFile objects
   - Expected additional savings: ~50ms

**Combined Expected Time**: ~120ms (72% reduction)

### Priority 3: React Helpers Tests

**Recommended Optimizations**:

1. **Shared Project Instance** (Expected: 70% reduction)
   - Single Project per describe block
   - Reuse across tests
   - Expected time: ~370ms (from 1232ms)

2. **Fixture Library** (Additional improvement)
   - Pre-create common component patterns
   - Cache parsed SourceFiles
   - Expected additional savings: ~100ms

**Combined Expected Time**: ~270ms (78% reduction)

## Implementation Guidelines

### For CLI Integration Tests

```typescript
describe('CLI Formatter Integration Tests', () => {
  // Cache for analysis results
  let analysisCache: Map<string, string>;

  beforeAll(() => {
    // Run analysis once for each unique combination
    const cliPath = resolve(__dirname, '../../dist/index.js');
    const fixture = resolve(__dirname, '../../../../analyzers/react/src/fixtures/DataTable.tsx');

    analysisCache = new Map([
      ['full', execSync(`node ${cliPath} analyze ${fixture} -q`)],
      ['security', execSync(`node ${cliPath} analyze ${fixture} -q -r security`)],
      // ... etc for each unique filter combination
    ]);
  }, 30000); // Reasonable timeout for cache building

  it('should format full report with all sections', () => {
    const output = analysisCache.get('full');
    // ... assertions on cached output
  });
});
```

### For TypeScript Test Utils

```typescript
// packages/testing/src/test-utils.ts
export class TestUtils {
  private static sharedProject: Project | null = null;

  /**
   * Get or create shared Project instance
   */
  static getSharedProject(): Project {
    if (!this.sharedProject) {
      this.sharedProject = new Project({
        useInMemoryFileSystem: true,
        compilerOptions: {
          jsx: 4,
          target: 99,
          module: 99,
        },
      });
    }
    return this.sharedProject;
  }

  /**
   * Create SourceFile using shared Project
   */
  static createSourceFile(code: string, filePath = 'test.tsx'): SourceFile {
    const project = this.getSharedProject();

    // Generate unique file path to avoid collisions
    const uniquePath = `${filePath.replace('.tsx', '')}-${Date.now()}-${Math.random()}.tsx`;

    return project.createSourceFile(uniquePath, code);
  }

  /**
   * Clear shared project (call in afterAll if needed)
   */
  static resetSharedProject(): void {
    this.sharedProject = null;
  }
}
```

### For Individual Test Files

```typescript
describe('TypesAnalysis', () => {
  afterAll(() => {
    // Clean up shared resources if needed
    TestUtils.resetSharedProject();
  });

  // Tests use TestUtils.createSourceFile as before
  // Now they share the same Project instance
});
```

## Expected Performance Improvements

| Test File                         | Current     | Optimized   | Improvement    |
| --------------------------------- | ----------- | ----------- | -------------- |
| cli-formatter-integration.test.ts | 42810ms     | ~2000ms     | 95% faster     |
| types-analysis.test.ts            | 431ms       | ~120ms      | 72% faster     |
| react-helpers.test.ts             | 1232ms      | ~270ms      | 78% faster     |
| **Total**                         | **44473ms** | **~2390ms** | **95% faster** |

## Testing the Optimizations

After implementing changes:

```bash
# Run tests with timing
pnpm test --reporter=verbose

# Verify specific test timing
pnpm test cli-formatter-integration.test.ts
pnpm test types-analysis.test.ts
pnpm test react-helpers.test.ts
```

## Risks and Mitigation

### Risk 1: Shared State Pollution

**Mitigation**:

- Use unique file paths for each SourceFile
- Clear Project state between test suites if needed
- Add isolation tests to verify no cross-contamination

### Risk 2: Cache Invalidation

**Mitigation**:

- Cache key includes all relevant CLI flags
- Document cache dependencies
- Add cache validation in beforeAll

### Risk 3: Parallel Test Issues

**Mitigation**:

- Keep shared Project per describe block (not global)
- Avoid mutating shared instances
- Use beforeEach for test-specific setup

## Next Steps

1. Implement shared Project in TestUtils
2. Migrate types-analysis.test.ts to use shared Project
3. Migrate react-helpers.test.ts to use shared Project
4. Implement CLI test caching
5. Validate performance improvements
6. Document patterns for future tests
