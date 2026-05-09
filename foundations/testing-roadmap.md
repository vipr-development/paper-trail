# Testing Roadmap - Visual Guide

**Quick visual reference for testing priorities and dependencies**

## Priority Distribution

```mermaid
pie title Test Priority Distribution
    "P0 Critical" : 275
    "P1 High" : 160
    "P2 Medium" : 40
    "Benchmarks" : 11
```

## Test Coverage Gap

```mermaid
pie title Current vs Required Tests
    "Existing Tests" : 200
    "Missing Tests (P0)" : 275
    "Missing Tests (P1)" : 160
    "Missing Tests (P2)" : 40
```

## Sprint Timeline

```mermaid
gantt
    title Testing Roadmap (4-6 Weeks)
    dateFormat YYYY-MM-DD
    section Sprint 1
    temporal-analysis.test.ts (55 tests)     :s1t1, 2026-01-13, 3d
    coupling-analysis.test.ts (35 tests)     :s1t2, 2026-01-13, 2d
    identity-analysis.test.ts (40 tests)     :s1t3, 2026-01-15, 2d
    analysis-cache.test.ts (30 tests)        :s1t4, 2026-01-17, 2d
    analysis-engine additions (30 tests)     :s1t5, 2026-01-19, 2d
    Integration tests (15 tests)             :s1t6, 2026-01-21, 2d
    section Sprint 2
    react-helpers.test.ts (70 tests)         :s2t1, 2026-01-23, 4d
    ast-helpers.test.ts (45 tests)           :s2t2, 2026-01-27, 3d
    scoring.test.ts additions (15 tests)     :s2t3, 2026-01-30, 1d
    plugin.ts core (25 tests)                :s2t4, 2026-01-31, 2d
    accessibility-helpers.test.ts (30 tests) :s2t5, 2026-02-02, 2d
    section Sprint 3
    Verify existing tests                    :s3t1, 2026-02-04, 3d
    Performance benchmarks                   :s3t2, 2026-02-07, 2d
    Coverage review                          :s3t3, 2026-02-09, 2d
    Bug fixes                                :s3t4, 2026-02-11, 2d
```

## Complexity vs Test Coverage

```mermaid
graph TD
    A[Files by Complexity] --> B{Cyclomatic Complexity}
    B -->|35-50| C[Critical Priority]
    B -->|20-35| D[High Priority]
    B -->|10-20| E[Medium Priority]

    C --> C1[temporal-analysis 35<br/>NO TESTS]
    C --> C2[analysis-engine 50<br/>PARTIAL]
    C --> C3[dataflow-analysis 40<br/>HAS TESTS]

    D --> D1[react-helpers 50<br/>NO TESTS]
    D --> D2[hook-analysis 25<br/>HAS TESTS]
    D --> D3[coupling-analysis 18<br/>NO TESTS]
    D --> D4[identity-analysis 20<br/>NO TESTS]

    E --> E1[analysis-cache 20<br/>NO TESTS]
    E --> E2[ast-helpers 28<br/>NO TESTS]
    E --> E3[scoring 18<br/>HAS TESTS]

    style C1 fill:#ff0000,color:#fff
    style C2 fill:#ff6600,color:#fff
    style C3 fill:#00cc00,color:#fff
    style D1 fill:#ff6600,color:#fff
    style D2 fill:#00cc00,color:#fff
    style D3 fill:#ff6600,color:#fff
    style D4 fill:#ff6600,color:#fff
    style E1 fill:#ff6600,color:#fff
    style E2 fill:#ff6600,color:#fff
    style E3 fill:#00cc00,color:#fff
```

Legend:

- 🔴 Red: Critical - No tests, high complexity
- 🟠 Orange: High risk - Partial or no tests
- 🟢 Green: Has tests (may need verification)

## Analysis Engine Test Dependencies

