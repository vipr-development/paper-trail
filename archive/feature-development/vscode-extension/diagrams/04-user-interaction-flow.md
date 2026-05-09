---
id: 04-user-interaction-flow
---

# User Interaction Flow

This diagram visualizes how users interact with the extension through commands, CodeLens, diagnostics, and the webview.

## User Journey: Analyzing a File

```mermaid
---
title: User Journey - File Analysis
config:
  theme: neutral
---
journey
    accTitle: User Journey for Analyzing a File
    accDescr: Shows the complete user experience from opening a file to viewing and acting on analysis results

    title User Experience: Analyzing a File with Vipr

    section File Opening
      Open TypeScript file: 5: User
      Extension activates: 3: Extension
      Auto-analyze in background: 4: Extension
      Status bar updates: 5: User

    section Viewing Results
      See CodeLens hints: 5: User
      Hover over gutter icon: 5: User
      View diagnostic in Problems panel: 4: User
      Click CodeLens to navigate: 5: User

    section Dashboard Interaction
      Open sidebar dashboard: 5: User
      View metrics grid: 5: User
      Explore trend charts: 4: User
      Filter by category: 5: User

    section Taking Action
      Click quick fix: 5: User
      Apply automated fix: 4: Extension
      Request AI suggestion: 5: User
      Review AI-generated code: 4: User

    section History Analysis
      Run show history command: 5: User
      View temporal trends: 5: User
      Identify regression commit: 3: Extension
      Navigate to git commit: 4: User
```

## Command Palette Interactions

