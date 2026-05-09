---
id: 05-extension-lifecycle
title: Extension Lifecycle
sidebar_label: Extension Lifecycle
---

# Extension Lifecycle

This diagram shows the complete lifecycle of the Vipr VSCode extension from activation to deactivation.

## Activation Sequence

```mermaid
---
title: Vipr Extension Activation Sequence
config:
  theme: neutral
---
sequenceDiagram
    accTitle: Extension Activation Sequence
    accDescr: Shows the initialization order and dependencies during extension activation

    participant VSCode as VSCode Host
    participant Extension as Extension (extension.ts)
    participant Config as Config Manager
    participant License as License Validator
    participant Storage as Storage Service
    participant Git as Git History Service
    participant Providers as UI Providers
    participant Commands as Command Registry
    participant Compatibility as Compatibility Layer
    participant AI as AI Features

    VSCode->>Extension: activate(context)
    Note over Extension: Activation Start<br/>Record timestamp

    %% Phase 1: Core initialization
    Extension->>Extension: Create output channel
    Extension->>Compatibility: Detect editor fork
    Compatibility-->>Extension: Editor info (VSCode/Cursor/Windsurf)
    Extension->>Compatibility: Log compatibility info

    Extension->>Config: Initialize ConfigManager
    Config-->>Extension: Config ready

    Extension->>Config: Get performance settings
    Config-->>Extension: Cache settings

    %% Phase 2: Performance components
    Extension->>Extension: Initialize ResultCache
    Extension->>Extension: Initialize MemoryMonitor
    Extension->>Extension: Start memory monitoring

    %% Phase 3: History tracking
    Extension->>Config: Check if history enabled
    Config-->>Extension: History enabled: true

    Extension->>Storage: Initialize StorageService
    Storage->>Storage: Load SQLite WASM
    Storage->>Storage: Create/load database
    Storage->>Storage: Run schema migrations
    Storage-->>Extension: Storage ready

    Extension->>Git: Initialize GitHistoryService
    Git->>Git: Check if git repository
    Git-->>Extension: Git ready (if repo exists)

    alt Is Git Repository
        Extension->>Extension: Initialize RegressionDetector
        Note over Extension: History tracking enabled
    else Not Git Repository
        Note over Extension: History tracking disabled
    end

    %% Phase 4: Core services
    Extension->>License: Initialize LicenseValidator
    Extension->>License: Validate license key
    License-->>Extension: License tier info

    Extension->>Extension: Initialize AnalysisManager
    Extension->>Extension: Create diagnostic provider
    Extension->>Extension: Create decoration provider
    Extension->>Extension: Create code action provider
    Extension->>Extension: Create dashboard provider
    Extension->>Extension: Create file tree provider
    Extension->>Extension: Create status bar
    Extension->>Extension: Create on-save handler

    Note over Extension: Engine and PluginLoader<br/>are LAZY loaded<br/>(not initialized yet)

    %% Phase 5: Register providers
    Extension->>VSCode: Register CodeLens provider
    Extension->>VSCode: Register CodeAction provider
    Extension->>VSCode: Register Dashboard webview
    Extension->>VSCode: Register File Tree view

    %% Phase 6: Register commands
    Extension->>Commands: Register all commands
    Commands->>VSCode: Register command: vipr.analyzeFile
    Commands->>VSCode: Register command: vipr.analyzeWorkspace
    Commands->>VSCode: Register command: vipr.showHistory
    Commands->>VSCode: Register command: vipr.findRegression
    Commands->>VSCode: Register command: vipr.cleanupHistory
    Commands->>VSCode: Register command: vipr.exportReport
    Commands->>VSCode: Register command: vipr.fixWithAI
    Commands-->>Extension: Commands registered

    %% Phase 7: AI features (conditional)
    Extension->>Compatibility: Check AI feature support
    Compatibility-->>Extension: AI supported: true

    Extension->>Compatibility: Initialize AI features
    Compatibility-->>Extension: AI initialized

    Extension->>AI: Register chat participant
    AI-->>Extension: Chat participant registered

    Extension->>AI: Register language model tools
    AI-->>Extension: LM tools registered

    %% Phase 8: Event subscriptions
    Extension->>VSCode: Subscribe to onDidSaveTextDocument
    Extension->>VSCode: Subscribe to onDidChangeActiveTextEditor
    Extension->>VSCode: Subscribe to onDidChangeConfiguration
    VSCode-->>Extension: Event subscriptions active

    %% Phase 9: Initial auto-analysis
    Extension->>VSCode: Get active editor
    VSCode-->>Extension: Active editor (if any)

    alt Has Active Editor
        Extension->>Extension: Check cache for editor
        Extension->>Extension: Trigger background analysis
    end

    %% Phase 10: Activation complete
    Note over Extension: Calculate activation time
    Extension->>Extension: Log performance stats
    Extension->>Compatibility: Show welcome message
    Compatibility-->>VSCode: Display toast notification

    VSCode-->>Extension: Activation complete
```

