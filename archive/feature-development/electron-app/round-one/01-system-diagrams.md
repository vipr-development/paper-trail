---
id: 01-system-diagrams
---

# system diagrams

comprehensive mermaid diagrams for electron desktop application architecture

---

## overview architecture

```mermaid
---
title: vipr desktop application - component architecture
config:
  theme: base
  layout: elk
---
graph TB
    accTitle: Vipr Desktop Application Component Architecture
    accDescr: Shows the complete architecture of the Vipr Electron desktop app including main process, renderer process, analysis engine, plugins, and data persistence

    subgraph electron["Electron Desktop App"]
        subgraph main["Main Process (Node.js)"]
            mainEntry[Main Entry Point]
            ipcHandlers[IPC Handlers]
            dbManager[SQLite Manager]
            fsWatcher[File Watcher<br/>chokidar]
            analysisCoord[Analysis Coordinator]
        end

        subgraph preload["Preload Scripts"]
            contextBridge[Context Bridge API]
        end

        subgraph renderer["Renderer Process (React)"]
            reactApp[React Application]
            dashboard[Dashboard Views]
            fileDetail[File Detail Views]
            settings[Settings UI]
            reports[Report Generator]
        end
    end

    subgraph engine["@vipr/engine"]
        analysisEngine[Analysis Engine]
        engineCache[Engine Cache]
        pluginOrch[Plugin Orchestrator]
    end

    subgraph plugins["Analyzer Plugins"]
        corePlugin["@vipr/core<br/>Cyclomatic/Halstead/MI"]
        reactPlugin["@vipr/react<br/>Component/Hooks/A11y"]
        nextjsPlugin["@vipr/nextjs<br/>App Router/RSC"]
    end

    subgraph common["@vipr/common"]
        types[Shared Types]
        presenterReg[Presenter Registry]
        utils[Utilities]
    end

    subgraph loader["@vipr/plugin-loader"]
        workspaceScanner[Workspace Scanner]
        dynamicLoader[Dynamic Loader]
        pluginRegistry[Plugin Registry]
    end

    subgraph persistence["Data Layer"]
        sqlite[(SQLite DB<br/>better-sqlite3)]
        fileSchema[files table]
        analysisSchema[analyses table]
        snapshotSchema[snapshots table]
        notesSchema[notes table]
    end

    subgraph mcp["MCP Server (Optional)"]
        mcpServer[@vipr/mcp-analyzer]
        mcpTools[Analysis Tools]
    end

    %% Main connections
    mainEntry --> ipcHandlers
    mainEntry --> dbManager
    mainEntry --> fsWatcher
    mainEntry --> analysisCoord

    ipcHandlers <-.IPC.-> contextBridge
    contextBridge <-.Secure API.-> reactApp

    reactApp --> dashboard
    reactApp --> fileDetail
    reactApp --> settings
    reactApp --> reports

    analysisCoord --> analysisEngine
    analysisCoord --> dbManager

    analysisEngine --> pluginOrch
    analysisEngine --> engineCache

    pluginOrch --> corePlugin
    pluginOrch --> reactPlugin
    pluginOrch --> nextjsPlugin

    corePlugin -.uses.-> common
    reactPlugin -.uses.-> common
    nextjsPlugin -.uses.-> common

    analysisEngine -.loads via.-> loader

    dbManager --> sqlite
    sqlite --> fileSchema
    sqlite --> analysisSchema
    sqlite --> snapshotSchema
    sqlite --> notesSchema

    fsWatcher -.file changes.-> analysisCoord

    mainEntry -.optional.-> mcpServer
    mcpServer --> mcpTools
    mcpTools --> sqlite

    classDef processNode fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef engineNode fill:#8b5cf6,stroke:#6d28d9,color:#fff
    classDef pluginNode fill:#10b981,stroke:#059669,color:#fff
    classDef dataNode fill:#f59e0b,stroke:#d97706,color:#fff
    classDef mcpNode fill:#ec4899,stroke:#be185d,color:#fff

    class mainEntry,ipcHandlers,dbManager,fsWatcher,analysisCoord,contextBridge,reactApp processNode
    class analysisEngine,engineCache,pluginOrch engineNode
    class corePlugin,reactPlugin,nextjsPlugin pluginNode
    class sqlite,fileSchema,analysisSchema,snapshotSchema,notesSchema dataNode
    class mcpServer,mcpTools mcpNode
```