```mermaid
graph TB
    subgraph Core Tests
        E[analysis-engine tests]
        C[analysis-cache tests]
        A[ast-helpers tests]
        S[scoring tests]
    end

    subgraph React Tests
        T[temporal-analysis tests]
        D[dataflow-analysis tests]
        H[hook-analysis tests]
        RH[react-helpers tests]
        CO[coupling-analysis tests]
        I[identity-analysis tests]
    end

    subgraph Integration
        INT[Integration tests]
    end

    C --> E
    A --> E
    S --> E

    RH --> T
    RH --> D
    RH --> H
    RH --> CO
    RH --> I

    E --> INT
    T --> INT
    D --> INT
    H --> INT

    style E fill:#ff6600
    style C fill:#ff6600
    style RH fill:#ff6600
    style T fill:#ff0000
```

## Critical Path Analysis

```mermaid
sequenceDiagram
    participant User
    participant Engine as AnalysisEngine
    participant Cache as AnalysisCache
    participant Plugin as ReactPlugin
    participant Analysis as Temporal Analysis
    participant Helpers as react-helpers

    User->>Engine: analyzeFile(path)
    Engine->>Cache: get(cacheKey)
    alt Cache Hit
        Cache-->>Engine: Cached Result
        Engine-->>User: Return Result
    else Cache Miss
        Engine->>Plugin: canHandle(sourceFile)
        Plugin-->>Engine: true
        Engine->>Plugin: analyze(sourceFile)
        Plugin->>Analysis: execute(sourceFile)
        Analysis->>Helpers: findReactComponents()
        Helpers-->>Analysis: components
        Analysis->>Helpers: detectTimeoutInCallback()
        Helpers-->>Analysis: hasTimeout
        Analysis-->>Plugin: AnalysisResult
        Plugin-->>Engine: PluginResult
        Engine->>Cache: set(cacheKey, result)
        Engine-->>User: Aggregated Result
    end

    Note over Engine,Cache: NEEDS TESTS:<br/>Cache miss, hits,<br/>invalidation
    Note over Plugin,Analysis: NEEDS TESTS:<br/>Temporal analysis<br/>Risk assessment
    Note over Helpers: NEEDS TESTS:<br/>All helper functions
```

## Test Distribution by Package

```mermaid
graph LR
    A[Total: 486 Tests] --> B[@analyzers/core<br/>200-235 tests]
    A --> C[@analyzers/react<br/>325-370 tests]
    A --> D[Integration<br/>25-35 tests]

    B --> B1[analysis-engine<br/>50-60 tests]
    B --> B2[analysis-cache<br/>25-30 tests]
    B --> B3[ast-helpers<br/>40-45 tests]
    B --> B4[scoring<br/>30-35 tests]
    B --> B5[plugin<br/>20-25 tests]
    B --> B6[base-analyzer<br/>15-20 tests]

    C --> C1[temporal-analysis<br/>50-55 tests]
    C --> C2[dataflow-analysis<br/>45-50 tests]
    C --> C3[hook-analysis<br/>40-45 tests]
    C --> C4[react-helpers<br/>60-70 tests]
    C --> C5[coupling-analysis<br/>30-35 tests]
    C --> C6[identity-analysis<br/>35-40 tests]
    C --> C7[plugin<br/>40-45 tests]
    C --> C8[accessibility-helpers<br/>25-30 tests]

    style B1 fill:#ff6600
    style B2 fill:#ff6600
    style B3 fill:#ff9900
    style C1 fill:#ff0000
    style C4 fill:#ff6600
    style C5 fill:#ff6600
    style C6 fill:#ff6600
```

## Risk Assessment Matrix

```mermaid
quadrantChart
    title Code Complexity vs Test Coverage
    x-axis Low Complexity --> High Complexity
    y-axis Low Coverage --> High Coverage
    quadrant-1 High Complexity, Good Coverage (Verify)
    quadrant-2 High Complexity, Poor Coverage (CRITICAL)
    quadrant-3 Low Complexity, Poor Coverage (Medium)
    quadrant-4 Low Complexity, Good Coverage (Maintain)

    temporal-analysis: [0.88, 0.0]
    analysis-engine: [1.0, 0.4]
    dataflow-analysis: [0.85, 0.7]
    react-helpers: [0.95, 0.0]
    hook-analysis: [0.55, 0.75]
    coupling-analysis: [0.40, 0.0]
    identity-analysis: [0.45, 0.0]
    analysis-cache: [0.42, 0.0]
    scoring: [0.38, 0.8]
    plugin-core: [0.32, 0.0]
    ast-helpers: [0.60, 0.0]
    accessibility-helpers: [0.35, 0.0]
```