## First Analysis Flow (Lazy Loading)

```mermaid
---
title: First Analysis Triggers Lazy Initialization
config:
  theme: neutral
---
sequenceDiagram
    accTitle: First Analysis with Lazy Loading
    accDescr: Shows how the first analysis run triggers lazy initialization of heavy components

    actor User
    participant Editor as VSCode Editor
    participant Command as Analyze Command
    participant Extension as Extension State
    participant LazyLoader as Lazy Loader
    participant Engine as Analysis Engine
    participant PluginLoader as Plugin Loader
    participant Worker as Worker Manager

    User->>Editor: Save file or run command
    Editor->>Command: Trigger analysis
    Command->>Extension: getOrCreateEngine()

    Extension->>Extension: Check if engine exists
    Note over Extension: engine = undefined<br/>(not yet initialized)

    Extension->>Extension: Log lazy load start
    Extension->>Engine: new AnalysisEngine()
    Note over Engine: Engine initialization<br/>~100-200ms

    Extension->>PluginLoader: new VscodePluginLoader()
    Extension->>PluginLoader: loadBundledPlugins()
    PluginLoader->>PluginLoader: Discover plugins
    PluginLoader->>PluginLoader: Load core analyzer
    PluginLoader->>PluginLoader: Load React analyzer
    PluginLoader-->>Extension: Plugins loaded

    Extension->>Extension: Store engine reference
    Extension->>Extension: Log initialization time
    Extension-->>Command: Return engine

    Command->>Engine: analyzeFile(content)
    Engine->>Engine: Check if should use worker
    Engine->>Worker: Initialize worker thread
    Worker->>Worker: Load analysis-worker.ts
    Worker-->>Engine: Worker ready

    Engine->>Worker: Post message: analyze
    Worker->>Worker: Run analyzers
    Worker->>Worker: Aggregate results
    Worker-->>Engine: Analysis complete

    Engine-->>Command: AggregatedResult
    Command->>Command: Update providers
    Command->>Command: Save snapshot
    Command-->>User: Show results
```

## Runtime Event Handling