---

## analysis pipeline flow

```mermaid
---
title: analysis pipeline data flow
config:
  theme: base
---
sequenceDiagram
    accTitle: Analysis Pipeline Data Flow
    accDescr: Shows the complete flow from user opening a repository through analysis execution to UI presentation

    actor User
    participant UI as Renderer UI
    participant IPC as IPC Bridge
    participant Main as Main Process
    participant FS as File Watcher
    participant Coord as Analysis Coordinator
    participant DB as SQLite Manager
    participant Engine as Analysis Engine
    participant Loader as Plugin Loader
    participant Plugins as Analyzer Plugins
    participant Cache as Cache Layer

    User->>+UI: Open Repository
    UI->>+IPC: openRepository(path)
    IPC->>+Main: Handle Request

    Main->>+FS: Start Watching
    Main->>+Coord: Initialize Analysis

    Coord->>+DB: Check Existing Data
    DB->>DB: Query files table

    alt Fresh Repository
        DB-->>Coord: No Data
        Coord->>+Engine: Full Analysis Required
    else Cached Repository
        DB-->>Coord: Cached Results
        Coord->>Coord: Validate Cache (SHA)
        Coord->>+Engine: Incremental Analysis
    end

    Engine->>+Loader: Load Plugins
    Loader->>Loader: Scan Workspace
    Loader->>Loader: Validate Packages
    Loader->>Loader: Dynamic Import
    Loader-->>-Engine: Plugin Instances

    Engine->>Engine: Register Plugins
    Engine->>Engine: Get File List

    loop For Each File
        Engine->>+Cache: Check Cache

        alt Cache Hit
            Cache-->>Engine: Cached Result
        else Cache Miss
            Engine->>+Plugins: canHandle(file)
            Plugins-->>-Engine: Applicable Plugins

            par Parallel Execution
                Engine->>Plugins: core.analyze()
                Engine->>Plugins: react.analyze()
                Engine->>Plugins: nextjs.analyze()
            end

            Plugins-->>Engine: Plugin Results
            Engine->>Engine: Aggregate Results
            Engine->>-Cache: Store Result
        end

        Engine->>+Coord: File Result
        Coord->>+DB: Persist Analysis
        DB->>DB: INSERT/UPDATE files
        DB->>DB: INSERT analyses
        DB-->>-Coord: Success

        Coord->>IPC: Progress Update
        IPC->>UI: Update Progress Bar
    end

    Engine-->>-Coord: Complete
    Coord->>+DB: Create Snapshot
    DB->>DB: INSERT snapshot (git SHA)
    DB-->>-Coord: Snapshot ID

    Coord-->>-Main: Analysis Complete
    Main->>IPC: analysisComplete(results)
    IPC-->>-UI: Display Results
    UI-->>-User: Show Dashboard

    FS->>FS: Detect File Change
    FS->>Coord: File Modified Event
    Coord->>Engine: Re-analyze Single File
    Engine->>Plugins: Incremental Analysis
    Plugins-->>Engine: Updated Result
    Engine->>Coord: Updated Result
    Coord->>DB: Update Analysis
    Coord->>IPC: File Update Event
    IPC->>UI: Refresh UI
```

---

## electron process architecture