Quadrant Interpretation:

- **Q1 (Top-Right):** High complexity, good coverage - Verify and maintain
- **Q2 (Top-Left):** High complexity, poor coverage - CRITICAL PRIORITY
- **Q3 (Bottom-Left):** Low complexity, poor coverage - Medium priority
- **Q4 (Bottom-Right):** Low complexity, good coverage - Maintain

## Sprint Capacity Planning

```mermaid
graph LR
    S1[Sprint 1<br/>275 Tests<br/>2 Weeks] --> S1D1[Dev 1: temporal + coupling<br/>~90 tests]
    S1 --> S1D2[Dev 2: identity + cache + engine<br/>~100 tests]
    S1 --> S1D3[Both: Integration<br/>~15 tests]

    S2[Sprint 2<br/>160 Tests<br/>2 Weeks] --> S2D1[Dev 1: react-helpers<br/>~70 tests]
    S2 --> S2D2[Dev 2: ast-helpers + plugin<br/>~70 tests]
    S2 --> S2D3[Both: accessibility<br/>~30 tests]

    S3[Sprint 3<br/>40 Tests<br/>1-2 Weeks] --> S3D1[Verify existing tests<br/>Review coverage]
    S3 --> S3D2[Performance benchmarks<br/>11 benchmarks]
    S3 --> S3D3[Bug fixes<br/>As discovered]
```

## Test Types Distribution

```mermaid
pie title Test Types in Sprint 1
    "Unit Tests" : 230
    "Integration Tests" : 15
    "Edge Case Tests" : 30
```

```mermaid
pie title Test Types in Sprint 2
    "Unit Tests" : 130
    "Helper Function Tests" : 100
    "Integration Tests" : 10
```

## Coverage Targets by Sprint

```mermaid
xychart-beta
    title Code Coverage Progress
    x-axis [Current, Sprint 1, Sprint 2, Sprint 3]
    y-axis "Coverage %" 0 --> 100
    line [37, 65, 80, 85]
    line [35, 60, 75, 80]
```

Legend:

- Blue: Line coverage
- Orange: Branch coverage

Target: 85% line, 80% branch

## File Prioritization Matrix

| Priority | File                     | Tests Needed | Sprint | Team Member |
| -------- | ------------------------ | ------------ | ------ | ----------- |
| 🔴 P0.1  | temporal-analysis.ts     | 55           | 1      | Dev 1       |
| 🔴 P0.2  | coupling-analysis.ts     | 35           | 1      | Dev 1       |
| 🔴 P0.3  | identity-analysis.ts     | 40           | 1      | Dev 2       |
| 🔴 P0.4  | analysis-cache.ts        | 30           | 1      | Dev 2       |
| 🔴 P0.5  | analysis-engine.ts       | 30           | 1      | Dev 2       |
| 🔴 P0.6  | Integration tests        | 15           | 1      | Both        |
| 🟠 P1.1  | react-helpers.ts         | 70           | 2      | Dev 1       |
| 🟠 P1.2  | ast-helpers.ts           | 45           | 2      | Dev 2       |
| 🟠 P1.3  | plugin.ts (core)         | 25           | 2      | Dev 2       |
| 🟠 P1.4  | accessibility-helpers.ts | 30           | 2      | Both        |
| 🟠 P1.5  | scoring.ts additions     | 15           | 2      | Dev 2       |
| 🟡 P2.1  | Verify existing          | 30           | 3      | Both        |
| 🟡 P2.2  | Benchmarks               | 11           | 3      | Both        |

## Quick Decision Tree

