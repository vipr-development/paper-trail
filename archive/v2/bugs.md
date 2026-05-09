# Welcome

- After restarting the application (either production or dev mode), workspaces with prior analysis should not re-analyze all files. Expected: Only changed files should be re-analyzed; unchanged files should use cache. We should have hashes to assist with this.

## Initial Assessment Modal

- The total that shows over the welcome page during initial analysis should have a close icon in the top right corner of it. When that is clicked on, the initial analysis is aborted, and whatever was stored in that process is removed from the database. This should be respectful of true initial analysis, where a workspace is selected from the file system, and should also be mindful of when initial analyses have to be rerun because the data is stale or it has been lost. Either way, the graceful process of canceling the current analysis should be done, and it should be done without corrupting the data for that particular workspace that is being analyzed.
- On the hotspot review pane of the initial assessment modal, the top five hotspots consistently seem to show files that have a high score. I understand that this is an intersection of files that have worst health scores, and so I'm really confused as to why we're seeing files that have such a high health score. For example, the top file in that list has a score of 88 out of 100, but when I go into the application that had been analyzed, there are files with a much lower score. Can we just double-check the algorithm for this and ensure that we're showing the top five hotspot files? code-complexity-analyzer subagent can assist
- On the Action Planning pane of the InitialAssessmentWizard, we need to make sure that the numbers are using `toLocaleString` so that we have the proper formatting.
- We should rename the task list or to-do list the follow-up items. You need to update any file names and references so that the list is called "Follow-up".
- On the Action Planning pane of the initial assessment modal, we need to let the user know that we have added the action items from that pane into the Follow-up Items, which will be in the Follow-up page. When we perform this initial assessment, we should add these to that list for them automatically.

- We should create a Widget in the Widget Library. After the initial assessment has been opened, that will be one of the first modules that shows up on the overview page by default, with check boxes to the left of them. The user crosses items off this list. Those items are crossed off in the database and don't show up in the follow-up page. It goes to the follow-up page. These items are also listed there. This module spans two columns wide as a medium-sized module and it shows items that they can work on. When we first launch the application for the user, this list will be a list of items for welcoming them to the Vipr application. The user is able to choose the Configure option from the dropdown for that widget and change the list from the follow-up page. They should see different items in the list. The list they will be shown by default on the overview screen is the "Vipr Onboarding" list. It will include one item for going through the walkthrough of the application, another item for configuring widgets on the overview page, and then the remainder of the items are the items provided to them in the action planning pane of the initial assessment modal.

We should also make sure that these list items are smart. Using deep linking as part of react-router, if I have an action item that's focused on something like "address 1049 critical issues", when I click on that action item, it should take me to the issues page and show me that exact number of critical issues. If I need to fix dead code, it will take me to the anti-patterns detail page for dead code and show me the dead code issues. If I need a refactor high-impact files, it will take me to the files view, filtering down those high-impact files as the list of things for me to work on. To do this, we need to create better deep linking for our pages so that we can filter the results as granularly as possible based on the filters that are provided on those pages.

# Overview

When adding tables to the overview page from the widget library, those tables should not be shown as separate sections on the page. They should instead be added as additional tabs in the tabbed section below where the tables are. By default we show the top issues table and at-risk files tables; however if the user goes into the customize button and chooses a table from the list and adds it to the dashboard, it will add up as a third tab at the bottom of the page next to the first two. This will default to showing 10 results for that specific table.

To accommodate this functionality we should add a button to the right side across from the tabs that is a gear icon. When the user clicks on the gear icon they should see the tabs expand to show an icon in the tab to delete the table from the list of tab-to-tables. When they click away from the tabs to blur by clicking on the background, that functionality goes away. I prefer to use an icon like a minus sign rather than a trash can. The tabs themselves should be shaking when in a mode for deletion, like an iOS app would if it was queued for deletion. That way, the user knows the tabs are being edited and they can remove them. As with the other tables, sorting with table columns through the headers should apply.