```mermaid
---
title: electron three-process model
config:
  theme: base
---
graph TB
    accTitle: Electron Three-Process Architecture
    accDescr: Shows the Electron main process, renderer process, preload script, and their security boundaries

    subgraph security["Security Boundary"]
        subgraph main["Main Process (Node.js)<br/>Full System Access"]
            direction TB
            mainApp[app.ts<br/>Entry Point]
            windowMgr[Window Manager]
            ipcMain[IPC Main Handlers]

            subgraph mainModules["Core Modules"]
                direction LR
                sqliteDb[db/database.ts<br/>SQLite Operations]
                fileWatcher[fs/watcher.ts<br/>File Monitoring]
                analysisRunner[analysis/runner.ts<br/>Engine Coordinator]
                mcpManager[mcp/server.ts<br/>MCP Server]
            end
        end

        subgraph preloadLayer["Preload Scripts<br/>Context Bridge"]
            preloadApi[preload/index.ts<br/>Exposed API]
            apiDef[API Definitions]
        end

        subgraph renderer["Renderer Process (Chromium)<br/>Sandboxed"]
            direction TB
            reactRoot[App.tsx<br/>React Root]

            subgraph rendererModules["UI Modules"]
                direction LR
                pages[pages/<br/>Route Components]
                components[components/<br/>Shared UI]
                hooks[hooks/<br/>Custom Hooks]
                stores[stores/<br/>State Management]
            end

            electronApi[window.electron API]
        end
    end

    %% Process connections
    mainApp --> windowMgr
    windowMgr --> ipcMain

    ipcMain <-.IPC Channel.-> preloadApi

    preloadApi --> apiDef
    apiDef -."contextBridge.expose".-> electronApi

    electronApi --> reactRoot
    reactRoot --> pages
    reactRoot --> components
    reactRoot --> hooks
    reactRoot --> stores

    %% Main process internal
    ipcMain --> sqliteDb
    ipcMain --> fileWatcher
    ipcMain --> analysisRunner
    ipcMain --> mcpManager

    fileWatcher -.file events.-> analysisRunner
    analysisRunner --> sqliteDb

    %% Security annotations
    note1[/"Node.js APIs<br/>File System<br/>SQLite<br/>Analysis Engine"/]
    note2[/"Secure Bridge<br/>No Direct Access<br/>Type-Safe API"/]
    note3[/"Web APIs Only<br/>No Node.js Access<br/>Content Security Policy"/]

    mainModules -.-> note1
    preloadApi -.-> note2
    rendererModules -.-> note3

    classDef mainClass fill:#ef4444,stroke:#b91c1c,color:#fff
    classDef preloadClass fill:#f59e0b,stroke:#d97706,color:#fff
    classDef rendererClass fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef noteClass fill:#6b7280,stroke:#4b5563,color:#fff

    class mainApp,windowMgr,ipcMain,sqliteDb,fileWatcher,analysisRunner,mcpManager mainClass
    class preloadApi,apiDef preloadClass
    class reactRoot,pages,components,hooks,stores,electronApi rendererClass
    class note1,note2,note3 noteClass
```

---

## sqlite persistence schema

```mermaid
---
title: sqlite database schema
config:
  theme: base
---
erDiagram
    accTitle: SQLite Database Schema
    accDescr: Shows the complete database schema with tables for files, analyses, snapshots, notes, and exclusions

    FILES {
        INTEGER id PK
        TEXT path UK "Unique file path"
        TEXT sha "Content SHA-256"
        INTEGER analyzed_at "Unix timestamp"
        TEXT git_sha "Git commit SHA"
        TEXT git_author "Git commit author"
        INTEGER git_date "Git commit timestamp"
        TEXT file_type "Detected file type"
        JSON technologies "Array of technologies"
        INTEGER created_at
        INTEGER updated_at
    }

    ANALYSES {
        INTEGER id PK
        INTEGER file_id FK
        TEXT plugin_id "Plugin identifier"
        INTEGER score "0-100 quality score"
        JSON result "Full PluginResult JSON"
        JSON insights "Array of insights"
        JSON metrics "Plugin-specific metrics"
        INTEGER execution_time_ms
        INTEGER created_at
    }

    SNAPSHOTS {
        INTEGER id PK
        TEXT git_sha UK "Git commit SHA"
        TEXT git_author
        TEXT git_message
        INTEGER git_date
        INTEGER file_count
        REAL avg_score "Average quality score"
        JSON summary "Aggregate metrics"
        INTEGER created_at
    }

    SNAPSHOT_FILES {
        INTEGER id PK
        INTEGER snapshot_id FK
        INTEGER file_id FK
        INTEGER overall_score
        JSON plugin_results "Map of plugin results"
    }

    NOTES {
        INTEGER id PK
        TEXT target_type "file | issue | abstraction"
        TEXT target_id "Composite key"
        TEXT content "User note text"
        INTEGER created_at
        INTEGER updated_at
    }

    EXCLUSIONS {
        INTEGER id PK
        TEXT issue_type "Type of excluded issue"
        TEXT file_path "Optional file path"
        TEXT plugin_id "Optional plugin filter"
        TEXT reason "User-provided reason"
        INTEGER created_at
    }

    METADATA {
        TEXT key PK
        TEXT value "JSON-encoded value"
        INTEGER updated_at
    }

    FILES ||--o{ ANALYSES : "has many"
    FILES ||--o{ SNAPSHOT_FILES : "captured in"
    SNAPSHOTS ||--o{ SNAPSHOT_FILES : "contains"
    FILES ||--o{ NOTES : "annotated by"
```

