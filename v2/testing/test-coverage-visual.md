# Test Coverage Visual Reference

Visual representation of test coverage status for the React analyzer package.

## Coverage Heatmap

```mermaid
graph TB
    subgraph "React Analyzer Package"
        subgraph "src/analyses [100% Tested]"
            A1[accessibility-analysis ✓]
            A2[anti-pattern-analysis ✓]
            A3[coupling-analysis ✓]
            A4[dataflow-analysis ✓]
            A5[hook-analysis ✓]
            A6[identity-analysis ✓]
            A7[migration-analysis ✓]
            A8[performance-analysis ✓]
            A9[reliability-analysis ✓]
            A10[security-analysis ✓]
            A11[structural-analysis ✓]
            A12[technical-debt-analysis ✓]
            A13[temporal-analysis ✓]
            A14[types-analysis ✓]
        end

        subgraph "src/presenters [0% Tested]"
            P1[overview-presenter ✗]
            P2[performance-presenter ✗]
            P3[accessibility-presenter ✗]
            P4[migration-presenter ✗]
            P5[reliability-presenter ✗]
            P6[security-presenter ✗]
            P7[dataflow-presenter ✗]
            P8[anti-pattern-presenter ✗]
            P9[base-presenter ✗]
        end

        subgraph "src/utils [100% Tested]"
            U1[accessibility-helpers ✓]
            U2[react-helpers ✓]
        end

        subgraph "src/constants [0% Tested]"
            C1[weights ✗]
            C2[thresholds ✗]
            C3[dependencies △]
            C4[index △]
        end

        subgraph "Root Files"
            R1[plugin ✓]
        end
    end

    style A1 fill:#90EE90
    style A2 fill:#90EE90
    style A3 fill:#90EE90
    style A4 fill:#90EE90
    style A5 fill:#90EE90
    style A6 fill:#90EE90
    style A7 fill:#90EE90
    style A8 fill:#90EE90
    style A9 fill:#90EE90
    style A10 fill:#90EE90
    style A11 fill:#90EE90
    style A12 fill:#90EE90
    style A13 fill:#90EE90
    style A14 fill:#90EE90

    style P1 fill:#FFB6C6
    style P2 fill:#FFB6C6
    style P3 fill:#FFB6C6
    style P4 fill:#FFB6C6
    style P5 fill:#FFB6C6
    style P6 fill:#FFB6C6
    style P7 fill:#FFB6C6
    style P8 fill:#FFB6C6
    style P9 fill:#FFB6C6

    style U1 fill:#90EE90
    style U2 fill:#90EE90

    style C1 fill:#FFB6C6
    style C2 fill:#FFB6C6
    style C3 fill:#FFD700
    style C4 fill:#FFD700

    style R1 fill:#90EE90
```

**Legend:**

- Green (✓): Tested
- Red (✗): Not Tested
- Yellow (△): Low Priority / Optional

## Architecture Flow

```mermaid
flowchart LR
    subgraph Input
        SF[Source File]
    end

    subgraph Plugin[Plugin Layer - TESTED]
        P[ReactAnalyzerPlugin]
    end

    subgraph Analysis[Analysis Layer - TESTED]
        A1[14 Analysis Modules]
        A2[All Tested ✓]
    end

    subgraph Presentation[Presentation Layer - NOT TESTED]
        PR1[9 Presenters]
        PR2[0% Coverage ✗]
    end

    subgraph Output
        OP[Report Presentation]
    end

    SF --> P
    P --> A1
    A1 --> A2
    A2 --> PR1
    PR1 --> PR2
    PR2 --> OP

    style P fill:#90EE90
    style A1 fill:#90EE90
    style A2 fill:#90EE90
    style PR1 fill:#FFB6C6
    style PR2 fill:#FFB6C6
```

## Test Coverage Matrix

| Module Type | Total Files | Tested | Untested | Coverage % | Priority |
| ----------- | ----------- | ------ | -------- | ---------- | -------- |
| Analyses    | 14          | 14     | 0        | 100%       | Maintain |
| Presenters  | 9           | 0      | 9        | 0%         | Critical |
| Utilities   | 2           | 2      | 0        | 100%       | Maintain |
| Constants   | 4           | 0      | 2        | 0%         | High     |
| Plugin Core | 1           | 1      | 0        | 100%       | Maintain |
| Types       | 19          | 0      | 19       | N/A        | Low      |
| **Total**   | **49**      | **17** | **30**   | **35%**    | -        |

