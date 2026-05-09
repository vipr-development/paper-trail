# Velocity Trends Demo Guide

Quick reference for demonstrating the Velocity Trends feature using mock data.

## Quick Start

The Velocity Trends page is fully demoable without real repository data. All API calls return realistic, parameter-aware mock data.

## Features Demonstrated

### 1. Main Repository Trend

**Location:** Top chart on Velocity Trends page

**What to Show:**

- Switch between metrics (Health Score, Complexity, Maintainability, etc.)
- Each metric shows a different trend direction
- Change time ranges to see data density adjust

**Demo Script:**

```
1. Select "Health Score" - Shows improving trend (58 → 72)
2. Select "Complexity Score" - Shows degrading complexity (improving health)
3. Change to "30 days" - Chart shows 16 data points
4. Change to "1 year" - Chart shows 53 data points (weekly)
```

### 2. File Leaderboard

**Location:** Right sidebar

**What to Show:**

- 20 files sorted by velocity
- Mix of improving, stable, and degrading files
- Different file types (components, utils, services, hooks)

**Top Performers:**

- `src/utils/formatters.ts` - Velocity: +3.2 (rapid improvement)
- `src/components/charts/LineChart.tsx` - Velocity: +2.8
- `src/hooks/useVelocityTrend.ts` - Velocity: +2.1

**Worst Performers:**

- `src/utils/legacy/data-processor.ts` - Velocity: -4.5 (rapid degradation)
- `src/components/widgets/ComplexWidget.tsx` - Velocity: -3.9
- `src/services/data/cache.service.ts` - Velocity: -3.4

### 3. Inflection Points

**Location:** Timeline below main chart

**What to Show:**

- 7 meaningful events that changed velocity
- Click on points to see event details
- Points are filtered by selected time range

**Notable Events:**

1. **Component Architecture Refactor** (12 days ago)
   - Changed slope from 0.8 to 2.9
   - Magnitude: 2.1

2. **Major Refactoring Initiative** (45 days ago)
   - Changed slope from -2.8 to 1.5
   - Magnitude: 4.3 (biggest impact)

3. **Feature Rush Before Release** (58 days ago)
   - Changed slope from 1.1 to -0.9
   - Negative impact event

### 4. Directory Trends

**Location:** Directory comparison section

**What to Show:**

- Compare multiple directories
- Each shows different trend pattern
- File counts vary (15-48 files)

**Default Directories:**

- `src/components` - Improving (48 files)
- `src/utils` - Degrading (22 files)
- `src/pages` - Stable (15 files)
- `src/services` - Volatile (31 files)
- `src/hooks` - Improving (48 files)
- `src/contexts` - Degrading (22 files)

### 5. Metric Breakdown

**Location:** Metric breakdown panel

**What to Show:**

- 7 different metrics tracked
- Each with its own velocity
- Categories: complexity, quality

**Metrics:**

1. **Cyclomatic Complexity** - Improving (-0.8/week)
2. **Maintainability Index** - Improving (+1.2/week)
3. **Halstead Bugs** - Improving (-0.05/week)
4. **Test Coverage** - Improving (+2.1/week) - Best improvement
5. **Dependency Count** - Stable (+0.3/week)
6. **Code Duplication** - Improving (-0.4/week)
7. **Type Coverage** - Improving (+1.5/week)

### 6. Multi-Metric Comparison

**Location:** Select multiple metrics for comparison

**What to Show:**

- Overlay 2-3 metrics on same chart
- Compare health vs complexity trends
- Different colors for each metric

**Demo Script:**

```
1. Select "Health Score" + "Maintainability Score"
2. Both show improving trends
3. Add "Complexity Score" to see inverse relationship
4. Complexity decreases while health increases (both good)
```

## Demo Scenarios

### Scenario 1: Show Overall Health Improvement

**Goal:** Demonstrate codebase is getting healthier

**Steps:**

1. Set metric to "Health Score"
2. Set timeRange to "90 days"
3. Point out steady improvement: 58 → 72
4. Highlight high confidence (R² = 0.84)
5. Show velocity: +2.5 points per week