```mermaid
---
title: Command Palette User Flow
config:
  theme: neutral
---
flowchart TD
    accTitle: Command Palette Interactions
    accDescr: Shows available commands and their outcomes from the user perspective

    Start([User Opens Command Palette<br/>Cmd+Shift+P / Ctrl+Shift+P]) --> SelectCommand{Select Command}

    SelectCommand -->|Vipr: Analyze File| AnalyzeFile[Analyze Current File]
    SelectCommand -->|Vipr: Analyze Workspace| AnalyzeWorkspace[Analyze All Files]
    SelectCommand -->|Vipr: Show History| ShowHistory[Open History Panel]
    SelectCommand -->|Vipr: Find Regression| FindRegression[Detect Regression Commit]
    SelectCommand -->|Vipr: Cleanup History| CleanupHistory[Delete Old Snapshots]
    SelectCommand -->|Vipr: Export Report| ExportReport[Generate PDF Report]
    SelectCommand -->|Vipr: Fix with AI| FixWithAI[AI-Powered Quick Fix]
    SelectCommand -->|Vipr: Clear Cache| ClearCache[Clear Analysis Cache]
    SelectCommand -->|Vipr: License Status| LicenseStatus[Show License Info]

    %% Analyze File flow
    AnalyzeFile --> RunAnalysis[Run Analysis Engine]
    RunAnalysis --> UpdateProviders[Update All UI Providers]
    UpdateProviders --> ShowResults[Display Results in:<br/>• Problems Panel<br/>• CodeLens<br/>• Gutter Decorations<br/>• Dashboard]
    ShowResults --> SaveSnapshot[Save to SQLite]
    SaveSnapshot --> NotifyComplete[Show Completion Toast]
    NotifyComplete --> End([Done])

    %% Analyze Workspace flow
    AnalyzeWorkspace --> SelectFiles[Scan Workspace<br/>for TS/JS files]
    SelectFiles --> ProgressBar[Show Progress Bar]
    ProgressBar --> BatchAnalysis[Analyze in Batches]
    BatchAnalysis --> UpdateFileTree[Update File Tree Navigator]
    UpdateFileTree --> WorkspaceReport[Show Summary Report]
    WorkspaceReport --> End

    %% Show History flow
    ShowHistory --> CheckHistory{Has History?}
    CheckHistory -->|No| NoHistoryMsg[Info: Analyze file<br/>to start tracking]
    NoHistoryMsg --> End
    CheckHistory -->|Yes| OpenPanel[Open History Panel]
    OpenPanel --> DisplayTrends[Show Trend Chart]
    DisplayTrends --> DisplaySnapshots[List All Snapshots]
    DisplaySnapshots --> InteractiveView[Interactive Visualization]
    InteractiveView --> End

    %% Find Regression flow
    FindRegression --> CheckSnapshots{Has 2+ Snapshots?}
    CheckSnapshots -->|No| InsufficientData[Error: Need more history]
    InsufficientData --> End
    CheckSnapshots -->|Yes| RunBinarySearch[Binary Search Algorithm]
    RunBinarySearch --> ShowProgress[Progress Notification]
    ShowProgress --> IdentifyCommit{Regression Found?}
    IdentifyCommit -->|Yes| ShowCommitDetails[Display Commit Info:<br/>• Author<br/>• Date<br/>• Message<br/>• Impact Score]
    ShowCommitDetails --> OfferActions[Offer Actions:<br/>• View Diff<br/>• Checkout Commit<br/>• Blame Authors]
    OfferActions --> End
    IdentifyCommit -->|No| NoRegression[Info: No clear regression]
    NoRegression --> End

    %% Cleanup History flow
    CleanupHistory --> PromptRetention[Prompt for Retention Days<br/>Default: 90]
    PromptRetention --> DeleteOld[Delete Snapshots<br/>Older Than Threshold]
    DeleteOld --> ShowDeleted[Show Count of Deleted Records]
    ShowDeleted --> End

    %% Export Report flow
    ExportReport --> SelectFormat{Select Format}
    SelectFormat -->|PDF| GeneratePDF[Generate PDF Report]
    SelectFormat -->|JSON| ExportJSON[Export JSON Data]
    SelectFormat -->|Markdown| ExportMarkdown[Export Markdown Report]
    GeneratePDF --> SaveDialog[Show Save Dialog]
    ExportJSON --> SaveDialog
    ExportMarkdown --> SaveDialog
    SaveDialog --> FileWritten[Write File to Disk]
    FileWritten --> OpenFile[Offer to Open File]
    OpenFile --> End

    %% Fix with AI flow
    FixWithAI --> CheckAI{AI Available?}
    CheckAI -->|No| NoAIMsg[Error: Copilot not available]
    NoAIMsg --> End
    CheckAI -->|Yes| SelectIssue[Select Issue from Diagnostics]
    SelectIssue --> GenerateFix[Generate Fix with LLM]
    GenerateFix --> ShowDiff[Show Inline Diff Preview]
    ShowDiff --> UserReview{Accept Fix?}
    UserReview -->|Accept| ApplyFix[Apply Code Changes]
    ApplyFix --> Reanalyze[Re-analyze File]
    Reanalyze --> End
    UserReview -->|Reject| End

    %% Clear Cache flow
    ClearCache --> ClearMemory[Clear Result Cache]
    ClearMemory --> ClearEngine[Clear Engine Cache]
    ClearEngine --> ShowConfirm[Show Success Toast]
    ShowConfirm --> End

    %% License Status flow
    LicenseStatus --> ValidateLicense[Validate License Key]
    ValidateLicense --> ShowModal[Show Modal Dialog:<br/>• Tier<br/>• Features<br/>• Expiration]
    ShowModal --> End

    %% Styling
    classDef userAction fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef systemAction fill:#10b981,stroke:#059669,color:#fff
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff
    classDef output fill:#8b5cf6,stroke:#7c3aed,color:#fff
    classDef error fill:#ef4444,stroke:#dc2626,color:#fff

    class Start,SelectCommand,SelectFormat,UserReview userAction
    class RunAnalysis,UpdateProviders,SaveSnapshot,BatchAnalysis,RunBinarySearch,GenerateFix systemAction
    class CheckHistory,CheckSnapshots,IdentifyCommit,CheckAI decision
    class ShowResults,DisplayTrends,ShowCommitDetails,ShowDiff,ShowModal output
    class NoHistoryMsg,InsufficientData,NoAIMsg,NoRegression error
```

## CodeLens and Gutter Interactions

```mermaid
---
title: Inline UI Interactions
config:
  theme: neutral
---
sequenceDiagram
    accTitle: CodeLens and Gutter Interactions
    accDescr: Shows how users interact with inline hints, decorations, and quick fixes

    actor User
    participant Editor as VSCode Editor
    participant CodeLens as CodeLens Provider
    participant Gutter as Decoration Provider
    participant Diagnostic as Diagnostic Provider
    participant QuickFix as Code Action Provider
    participant AI as AI Service

    %% CodeLens interaction
    User->>Editor: View file
    Editor->>CodeLens: Request CodeLens
    CodeLens->>CodeLens: Get cached analysis
    CodeLens-->>Editor: Show inline hints

    Note over Editor: 📊 Score: 75/100 (+5)<br/>🔴 XSS vulnerability • Introduced 2d ago by Alice

    User->>Editor: Click CodeLens
    Editor->>CodeLens: Execute command
    CodeLens->>Editor: Navigate to line<br/>or show details

    %% Gutter decoration interaction
    User->>Editor: View gutter icons
    Editor->>Gutter: Render decorations
    Gutter-->>Editor: Show colored icons<br/>🔴 Critical<br/>🟡 Warning

    User->>Editor: Hover over gutter icon
    Editor->>Diagnostic: Get hover content
    Diagnostic-->>Editor: Show hover tooltip

    Note over Editor: Hover Tooltip:<br/>Security: XSS Vulnerability<br/>Line 42<br/>eval() detected<br/>Click for quick fixes

    %% Quick fix interaction
    User->>Editor: Click quick fix lightbulb
    Editor->>QuickFix: Request code actions
    QuickFix->>QuickFix: Get available fixes
    QuickFix-->>Editor: Return action list

    alt Has AI-powered fix
        QuickFix-->>Editor: • Fix with AI<br/>• Suppress warning<br/>• View documentation
    else Manual fix only
        QuickFix-->>Editor: • Suppress warning<br/>• View documentation
    end

    User->>Editor: Select "Fix with AI"
    Editor->>AI: Generate fix
    AI->>AI: Call Language Model API
    AI-->>Editor: Suggested code change

    Editor->>Editor: Show inline diff preview
    User->>Editor: Accept fix
    Editor->>Editor: Apply code changes
    Editor->>Editor: Re-analyze file
    Editor-->>User: Updated decorations
```