- The Complexity Distribution chart inside the Complexity Distribution widget, for some reason, has a black line or a black border around the chart itself. That color is not correct in dark mode. It should be the same color as the background of the individual widgets.In light mode, it looks fine.
- In light mode, the HTML legend labels for the complexity distribution chart should have a light border around them. Right now they only have a light shadow at the bottom, and so you can't entirely make out the pill itself.
- The TypeScript Health widget has all of the components, but it is not really positioned correctly.
- The score and its denominator are too small and are at the very top left underneath the header of the module. That should be centered in the center of the module.
- Underneath it should be a score text like "Excellent". The "mid." followed by the number of files should be a slightly bigger font than they are.
- The TypeScript Health metric bar chart that sits at the very bottom of that container is too thin. It should be thicker. It shouldn't necessarily need to span the entire width of its container. It should probably be reduced to about 75%, and it should sit underneath the score and its sub-labels underneath. All in all, all three of these things should sit together more towards the center of the module and take up a little bit more size and thickness.
- Issue distribution widget on the dashboard looks great with one exception. The right side of that chart is butting up too close to the right side of the container. While keeping the left where it is, we should make some adjustments so that that chart feels more centered and that the right side of that module has a bit more spacing.

-

# Budget

# Cleanup

- Use of `console.log` in tests and scripts should try to make use of the shared logger package.
- licensing-server shouldn't log data into the console
- desktop tests have `workspace-analysis-service` warnings that need fixing:

```bash
@vipr/desktop:test: [warn] [worktree-analysis-service] Failed to analyze file in worktree {
@vipr/desktop:test:   worktreePath: '/repo/wt1',
@vipr/desktop:test:   filePath: 'src/changed.ts',
@vipr/desktop:test:   error: TypeError: Cannot read properties of undefined (reading 'size')
@vipr/desktop:test:       at WorktreeAnalysisService.analyzeFilesOnDisk (/Users/jamesleebaker/Codespace/vipr-app/clients/desktop/src/main/analysis/worktree-analysis-service.ts:294:32)
@vipr/desktop:test:       at WorktreeAnalysisService.analyzeWorktree (/Users/jamesleebaker/Codespace/vipr-app/clients/desktop/src/main/analysis/worktree-analysis-service.ts:97:24)
@vipr/desktop:test:       at /Users/jamesleebaker/Codespace/vipr-app/clients/desktop/src/main/analysis/worktree-analysis-service.test.ts:164:7
@vipr/desktop:test:       at runTest (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:781:11)
@vipr/desktop:test:       at runSuite (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:909:15)
@vipr/desktop:test:       at runSuite (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:909:15)
@vipr/desktop:test:       at runSuite (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:909:15)
@vipr/desktop:test:       at runFiles (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:958:5)
@vipr/desktop:test:       at startTests (file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/@vitest/runner/dist/index.js:967:3)
@vipr/desktop:test:       at file:///Users/jamesleebaker/Codespace/vipr-app/node_modules/vitest/dist/chunks/runtime-runBaseTests.oAvMKtQC.js:116:7
@vipr/desktop:test: }
```

`pnpm dlx @turbo/codemod@latest update`

---

I renamed my git branch locally for the analyzed project and saw this. Is this something we can be resilient of?

```text
[branch-diff-service] Failed to get changed files between branches {
base: '0.27.1',
feature: '0.28.0',
error: Error: Command failed: git diff --name-only 0.27.1...0.28.0
fatal: ambiguous argument '0.27.1...0.28.0': unknown revision or path not in the working tree.
Use '--' to separate paths from revisions, like this:
'git <command> [<revision>...] -- [<file>...]'

      at genericNodeError (node:internal/errors:983:15)
      at wrappedFn (node:internal/errors:537:14)
      at ChildProcess.exithandler (node:child_process:417:12)
      at ChildProcess.emit (node:events:519:28)
      at maybeClose (node:internal/child_process:1101:16)
      at Socket.<anonymous> (node:internal/child_process:456:11)
      at Socket.emit (node:events:519:28)
      at Pipe.<anonymous> (node:net:346:12) {
    code: 128,
    killed: false,
    signal: null,
    cmd: 'git diff --name-only 0.27.1...0.28.0',
    stdout: '',
    stderr: "fatal: ambiguous argument '0.27.1...0.28.0': unknown revision or path not in the working tree.\n" +
      "Use '--' to separate paths from revisions, like this:\n" +
      "'git <command> [<revision>...] -- [<file>...]'\n"

}
}
[repository-handlers] New commit on 0.28.0: 20080c43
```

---