---

## plugin loading and coordination

```mermaid
---
title: plugin loading and coordination flow
config:
  theme: base
---
flowchart TD
    accTitle: Plugin Loading and Coordination Flow
    accDescr: Shows how plugins are discovered, loaded, validated, and coordinated during analysis

    Start([Application Start]) --> InitEngine[Initialize Analysis Engine]
    InitEngine --> CreateLoader[Create Plugin Loader]

    CreateLoader --> ScanWS[Workspace Scanner:<br/>Find analyzer packages]
    ScanWS --> FoundPkgs{Packages Found?}

    FoundPkgs -->|No| Error1[Error: No Analyzers]
    FoundPkgs -->|Yes| ValidatePkgs[Plugin Validator:<br/>Check package.json]

    ValidatePkgs --> ValidPkgs{Valid?}
    ValidPkgs -->|No| Error2[Error: Invalid Package]
    ValidPkgs -->|Yes| DynamicLoad[Dynamic Loader:<br/>Import ES modules]

    DynamicLoad --> Instantiate[Plugin Instantiator:<br/>Create instances]
    Instantiate --> Register[Register with Engine]

    Register --> RegPresenters[Register Presenters<br/>with Presenter Registry]

    %% File analysis flow
    RegPresenters --> AnalyzeFile[Analyze File Request]
    AnalyzeFile --> GetSource[Get SourceFile from ts-morph]

    GetSource --> FindPlugins[Find Applicable Plugins]
    FindPlugins --> CheckCore{Core Plugin<br/>canHandle?}

    CheckCore -->|Yes| CoreRun[Core: Always runs<br/>for .ts/.tsx/.js/.jsx]
    CheckCore -->|No| SkipCore[Skip Core]

    CoreRun --> CheckReact{React Plugin<br/>canHandle?}
    SkipCore --> CheckReact

    CheckReact -->|Has React imports| CheckNextJS{Next.js Indicators?}
    CheckReact -->|No React| SkipReact[Skip React]

    CheckNextJS -->|Yes: app/, pages/,<br/>Next.js imports| NextJSRun[Next.js: Runs<br/>React defers]
    CheckNextJS -->|No: Pure React| ReactRun[React: Runs]

    NextJSRun --> ParallelExec[Parallel Execution]
    ReactRun --> ParallelExec
    SkipReact --> ParallelExec

    ParallelExec --> RunCore[Core.getAnalyses]
    ParallelExec --> RunReact[React.getAnalyses]
    ParallelExec --> RunNextJS[NextJS.getAnalyses]

    RunCore --> CoreAnalyses[Cyclomatic<br/>Halstead<br/>Maintainability]
    RunReact --> ReactAnalyses[Structural<br/>Hooks<br/>Performance<br/>A11y<br/>Security]
    RunNextJS --> NextJSAnalyses[App Router<br/>Server Components<br/>Data Fetching]

    CoreAnalyses --> Aggregate[Aggregate Results]
    ReactAnalyses --> Aggregate
    NextJSAnalyses --> Aggregate

    Aggregate --> CalcScore[Calculate Composite Score]
    CalcScore --> MergeInsights[Merge Insights]
    MergeInsights --> DetectFileType[Detect File Type<br/>from Plugin Results]

    DetectFileType --> Result([Aggregated Result])

    classDef startEnd fill:#10b981,stroke:#059669,color:#fff
    classDef process fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff
    classDef error fill:#ef4444,stroke:#b91c1c,color:#fff
    classDef plugin fill:#8b5cf6,stroke:#6d28d9,color:#fff

    class Start,Result startEnd
    class InitEngine,CreateLoader,ScanWS,ValidatePkgs,DynamicLoad,Instantiate,Register,RegPresenters,AnalyzeFile,GetSource,FindPlugins,ParallelExec,Aggregate,CalcScore,MergeInsights,DetectFileType process
    class FoundPkgs,ValidPkgs,CheckCore,CheckReact,CheckNextJS decision
    class Error1,Error2 error
    class CoreRun,ReactRun,NextJSRun,RunCore,RunReact,RunNextJS,CoreAnalyses,ReactAnalyses,NextJSAnalyses plugin
```