## Dashboard Webview Interactions

```mermaid
---
title: Dashboard Webview User Flow
config:
  theme: neutral
---
stateDiagram-v2
    accTitle: Dashboard Webview Interactions
    accDescr: Shows the user interaction flow within the dashboard sidebar

    [*] --> Loading: Open Dashboard

    Loading --> NoAnalysis: No cached results
    Loading --> ShowMetrics: Has cached results

    NoAnalysis --> PromptAnalyze: Show empty state
    PromptAnalyze --> RunningAnalysis: Click "Analyze File"
    RunningAnalysis --> ShowMetrics: Analysis complete

    ShowMetrics --> InteractWithDashboard: Display metrics

    state InteractWithDashboard {
        [*] --> ViewOverview
        ViewOverview --> FilterByCategory: Click category tab
        FilterByCategory --> ViewCategoryMetrics: Show filtered data

        ViewCategoryMetrics --> ViewChart: Click chart
        ViewChart --> DrillDown: Hover data points

        DrillDown --> ViewIssuesList: Click metric
        ViewIssuesList --> NavigateToCode: Click issue
        NavigateToCode --> [*]: Jump to editor

        ViewOverview --> ExportData: Click export button
        ExportData --> [*]: Generate report
    }

    InteractWithDashboard --> RefreshData: File changes detected
    RefreshData --> RunningAnalysis: Auto re-analyze
    RunningAnalysis --> ShowMetrics: Update dashboard

    state ShowMetrics {
        [*] --> RenderMetricsGrid
        RenderMetricsGrid --> RenderCharts
        RenderCharts --> RenderIssuesList
        RenderIssuesList --> [*]
    }

    note right of NoAnalysis
        Empty state with call-to-action
        button to trigger analysis
    end note

    note right of InteractWithDashboard
        Interactive Lit components
        with Chart.js visualizations
    end note

    note right of RefreshData
        Real-time updates when
        file content changes
    end note
```

## File Tree Navigator Interactions

```mermaid
---
title: File Tree Navigator User Flow
config:
  theme: neutral
---
flowchart LR
    accTitle: File Tree Navigator Interactions
    accDescr: Shows user interactions with the file tree sidebar including sorting, filtering, and navigation

    Start([Open File Tree View]) --> Display[Display Tree Structure]

    Display --> UserAction{User Action}

    UserAction -->|Click File| OpenFile[Open File in Editor]
    OpenFile --> AutoAnalyze[Auto-analyze if needed]
    AutoAnalyze --> UpdateTree[Update tree item icon]
    UpdateTree --> Display

    UserAction -->|Click Folder| ExpandCollapse[Expand/Collapse Folder]
    ExpandCollapse --> Display

    UserAction -->|Click Sort by Name| SortName[Sort Alphabetically]
    SortName --> Refresh[Refresh Tree View]
    Refresh --> Display

    UserAction -->|Click Sort by Score| SortScore[Sort by Analysis Score<br/>Worst first]
    SortScore --> Refresh

    UserAction -->|Click Filter: All| FilterAll[Show All Files]
    FilterAll --> Refresh

    UserAction -->|Click Filter: Critical| FilterCritical[Show Only Critical Issues]
    FilterCritical --> Refresh

    UserAction -->|Click Filter: Warning| FilterWarning[Show Warnings + Critical]
    FilterWarning --> Refresh

    UserAction -->|Right-click File| ContextMenu[Show Context Menu]
    ContextMenu --> ContextAction{Select Action}

    ContextAction -->|Analyze File| TriggerAnalysis[Run Analysis]
    TriggerAnalysis --> Refresh

    ContextAction -->|Show Problems| OpenProblems[Focus Problems Panel]
    OpenProblems --> Display

    ContextAction -->|View History| OpenHistory[Show History Panel]
    OpenHistory --> Display

    ContextAction -->|Export Report| ExportFile[Export File Report]
    ExportFile --> Display

    %% Styling
    classDef user fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef action fill:#10b981,stroke:#059669,color:#fff
    classDef decision fill:#f59e0b,stroke:#d97706,color:#fff

    class UserAction,ContextAction decision
    class OpenFile,SortName,SortScore,FilterAll,FilterCritical,FilterWarning action
    class TriggerAnalysis,OpenProblems,OpenHistory,ExportFile action
```

