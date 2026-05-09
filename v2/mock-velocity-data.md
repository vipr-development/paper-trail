# Mock Velocity Data System

This document describes the comprehensive mock data system for the Velocity Trends feature, enabling full demo functionality without real repository data.

## Overview

The mock velocity system provides realistic, parameter-aware data for:

- Repository trends across 5 metrics and 5 time ranges
- File leaderboard with 20+ diverse entries
- Directory trends with varied patterns
- Inflection points with meaningful events
- Metric breakdown across multiple categories
- Multi-metric comparisons

## Architecture

### Files

- `/src/renderer/mocks/fixtures/velocity.fixtures.ts` - Mock data generation and fixtures
- `/src/renderer/mocks/mock-ipc-api.ts` - Parameter-aware mock API implementation
- `/src/shared/ipc-types.ts` - Type definitions for all velocity data structures

### Key Functions

#### `getVelocityTrend(metric: string, timeRange: TimeRange)`

Returns trend data for a specific metric and time range. Dynamically generates appropriate data point density based on the time range.

**Metrics:**

- `health_score` - Overall health (improving pattern)
- `complexity_score` - Code complexity (degrading pattern)
- `maintainability_score` - Maintainability index (improving pattern)
- `issue_count` - Issue count (degrading/improving)
- `critical_count` - Critical issues (stable pattern)

**Time Ranges and Data Density:**

- `7d` - 8 data points (daily snapshots)
- `30d` - 16 data points (every 2 days)
- `90d` - 31 data points (every 3 days)
- `1y` - 53 data points (weekly)
- `all` - 65 data points (varying intervals over 2 years)

**Example:**

```typescript
const { velocity, trendData } = getVelocityTrend('health_score', '30d');
// Returns 16 data points showing improving health trend
```

#### `getDirectoryTrends(directories: string[], metric: string, timeRange: TimeRange)`

Returns velocity trends for multiple directories. Each directory has a different pattern:

- Index 0 (mod 4): Improving trend
- Index 1 (mod 4): Degrading trend
- Index 2 (mod 4): Stable trend
- Index 3 (mod 4): Volatile trend

**Example:**

```typescript
const trends = getDirectoryTrends(
  ['src/components', 'src/utils', 'src/pages'],
  'health_score',
  '90d'
);
// Returns 3 directory trends with different patterns
```

#### `getMultiMetricTrend(metrics: string[], timeRange: TimeRange)`

Returns trend data for multiple metrics at once, enabling multi-metric chart comparisons.

**Example:**

```typescript
const multiData = getMultiMetricTrend(['health_score', 'maintainability_score'], '30d');
// Returns { health_score: [...], maintainability_score: [...] }
```

## Mock API Behavior

### `velocity.getRepoTrend(payload)`

**Input:** `{ metric: string, timeRange: TimeRange }`

**Behavior:** Returns trend data based on the specific metric and time range requested. Different metrics show different trend directions:

- Health/Maintainability: Improving trends
- Complexity: Degrading trend (getting better by reducing)
- Critical Count: Stable trend
- Issue Count: Degrading count (improving health)

### `velocity.getLeaderboard(payload)`

**Input:** `{ metric: string, timeRange: TimeRange, limit?: number }`

**Behavior:**

- Returns 20 diverse files (or `limit` if specified)
- Sorted based on metric type:
  - Health/Maintainability: Most improving first (highest velocity)
  - Issue/Critical counts: Most degrading first (fastest degradation)
  - Default: Absolute velocity magnitude

**Leaderboard includes:**

- 5 rapidly improving files (velocity > 1.4)
- 4 stable improving files (0.5 < velocity < 1.4)
- 5 stable files (-0.4 < velocity < 0.4)
- 6 degrading files (velocity < -0.8)

### `velocity.getInflectionPoints(payload)`

**Input:** `{ metric: string, timeRange: TimeRange }`

**Behavior:**

- Returns 7 inflection points
- Filters by time range (only returns points within range)
- Each point has associated event metadata in `inflectionPointEvents`

**Events include:**

- Component Architecture Refactor
- TypeScript Strict Mode Enabled
- Tech Debt Sprint
- Major Refactoring Initiative
- Feature Rush Before Release
- Dependency Update Cascade
- Testing Framework Migration

### `velocity.getDirectoryTrends(payload)`

**Input:** `{ directories: string[], metric: string, timeRange: TimeRange }`

**Behavior:**

- If `directories` array is empty, uses default list: `src/components`, `src/utils`, `src/pages`, `src/services`, `src/hooks`, `src/contexts`
- Each directory gets a different pattern based on its index
- Returns trends with varied file counts (15-48 files per directory)

### `velocity.getMetricBreakdown(payload)`