```mermaid
graph TD
    Start[Which file should I test next?] --> Q1{Is it P0?}
    Q1 -->|Yes| Q2{Has NO tests?}
    Q1 -->|No| Q5{Is it P1?}

    Q2 -->|Yes| A1[temporal-analysis<br/>coupling-analysis<br/>identity-analysis<br/>analysis-cache]
    Q2 -->|No| A2[analysis-engine<br/>Add 30 tests]

    Q5 -->|Yes| Q6{Helper library?}
    Q5 -->|No| A7[Move to P2<br/>or verify existing]

    Q6 -->|Yes| A5[react-helpers<br/>ast-helpers<br/>accessibility-helpers]
    Q6 -->|No| A6[plugin.ts core<br/>scoring additions]

    style A1 fill:#ff0000,color:#fff
    style A2 fill:#ff6600,color:#fff
    style A5 fill:#ff9900,color:#fff
    style A6 fill:#ff9900,color:#fff
    style A7 fill:#ffcc00
```

## Dependencies Between Test Files

```mermaid
graph TD
    RH[react-helpers.test.ts<br/>Foundation] --> TA[temporal-analysis.test.ts]
    RH --> DA[dataflow-analysis.test.ts]
    RH --> HA[hook-analysis.test.ts]
    RH --> CA[coupling-analysis.test.ts]
    RH --> IA[identity-analysis.test.ts]

    AC[analysis-cache.test.ts] --> AE[analysis-engine.test.ts]
    AST[ast-helpers.test.ts] --> AE
    SC[scoring.test.ts] --> AE

    AE --> INT[integration.test.ts]
    TA --> INT
    DA --> INT
    HA --> INT

    style RH fill:#ff6600,stroke:#333,stroke-width:4px
    style AE fill:#ff6600,stroke:#333,stroke-width:4px
    style INT fill:#4169E1,stroke:#333,stroke-width:4px
```

**Key Insight:** react-helpers.ts is a foundational dependency. Test it early in Sprint 2 to unblock other React analysis tests.

## Weekly Milestones

```mermaid
gantt
    title Weekly Testing Milestones
    dateFormat YYYY-MM-DD
    section Week 1
    temporal-analysis complete     :milestone, m1, 2026-01-17, 0d
    coupling-analysis complete      :milestone, m2, 2026-01-17, 0d
    section Week 2
    identity + cache + engine      :milestone, m3, 2026-01-24, 0d
    Integration tests complete     :milestone, m4, 2026-01-24, 0d
    section Week 3
    react-helpers complete         :milestone, m5, 2026-01-31, 0d
    section Week 4
    All P1 tests complete          :milestone, m6, 2026-02-07, 0d
    section Week 5-6
    Benchmarks & verification      :milestone, m7, 2026-02-14, 0d
```

## Success Metrics Dashboard

Track these weekly:

| Metric          | Target | Current | Sprint 1 | Sprint 2 | Sprint 3 |
| --------------- | ------ | ------- | -------- | -------- | -------- |
| Line Coverage   | 85%    | 37%     | 65%      | 80%      | 85%      |
| Branch Coverage | 80%    | 35%     | 60%      | 75%      | 80%      |
| Tests Written   | 486    | ~200    | 475      | 635      | 686      |
| P0 Tests Done   | 275    | 0       | 275      | 275      | 275      |
| Bugs Found      | -      | 0       | TBD      | TBD      | TBD      |
| Benchmarks      | 11     | 0       | 0        | 0        | 11       |

## Resource Allocation

```mermaid
pie title Engineer Hours by Sprint
    "Sprint 1 (275 tests)" : 50
    "Sprint 2 (160 tests)" : 45
    "Sprint 3 (40 tests)" : 15
    "Benchmarks" : 10
    "Buffer for bugs" : 17
```

Total: 137 hours (17 days @ 8h/day with 2 engineers = 8.5 days)

## Conclusion

This visual roadmap provides:

- Clear prioritization (P0 → P1 → P2)
- Sprint-by-sprint plan (3 sprints, 4-6 weeks)
- Resource allocation (2 developers)
- Dependencies between test files
- Success metrics and milestones

**Next Action:** Begin Sprint 1 with temporal-analysis.test.ts (highest risk file).

---

**Related Documents:**

- `testing-summary.md` - Executive summary
- `testing-priority-checklist.md` - Detailed checklists
- `detailed-test-specifications.md` - Test specifications
- `complexity-matrix.md` - Complexity metrics
