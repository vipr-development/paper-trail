---
id: 03-git-integration-flow
title: Git Integration Flow
sidebar_label: Git Integration Flow
---

# Git Integration Flow

This diagram shows how git history is leveraged for regression detection and attribution.

## Regression Detection Flow

```mermaid
---
title: Temporal Regression Detection with Binary Search
config:
  theme: neutral
---
sequenceDiagram
    accTitle: Regression Detection Flow
    accDescr: Shows binary search algorithm to find the commit that introduced a regression

    actor User
    participant Command as Find Regression Command
    participant Detector as Regression Detector
    participant Git as Git History Service
    participant Storage as Storage Service
    participant Engine as Analysis Engine
    participant Progress as Progress Notification

    User->>Command: vipr.findRegression
    Command->>Storage: Get previous snapshot
    Storage-->>Command: AnalysisSnapshot (baseline)

    alt No previous snapshot
        Command-->>User: No history to compare
    else Has baseline
        Command->>Git: Get file commit history
        Git->>Git: git log --format<br/>-n 50 -- file.ts
        Git-->>Command: GitCommit[]

        Command->>Detector: Find regression commit
        Detector->>Progress: Show progress notification
        Progress-->>User: Finding regression commit...

        %% Binary search loop
        loop Binary Search (max depth: 10)
            Detector->>Detector: Calculate midpoint

            Detector->>Storage: Check for cached snapshot
            Storage-->>Detector: Snapshot or null

            alt Snapshot exists
                Detector->>Detector: Use cached score
            else No snapshot
                Detector->>Git: Get file at commit
                Git->>Git: git show commit:file
                Git-->>Detector: File content

                Detector->>Engine: Analyze historical version
                Engine-->>Detector: Analysis score

                Detector->>Storage: Save historical snapshot
                Storage-->>Detector: Snapshot saved
            end

            Detector->>Progress: Update progress<br/>Checking commit abc123...
            Progress-->>User: Progress update

            alt Score < threshold (regression found)
                Detector->>Detector: Search earlier commits<br/>right = mid - 1
            else Score >= threshold (still good)
                Detector->>Detector: Search later commits<br/>left = mid + 1
            end
        end

        Detector-->>Command: RegressionCommit or null

        alt Regression found
            Command->>Git: Get full commit info
            Git-->>Command: Commit metadata
            Command-->>User: Show regression details<br/>with author, date, message
        else No regression in range
            Command-->>User: No clear regression found
        end
    end
```

## Git Blame Attribution Flow

```mermaid
---
title: Finding Attribution with Git Blame
config:
  theme: neutral
---
flowchart TD
    accTitle: Git Blame Attribution Flow
    accDescr: Shows how findings are attributed to specific commits and authors using git blame with caching

    Start([Finding Detected]) --> HasLocation{Finding has<br/>line number?}

    HasLocation -->|No| SkipAttribution[Skip Attribution<br/>Category-level finding]
    SkipAttribution --> End([Done])

    HasLocation -->|Yes| GetCommit[Get Current Commit<br/>git rev-parse HEAD]
    GetCommit --> CheckCache{Check Blame Cache<br/>file + line + commit}

    CheckCache -->|Cache Hit| UseCached[Use Cached Blame<br/>commit, author, date]
    UseCached --> UpdateFinding[Update Finding<br/>with attribution data]
    UpdateFinding --> End

    CheckCache -->|Cache Miss| RunBlame[Run Git Blame<br/>git blame -L line,line<br/>--porcelain commit file]

    RunBlame --> ParseOutput[Parse Porcelain Output<br/>Extract commit, author, date]
    ParseOutput --> CacheResult[Store in Blame Cache<br/>TTL: 24 hours]
    CacheResult --> UpdateFinding

    %% Batch optimization
    BatchFindings([Multiple Findings]) -.-> ConsolidateRanges[Consolidate to Ranges<br/>Minimize git calls]
    ConsolidateRanges -.-> RunBlame

    %% Styling
    classDef cache fill:#10b981,stroke:#059669,color:#fff
    classDef git fill:#8b5cf6,stroke:#7c3aed,color:#fff
    classDef update fill:#3b82f6,stroke:#1e40af,color:#fff

    class CheckCache,UseCached,CacheResult cache
    class GetCommit,RunBlame,ParseOutput git
    class UpdateFinding update
```

