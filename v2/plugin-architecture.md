# Vipr Plugin Architecture

**Purpose**: Architecture documentation for the Vipr plugin system, presenter registry pattern, and dynamic discovery mechanism.

**Last Updated**: 2026-01-16

**Audience**: LLMs, developers, contributors working on Vipr architecture

## Overview

Vipr uses a plugin-based architecture to decouple analysis logic from presentation and client implementations. Plugins are discovered dynamically, and report presenters are registered at runtime, enabling extensibility without modifying client code.

## Diagrams

### 1. Package Structure and Boundaries

```mermaid
---
title: Vipr Package Structure (C4 Container)
config:
  theme: forest
---
C4Container
    accTitle: Vipr Package Architecture
    accDescr: Shows the high-level package structure with clear boundaries between analyzers, shared packages, and client applications

    Container_Boundary(analyzers, "Analyzers") {
        Container(react, "React Analyzer", "TypeScript", "React-specific analysis and presenters")
        Container(core, "Core Analyzer", "TypeScript", "JS/TS core metrics (cyclomatic, Halstead)")
    }

    Container_Boundary(packages, "Shared Packages") {
        Container(common, "@vipr/common", "TypeScript", "Shared types, interfaces, registry, utilities")
        Container(logging, "@vipr/logging", "TypeScript", "Logging utilities")
        Container(engine, "@vipr/engine", "TypeScript", "Analysis orchestration engine")
    }

    Container_Boundary(clients, "Client Applications") {
        Container(cli, "CLI Client", "TypeScript", "Command-line interface")
        Container(vscode, "VS Code Extension", "TypeScript", "Editor integration")
        Container(desktop, "Desktop App", "Electron", "GUI application")
    }

    Rel(react, common, "Implements interfaces from", "ITechnologyPlugin, IReportPresenter")
    Rel(core, common, "Implements interfaces from", "ITechnologyPlugin, IReportPresenter")
    Rel(engine, common, "Uses", "Types, interfaces")

    Rel(cli, common, "Direct import", "PresenterRegistry, types")
    Rel(cli, engine, "Direct import", "AnalysisEngine")
    Rel(cli, logging, "Uses", "Logger")

    Rel(vscode, common, "Uses", "PresenterRegistry, types")
    Rel(desktop, common, "Uses", "PresenterRegistry, types")

    Rel(cli, react, "DYNAMIC IMPORT", "via CliPluginLoader")
    Rel(cli, core, "DYNAMIC IMPORT", "via CliPluginLoader")
```

**Key Points**:

- Analyzers are isolated plugins (dynamically imported by CliPluginLoader)
- `@vipr/engine` is directly imported by clients for orchestration
- `@vipr/common` provides shared types, utilities, and registry
- Clients dynamically import analyzer plugins via CliPluginLoader (NOT compile-time imports)

### 2. Plugin Discovery and Registration Flow

```mermaid
---
title: Plugin Discovery and Registration
config:
  theme: forest
  layout: elk
---
sequenceDiagram
    accTitle: Plugin Discovery and Registration Flow
    accDescr: Shows how plugins are discovered at runtime and registered with the engine and presenter registry

    participant CLI as CLI Application
    participant Loader as CliPluginLoader
    participant Engine as AnalysisEngine
    participant Registry as PresenterRegistry
    participant ReactPlugin as ReactAnalyzerPlugin
    participant CorePlugin as CoreAnalyzerPlugin

    CLI->>+Loader: new CliPluginLoader(engine)
    CLI->>Loader: loadBundledPlugins()

    Loader->>+Loader: discoverBundledPlugins()
    Loader->>ReactPlugin: import('@vipr/react')
    Loader->>ReactPlugin: new ReactAnalyzerPlugin()
    activate ReactPlugin
    ReactPlugin->>ReactPlugin: registerAnalyses()
    ReactPlugin->>ReactPlugin: createReactPresenters()
    ReactPlugin-->>Loader: plugin instance
    deactivate ReactPlugin

    Loader->>CorePlugin: import('@vipr/core')
    Loader->>CorePlugin: new CoreAnalyzerPlugin()
    activate CorePlugin
    CorePlugin->>CorePlugin: registerAnalyses()
    CorePlugin->>CorePlugin: createCorePresenters()
    CorePlugin-->>Loader: plugin instance
    deactivate CorePlugin
    Loader-->>-Loader: [ReactPlugin, CorePlugin]

    loop For each discovered plugin
        Loader->>+Engine: registerPlugin(plugin)
        Engine->>Engine: Store plugin by ID
        Engine-->>-Loader: void

        Loader->>+Registry: registerFromPlugin(plugin)
        Registry->>ReactPlugin: getReportPresenters()
        ReactPlugin-->>Registry: [SecurityPresenter, A11yPresenter, ...]

        loop For each presenter
            Registry->>Registry: register(presenter)
            Note over Registry: Store by key:<br/>"pluginId:reportType"
        end
        Registry-->>-Loader: void
    end

    Loader-->>CLI: Plugins loaded
```