## Lines of Code Distribution

```mermaid
pie title Lines of Code by Test Status
    "Tested (Analyses)" : 14121
    "Tested (Utils)" : 400
    "Tested (Plugin)" : 600
    "Untested (Presenters)" : 2000
    "Untested (Constants)" : 500
    "Types (N/A)" : 2000
```

## Priority-Based Roadmap

```mermaid
gantt
    title Test Coverage Implementation Roadmap
    dateFormat YYYY-MM-DD
    section Phase 1 - Critical
    Presenter Tests           :crit, p1, 2026-01-20, 10d
    Constants Validation      :crit, p2, 2026-01-25, 3d
    section Phase 2 - High
    Integration Tests         :p3, 2026-01-30, 6d
    Edge Case Coverage        :p4, 2026-02-03, 8d
    section Phase 3 - Medium
    Performance Tests         :p5, 2026-02-11, 5d
    Type Guard Tests          :p6, 2026-02-14, 3d
```

## Test Impact Analysis

```mermaid
graph TD
    subgraph High Impact
        H1[Presenter Tests]
        H2[Integration Tests]
        H3[Constants Validation]
    end

    subgraph Medium Impact
        M1[Edge Case Coverage]
        M2[Performance Tests]
    end

    subgraph Low Impact
        L1[Type Guard Tests]
    end

    H1 -->|Affects All Clients| CLIENT[CLI, VSCode, Desktop]
    H2 -->|Validates Workflows| RELIABILITY[System Reliability]
    H3 -->|Ensures Accuracy| SCORING[Scoring Correctness]

    M1 -->|Prevents Crashes| ROBUSTNESS[Production Robustness]
    M2 -->|Maintains Speed| PERFORMANCE[User Experience]

    L1 -->|Extra Safety| SAFETY[Type Safety]

    style H1 fill:#FF6B6B
    style H2 fill:#FF6B6B
    style H3 fill:#FF6B6B
    style M1 fill:#FFA500
    style M2 fill:#FFA500
    style L1 fill:#FFD700
```

## Presenter Dependencies

```mermaid
graph LR
    subgraph Core
        BP[base-presenter ✗]
    end

    subgraph Concrete Presenters
        OP[overview-presenter ✗]
        PP[performance-presenter ✗]
        AP[accessibility-presenter ✗]
        MP[migration-presenter ✗]
        RP[reliability-presenter ✗]
        SP[security-presenter ✗]
        DP[dataflow-presenter ✗]
        APP[anti-pattern-presenter ✗]
    end

    BP --> OP
    BP --> PP
    BP --> AP
    BP --> MP
    BP --> RP
    BP --> SP
    BP --> DP
    BP --> APP

    style BP fill:#FFB6C6
    style OP fill:#FFB6C6
    style PP fill:#FFB6C6
    style AP fill:#FFB6C6
    style MP fill:#FFB6C6
    style RP fill:#FFB6C6
    style SP fill:#FFB6C6
    style DP fill:#FFB6C6
    style APP fill:#FFB6C6
```

**Note:** All presenters depend on `base-presenter`, so testing it first establishes patterns for all others.

## Analysis-to-Presenter Mapping

```mermaid
graph LR
    subgraph Analyses [Tested ✓]
        AA[accessibility-analysis]
        PA[performance-analysis]
        MA[migration-analysis]
        RA[reliability-analysis]
        SA[security-analysis]
        DA[dataflow-analysis]
        APA[anti-pattern-analysis]
        OA[Other Analyses]
    end

    subgraph Presenters [Not Tested ✗]
        AAP[accessibility-presenter]
        PAP[performance-presenter]
        MAP[migration-presenter]
        RAP[reliability-presenter]
        SAP[security-presenter]
        DAP[dataflow-presenter]
        APAP[anti-pattern-presenter]
        OAP[overview-presenter]
    end

    AA -->|Transforms| AAP
    PA -->|Transforms| PAP
    MA -->|Transforms| MAP
    RA -->|Transforms| RAP
    SA -->|Transforms| SAP
    DA -->|Transforms| DAP
    APA -->|Transforms| APAP
    OA -->|Aggregates| OAP

    style AA fill:#90EE90
    style PA fill:#90EE90
    style MA fill:#90EE90
    style RA fill:#90EE90
    style SA fill:#90EE90
    style DA fill:#90EE90
    style APA fill:#90EE90
    style OA fill:#90EE90

    style AAP fill:#FFB6C6
    style PAP fill:#FFB6C6
    style MAP fill:#FFB6C6
    style RAP fill:#FFB6C6
    style SAP fill:#FFB6C6
    style DAP fill:#FFB6C6
    style APAP fill:#FFB6C6
    style OAP fill:#FFB6C6
```