## Historical Analysis Workflow

```mermaid
---
title: Analyzing File at Past Commits
config:
  theme: neutral
---
stateDiagram-v2
    accTitle: Historical Analysis State Machine
    accDescr: Shows the states and transitions when analyzing historical file versions

    [*] --> CheckSnapshot: Request historical analysis

    CheckSnapshot --> LoadFromCache: Snapshot exists in DB
    CheckSnapshot --> FetchFromGit: No snapshot

    FetchFromGit --> ValidateCommit: Get file content
    ValidateCommit --> FileExists: git show successful
    ValidateCommit --> FileNotFound: File didn't exist

    FileNotFound --> [*]: Skip this commit

    FileExists --> RunAnalysis: Analyze content
    RunAnalysis --> SaveSnapshot: Store results
    SaveSnapshot --> LoadFromCache: Add to cache

    LoadFromCache --> ReturnResults: Snapshot ready
    ReturnResults --> [*]: Analysis complete

    state RunAnalysis {
        [*] --> LoadPlugins
        LoadPlugins --> ExecuteAnalyzers
        ExecuteAnalyzers --> AggregateResults
        AggregateResults --> [*]
    }

    state SaveSnapshot {
        [*] --> InsertSnapshot
        InsertSnapshot --> InsertMetrics
        InsertMetrics --> InsertFindings
        InsertFindings --> Commit
        Commit --> [*]
    }

    note right of CheckSnapshot
        First check if we've already
        analyzed this file at this commit
    end note

    note right of FetchFromGit
        Expensive operation -
        requires git show + full analysis
    end note

    note right of SaveSnapshot
        Cache for future comparisons
        and trend visualizations
    end note
```

## Git History Service Architecture

```mermaid
---
title: Git History Service Operations
config:
  theme: neutral
  layout: elk
---
graph LR
    accTitle: Git History Service Architecture
    accDescr: Shows the operations provided by Git History Service and their relationships

    subgraph GitHistoryService["Git History Service"]
        direction TB

        subgraph MetadataOps["Metadata Operations"]
            GetCurrentCommit["getCurrentCommit()<br/>→ commit hash"]
            GetCommitInfo["getCommitInfo(hash)<br/>→ author, date, message"]
            GetFileHistory["getFileHistory(file)<br/>→ GitCommit[]"]
            GetCommitsBetween["getCommitsBetween(from, to)<br/>→ GitCommit[]"]
        end

        subgraph ContentOps["Content Operations"]
            GetFileAtCommit["getFileAtCommit(file, commit)<br/>→ file content"]
        end

        subgraph BlameOps["Attribution Operations"]
            GetBlameForLine["getBlameForLine(file, line)<br/>→ GitBlame"]
        end

        subgraph UtilityOps["Utility Operations"]
            IsGitRepo["isGitRepository()<br/>→ boolean"]
        end
    end

    subgraph GitCommands["Git CLI Commands"]
        RevParse["git rev-parse HEAD"]
        Show["git show commit:file<br/>git show -s --format"]
        Log["git log --format"]
        Blame["git blame -L --porcelain"]
        RevParseGitDir["git rev-parse --git-dir"]
    end

    subgraph Cache["Storage Service Cache"]
        BlameCache["Git Blame Cache<br/>(file + line + commit)"]
        SnapshotCache["Analysis Snapshots<br/>(file + commit)"]
    end

    %% Connections
    GetCurrentCommit --> RevParse
    GetCommitInfo --> Show
    GetFileHistory --> Log
    GetCommitsBetween --> Log
    GetFileAtCommit --> Show
    IsGitRepo --> RevParseGitDir

    GetBlameForLine --> BlameCache
    BlameCache -.cache miss.-> Blame
    Blame --> BlameCache

    GetFileAtCommit -.used by.-> SnapshotCache

    %% Styling
    classDef metadata fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef content fill:#10b981,stroke:#059669,color:#fff
    classDef blame fill:#f59e0b,stroke:#d97706,color:#fff
    classDef utility fill:#64748b,stroke:#475569,color:#fff
    classDef git fill:#8b5cf6,stroke:#7c3aed,color:#fff
    classDef cache fill:#ec4899,stroke:#db2777,color:#fff

    class GetCurrentCommit,GetCommitInfo,GetFileHistory,GetCommitsBetween metadata
    class GetFileAtCommit content
    class GetBlameForLine blame
    class IsGitRepo utility
    class RevParse,Show,Log,Blame,RevParseGitDir git
    class BlameCache,SnapshotCache cache
```