**Key Mechanisms**:

- `CliPluginLoader.discoverBundledPlugins()` dynamically imports analyzer packages
- Each plugin constructor creates its presenters via factory function
- `registerFromPlugin()` extracts presenters via `getReportPresenters()`
- Registry stores presenters with composite key `{pluginId}:{reportType}`

### 3. Presenter Registry Pattern

```mermaid
---
title: Presenter Registry Architecture
config:
  theme: forest
---
classDiagram
    accTitle: Presenter Registry Class Structure
    accDescr: Shows the interfaces and classes that implement the presenter registry pattern for dynamic report discovery

    class IReportPresenter {
        <<interface>>
        +reportType: string
        +pluginId: string
        +canPresent(data): boolean
        +present(data): ReportPresentation
        +getMetadata(): IReportMetadata
    }

    class IReportMetadata {
        <<interface>>
        +reportType: string
        +pluginId: string
        +label: string
        +hint: string
        +icon?: string
        +order: number
        +categories?: string[]
    }

    class IPresenterRegistry {
        <<interface>>
        +register(presenter)
        +registerFromPlugin(plugin)
        +getAvailableReports(): IReportMetadata[]
        +get(pluginId, reportType): IReportPresenter?
        +getByPlugin(pluginId): IReportPresenter[]
        +getAll(): IReportPresenter[]
        +getReportTypes(pluginId): string[]
    }

    class PresenterRegistry {
        -presenters: Map~string, IReportPresenter~
        -getKey(pluginId, reportType): string
        +register(presenter)
        +registerFromPlugin(plugin)
        +getAvailableReports(): IReportMetadata[]
        +get(pluginId, reportType): IReportPresenter?
        +getByPlugin(pluginId): IReportPresenter[]
    }

    class SecurityPresenter {
        +reportType: "security"
        +pluginId: "react"
        +canPresent(data): boolean
        +present(data): ReportPresentation
        +getMetadata(): IReportMetadata
    }

    class AccessibilityPresenter {
        +reportType: "accessibility"
        +pluginId: "react"
        +canPresent(data): boolean
        +present(data): ReportPresentation
        +getMetadata(): IReportMetadata
    }

    class CyclomaticPresenter {
        +reportType: "cyclomatic"
        +pluginId: "core"
        +canPresent(data): boolean
        +present(data): ReportPresentation
        +getMetadata(): IReportMetadata
    }

    IPresenterRegistry <|.. PresenterRegistry: implements
    IReportPresenter <|.. SecurityPresenter: implements
    IReportPresenter <|.. AccessibilityPresenter: implements
    IReportPresenter <|.. CyclomaticPresenter: implements
    PresenterRegistry "1" *-- "many" IReportPresenter: manages
    IReportPresenter ..> IReportMetadata: returns
```

**Registry Benefits**:

- **Dynamic Discovery**: Clients query registry for available reports instead of hardcoding
- **Decoupling**: Presenters defined in plugins, consumed through interface
- **Extensibility**: New plugins automatically register their presenters
- **Metadata-Driven**: UI elements (labels, icons, order) come from presenter metadata

### 4. Report Discovery and Rendering Flow

