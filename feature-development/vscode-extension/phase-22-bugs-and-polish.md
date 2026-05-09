- [x] When I right-click a file in the explore and choose the option to analyze that file, the file analysis runs but I would also expect the sidebar to open and show the visual report of that file. Presently, it does not. Please fix that.
- [x] When I click on the vipr icon in the explorer or in the footer bar, it should open the current file in the pane and show the analysis report. If it has been cached, show the cached report.
- [x] The cached report should invalidate and the report be regenerated when the sidebar is open and the active file has been modified and saved.

- [x] Change "Vipr Analysis Dashboard" to Vipr Code Quality Report"
- [x]We should have the file name (in the IDE's monofont), file name only, below the "Vipr Analysis Dashboard"
- [x] Remove the "1 Files" pill since this is only for one file.
- [x] Put the `{X} Issues` pill below where the file name will go. To the left of it, create a pill for the file type. Analyzers should provide this type to you in their analysis data with the file path. Supported file types include, but are not limited to: Component, Hook, Server Action, React Server Component, Utility, Page, App Router, etc. We'll need to define these support types in each analyzer and have them share a common base interface that the respective clients (CLI, VSCode Extension, and Desktop) support.

Use these subagents to properly address these concerns:

@typescript-refactoring-expert
@vscode-extension-optimizer
@vitest-engineer
@vscode-lit-ux-engineer
