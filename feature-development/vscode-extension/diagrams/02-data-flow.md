---
id: 02-data-flow
---

# Data Flow Diagram

This diagram shows how analysis snapshots flow through the system for temporal analysis.

## Analysis Snapshot Data Flow

```mermaid
---
title: Vipr Analysis Snapshot Data Flow
config:
  theme: neutral
---
sequenceDiagram
    accTitle: Analysis Snapshot Data Flow
    accDescr: Shows how analysis results are stored, retrieved, and used for temporal comparison and regression detection

    actor User
    participant Editor as VSCode Editor
    participant OnSave as OnSave Handler
    participant AnalysisManager as Analysis Manager
    participant Engine as Analysis Engine
    participant Storage as Storage Service
    participant Database as SQLite Database
    participant Git as Git History Service
    participant GitRepo as Git Repository

    %% Analysis Trigger
    User->>Editor: Save file or run command
    Editor->>OnSave: Document saved event
    OnSave->>AnalysisManager: Trigger analysis

    %% Check cache
    AnalysisManager->>AnalysisManager: Compute content hash
    AnalysisManager->>AnalysisManager: Check result cache

    alt Cache hit
        AnalysisManager-->>Editor: Return cached result
    else Cache miss
        %% Run analysis
        AnalysisManager->>Engine: Analyze file
        Engine->>Engine: Load plugins<br/>(lazy initialization)
        Engine->>Engine: Run analyzers
        Engine-->>AnalysisManager: AggregatedResult

        %% Store in result cache
        AnalysisManager->>AnalysisManager: Store in result cache<br/>(content hash → result)

        %% Get git context
        AnalysisManager->>Git: Get current commit info
        Git->>GitRepo: git rev-parse HEAD
        GitRepo-->>Git: commit hash
        Git->>GitRepo: git show -s --format
        GitRepo-->>Git: commit metadata
        Git-->>AnalysisManager: GitCommit

        %% Store snapshot
        AnalysisManager->>Storage: Save snapshot
        Storage->>Database: INSERT analysis_snapshots
        Storage->>Database: INSERT metric_scores
        Storage->>Database: INSERT findings
        Database-->>Storage: snapshot_id
        Storage-->>AnalysisManager: Snapshot saved

        %% Update UI providers
        AnalysisManager->>Editor: Emit state change event
        AnalysisManager-->>Editor: Update diagnostics
        AnalysisManager-->>Editor: Update CodeLens
        AnalysisManager-->>Editor: Update decorations
        AnalysisManager-->>Editor: Update dashboard
    end

    %% Show results to user
    Editor->>User: Display analysis results
```

## Historical Comparison Flow

```mermaid
---
title: Historical Analysis Comparison
config:
  theme: neutral
---
sequenceDiagram
    accTitle: Historical Analysis Comparison
    accDescr: Shows how current analysis is compared with previous snapshots to detect regressions

    actor User
    participant Command as Show History Command
    participant Storage as Storage Service
    participant Database as SQLite Database
    participant HistoryPanel as History Panel
    participant Chart as Trend Chart

    User->>Command: vipr.showHistory
    Command->>Storage: Get snapshots for file
    Storage->>Database: SELECT * FROM analysis_snapshots<br/>WHERE file_path = ?<br/>ORDER BY commit_date DESC
    Database-->>Storage: List of snapshots
    Storage-->>Command: AnalysisSnapshot[]

    alt No history available
        Command-->>User: Show info message
    else History exists
        Command->>HistoryPanel: Create panel with snapshots
        HistoryPanel->>HistoryPanel: Prepare state
        HistoryPanel->>HistoryPanel: Generate HTML
        HistoryPanel-->>User: Display history panel

        %% Calculate deltas
        loop For each snapshot pair
            HistoryPanel->>HistoryPanel: Calculate score delta
            HistoryPanel->>HistoryPanel: Identify trend direction
        end

        %% Render visualization
        HistoryPanel->>Chart: Render trend chart
        Chart->>Chart: Scale data points
        Chart->>Chart: Draw line path
        Chart->>Chart: Add data points
        Chart-->>HistoryPanel: SVG chart
        HistoryPanel-->>User: Interactive visualization
    end
```

## Metric Storage Schema