```mermaid
---
title: Dynamic Report Discovery and Rendering
config:
  theme: forest
  layout: elk
---
sequenceDiagram
    accTitle: Report Discovery and Rendering Flow
    accDescr: Shows how formatters dynamically discover available reports and render them without hardcoded mappings

    participant CLI as CLI Command
    participant Formatter as CliFormatter
    participant Registry as PresenterRegistry
    participant Presenter as IReportPresenter
    participant Renderer as CliReportRenderer

    CLI->>+Formatter: new CliFormatter(registry, options)
    Note over Formatter: NO hardcoded report types<br/>NO report arrays<br/>NO normalizeReportType()

    CLI->>Formatter: format(aggregatedResult)

    alt User specified reportType(s)
        Formatter->>+Registry: get(pluginId, reportType)
        Registry-->>-Formatter: SecurityPresenter
    else Show all reports
        Formatter->>+Registry: getAvailableReports()
        Registry->>Registry: Collect metadata from all presenters
        Registry->>Registry: Sort by order field
        Registry-->>-Formatter: [IReportMetadata, ...]
        Note over Formatter: Iterate metadata in display order
    end

    loop For each report
        Formatter->>+Presenter: canPresent(pluginResult)
        Presenter->>Presenter: Check if data exists
        Presenter-->>-Formatter: true

        Formatter->>+Presenter: present(pluginResult)
        Presenter->>Presenter: Extract data from analysisBreakdown
        Presenter->>Presenter: Build ReportPresentation
        Presenter-->>-Formatter: ReportPresentation

        Formatter->>+Renderer: render(presentation)
        Renderer->>Renderer: Format sections, items, metrics
        Renderer-->>-Formatter: Formatted string
    end

    Formatter-->>CLI: Complete report output
```

**Critical Design Decisions**:

- Formatters query `registry.getAvailableReports()` instead of maintaining hardcoded lists
- Report order determined by `IReportMetadata.order` field, not hardcoded arrays
- Presenters decide if they can present data via `canPresent()`
- No `normalizeReportType()` function or alias mapping needed

### 5. Data Flow from Analysis to Output

```mermaid
---
title: Data Flow - Analysis to Presentation
config:
  theme: forest
---
flowchart TD
    accTitle: Data Flow from Analysis to Formatted Output
    accDescr: Shows how analysis data flows through the plugin system, presenters, and formatters to produce final output

    Start([User runs analysis]) --> Engine[AnalysisEngine]

    Engine --> ReactPlugin[ReactAnalyzerPlugin.analyze]
    Engine --> CorePlugin[CoreAnalyzerPlugin.analyze]

    ReactPlugin --> ReactAnalyses[Run analyses:<br/>- Security<br/>- Accessibility<br/>- Performance<br/>etc.]
    CorePlugin --> CoreAnalyses[Run analyses:<br/>- Cyclomatic<br/>- Halstead<br/>- Maintainability]

    ReactAnalyses --> PluginResult1[PluginResult<br/>+ analysisBreakdown Map]
    CoreAnalyses --> PluginResult2[PluginResult<br/>+ analysisBreakdown Map]

    PluginResult1 --> Aggregated[AggregatedResult]
    PluginResult2 --> Aggregated

    Aggregated --> FormatterChoice{Formatter<br/>queries registry}

    FormatterChoice --> GetMeta[registry.getAvailableReports]
    GetMeta --> ReportMeta["IReportMetadata array<br/>sorted by order"]

    ReportMeta --> GetPresenter[registry.get or iterate]
    GetPresenter --> Presenter[IReportPresenter]

    Presenter --> CanPresent{canPresent?}
    CanPresent -->|Yes| Present[present]
    CanPresent -->|No| Skip[Skip this report]

    Present --> ExtractData[Extract from<br/>analysisBreakdown]
    ExtractData --> BuildModel[Build ReportPresentation<br/>- sections<br/>- metrics<br/>- items]

    BuildModel --> Render[Formatter renders<br/>presentation model]
    Render --> Output([Formatted output])

    Skip --> NextReport{More reports?}
    Output --> NextReport
    NextReport -->|Yes| GetPresenter
    NextReport -->|No| Done([Complete])

    classDef plugin fill:#2563eb,stroke:#1e40af,color:#fff
    classDef data fill:#16a34a,stroke:#15803d,color:#fff
    classDef registry fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef output fill:#64748b,stroke:#475569,color:#fff

    class ReactPlugin,CorePlugin plugin
    class PluginResult1,PluginResult2,Aggregated,ReportMeta data
    class FormatterChoice,GetMeta,GetPresenter,Presenter registry
    class Render,Output output
```

**Data Transformations**:

1. **Analysis**: Plugin runs analyses, stores results in `analysisBreakdown` Map
2. **Aggregation**: Engine combines plugin results into `AggregatedResult`
3. **Discovery**: Formatter queries registry for available reports
4. **Presentation**: Presenter extracts data and builds `ReportPresentation`
5. **Rendering**: Formatter renders presentation model to final format

### 6. Anti-Patterns and Correct Patterns