```mermaid
---
title: Runtime Event Handling
config:
  theme: neutral
---
flowchart TD
    accTitle: Extension Runtime Event Handling
    accDescr: Shows how the extension responds to various VSCode events during runtime

    subgraph Events["VSCode Events"]
        SaveEvent([onDidSaveTextDocument])
        EditorChange([onDidChangeActiveTextEditor])
        ConfigChange([onDidChangeConfiguration])
        PeriodicCleanup([Periodic Timer<br/>Every 5 minutes])
    end

    subgraph SaveHandler["Save Event Handler"]
        SaveEvent --> CheckAutoAnalyze{Auto-analyze<br/>enabled?}
        CheckAutoAnalyze -->|Yes| ComputeHash[Compute content hash]
        CheckAutoAnalyze -->|No| SkipAnalysis[Skip analysis]

        ComputeHash --> CheckCache{Result<br/>cached?}
        CheckCache -->|Yes| SkipAnalysis
        CheckCache -->|No| TriggerAnalysis[Trigger analysis]

        TriggerAnalysis --> SaveSnapshot[Save snapshot to DB]
        SaveSnapshot --> UpdateProviders[Update all providers]
        UpdateProviders --> Done1([Done])
        SkipAnalysis --> Done1
    end

    subgraph EditorHandler["Active Editor Change"]
        EditorChange --> HasEditor{Has active<br/>editor?}
        HasEditor -->|No| ResetStatusBar[Reset status bar]
        ResetStatusBar --> ClearHistory[Clear history context]
        ClearHistory --> Done2([Done])

        HasEditor -->|Yes| CheckEditorCache{Cached result<br/>exists?}
        CheckEditorCache -->|Yes| ShowCached[Show cached results]
        ShowCached --> UpdateHistory[Update history context]
        UpdateHistory --> Done2

        CheckEditorCache -->|No| BackgroundAnalysis[Trigger background analysis]
        BackgroundAnalysis --> Done2
    end

    subgraph ConfigHandler["Configuration Change"]
        ConfigChange --> WhichConfig{Which setting<br/>changed?}

        WhichConfig -->|License Key| ValidateNewKey[Validate new license]
        ValidateNewKey --> LogChange[Log license tier]
        LogChange --> Done3([Done])

        WhichConfig -->|Show Decorations| ToggleDecorations{Enabled?}
        ToggleDecorations -->|Yes| ReapplyDecorations[Reapply decorations]
        ToggleDecorations -->|No| ClearDecorations[Clear all decorations]
        ReapplyDecorations --> Done3
        ClearDecorations --> Done3

        WhichConfig -->|Other| Done3
    end

    subgraph CleanupHandler["Periodic Cleanup"]
        PeriodicCleanup --> CleanResultCache[Clean result cache]
        CleanResultCache --> CheckMemory{Memory usage<br/>high?}

        CheckMemory -->|Yes| AggressiveCleanup[Aggressive cleanup]
        AggressiveCleanup --> TerminateWorker[Terminate worker]
        TerminateWorker --> ClearLazyCache[Clear lazy loader]
        ClearLazyCache --> Done4([Done])

        CheckMemory -->|No| Done4
    end

    %% Styling
    classDef event fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff
    classDef action fill:#10b981,stroke:#059669,color:#fff
    classDef cleanup fill:#ef4444,stroke:#dc2626,color:#fff

    class SaveEvent,EditorChange,ConfigChange,PeriodicCleanup event
    class CheckAutoAnalyze,CheckCache,HasEditor,CheckEditorCache,WhichConfig,ToggleDecorations,CheckMemory decision
    class TriggerAnalysis,UpdateProviders,BackgroundAnalysis,ReapplyDecorations action
    class AggressiveCleanup,TerminateWorker,ClearLazyCache cleanup
```

## Deactivation Sequence

