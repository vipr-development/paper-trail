# TODO - this phase is focused on understanding how an application is initially analyzed and its findings stored in the sqlite database and how subsequent analyses occur. The goal is to gain clarity on if ingestion of analytics is correct and if incremental analysis is properly conducted over time.

Presently, when the application is loaded fresh in dev mode and a folder is selected for analysis, even if it already exists in the list of existing workspaces, it is re-analyzed in full. This is not the intended behavior of the application. Though a full scan is conducted, only files that have changed—supposedly indicated through some type of hashing logic—are re-analyzed; all others are skipped. This doesn't seem consistent with how the application currently behaves.

Especially with the introduction of git-level analysis and adaptability to git branching and worktrees, the application should be resilient in what it analyses, re-analyses, garbage collects/archives, and skips analysis on.