## Test Effort Distribution

```mermaid
graph TD
    subgraph Total Effort: 52-70 hours
        P1[Phase 1: Presenters<br/>16-20 hours<br/>38%]
        P2[Phase 1: Constants<br/>4-6 hours<br/>9%]
        P3[Phase 2: Integration<br/>8-12 hours<br/>19%]
        P4[Phase 2: Edge Cases<br/>12-16 hours<br/>25%]
        P5[Phase 3: Performance<br/>8-10 hours<br/>16%]
        P6[Phase 3: Type Guards<br/>4-6 hours<br/>9%]
    end

    style P1 fill:#FF6B6B
    style P2 fill:#FF6B6B
    style P3 fill:#FFA500
    style P4 fill:#FFA500
    style P5 fill:#FFD700
    style P6 fill:#FFD700
```

## Coverage Progress Tracker

**Current State:** Week 0

```
Analyses:     ████████████████████ 100%
Utilities:    ████████████████████ 100%
Plugin:       ████████████████████ 100%
Presenters:   ░░░░░░░░░░░░░░░░░░░░   0%
Constants:    ░░░░░░░░░░░░░░░░░░░░   0%
Integration:  ████░░░░░░░░░░░░░░░░  20%
─────────────────────────────────────
Overall:      ███████░░░░░░░░░░░░░  35%
```

**Target State:** Week 6

```
Analyses:     ████████████████████ 100%
Utilities:    ████████████████████ 100%
Plugin:       ████████████████████ 100%
Presenters:   ███████████████████░  95%
Constants:    ████████████████░░░░  80%
Integration:  ████████████████████ 100%
─────────────────────────────────────
Overall:      ███████████████████░  95%
```

## Risk Assessment Matrix

| Gap                          | Impact | Likelihood | Risk Level | Priority |
| ---------------------------- | ------ | ---------- | ---------- | -------- |
| Presenter bugs in production | High   | Medium     | High       | Critical |
| Incorrect scoring weights    | High   | Low        | Medium     | High     |
| Integration failures         | High   | Low        | Medium     | High     |
| Edge case crashes            | Medium | Medium     | Medium     | Medium   |
| Performance regression       | Low    | Low        | Low        | Low      |

## Quick Reference: What to Test First

### Week 1-2: Critical Path

1. `base-presenter.test.ts` - Foundation for all presenters
2. `overview-presenter.test.ts` - Most complex, highest usage
3. `performance-presenter.test.ts` - High user visibility
4. `weights.test.ts` - Scoring accuracy
5. `thresholds.test.ts` - Scoring accuracy

### Week 3-4: High Impact

6. `accessibility-presenter.test.ts`
7. `migration-presenter.test.ts`
8. `reliability-presenter.test.ts`
9. `security-presenter.test.ts`
10. Expand `react.integration.ts`

### Week 5-6: Polish

11. `dataflow-presenter.test.ts`
12. `anti-pattern-presenter.test.ts`
13. Enhance `react.benchmark.ts`
14. Edge case coverage in analyses

## Success Visualization

```mermaid
graph LR
    START[Current: 35% Coverage] -->|Phase 1| P1[60% Coverage]
    P1 -->|Phase 2| P2[85% Coverage]
    P2 -->|Phase 3| END[95% Coverage]

    style START fill:#FFB6C6
    style P1 fill:#FFA500
    style P2 fill:#FFD700
    style END fill:#90EE90
```

## Key Metrics Dashboard

| Metric                | Current | Target     | Status       |
| --------------------- | ------- | ---------- | ------------ |
| Test Files            | 17      | 28         | 61%          |
| Presenter Coverage    | 0%      | 95%        | Needs Work   |
| Constants Coverage    | 0%      | 80%        | Needs Work   |
| Integration Scenarios | 3       | 15+        | Needs Work   |
| Total Coverage        | 35%     | 95%        | In Progress  |
| Test Suite Speed      | Unknown | `&lt;30`s` | To Establish |

---

**Legend:**

- ✓ = Tested
- ✗ = Not Tested
- △ = Low Priority
- Green = Good
- Yellow = Caution
- Red = Action Required