### Scenario 2: Identify Problem Areas

**Goal:** Find files that need attention

**Steps:**

1. Keep "Health Score" selected
2. Scroll to bottom of leaderboard
3. Point out degrading files:
   - `data-processor.ts` (-4.5/week)
   - `ComplexWidget.tsx` (-3.9/week)
   - `cache.service.ts` (-3.4/week)
4. These are candidates for refactoring

### Scenario 3: Show Impact of Refactoring

**Goal:** Demonstrate refactoring initiative paid off

**Steps:**

1. Select "90 days" timeRange
2. Click inflection point at day 45
3. Show "Major Refactoring Initiative"
4. Before slope: -2.8 (degrading)
5. After slope: +1.5 (improving)
6. Magnitude: 4.3 point swing

### Scenario 4: Compare Directory Health

**Goal:** Show which parts of codebase are healthiest

**Steps:**

1. Select directories to compare
2. `src/components` - Improving trend
3. `src/utils` - Degrading trend
4. `src/pages` - Stable trend
5. Recommend focusing on `utils` directory

### Scenario 5: Track Multiple Metrics

**Goal:** Show comprehensive view of codebase health

**Steps:**

1. Open Metric Breakdown
2. Point out Test Coverage improving fastest (+2.1/week)
3. Type Coverage also strong (+1.5/week)
4. Maintainability improving (+1.2/week)
5. Overall positive momentum

## Time Range Behavior

| Range | Data Points | Interval | Use Case         |
| ----- | ----------- | -------- | ---------------- |
| 7d    | 8           | Daily    | Recent changes   |
| 30d   | 16          | 2 days   | Monthly trend    |
| 90d   | 31          | 3 days   | Quarterly view   |
| 1y    | 53          | Weekly   | Annual trend     |
| all   | 65          | 11 days  | Complete history |

## Metric Patterns

| Metric          | Pattern   | Direction | Meaning             |
| --------------- | --------- | --------- | ------------------- |
| Health Score    | Improving | ↗         | Getting better      |
| Complexity      | Degrading | ↘         | Reducing (good)     |
| Maintainability | Improving | ↗         | Getting better      |
| Issue Count     | Degrading | ↘         | Fewer issues (good) |
| Critical Count  | Stable    | →         | Consistent          |

## Tips for Effective Demos

1. **Start Broad:** Begin with "Health Score" + "90 days" to show overall trend
2. **Drill Down:** Use leaderboard to identify specific files
3. **Show Context:** Click inflection points to explain velocity changes
4. **Compare:** Use directory trends to show uneven distribution
5. **Metrics:** Show metric breakdown to validate single-metric trends

## Common Questions

**Q: Why is "Complexity Score" degrading?**
A: Degrading complexity is good - it means code is getting simpler. Lower complexity = better.

**Q: What's a good velocity value?**
A: For health/maintainability: Positive (`>0.5`) is good. For complexity/issues: Negative (`<-0.5`) is good.

**Q: What does R² (confidence) mean?**
A: Higher R² (closer to 1.0) means more consistent trend. Low R² means volatile/noisy data.

**Q: Why do some files have negative velocity?**
A: Negative velocity means the metric is decreasing. For health, that's bad. For complexity, that's good.

## Mock Data Characteristics

- **Realistic Noise:** All trends include +/- random variation
- **Varied Patterns:** Different metrics show different trend directions
- **Meaningful Events:** Inflection points have real-world event names
- **Diverse Files:** Leaderboard includes components, utils, services, hooks, pages
- **Proper Density:** Data point count matches time range expectations

## Troubleshooting

**Issue:** No data showing

- **Fix:** Ensure you're in mock mode (no real repository open)

**Issue:** Same data for all metrics

- **Fix:** Clear cache and restart app - mock data should be parameter-aware

**Issue:** Inflection points not visible

- **Fix:** Extend time range to "90 days" or "all"

**Issue:** Leaderboard only shows 2 files

- **Fix:** Update to latest mock data (should show 20 files)