```mermaid
---
title: SQLite Database Schema for Temporal Tracking
config:
  theme: neutral
  layout: elk
---
erDiagram
    accTitle: Vipr History Database Schema
    accDescr: SQLite schema showing relationships between analysis snapshots, metric scores, findings, and git blame cache

    analysis_snapshots ||--o{ metric_scores : "has many"
    analysis_snapshots ||--o{ findings : "has many"

    analysis_snapshots {
        int id PK
        text file_path
        text commit_hash
        int commit_date "Unix timestamp"
        text commit_author
        text commit_message
        real overall_score
        int analyzed_at "Unix timestamp"
    }

    metric_scores {
        int id PK
        int snapshot_id FK
        text category "security, performance, etc"
        text metric_name "xss_count, cyclomatic, etc"
        real score_value "0-100 percentage"
        int count_value "Integer counts"
    }

    findings {
        int id PK
        int snapshot_id FK
        text category "MetricCategory"
        text finding_type "eval_usage, etc"
        text severity "critical, high, medium, low"
        int line_number
        int column_number
        text message
        text introduced_commit "From git blame"
        int introduced_date "Unix timestamp"
        text introduced_author
    }

    git_blame_cache {
        int id PK
        text file_path
        int line_number
        text as_of_commit "Context commit"
        text blame_commit "Actual commit"
        text blame_author
        int blame_date "Unix timestamp"
        int cached_at "Unix timestamp"
    }
```

## Cache Invalidation Strategy

```mermaid
---
title: Multi-Layer Caching Strategy
config:
  theme: neutral
---
flowchart TD
    accTitle: Vipr Caching Strategy
    accDescr: Shows the multi-layer caching approach with content-based result cache and SQLite storage

    Start([File Analysis Request]) --> ComputeHash[Compute Content Hash<br/>SHA-256 of file content]

    ComputeHash --> CheckResultCache{Check Result Cache<br/>content hash → result}

    CheckResultCache -->|Hit| ReturnCached[Return Cached Result<br/>Update UI immediately]
    CheckResultCache -->|Miss| CheckStorage{Check SQLite Storage<br/>file + commit hash}

    CheckStorage -->|Hit| LoadSnapshot[Load Snapshot<br/>with metrics + findings]
    LoadSnapshot --> StoreInCache[Store in Result Cache]
    StoreInCache --> ReturnCached

    CheckStorage -->|Miss| RunAnalysis[Run Full Analysis<br/>via Analysis Engine]
    RunAnalysis --> StoreResults[Store in Both Caches]

    StoreResults --> UpdateResultCache[Update Result Cache<br/>TTL: 10 minutes]
    StoreResults --> SaveSnapshot[Save to SQLite<br/>with git metadata]

    UpdateResultCache --> ReturnCached
    SaveSnapshot --> ReturnCached

    ReturnCached --> UpdateUI[Update All Providers<br/>Diagnostics, CodeLens,<br/>Decorations, Dashboard]
    UpdateUI --> End([Done])

    %% Periodic cleanup
    Cleanup([Periodic Cleanup<br/>Every 5 minutes]) -.-> CleanResultCache[Clean Expired Cache Entries]
    CleanResultCache -.-> CleanSnapshots[Clean Old Snapshots<br/>retention: 90 days]

    %% Styling
    classDef cacheHit fill:#10b981,stroke:#059669,color:#fff
    classDef cacheMiss fill:#f59e0b,stroke:#d97706,color:#fff
    classDef storage fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef analysis fill:#8b5cf6,stroke:#7c3aed,color:#fff

    class ReturnCached,StoreInCache cacheHit
    class RunAnalysis analysis
    class CheckStorage,SaveSnapshot storage
    class CheckResultCache,UpdateResultCache cacheMiss
```

## Key Insights

### Performance Optimizations

1. **Content-Based Hashing**: Files with identical content reuse cached results
2. **Lazy SQLite Writes**: Batch database writes to reduce I/O
3. **Git Blame Caching**: Expensive git operations cached in database
4. **TTL-Based Expiration**: Result cache expires after 10 minutes (configurable)

### Storage Strategy

1. **Snapshot on Every Analysis**: Each analysis run creates a new snapshot
2. **Deduplication**: Same file + commit hash combination is updated, not duplicated
3. **Flexible Metrics**: Schema supports evolving metric types without migration
4. **Attribution Data**: Findings include git blame for accountability

### Data Flow Guarantees

1. **Eventual Consistency**: UI updates immediately from cache, storage is async
2. **No Data Loss**: Failed storage operations don't block analysis
3. **Graceful Degradation**: Extension works without git repository
4. **Offline Support**: Result cache works without network/git access