```mermaid
---
title: Extension Deactivation Sequence
config:
  theme: neutral
---
sequenceDiagram
    accTitle: Extension Deactivation Sequence
    accDescr: Shows the cleanup and disposal process when the extension is deactivated

    participant VSCode as VSCode Host
    participant Extension as Extension (extension.ts)
    participant MemoryMonitor as Memory Monitor
    participant Worker as Worker Manager
    participant Cache as Result Cache
    participant Storage as Storage Service
    participant LazyLoader as Lazy Loader

    VSCode->>Extension: deactivate()
    Extension->>Extension: Log deactivation start

    %% Phase 1: Stop monitoring
    Extension->>MemoryMonitor: Stop monitoring
    MemoryMonitor->>MemoryMonitor: Clear interval
    MemoryMonitor-->>Extension: Stopped

    %% Phase 2: Cleanup workers
    Extension->>Worker: Dispose worker manager
    Worker->>Worker: Terminate active worker
    Worker->>Worker: Clear message queue
    Worker-->>Extension: Disposed

    %% Phase 3: Log cache stats
    Extension->>Cache: Get cache statistics
    Cache-->>Extension: hits, misses, hit rate
    Extension->>Extension: Log cache performance

    %% Phase 4: Close database
    Extension->>Storage: Dispose storage service
    Storage->>Storage: Save pending changes
    Storage->>Storage: Close SQLite database
    Storage->>Storage: Clear cache
    Storage-->>Extension: Disposed

    %% Phase 5: Clear lazy loader
    Extension->>LazyLoader: Clear all cached modules
    LazyLoader->>LazyLoader: Release module references
    LazyLoader-->>Extension: Cleared

    %% Phase 6: Clear state
    Extension->>Extension: Set state = undefined
    Extension->>Extension: Log deactivation complete

    Extension-->>VSCode: Deactivation complete
```

## Memory Management

```mermaid
---
title: Memory Management and Cleanup
config:
  theme: neutral
---
flowchart TD
    accTitle: Memory Management Strategy
    accDescr: Shows how the extension manages memory and performs cleanup

    Start([Extension Running]) --> Monitor{Memory Monitor<br/>Enabled?}

    Monitor -->|Yes| CheckInterval[Check Every 5 Minutes]
    Monitor -->|No| Skip[No Monitoring]

    CheckInterval --> GetHeapUsage[Get Heap Usage]
    GetHeapUsage --> CalculateThreshold{Heap Used ><br/>Threshold?}

    CalculateThreshold -->|Below| LogStats[Log Memory Stats]
    LogStats --> Wait[Wait 5 Minutes]
    Wait --> CheckInterval

    CalculateThreshold -->|Above| TriggerCleanup[Trigger Cleanup]
    TriggerCleanup --> Phase1[Phase 1: Clear Result Cache]

    Phase1 --> Phase2[Phase 2: Clear Lazy Loader]
    Phase2 --> Phase3[Phase 3: Terminate Worker]
    Phase3 --> Phase4[Phase 4: Force GC<br/>if available]

    Phase4 --> Recheck[Recheck Memory]
    Recheck --> Improved{Memory<br/>Released?}

    Improved -->|Yes| LogSuccess[Log Success]
    LogSuccess --> Wait

    Improved -->|No| LogWarning[Log Warning]
    LogWarning --> NotifyUser[Show Warning Toast]
    NotifyUser --> Wait

    %% User-triggered cleanup
    UserCleanup([User Runs vipr.internal.cleanup]) --> ManualPhase1[Clear Result Cache]
    ManualPhase1 --> ManualPhase2[Clear Lazy Loader]
    ManualPhase2 --> ManualPhase3[Terminate Worker]
    ManualPhase3 --> ShowComplete[Show Completion Message]
    ShowComplete --> Completed([Done])

    %% Styling
    classDef monitor fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff
    classDef cleanup fill:#ef4444,stroke:#dc2626,color:#fff
    classDef success fill:#10b981,stroke:#059669,color:#fff

    class CheckInterval,GetHeapUsage monitor
    class CalculateThreshold,Improved decision
    class TriggerCleanup,Phase1,Phase2,Phase3,Phase4 cleanup
    class LogSuccess,ShowComplete success
```

## Lifecycle State Transitions