---

## file watching and incremental analysis

```mermaid
---
title: file watching and incremental analysis
config:
  theme: base
---
stateDiagram-v2
    accTitle: File Watching State Machine
    accDescr: Shows the state transitions for file watching and incremental analysis in the desktop app

    [*] --> Idle: App Start

    Idle --> Initializing: Open Repository
    Initializing --> Scanning: Start Watcher

    Scanning --> WatchActive: Initial Scan Complete

    state WatchActive {
        [*] --> Monitoring

        Monitoring --> FileChanged: File Modified/Created
        Monitoring --> FileDeleted: File Deleted
        Monitoring --> DirectoryChanged: Directory Event

        FileChanged --> Debouncing: Queue Change
        Debouncing --> Debouncing: More Changes (< 500ms)
        Debouncing --> ValidatingChange: Stable (> 500ms)

        ValidatingChange --> CheckingSHA: Read File
        CheckingSHA --> NoChange: SHA Match
        CheckingSHA --> AnalysisNeeded: SHA Different

        NoChange --> Monitoring

        AnalysisNeeded --> Analyzing: Run Engine

        state Analyzing {
            [*] --> LoadFromCache
            LoadFromCache --> CacheHit: Valid Cache
            LoadFromCache --> ExecutePlugins: Cache Miss

            CacheHit --> [*]

            ExecutePlugins --> RunCorePlugin
            RunCorePlugin --> RunReactPlugin
            RunReactPlugin --> RunNextJSPlugin
            RunNextJSPlugin --> AggregatePluginResults
            AggregatePluginResults --> [*]
        }

        Analyzing --> UpdatingDatabase: Analysis Complete
        UpdatingDatabase --> UpdatedFiles: Update files table
        UpdatedFiles --> UpdatedAnalyses: Update analyses table
        UpdatedAnalyses --> NotifyUI: Emit Event
        NotifyUI --> Monitoring

        FileDeleted --> RemovingRecord: Delete from DB
        RemovingRecord --> NotifyUI

        DirectoryChanged --> RescanDirectory: Check for new files
        RescanDirectory --> Monitoring
    }

    WatchActive --> Paused: User Pauses
    Paused --> WatchActive: User Resumes

    WatchActive --> Stopping: Close Repository
    Stopping --> Cleanup: Stop Watcher
    Cleanup --> [*]

    note right of Debouncing
        Prevents analysis thrashing
        during rapid edits
        (e.g., auto-save)
    end note

    note right of CheckingSHA
        SHA comparison avoids
        re-analyzing identical
        content after file moves
    end note

    note right of LoadFromCache
        Two-tier cache:
        1. Engine cache (in-memory)
        2. SQLite cache (persistent)
    end note
```

---

## mcp server integration

