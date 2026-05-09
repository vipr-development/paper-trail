# Metric Normalization Data Drift Fix

## Problem Statement

The code-complexity-analyzer agent identified critical data drift between VSCode and Desktop clients in how cyclomatic complexity metrics are normalized and displayed to users.

### Original Issue

**VSCode Extension** used hardcoded formula:

```typescript
complexity: Math.max(0, Math.min(100, 100 - rawComplexity * 2));
```

**Desktop Client** used range-based normalization with **incorrect** `invert: false`:

```typescript
'core-cyclomatic': {
  metricId: 'core-cyclomatic',
  label: 'Cyclomatic Complexity',
  range: { min: 1, max: 50 },
  invert: false, // ❌ WRONG - causes inverted display
  unit: 'paths',
},
```

### Impact of Data Drift

For a file with cyclomatic complexity of 1 (simple, good code):

- **VSCode showed**: 98% (excellent quality) ✓ conceptually correct direction
- **Desktop showed**: 0% (poor quality) ✗ completely wrong

For a file with cyclomatic complexity of 50 (complex, problematic code):

- **VSCode showed**: 0% (poor quality) ✓ conceptually correct direction
- **Desktop showed**: 100% (excellent quality) ✗ completely wrong

The same code quality appeared radically different between clients, undermining user trust.

## Root Cause Analysis

### Semantic Confusion: `invert` Parameter

The `invert` parameter controls how raw metric values map to quality scores:

- `invert: false` = "higher raw value → higher quality score"
- `invert: true` = "higher raw value → lower quality score"

For **cyclomatic complexity**:

- Raw value semantics: **higher complexity = worse code**
- Display semantics: **higher score should = better quality**
- Therefore: `invert: true` is required

### Why Desktop Was Wrong

Desktop config had `invert: false` with comment "Higher complexity is worse":

```typescript
invert: false, // Higher complexity is worse ❌ Comment is true, but invert value is wrong
```

This caused:

- Complexity 1 → Normalized 0% → Displayed as poor quality ✗
- Complexity 50 → Normalized 100% → Displayed as excellent quality ✗

The comment correctly identified that "higher complexity is worse," but the `invert` value was set backwards.

### Why VSCode Was Partially Correct

VSCode's hardcoded formula `100 - rawComplexity * 2` was conceptually correct (inverts the value), but had issues:

1. ❌ **Hardcoded** - violates architecture principle of using metadata
2. ❌ **Naive linear mapping** - doesn't use proper min/max range
3. ❌ **Assumes max=50** - not configurable
4. ✓ **Correct direction** - inverts the value for quality display

## Solution Implemented

### 1. Fixed Desktop Configuration

Updated `metricNormalizationConfig.ts`:

```typescript
'core-cyclomatic': {
  metricId: 'core-cyclomatic',
  label: 'Cyclomatic Complexity',
  range: { min: 1, max: 50 },
  invert: true, // ✅ FIXED - higher complexity = worse quality
  unit: 'paths',
  description: 'Number of linearly independent paths through code',
},
```

### 2. Updated VSCode to Use Range-Based Normalization

Replaced hardcoded formula in `dashboard-state-builder.ts`:

**Before**:

```typescript
complexity: Math.max(0, Math.min(100, 100 - rawComplexity * 2)),
```

**After**:

```typescript
// Added helper function
function normalizeMetric(value: number, min: number, max: number, invert: boolean): number {
  const clamped = Math.max(min, Math.min(max, value));
  const rangeSize = max - min;
  let normalized = rangeSize > 0 ? ((clamped - min) / rangeSize) * 100 : 50;

  if (invert) {
    normalized = 100 - normalized;
  }

  return Math.max(0, Math.min(100, normalized));
}

// Usage
complexity: normalizeMetric(rawComplexity, 1, 50, true),
```

### 3. Updated Tests

Updated 3 failing Desktop tests to reflect corrected behavior:

```typescript
// Test: should normalize cyclomatic complexity (1-50 range)
expect(normalizeComplexity(1, 'cyclomatic')).toBe(100); // Best quality
expect(normalizeComplexity(25.5, 'cyclomatic')).toBe(50); // Medium quality
expect(normalizeComplexity(50, 'cyclomatic')).toBe(0); // Worst quality

// Test: should clamp values to valid range
expect(normalizeComplexity(-10, 'cyclomatic')).toBe(100); // Clamped to 1, inverted
expect(normalizeComplexity(1000, 'cyclomatic')).toBe(0); // Clamped to 50, inverted

// Test: should return cyclomatic when requested
// (25 - 1) / (50 - 1) * 100 = ~48.98, then inverted: 100 - 48.98 = ~51.02
expect(score).toBeCloseTo(51.02, 1);
```

## Results

### Consistency Achieved

Both clients now show **identical** quality scores for the same complexity values:

| Cyclomatic Complexity | VSCode Score | Desktop Score | Interpretation |
| --------------------- | ------------ | ------------- | -------------- |
| 1 (simple)            | 100%         | 100%          | Excellent ✓    |
| 10                    | 81.6%        | 81.6%         | Good ✓         |
| 25                    | 51.0%        | 51.0%         | Moderate ✓     |
| 50 (complex)          | 0%           | 0%            | Poor ✓         |

### Test Results

- ✅ All 45 Desktop churnCalculationService tests pass
- ✅ VSCode extension typechecks successfully
- ✅ No regressions in unrelated tests

## Future Work

### Short-Term (Next PR)

Extract Desktop's `metricNormalizationConfig` to `@vipr/common`:

```typescript
// packages/common/src/config/metric-normalization.ts
export const METRIC_NORMALIZATION_CONFIGS: Record<string, MetricNormalizationConfig> = {
  'core-cyclomatic': {
    /* ... */
  },
  'core-maintainability': {
    /* ... */
  },
  // ... etc
};
```

Then import in both clients:

```typescript
import { normalizeMetric } from '@vipr/common/config/metric-normalization';
```

### Long-Term (Presenter Architecture)

Implement `getMetricMetadata()` on analyzer presenters:

```typescript
// analyzers/core/src/presenters/cyclomatic-presenter.ts
export class CyclomaticPresenter implements IReportPresenter {
  getMetricMetadata(): MetricNormalizationConfig {
    return {
      metricId: 'core-cyclomatic',
      label: 'Cyclomatic Complexity',
      range: { min: 1, max: 50 },
      invert: true,
      unit: 'paths',
      description: 'Number of linearly independent paths through code',
    };
  }
}
```

This eliminates hardcoded configs entirely and sources truth from the analyzers themselves.

## Files Changed

### Desktop Client

- `clients/desktop/src/main/config/metricNormalizationConfig.ts` - Fixed `invert: true` for cyclomatic
- `clients/desktop/src/main/services/churnCalculationService.test.ts` - Updated 3 test expectations

### VSCode Extension

- `clients/vscode-extension/src/views/dashboard/dashboard-state-builder.ts` - Added `normalizeMetric()` helper and replaced hardcoded formula

## Lessons Learned

1. **Comments can lie** - The Desktop config comment said "higher complexity is worse" but the code did the opposite
2. **Test what you mean** - Tests should validate business intent (quality display), not just formula mechanics
3. **Data drift is insidious** - Users experienced radically different views without anyone noticing
4. **Architecture matters** - Hardcoded values enabled this drift; metadata-driven approach prevents it

## Related Issues

- Code-complexity-analyzer agent audit (session 9f7d5b02-fe71-458a-92b8-ec3d96f72d5a)
- Churn-Complexity Quadrant Analysis remediation plan (Priority 1, Task 1.1)