## Keyboard Shortcuts and Quick Actions

```mermaid
---
title: Keyboard Shortcuts and Accessibility
config:
  theme: neutral
---
flowchart TD
    accTitle: Keyboard Shortcuts and Quick Actions
    accDescr: Shows available keyboard shortcuts and their associated actions

    subgraph Shortcuts["Keyboard Shortcuts"]
        direction TB

        CmdShiftP["Cmd/Ctrl + Shift + P<br/>Command Palette"] --> AllCommands[Access all Vipr commands]

        CmdShiftM["Cmd/Ctrl + Shift + M<br/>Problems Panel"] --> ShowProblems[View all diagnostics]

        F1["F1<br/>Command Palette"] --> AllCommands

        Escape["Escape<br/>Dismiss"] --> ClosePanel[Close active panel]

        Enter["Enter<br/>Navigate"] --> GoToIssue[Jump to selected issue]

        Arrow["Arrow Keys<br/>Navigate"] --> BrowseList[Browse issues list]

        CmdClick["Cmd/Ctrl + Click<br/>Open Link"] --> FollowLink[Open documentation<br/>or git commit]
    end

    subgraph QuickActions["Quick Actions (Editor)"]
        direction TB

        Lightbulb["💡 Lightbulb Icon<br/>Code Action"] --> QuickFixMenu[Show quick fixes]

        HoverMessage["Hover over Gutter<br/>Icon"] --> ShowTooltip[Display issue details]

        ClickCodeLens["Click CodeLens<br/>Hint"] --> ExecuteAction[Run associated command]

        RightClick["Right-click<br/>Context Menu"] --> ContextCommands[Show Vipr commands]
    end

    subgraph Accessibility["Accessibility Features"]
        direction TB

        ScreenReader["Screen Reader Support"] --> Announce[Announce diagnostics<br/>and results]

        HighContrast["High Contrast Mode"] --> ThemeAdapt[Adapt colors to theme]

        KeyboardOnly["Keyboard-only Navigation"] --> FullAccess[Access all features<br/>without mouse]

        FocusIndicators["Focus Indicators"] --> VisibleFocus[Clear focus states]
    end

    %% Styling
    classDef shortcut fill:#3b82f6,stroke:#1e40af,color:#fff
    classDef quick fill:#10b981,stroke:#059669,color:#fff
    classDef a11y fill:#f59e0b,stroke:#d97706,color:#fff

    class CmdShiftP,CmdShiftM,F1,Escape,Enter,Arrow,CmdClick shortcut
    class Lightbulb,HoverMessage,ClickCodeLens,RightClick quick
    class ScreenReader,HighContrast,KeyboardOnly,FocusIndicators a11y
```

## Key User Experience Principles

### Progressive Disclosure

1. **Auto-analysis on Save**: Silent background analysis doesn't interrupt workflow
2. **Inline Hints First**: CodeLens provides quick overview without opening panels
3. **Details on Demand**: Hover for tooltips, click for full information
4. **Contextual Actions**: Quick fixes appear when relevant

### Performance Perception

1. **Optimistic UI**: Show cached results immediately
2. **Background Processing**: Heavy analysis in worker threads
3. **Incremental Loading**: Dashboard renders progressively
4. **Progress Indicators**: Clear feedback for long operations

### Discoverability

1. **Command Palette**: All features accessible via search
2. **Context Menus**: Right-click for relevant actions
3. **Empty States**: Helpful prompts when no data available
4. **Onboarding**: Welcome messages guide new users

### Error Handling

1. **Graceful Degradation**: Features work without git/AI when possible
2. **Clear Error Messages**: Actionable feedback on failures
3. **Recovery Actions**: Suggest next steps (e.g., "Install Copilot")
4. **Silent Failures**: Non-critical errors logged, not shown

### Accessibility

1. **Keyboard Navigation**: Full functionality without mouse
2. **Screen Reader Support**: Semantic HTML and ARIA labels
3. **High Contrast**: Respects VSCode theme colors
4. **Focus Management**: Clear focus indicators and logical tab order