```mermaid
---
title: mcp server integration architecture
config:
  theme: base
---
graph LR
    accTitle: MCP Server Integration Architecture
    accDescr: Shows how the optional MCP server integrates with the desktop app and external IDEs

    subgraph desktop["Desktop App (Main Process)"]
        mainProc[Main Process]
        dbConn[SQLite Connection]
        mcpConfig[MCP Configuration]
    end

    subgraph mcpServer["@vipr/mcp-analyzer Server"]
        direction TB
        serverInit[Server Initialization]
        toolRegistry[Tool Registry]

        subgraph tools["Analysis Tools"]
            getFileAnalysis[get_file_analysis]
            searchIssues[search_issues]
            getRecommendations[get_recommendations]
            getSnapshot[get_snapshot]
            compareSnapshots[compare_snapshots]
        end

        stdioTransport[stdio Transport]
    end

    subgraph ide["External IDE"]
        claudeCode[Claude Code]
        cursor[Cursor]
        otherMCP[Other MCP Clients]
    end

    subgraph sqliteData["SQLite Database"]
        filesTable[files table]
        analysesTable[analyses table]
        snapshotsTable[snapshots table]
        notesTable[notes table]
    end

    %% Connections
    mainProc -->|Optional| mcpConfig
    mcpConfig -->|Start/Stop| serverInit

    serverInit --> toolRegistry
    toolRegistry --> getFileAnalysis
    toolRegistry --> searchIssues
    toolRegistry --> getRecommendations
    toolRegistry --> getSnapshot
    toolRegistry --> compareSnapshots

    serverInit --> stdioTransport

    stdioTransport <-.JSON-RPC.-> claudeCode
    stdioTransport <-.JSON-RPC.-> cursor
    stdioTransport <-.JSON-RPC.-> otherMCP

    getFileAnalysis --> dbConn
    searchIssues --> dbConn
    getRecommendations --> dbConn
    getSnapshot --> dbConn
    compareSnapshots --> dbConn

    dbConn --> filesTable
    dbConn --> analysesTable
    dbConn --> snapshotsTable
    dbConn --> notesTable

    mainProc -.writes.-> dbConn

    classDef desktopClass fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef mcpClass fill:#ec4899,stroke:#be185d,color:#fff
    classDef toolClass fill:#8b5cf6,stroke:#6d28d9,color:#fff
    classDef ideClass fill:#10b981,stroke:#059669,color:#fff
    classDef dataClass fill:#f59e0b,stroke:#d97706,color:#fff

    class mainProc,dbConn,mcpConfig desktopClass
    class serverInit,toolRegistry,stdioTransport mcpClass
    class getFileAnalysis,searchIssues,getRecommendations,getSnapshot,compareSnapshots toolClass
    class claudeCode,cursor,otherMCP ideClass
    class filesTable,analysesTable,snapshotsTable,notesTable dataClass
```

---

## presenter and report generation

```mermaid
---
title: presenter registry and report generation
config:
  theme: base
---
sequenceDiagram
    accTitle: Presenter Registry and Report Generation Flow
    accDescr: Shows how presenters are registered by plugins and used to generate reports in various formats

    participant Engine as Analysis Engine
    participant CorePlugin as Core Plugin
    participant ReactPlugin as React Plugin
    participant NextJSPlugin as Next.js Plugin
    participant Registry as Presenter Registry
    participant UI as Desktop UI
    participant ReportGen as Report Generator
    participant PDF as PDF Exporter

    Note over Engine,Registry: Plugin Registration Phase

    Engine->>CorePlugin: Initialize
    CorePlugin->>CorePlugin: createCorePresenters()
    CorePlugin->>Registry: register(CoreComplexityPresenter)
    CorePlugin->>Registry: register(CoreMetricsPresenter)

    Engine->>ReactPlugin: Initialize
    ReactPlugin->>ReactPlugin: createReactPresenters()
    ReactPlugin->>Registry: register(ReactComponentPresenter)
    ReactPlugin->>Registry: register(ReactHooksPresenter)
    ReactPlugin->>Registry: register(ReactA11yPresenter)

    Engine->>NextJSPlugin: Initialize
    NextJSPlugin->>NextJSPlugin: createNextJSPresenters()
    NextJSPlugin->>Registry: register(NextJSRouterPresenter)
    NextJSPlugin->>Registry: register(NextJSRSCPresenter)

    Note over UI,Registry: Report Discovery Phase

    UI->>Registry: getAvailableReports()
    Registry->>Registry: Collect all presenter metadata
    Registry->>Registry: Sort by order field
    Registry-->>UI: IReportMetadata[]<br/>[{label, hint, icon, categories}]

    UI->>UI: Render report selector<br/>(using metadata)

    Note over UI,PDF: Report Generation Phase

    UI->>ReportGen: Generate Report
    ReportGen->>Registry: get(pluginId, reportType)
    Registry-->>ReportGen: IReportPresenter instance

    ReportGen->>ReportGen: Pass AggregatedResult
    ReportGen->>ReportGen: presenter.present(result)
    ReportGen-->>UI: ReportPresentation<br/>{sections, metrics, items}

    UI->>UI: Render in Dashboard

    alt PDF Export
        UI->>PDF: Export to PDF
        PDF->>Registry: getByPlugin(pluginId)
        Registry-->>PDF: All presenters for plugin

        loop For each presenter
            PDF->>PDF: presenter.present(result)
            PDF->>PDF: Render as HTML
        end

        PDF->>PDF: Puppeteer HTML→PDF
        PDF-->>UI: PDF File
    end

    Note over Engine,PDF: Key Architecture Rule
    Note right of Registry: Clients NEVER import<br/>presenters directly.<br/>Always use registry.
```