```mermaid
---
title: Extension State Machine
config:
  theme: neutral
---
stateDiagram-v2
    accTitle: Extension Lifecycle State Machine
    accDescr: Shows the state transitions during the extension lifecycle

    [*] --> NotActivated: Extension installed

    NotActivated --> Activating: VSCode activates extension

    Activating --> InitializingCore: Load core services
    InitializingCore --> InitializingHistory: Initialize storage & git
    InitializingHistory --> RegisteringProviders: Setup UI providers
    RegisteringProviders --> RegisteringCommands: Register commands
    RegisteringCommands --> RegisteringAI: Setup AI features (if available)
    RegisteringAI --> Active: Activation complete

    Active --> LazyLoadingEngine: First analysis triggered
    LazyLoadingEngine --> Active: Engine initialized

    Active --> ProcessingAnalysis: Analysis running
    ProcessingAnalysis --> Active: Analysis complete

    Active --> ShowingHistory: User views history
    ShowingHistory --> Active: Panel closed

    Active --> DetectingRegression: User finds regression
    DetectingRegression --> Active: Search complete

    Active --> PerformingCleanup: Memory threshold exceeded
    PerformingCleanup --> Active: Cleanup complete

    Active --> Deactivating: VSCode deactivates

    Deactivating --> StoppingMonitors: Stop memory monitor
    StoppingMonitors --> DisposingWorkers: Terminate workers
    DisposingWorkers --> ClosingStorage: Close database
    ClosingStorage --> ClearingState: Clear all state
    ClearingState --> [*]: Deactivated

    note right of NotActivated
        Extension is installed
        but not yet activated
    end note

    note right of Active
        Main operational state
        Ready to handle events
    end note

    note right of LazyLoadingEngine
        First analysis triggers
        lazy initialization of
        heavy components
    end note

    note right of PerformingCleanup
        Automatic cleanup when
        memory usage is high
    end note
```

## Performance Metrics

The extension tracks several performance metrics during its lifecycle:

### Activation Time Breakdown

| Phase                 | Target        | Typical       | Notes                                  |
| --------------------- | ------------- | ------------- | -------------------------------------- |
| Core initialization   | &lt;50ms      | 30-40ms       | Config, License, Storage               |
| Provider registration | &lt;30ms      | 20-25ms       | Register all UI providers              |
| Command registration  | &lt;20ms      | 10-15ms       | Register all commands                  |
| AI feature setup      | &lt;50ms      | 30-40ms       | Conditional, only if Copilot available |
| **Total Activation**  | **&lt;150ms** | **100-120ms** | Without lazy-loaded components         |

### First Analysis Time

| Component                | Target        | Typical       | Notes                           |
| ------------------------ | ------------- | ------------- | ------------------------------- |
| Lazy load engine         | &lt;200ms     | 150-180ms     | One-time cost on first analysis |
| Plugin loading           | &lt;100ms     | 80-100ms      | Load core + React analyzers     |
| Worker initialization    | &lt;50ms      | 30-40ms       | Create worker thread            |
| Actual analysis          | Varies        | 100-500ms     | Depends on file size/complexity |
| **Total First Analysis** | **&lt;500ms** | **400-600ms** | Subsequent analyses much faster |

### Runtime Performance

| Operation               | Target    | Typical   | Notes                      |
| ----------------------- | --------- | --------- | -------------------------- |
| Cached result retrieval | &lt;5ms   | 2-3ms     | Content hash lookup        |
| Background analysis     | &lt;200ms | 100-150ms | For small-medium files     |
| History snapshot save   | &lt;50ms  | 30-40ms   | SQLite insert              |
| Git blame lookup        | &lt;100ms | 50-80ms   | With caching               |
| Regression detection    | Varies    | 2-5s      | Binary search with caching |

## Key Lifecycle Optimizations

### 1. Lazy Initialization

- Analysis Engine not created until first analysis
- Worker threads spawned on demand
- Reduces initial activation time by ~60%

### 2. Deferred Database Connections

- SQLite database opened during activation
- But migrations run lazily
- Reduces activation blocking

### 3. Progressive Feature Registration

- Core features registered immediately
- AI features registered conditionally
- Non-critical subscriptions deferred

### 4. Memory Monitoring

- Automatic cleanup at threshold
- Proactive worker termination
- Lazy loader cache clearing

### 5. Graceful Shutdown

- Pending snapshots flushed
- Database closed cleanly
- Workers terminated properly
- No zombie processes