**Input:** `{ timeRange: TimeRange }`

**Behavior:** Returns breakdown of 7 metrics:

- Cyclomatic Complexity (improving)
- Maintainability Index (improving)
- Halstead Bugs (improving)
- Test Coverage (improving)
- Dependency Count (stable)
- Code Duplication (improving)
- Type Coverage (improving)

## Trend Patterns

The system uses four trend patterns to simulate realistic velocity data:

### 1. Improving

Smooth upward trend with slight random noise (+/- 2 points). High R² (0.75-0.90).

### 2. Degrading

Smooth downward trend with slight random noise (+/- 2 points). High R² (0.75-0.90).

### 3. Stable

Oscillates around base value with minimal change (+/- 3 points). Very high R² (0.92-0.97).

### 4. Volatile

Large swings with overall trend direction (+/- 8 points). Low R² (0.45-0.65).

## Data Structures

### VelocityTrendPoint

```typescript
{
  timestamp: number; // Unix timestamp
  value: number; // Metric value (0-100)
  commitSha: string; // Generated commit SHA
}
```

### VelocityMetrics

```typescript
{
  slope: number; // Rate of change per week
  intercept: number; // Y-intercept of regression
  r2: number; // Confidence (0-1)
  currentValue: number; // Latest value
  trend: 'improving' | 'degrading' | 'stable';
  confidence: number; // Same as r2
}
```

### VelocityLeaderboardEntry

```typescript
{
  filePath: string;
  fileType: string | null;
  technologies: string[] | null;
  velocity: number;       // Slope value
  currentScore: number;
  trend: 'improving' | 'degrading' | 'stable';
  confidence: number;
}
```

### InflectionPoint

```typescript
{
  timestamp: number;
  commitSha: string;
  beforeSlope: number; // Velocity before change
  afterSlope: number; // Velocity after change
  magnitude: number; // Absolute change in slope
  significance: number; // Statistical significance (0-1)
}
```

## Testing the System

### Switch Metrics

Change the primary metric dropdown and observe different trend directions:

```
Health Score → Improving trend
Complexity Score → Degrading trend (complexity going down = good)
Maintainability → Improving trend
Issue Count → Degrading count (fewer issues = good)
Critical Count → Stable trend
```

### Change Time Ranges

Switch time ranges and observe appropriate data density:

```
7d  → 8 points (daily)
30d → 16 points (every 2 days)
90d → 31 points (every 3 days)
1y  → 53 points (weekly)
all → 65 points (varying intervals)
```

### Leaderboard Sorting

The leaderboard should show:

- Top entries: Files improving rapidly (positive velocity)
- Middle: Stable files (near-zero velocity)
- Bottom: Files degrading rapidly (negative velocity)

### Inflection Points

Click on inflection points in charts to see meaningful event descriptions like refactoring initiatives, tech debt sprints, etc.

### Directory Comparisons

Select multiple directories to see varied trends (some improving, some degrading, some stable).

## Extension Points

### Adding New Metrics

1. Add metric configuration to `metricConfigs`:

```typescript
const metricConfigs: Record<string, MetricConfig> = {
  your_metric: { pattern: 'improving', baseValue: 50, targetValue: 75 },
  // ...
};
```

2. The mock API will automatically handle the new metric

### Adding More Inflection Points

1. Add to `mockInflectionPoints` array
2. Add event metadata to `inflectionPointEvents`:

```typescript
inflect008xyz: {
  title: 'Your Event Title',
  description: 'Detailed description of what happened',
}
```

### Custom Directory Patterns

Modify `getDirectoryTrends` to assign specific patterns to specific directories instead of using modulo-based assignment.

## Common Scenarios

### Demo: Show Improving Health

```typescript
// Use health_score metric with 30d or 90d range
// Shows steady improvement with high confidence
getVelocityTrend('health_score', '90d');
```

### Demo: Show Degrading Complexity

```typescript
// Use complexity_score to show complexity reduction
// Degrading complexity = improving health
getVelocityTrend('complexity_score', '90d');
```

### Demo: Compare Multiple Metrics

```typescript
// Show health improving while complexity degrades (both good)
getMultiMetricTrend(['health_score', 'complexity_score'], '30d');
```

### Demo: Identify Problem Areas

```typescript
// Leaderboard shows files with negative velocity
// Use metric: 'health_score' to find degrading files
getLeaderboard({ metric: 'health_score', timeRange: '90d', limit: 20 });
```

## Notes

- All timestamps are relative to `Date.now()`, ensuring data is always current
- Commit SHAs are deterministically generated from metric name and index
- Random noise is applied at generation time, not deterministic between calls
- R² values are pattern-specific to simulate realistic confidence levels
- The system supports extending to any metric by adding to `metricConfigs`