---

## data flow summary

```mermaid
---
title: complete data flow from file to ui
config:
  theme: base
  layout: elk
---
graph TD
    accTitle: Complete Data Flow from File to UI
    accDescr: High-level overview of data transformation from source files through analysis to UI presentation

    FS[File System] -->|File Path| Watcher[Chokidar Watcher]
    Watcher -->|Change Event| Coord[Analysis Coordinator]

    Coord -->|Check Cache| DB[(SQLite)]
    DB -->|Cache Hit| UI
    DB -->|Cache Miss| Engine[Analysis Engine]

    Engine -->|ts-morph| AST[Abstract Syntax Tree]
    AST -->|Source File| Plugins[Analyzer Plugins]

    Plugins -->|Plugin Results| Aggregator[Result Aggregator]

    Aggregator -->|AggregatedResult| Presenters[Presenter Registry]
    Presenters -->|ReportPresentation| Formatters[UI Formatters]

    Formatters --> Dashboard[Dashboard View]
    Formatters --> FileDetail[File Detail View]
    Formatters --> Reports[PDF Reports]

    Aggregator -->|Persist| DB

    Dashboard --> User((User))
    FileDetail --> User
    Reports --> User

    User -->|Open File in IDE| IDE[External IDE]
    User -->|Export| Export[Export Service]
    User -->|Add Note| Notes[Notes Manager]

    Notes --> DB
    Export --> PDF[PDF File]
    Export --> JSON[JSON File]

    classDef inputClass fill:#10b981,stroke:#059669,color:#fff
    classDef processClass fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef dataClass fill:#f59e0b,stroke:#d97706,color:#fff
    classDef outputClass fill:#8b5cf6,stroke:#6d28d9,color:#fff
    classDef userClass fill:#ec4899,stroke:#be185d,color:#fff

    class FS,Watcher inputClass
    class Coord,Engine,AST,Plugins,Aggregator,Presenters,Formatters processClass
    class DB,Notes dataClass
    class Dashboard,FileDetail,Reports,PDF,JSON outputClass
    class User,IDE userClass
```

---

## key architecture principles

### plugin isolation

- **No direct imports**: Desktop app NEVER imports analyzer code directly
- **Dynamic loading**: Plugins loaded at runtime via `@vipr/plugin-loader`
- **Registry pattern**: All access mediated through `PresenterRegistry`

### data persistence strategy

- **Content-addressed caching**: SHA-256 hashing for cache invalidation
- **Snapshot history**: Git-aware versioning for regression tracking
- **Incremental updates**: Only re-analyze modified files

### security boundaries

- **Main process**: Full Node.js access, runs analysis engine, manages SQLite
- **Preload scripts**: Context bridge, type-safe API exposure
- **Renderer process**: Sandboxed Chromium, React UI, no direct file access