## Regression Detection Algorithm

```mermaid
---
title: Binary Search for Regression Detection
config:
  theme: neutral
---
flowchart TD
    accTitle: Binary Search Regression Detection Algorithm
    accDescr: Detailed algorithm for finding the first bad commit using binary search

    Start([Start Regression Detection]) --> GetHistory[Get Commit History<br/>git log -n 50 file.ts]
    GetHistory --> ValidateHistory{History has<br/>2+ commits?}

    ValidateHistory -->|No| NoHistory[Return null<br/>Insufficient history]
    NoHistory --> End([End])

    ValidateHistory -->|Yes| Initialize[Initialize Search<br/>left = 0<br/>right = commits.length - 1<br/>depth = 0<br/>maxDepth = 10]

    Initialize --> SearchLoop{left <= right<br/>AND depth < maxDepth}

    SearchLoop -->|No| ReturnResult[Return Best Match<br/>or null if none found]
    ReturnResult --> End

    SearchLoop -->|Yes| CalcMid[Calculate Midpoint<br/>mid = floor((left + right) / 2)]
    CalcMid --> GetCommit[Get Commit at Index<br/>commits[mid]]

    GetCommit --> CheckSnapshot{Snapshot exists<br/>for this commit?}

    CheckSnapshot -->|Yes| GetCachedScore[Get Score from Cache]
    GetCachedScore --> EvaluateScore

    CheckSnapshot -->|No| FetchContent[Fetch File Content<br/>git show commit:file]
    FetchContent --> ValidateContent{Content exists?}

    ValidateContent -->|No| SkipCommit[Skip Commit<br/>right = mid - 1]
    SkipCommit --> IncrementDepth

    ValidateContent -->|Yes| AnalyzeContent[Analyze Historical Version<br/>Run full analysis]
    AnalyzeContent --> StoreSnapshot[Store Snapshot<br/>for future use]
    StoreSnapshot --> EvaluateScore{Score < threshold<br/>(e.g., 70)?}

    EvaluateScore -->|Bad Score| FoundRegression[Potential Regression<br/>Store as candidate<br/>right = mid - 1<br/>Search for earlier bad commit]
    FoundRegression --> IncrementDepth

    EvaluateScore -->|Good Score| NotRegression[Commit is Good<br/>left = mid + 1<br/>Search for later regression]
    NotRegression --> IncrementDepth

    IncrementDepth[Increment Depth<br/>depth++<br/>Update Progress UI] --> SearchLoop

    %% Styling
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff
    classDef process fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef cache fill:#10b981,stroke:#059669,color:#fff
    classDef found fill:#ef4444,stroke:#dc2626,color:#fff

    class ValidateHistory,CheckSnapshot,ValidateContent,EvaluateScore,SearchLoop decision
    class CalcMid,GetCommit,FetchContent,AnalyzeContent,IncrementDepth process
    class GetCachedScore,StoreSnapshot cache
    class FoundRegression found
```

## Key Design Decisions

### Binary Search Efficiency

1. **Depth Limit**: Maximum 10 iterations prevents excessive git operations
2. **Caching**: Previously analyzed commits reuse stored snapshots
3. **Progressive Search**: Finds the earliest regression, not just any bad commit
4. **Cancellable**: User can cancel long-running searches

### Git Blame Optimization

1. **Batching**: Multiple findings consolidated into range-based blame calls
2. **Cache Duration**: 24-hour TTL balances accuracy vs performance
3. **Porcelain Format**: Structured output easier to parse than default format
4. **Fallback**: Gracefully handles missing or moved files

### Historical Analysis Challenges

1. **File Existence**: File may not exist in old commits
2. **Line Number Drift**: Findings from current analysis may not map to historical lines
3. **Plugin Availability**: Historical analysis uses current plugin versions
4. **Performance**: Full analysis of old commits is expensive (mitigated by caching)

### Storage Considerations

1. **Snapshot Deduplication**: Same file + commit only stored once
2. **Retention Policy**: Configurable cleanup (default: 90 days)
3. **Incremental Growth**: Database grows with each analysis, bounded by retention
4. **Query Performance**: Indexes on file_path + commit_date for fast lookups