```mermaid
---
title: Anti-Patterns vs Correct Patterns
config:
  theme: base
---
flowchart TB
    accTitle: Anti-Patterns to Avoid vs Correct Patterns
    accDescr: Comparison of forbidden practices versus the correct plugin-based approach

    subgraph AntiPatterns [" ANTI-PATTERNS (FORBIDDEN)"]
        AP1["Direct Import<br/>import { SecurityPresenter }<br/>from '@vipr/react'"]
        AP2["Hardcoded Array<br/>const REPORT_TYPES =<br/>['security', 'accessibility']"]
        AP3["Hardcoded Mapping<br/>function normalizeReportType<br/>aliases: sec → security"]
        AP4["Creating Presenters<br/>new SecurityPresenter<br/>in client code"]
        AP5["Bypassing Registry<br/>Direct presenter access<br/>without discovery"]
    end

    subgraph CorrectPatterns [" CORRECT PATTERNS"]
        CP1["Plugin System<br/>import { CliPluginLoader }<br/>from '@vipr/cli'"]
        CP2["Dynamic Discovery<br/>registry.getAvailableReports<br/>query at runtime"]
        CP3["Metadata-Driven<br/>metadata.label, order<br/>from presenters"]
        CP4["Registry Access<br/>registry.get(pluginId, type)<br/>or registry.getByPlugin"]
        CP5["Plugin Registration<br/>registerFromPlugin<br/>auto-discovers presenters"]
    end

    AP1 -.->|Instead use| CP1
    AP2 -.->|Instead use| CP2
    AP3 -.->|Instead use| CP3
    AP4 -.->|Instead use| CP4
    AP5 -.->|Instead use| CP5

    classDef anti fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef correct fill:#16a34a,stroke:#15803d,color:#fff

    class AP1,AP2,AP3,AP4,AP5 anti
    class CP1,CP2,CP3,CP4,CP5 correct
```

## How to Use

1. **View Online**: Copy diagram code to [Mermaid Live Editor](https://mermaid.live)
2. **VS Code**: Install Mermaid extension and preview this file
3. **GitHub**: Renders automatically in markdown files

## Customization

- **Change theme**: Modify `config: theme:` in frontmatter (`default`, `forest`, `dark`, `neutral`)
- **Adjust layout**: Switch to `elk` for complex diagrams in `config: layout: elk`
- **Colors**: Modify `classDef` statements for different color schemes

## Design Principles

### 1. Decoupling

Clients never import from analyzers directly. All communication flows through `@vipr/common` interfaces and the plugin system.

```typescript
// WRONG: Direct import
import { SecurityPresenter } from '@vipr/react';

// CORRECT: Plugin system
import { AnalysisEngine } from '@vipr/engine';

const engine = new AnalysisEngine();
const loader = new CliPluginLoader(engine);
await loader.loadBundledPlugins();
const registry = loader.getPresenterRegistry();
```

### 2. Dynamic Discovery

No hardcoded report types or presenter arrays. Formatters query the registry for available reports.

```typescript
// WRONG: Hardcoded array
const REPORT_TYPES = ['security', 'accessibility', 'performance'];

// CORRECT: Dynamic discovery
const availableReports = registry.getAvailableReports();
for (const metadata of availableReports) {
  const presenter = registry.get(metadata.pluginId, metadata.reportType);
  // ...
}
```

### 3. Registry Pattern

Central source of truth for available reports. Presenters register themselves, clients discover through queries.

```typescript
// Plugin exposes presenters
class ReactAnalyzerPlugin implements ITechnologyPlugin {
  getReportPresenters(): IReportPresenter[] {
    return createReactPresenters();
  }
}

// Loader registers them
registry.registerFromPlugin(plugin);

// Clients discover them
const presenter = registry.get('react', 'security');
```

### 4. Metadata-Driven

UI elements (labels, icons, display order) come from presenter metadata, not client code.

```typescript
class SecurityPresenter implements IReportPresenter {
  getMetadata(): IReportMetadata {
    return {
      reportType: 'security',
      pluginId: 'react',
      label: 'Security',
      hint: 'Security vulnerabilities and risks',
      icon: '🔒',
      order: 20,
      categories: ['security', 'reliability'],
    };
  }
}
```

## Related Documentation

- `/packages/common/src/types/presentation/presenter.ts` - Presenter interfaces
- `/packages/common/src/presenters/presenter-registry.ts` - Registry implementation
- `/clients/cli/src/plugins/loader.ts` - Plugin discovery and loading
- `/clients/cli/src/formatters/cli-formatter.ts` - Dynamic report rendering

## Notes

This architecture enables:

- New analyzers to be added without modifying client code
- Plugins to define their own report types and presentation logic
- Clients to dynamically discover available reports at runtime
- Consistent presentation model across different clients (CLI, VS Code, Desktop)