### performance optimization

- **Two-tier caching**: In-memory engine cache + persistent SQLite cache
- **Parallel execution**: Plugins run concurrently, analyses run concurrently
- **Debounced file watching**: Prevents analysis thrashing during rapid edits

### extensibility

- **Plugin architecture**: New analyzers extend via `ITechnologyPlugin`
- **Presenter pattern**: Custom reports via `IReportPresenter`
- **MCP integration**: Optional server for IDE integration

---

## technology stack

| Layer                 | Technology                | Purpose                      |
| --------------------- | ------------------------- | ---------------------------- |
| **Desktop Framework** | Electron + Electron Forge | Cross-platform desktop app   |
| **UI Framework**      | React + TypeScript        | Renderer process UI          |
| **Build System**      | Vite                      | Fast dev server and bundling |
| **Analysis Engine**   | ts-morph + custom plugins | AST parsing and analysis     |
| **Plugin System**     | Dynamic ES module loading | Runtime plugin discovery     |
| **Persistence**       | better-sqlite3            | Embedded database            |
| **File Watching**     | chokidar                  | Cross-platform FS events     |
| **MCP Protocol**      | @modelcontextprotocol/sdk | IDE integration              |
| **State Management**  | React Context + Hooks     | UI state coordination        |
| **PDF Generation**    | Puppeteer / electron-pdf  | Report exports               |

---

## critical file paths

```
clients/desktop/
├── src/
│   ├── main/                    # Main process (Node.js)
│   │   ├── index.ts             # Electron app entry point
│   │   ├── ipc/
│   │   │   ├── handlers.ts      # IPC request handlers
│   │   │   └── channels.ts      # Channel definitions
│   │   ├── db/
│   │   │   ├── database.ts      # SQLite connection manager
│   │   │   ├── schema.ts        # Table schemas and migrations
│   │   │   └── queries.ts       # Prepared statements
│   │   ├── analysis/
│   │   │   ├── coordinator.ts   # Analysis orchestration
│   │   │   └── engine-wrapper.ts # Engine lifecycle management
│   │   ├── fs/
│   │   │   └── watcher.ts       # File watching with chokidar
│   │   └── mcp/
│   │       └── server.ts        # Optional MCP server
│   │
│   ├── preload/
│   │   └── index.ts             # Context bridge API
│   │
│   └── renderer/                # Renderer process (React)
│       ├── App.tsx              # React root component
│       ├── pages/               # Route pages
│       │   ├── Dashboard.tsx
│       │   ├── FileDetail.tsx
│       │   └── Settings.tsx
│       ├── components/          # Shared components
│       ├── hooks/               # Custom React hooks
│       ├── stores/              # State management
│       └── services/            # API clients
│
├── forge.config.ts              # Electron Forge configuration
├── vite.main.config.ts          # Vite config for main process
└── vite.renderer.config.ts      # Vite config for renderer
```

---

## implementation checklist

### phase 1: foundation

- [ ] Set up Electron Forge with TypeScript + Vite
- [ ] Configure three-process architecture (main/preload/renderer)
- [ ] Implement secure IPC communication layer
- [ ] Initialize SQLite with schema migrations
- [ ] Integrate `@vipr/engine` in main process

### phase 2: core analysis

- [ ] Implement plugin loader integration
- [ ] Build analysis coordinator with caching
- [ ] Set up file watcher with debouncing
- [ ] Create database persistence layer
- [ ] Implement snapshot management

### phase 3: ui development

- [ ] Build React application shell
- [ ] Create dashboard with summary cards
- [ ] Implement file detail views
- [ ] Add settings interface
- [ ] Integrate presenter registry for reports

### phase 4: advanced features

- [ ] PDF export service
- [ ] Historical snapshots and regression detection
- [ ] Notes and annotations system
- [ ] Issue exclusions management
- [ ] MCP server integration

### phase 5: polish

- [ ] Error handling and recovery
- [ ] Progress indicators and notifications
- [ ] IDE integration (open files)
- [ ] User preferences persistence
- [ ] Performance optimization
