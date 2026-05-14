# Performance Baselines

> Raw `Performance summary` log lines captured from `pnpm --filter @vipr/desktop dev`
> after Phase 0.5 instrumentation landed. Used as the comparison floor for every
> subsequent phase. Re-capture using [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md).

## Test conditions

- **Repository analyzed**: `/Users/jamesleebaker/Codespace/vipr/clients/desktop` (the desktop client itself)
- **Files**: 1,013 source files within scope
- **Backfill window**: 90 days, 394 commits, full coverage
- **Machine**: macOS, multi-core (24 logical cores observed)
- **State**: foreground run after deleting `vipr-default.db` (warm OS file cache; not a true cold start, but a consistent reference point)
- **Branch**: `claude/unruffled-hawking-f89a04` at commit `b193bed2`

Foreground and backfill ran sequentially (not overlapping). The user reported
~64s gap between foreground end and backfill start. So foreground numbers are
CPU-uncontended. Backfill numbers are also CPU-uncontended (no foreground in
flight).

## Foreground baseline

```jsonc
{
  "scope": "foreground",
  "label": "5506e422-9bce-4726-9d6b-4158498b0dfb",
  "startedAt": 1778349708591,
  "endedAt": 1778349986319,
  "totalDurationMs": 277728,                   // 4 min 37.7 s
  "totalFiles": 1013,
  "totalCommits": 0,
  "totalBackfillFiles": 0,
  "totalBatches": 0,
  "filesPerSecond": 3.65,
  "peakInFlight": 2,
  "fileTime":         { "count": 1013, "mean": 548.0, "min": 7,  "max": 12665, "p50": 389,   "p95": 1496.8, "p99": 3795 },
  "utilityRoundTrip": { "count": 1013, "mean": 545.7, "min": 6,  "max": 12657, "p50": 388,   "p95": 1493,   "p99": 3792.2 },
  "dbPersist":        { "count": 1013, "mean": 1.2,   "min": 0,  "max": 11,    "p50": 1,     "p95": 3,      "p99": 7.9 },
  "eventEmit":        { "count": 1013, "mean": 0.2,   "min": 0,  "max": 2,     "p50": 0,     "p95": 1,      "p99": 1 },
  "needsAnalysis":    { "count": 1013, "mean": 0.5,   "min": 0,  "max": 5,     "p50": 0,     "p95": 1,      "p99": 2 },
  "ipcEventCounts": {
    "analysis:queue-updated": 1013,
    "analysis:phase":         2,
    "analysis:file-started":  1013,
    "analysis:file-completed": 1013
  },
  "dbOpCounts": {
    "updateAnalysisRun":  1015,
    "saveAnalysisResult": 1013
  },
  "slowestFiles": [
    { "filePath": "clients/desktop/src/renderer/pages/FileDetail.tsx",                      "totalMs": 12665, "utilityRoundTripMs": 12657 },
    { "filePath": "clients/desktop/src/renderer/pages/Faq.tsx",                              "totalMs": 12610, "utilityRoundTripMs": 12608 },
    { "filePath": "clients/desktop/.storybook/preview.tsx",                                  "totalMs": 5356,  "utilityRoundTripMs": 5350  },
    { "filePath": "clients/desktop/.storybook/main.ts",                                      "totalMs": 5353,  "utilityRoundTripMs": 5349  },
    { "filePath": "clients/desktop/src/renderer/pages/AntiPatterns.tsx",                     "totalMs": 4868,  "utilityRoundTripMs": 4865  },
    { "filePath": "clients/desktop/src/renderer/pages/AntiPatternDetail.tsx",                "totalMs": 4866,  "utilityRoundTripMs": 4859  },
    { "filePath": "clients/desktop/src/renderer/components/file-detail/ABAnalysisWizard.tsx","totalMs": 4430,  "utilityRoundTripMs": 4419  },
    { "filePath": "clients/desktop/src/renderer/components/facets/index.ts",                 "totalMs": 4384,  "utilityRoundTripMs": 4383  },
    { "filePath": "clients/desktop/src/renderer/pages/AlertInvestigation.tsx",               "totalMs": 4072,  "utilityRoundTripMs": 4067  },
    { "filePath": "clients/desktop/src/renderer/pages/ActionItems.tsx",                      "totalMs": 4054,  "utilityRoundTripMs": 4045  }
  ]
}
```

## Backfill baseline (90 days, 394 commits)

```jsonc
{
  "scope": "backfill",
  "label": "backfill:fa010c05-c56a-4b18-8b71-41a778139de9",
  "startedAt": 1778350024528,
  "endedAt":   1778355705985,
  "totalDurationMs": 5681457,                  // 94 min 41.5 s
  "totalCommits": 394,
  "totalBackfillFiles": 7330,
  "totalBatches": 521,
  "commitTime":       { "count": 394,  "mean": 14374.7, "p50": 3480.5, "p95": 62762.6, "p99": 98010.4, "max": 377160 },
  "backfillFileTime": { "count": 7330, "mean": 2037.7,  "p50": 1848.2, "p95": 4933.5,  "p99": 8183.4,  "max": 18009.5 },
  "batchTime":        { "count": 521,  "mean": 7565.3,  "p50": 3497.4, "p95": 23403.8, "p99": 28326.2, "max": 29977.9 },
  "batchSize":        { "count": 521,  "mean": 14.1,    "p50": 8,      "p95": 32,      "p99": 32,      "max": 32 },
  "ipcEventCounts": {
    "backfill:started":   1,
    "backfill:progress":  394,
    "backfill:completed": 1
  },
  "dbOpCounts": { "updateIndexingJob": 396 },
  "slowestCommits": [
    { "sha": "175a5c4361fe9a00d6ae65b1ca067acc6c9bee6e", "totalMs": 377160, "filesAnalyzed": 593  }, // 6:17 — investigate
    { "sha": "7a19bc12e98a2949d3e92c3af50aed6b0780f6d1", "totalMs": 219093, "filesAnalyzed": 984  }, // 3:39 — likely formatting/rename
    { "sha": "5f338212c8ea80f6b12cb617dd9d4a2d04ad8d04", "totalMs": 148062, "filesAnalyzed": 1015 },
    { "sha": "41256cdb4024425290242dedd63e09d36d39712c", "totalMs": 120415, "filesAnalyzed": 179  },
    { "sha": "3c76e45f084c67024dea8ff980c447b1db3c6b50", "totalMs": 96324,  "filesAnalyzed": 179  },
    { "sha": "26567f8975cb1a8886026b19817fa04e3bd60a7a", "totalMs": 92926,  "filesAnalyzed": 155  },
    { "sha": "ccbcef61aded3b79f9976a25bd9ea373f4914c01", "totalMs": 90925,  "filesAnalyzed": 201  },
    { "sha": "fffa08d4f0a92d637c87c7fa2f90245f8e01f1fc", "totalMs": 89219,  "filesAnalyzed": 1034 },
    { "sha": "ee69d759903ad8b23d622321a088d1cea2f14d4e", "totalMs": 88774,  "filesAnalyzed": 143  },
    { "sha": "1c3a77e40861da9122f654aa368deb29adb447ac", "totalMs": 85707,  "filesAnalyzed": 134  }
  ],
  "slowestBackfillFiles": [
    { "filePath": "src/renderer/components/file-detail/ABAnalysisWizard.tsx", "analyzeMs": 18009.5, "fromSha": "7181b0ff", "contentBytes": 39499 },
    { "filePath": "src/main/ipc/mock/page-mock-handlers.ts",                  "analyzeMs": 17523.4, "fromSha": "7181b0ff", "contentBytes": 35987 },  // BUG: should be excluded
    { "filePath": "src/renderer/components/settings/ExportSettingsSection.tsx","analyzeMs": 17074,  "fromSha": "921c4ff5", "contentBytes": 24769 },
    { "filePath": "src/renderer/components/layout/Sidebar.tsx",               "analyzeMs": 16454,   "fromSha": "2a23b12a", "contentBytes": 18384 },
    { "filePath": "src/renderer/components/settings/LicenseSection.tsx",      "analyzeMs": 15725,   "fromSha": "2a23b12a", "contentBytes": 11547 },
    { "filePath": "src/renderer/components/dependencies/graph-utils.ts",      "analyzeMs": 15599.5, "fromSha": "921c4ff5", "contentBytes": 4483  },
    { "filePath": "src/renderer/components/file-detail/ABAnalysisWizard.tsx", "analyzeMs": 15416.1, "fromSha": "5e2c6f32", "contentBytes": 34762 },
    { "filePath": "src/renderer/components/file-detail/FileDetailHeader.tsx", "analyzeMs": 15411.3, "fromSha": "5e2c6f32", "contentBytes": 5217  },
    { "filePath": "src/renderer/pages/FileDetail.tsx",                        "analyzeMs": 15310.3, "fromSha": "5e2c6f32", "contentBytes": 52536 },
    { "filePath": "src/renderer/pages/DependencyCascade.tsx",                 "analyzeMs": 15193.9, "fromSha": "86a8fec0", "contentBytes": 46130 }
  ]
}
```

## Cross-reading these numbers

- **Foreground wall clock = (mean per-file × file count) ÷ peak in-flight**
  `(548 × 1013) ÷ 2 = 277,562ms ≈ 277,728ms actual`
  Confirms analysis is **serial-blocked on per-file CPU work, two at a time**.

- **Backfill per-file looks 3.7× foreground but isn't.**
  `mean batchTime ÷ mean batchSize = 7565 ÷ 14.1 = 537ms per file` at the throughput level — essentially identical to foreground (548ms). The 2,038ms per-file `analyzeMs` is an artifact of `historicalBatchConcurrency: 4` running 4 files concurrently in the same V8 isolate. See [`03-backfill-concurrency.md`](./03-backfill-concurrency.md).

- **Same files appear in both slowest lists.** Per-file cost is intrinsic to file shape (large React page components, dense barrels), not state-dependent. Foreground and backfill share fix targets.

- **One commit (`175a5c43`) ate 6:17 alone** on 593 files (636ms/file — above mean). Worth investigating — `git show --stat 175a5c43 | head` to see what kind of change. Likely a refactor that touched many heavy page components.
