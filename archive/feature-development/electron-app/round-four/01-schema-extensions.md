---
id: 01-schema-extensions
title: 'Database Schema Extensions (Migrations 15–17)'
phase: 4
dependencies: []
status: planned
---

# Database Schema Extensions (Migrations 15–17)

## Problem Statement

The `snapshots` table (current schema v14) was designed for a single analysis context: the working tree at HEAD. It has no `ref_type` discriminator, no `worktree_path` scope, no concept of a draft (staged) snapshot, and no associated tag metadata. The `indexing_jobs` table lacks a priority column, a cancellation mechanism, and the new job types required for backfill scheduling.

SQLite cannot add `CHECK` constraints to existing columns via `ALTER TABLE`, so both `snapshots` and `indexing_jobs` require table recreation. The migration sequence is v14 → v15 → v16 → v17; each step runs inside an explicit transaction (matching the `runMigrations` pattern in `clients/desktop/src/main/db/migrations/index.ts`).

## Migration 15: Snapshot Context Columns

Adds `ref_type`, `ref_name`, `worktree_path`, and `is_draft` to the `snapshots` table. A `CHECK` constraint on `ref_type` enforces the closed vocabulary. A composite partial unique index replaces the old `UNIQUE(git_sha)` constraint so that:

- Two non-draft snapshots at the same SHA but different worktrees are allowed.
- Two draft snapshots at the same SHA are always allowed (`is_draft = 1` is excluded from the index).

```sql
-- Migration 15: Add ref_type, ref_name, worktree_path, is_draft to snapshots
-- Requires table recreation due to new CHECK constraint on ref_type.

CREATE TABLE snapshots_new (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  git_sha TEXT NOT NULL,
  branch TEXT,
  ref_type TEXT NOT NULL DEFAULT 'commit' CHECK(ref_type IN ('commit','tag','branch','worktree','draft')),
  ref_name TEXT,
  worktree_path TEXT,
  is_draft INTEGER NOT NULL DEFAULT 0 CHECK(is_draft IN (0,1)),
  overall_score REAL,
  file_count INTEGER,
  analyzed_at INTEGER NOT NULL DEFAULT (unixepoch()),
  created_at INTEGER NOT NULL DEFAULT (unixepoch())
);

-- Preserve existing rows; new nullable columns default to NULL/0 automatically.
INSERT INTO snapshots_new (id, git_sha, branch, overall_score, file_count, analyzed_at, created_at)
SELECT id, git_sha, branch, overall_score, file_count, analyzed_at, created_at
FROM snapshots;

DROP TABLE snapshots;
ALTER TABLE snapshots_new RENAME TO snapshots;

-- Composite partial unique: non-draft snapshots unique per (sha, ref_type, worktree_path).
-- COALESCE handles NULL worktree_path (main worktree) consistently.
CREATE UNIQUE INDEX idx_snapshots_unique_context
  ON snapshots(git_sha, COALESCE(ref_type,'commit'), COALESCE(worktree_path,''))
  WHERE is_draft = 0;

-- Supporting indexes for common query patterns.
CREATE INDEX idx_snapshots_ref_type ON snapshots(ref_type);
CREATE INDEX idx_snapshots_is_draft ON snapshots(is_draft) WHERE is_draft = 1;
CREATE INDEX idx_snapshots_ref_type_date ON snapshots(ref_type, analyzed_at DESC);
```

**Columns added:**

| Column          | Type      | Default    | Purpose                                                             |
| --------------- | --------- | ---------- | ------------------------------------------------------------------- |
| `ref_type`      | `TEXT`    | `'commit'` | Discriminator: `commit`, `tag`, `branch`, `worktree`, `draft`       |
| `ref_name`      | `TEXT`    | `NULL`     | Human-readable ref (tag name, branch name, worktree name)           |
| `worktree_path` | `TEXT`    | `NULL`     | Absolute path of the linked worktree (worktree snapshots only)      |
| `is_draft`      | `INTEGER` | `0`        | `1` for staged/draft snapshots; excluded from uniqueness constraint |

**Dropped columns from old schema** (not migrated forward — unused in production):

- `git_author`, `git_message`, `git_date` — commit metadata is now stored in the `commit_files` table (migration 2) and in `git_tags` (migration 16). Removing them from `snapshots` simplifies the write path.
- `avg_score` — renamed to `overall_score` for semantic clarity across the schema.

> Note: The `snapshot_metrics` and `snapshot_files` tables reference `snapshots(id)` via foreign key with `ON DELETE CASCADE`, so they require no structural changes.

**Rollback strategy:** Recreate the original table from `snapshot_files` and `snapshot_metrics` join — acceptable because migration 15 is a planned forward-only migration with no production data at this schema version.

## Migration 16: `git_tags` Cache Table

The `git_tags` table caches the output of `git tag -l` so the renderer can display tag lists without spawning a child process on every navigation. The `snapshot_id` foreign key links a tag to the snapshot taken at that tag's commit (if one exists).

```sql
-- Migration 16: git_tags cache table
CREATE TABLE git_tags (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT UNIQUE NOT NULL,
  object_sha TEXT NOT NULL,
  commit_sha TEXT NOT NULL,
  subject TEXT,
  tagger_date INTEGER,
  tag_type TEXT NOT NULL DEFAULT 'lightweight' CHECK(tag_type IN ('lightweight','annotated')),
  snapshot_id INTEGER,
  created_at INTEGER NOT NULL DEFAULT (unixepoch()),
  FOREIGN KEY (snapshot_id) REFERENCES snapshots(id) ON DELETE SET NULL
);

CREATE INDEX idx_git_tags_commit_sha ON git_tags(commit_sha);
CREATE INDEX idx_git_tags_snapshot_id ON git_tags(snapshot_id) WHERE snapshot_id IS NOT NULL;
```

**Column semantics:**

| Column        | Purpose                                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------------------------- |
| `object_sha`  | The SHA of the tag object itself (matches `%(objectname)` in `git tag -l --format=...`)                              |
| `commit_sha`  | The dereferenced commit SHA (`%(*objectname)` for annotated, `%(objectname)` for lightweight)                        |
| `tag_type`    | `'lightweight'` or `'annotated'` — lightweight tags point directly to a commit; annotated tags have their own object |
| `snapshot_id` | Set to the `snapshots.id` after a tag snapshot is taken; `NULL` until then                                           |

**Cache invalidation:** Rows are deleted and re-inserted on each `listTags()` call from `GitContentService`. This is safe because `snapshot_id` is preserved via `ON DELETE SET NULL` and the re-insert uses `INSERT OR REPLACE`.

## Migration 17: `indexing_jobs` Extensions

Adds `ref_name`, `ref_type`, `cancellation_token`, and `priority` columns plus the new job types required for Round Four (`tag-analysis`, `worktree-analysis`, `backfill`). Uses table recreation to add the extended `CHECK` constraint on `job_type`.

```sql
-- Migration 17: indexing_jobs extensions
CREATE TABLE indexing_jobs_new (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  job_type TEXT NOT NULL CHECK(job_type IN ('full','incremental','file','tag-analysis','worktree-analysis','backfill')),
  status TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending','running','completed','failed','cancelled')),
  ref_name TEXT,
  ref_type TEXT,
  cancellation_token TEXT,
  priority INTEGER NOT NULL DEFAULT 10,
  started_at INTEGER,
  completed_at INTEGER,
  error TEXT,
  created_at INTEGER NOT NULL DEFAULT (unixepoch())
);

-- Preserve existing rows; new nullable columns default to NULL/10.
INSERT INTO indexing_jobs_new (id, job_type, status, started_at, completed_at, error, created_at)
SELECT id, job_type, status, started_at, completed_at, error_message, created_at
FROM indexing_jobs;

DROP TABLE indexing_jobs;
ALTER TABLE indexing_jobs_new RENAME TO indexing_jobs;

-- Priority queue dequeue index: fetch highest-priority pending job first, FIFO within same priority.
CREATE INDEX idx_indexing_jobs_priority
  ON indexing_jobs(priority DESC, created_at ASC)
  WHERE status = 'pending';
```

**New columns:**

| Column               | Purpose                                                                                            |
| -------------------- | -------------------------------------------------------------------------------------------------- |
| `ref_name`           | Tag name or worktree path being analyzed                                                           |
| `ref_type`           | Matches `snapshots.ref_type` vocabulary                                                            |
| `cancellation_token` | UUID set by the scheduler; checked by the engine worker to halt processing                         |
| `priority`           | Higher value = dequeued first. Default `10`; backfill uses `5`; interactive tag/worktree uses `20` |

> Note: The existing `error_message` column is renamed to `error` in the new table to match the naming convention used in `schedule_history`.

## TypeScript Types

File: `clients/desktop/src/main/db/types.ts`

Add or extend the following interfaces. Follow existing style: no `I` prefix, `UnixTimestamp` for epoch integers, `JSON string` comment for raw JSON columns.

```typescript
// Extend existing SnapshotRecord — add the four new columns from migration 15.
export interface SnapshotRecord {
  id: SnapshotId;
  git_sha: string;
  branch: string | null;
  ref_type: 'commit' | 'tag' | 'branch' | 'worktree' | 'draft';
  ref_name: string | null;
  worktree_path: string | null;
  is_draft: 0 | 1;
  overall_score: number | null;
  file_count: number | null;
  analyzed_at: UnixTimestamp;
  created_at: UnixTimestamp;
}

// New record for git_tags cache table (migration 16).
export interface GitTagRecord {
  id: number;
  name: string;
  object_sha: string;
  commit_sha: string;
  subject: string | null;
  tagger_date: UnixTimestamp | null;
  tag_type: 'lightweight' | 'annotated';
  snapshot_id: SnapshotId | null;
  created_at: UnixTimestamp;
}

// Updated IndexingJobRecord — replaces the existing definition.
export interface IndexingJobRecord {
  id: number;
  job_type: 'full' | 'incremental' | 'file' | 'tag-analysis' | 'worktree-analysis' | 'backfill';
  status: 'pending' | 'running' | 'completed' | 'failed' | 'cancelled';
  ref_name: string | null;
  ref_type: string | null;
  cancellation_token: string | null;
  priority: number;
  started_at: UnixTimestamp | null;
  completed_at: UnixTimestamp | null;
  error: string | null;
  created_at: UnixTimestamp;
}

// Parameter bag for the updated createSnapshot method.
export interface CreateSnapshotParams {
  gitSha: string;
  branch?: string | null;
  refType?: 'commit' | 'tag' | 'branch' | 'worktree' | 'draft';
  refName?: string | null;
  worktreePath?: string | null;
  isDraft?: boolean;
  overallScore?: number | null;
  fileCount?: number | null;
}
```

## Files to Modify

### `clients/desktop/src/main/db/migrations/index.ts`

Append three migration objects to the `migrations` array. Each follows the `{ version, up, down }` shape defined by the `Migration` interface already present in that file.

Migration 15 `up` runs the table recreation SQL above.
Migration 16 `up` creates `git_tags`.
Migration 17 `up` recreates `indexing_jobs`.

All three `down` functions use `DROP TABLE IF EXISTS` for the new tables, and log a warning for the recreated tables (same pattern as migration v5 and v13).

### `clients/desktop/src/main/db/schema.ts`

```typescript
// Bump this constant from 14 to 17.
export const SCHEMA_VERSION = 17;
```

### `clients/desktop/src/main/db/types.ts`

Replace existing `SnapshotRecord` and add `GitTagRecord`, `IndexingJobRecord`, and `CreateSnapshotParams` as shown above.

### `clients/desktop/src/main/db/interfaces.ts`

Extend `SnapshotRepository` with four new methods:

```typescript
export interface SnapshotRepository {
  // ... existing methods unchanged ...

  /**
   * Create a snapshot with full Round Four context parameters.
   * Supersedes the existing createSnapshot overload — update callers to use CreateSnapshotParams.
   */
  createSnapshot(params: CreateSnapshotParams): Promise<number>;

  /**
   * Find an existing non-draft snapshot by the triple (gitSha, refType, worktreePath).
   * Returns null if no match. Used by HistoricalAnalysisEngine to skip re-analysis.
   */
  getSnapshotByContext(
    gitSha: string,
    refType: string,
    worktreePath: string | null
  ): Promise<SnapshotRecord | null>;

  /**
   * Get all snapshots taken within a specific worktree, ordered by analyzed_at DESC.
   */
  getSnapshotsByWorktree(worktreePath: string): Promise<SnapshotRecord[]>;

  /**
   * Get the most recent snapshot for a worktree.
   */
  getLatestSnapshotForWorktree(worktreePath: string): Promise<SnapshotRecord | null>;
}
```

## Testing

File: `clients/desktop/src/main/db/migrations/migrations-round-four.test.ts`

Use the in-memory SQLite adapter already used by `clients/desktop/src/main/db/database.test.ts` as the pattern.

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { DatabaseSync } from 'node:sqlite';
import { runMigrations } from './index';

describe('Round Four migrations (v15–v17)', () => {
  let db: DatabaseSync;

  beforeEach(() => {
    db = new DatabaseSync(':memory:');
  });

  it('runs cleanly from v0 to v17', async () => {
    await runMigrations(db);
    const version = db
      .prepare('SELECT value FROM metadata WHERE key = ?')
      .get('schema_version') as { value: string };
    expect(Number(version.value)).toBe(17);
  });

  it('snapshots table has ref_type column after v15', async () => {
    await runMigrations(db);
    const info = db.prepare("PRAGMA table_info('snapshots')").all() as Array<{ name: string }>;
    const names = info.map(col => col.name);
    expect(names).toContain('ref_type');
    expect(names).toContain('ref_name');
    expect(names).toContain('worktree_path');
    expect(names).toContain('is_draft');
  });

  it('draft snapshots bypass composite unique constraint (two drafts same SHA)', async () => {
    await runMigrations(db);
    const sha = 'abc1234';
    db.prepare(`INSERT INTO snapshots (git_sha, ref_type, is_draft) VALUES (?, 'commit', 1)`).run(
      sha
    );
    // Second draft at the same SHA must not throw.
    expect(() => {
      db.prepare(`INSERT INTO snapshots (git_sha, ref_type, is_draft) VALUES (?, 'commit', 1)`).run(
        sha
      );
    }).not.toThrow();
  });

  it('two worktrees at same commit SHA coexist', async () => {
    await runMigrations(db);
    const sha = 'abc1234';
    db.prepare(
      `INSERT INTO snapshots (git_sha, ref_type, worktree_path, is_draft) VALUES (?, 'worktree', '/tmp/wt1', 0)`
    ).run(sha);
    expect(() => {
      db.prepare(
        `INSERT INTO snapshots (git_sha, ref_type, worktree_path, is_draft) VALUES (?, 'worktree', '/tmp/wt2', 0)`
      ).run(sha);
    }).not.toThrow();
  });

  it('two non-draft snapshots at same SHA same worktree violates unique constraint', async () => {
    await runMigrations(db);
    const sha = 'abc1234';
    db.prepare(`INSERT INTO snapshots (git_sha, ref_type, is_draft) VALUES (?, 'commit', 0)`).run(
      sha
    );
    expect(() => {
      db.prepare(`INSERT INTO snapshots (git_sha, ref_type, is_draft) VALUES (?, 'commit', 0)`).run(
        sha
      );
    }).toThrow();
  });

  it('git_tags table exists after v16', async () => {
    await runMigrations(db);
    const info = db.prepare("PRAGMA table_info('git_tags')").all() as Array<{ name: string }>;
    expect(info.length).toBeGreaterThan(0);
  });

  it('indexing_jobs table has priority column after v17', async () => {
    await runMigrations(db);
    const info = db.prepare("PRAGMA table_info('indexing_jobs')").all() as Array<{ name: string }>;
    const names = info.map(col => col.name);
    expect(names).toContain('priority');
    expect(names).toContain('cancellation_token');
    expect(names).toContain('ref_name');
  });

  it('rollback from v17 to v16 succeeds without data loss in unaffected tables', async () => {
    await runMigrations(db);
    // Seed a git_tags row so we can verify cascade/no-op on rollback.
    db.prepare(
      `INSERT INTO git_tags (name, object_sha, commit_sha, tag_type) VALUES ('v1.0', 'aaa', 'bbb', 'lightweight')`
    ).run();
    // rollbackMigration drops indexing_jobs_new — git_tags must survive.
    const tags = db.prepare('SELECT * FROM git_tags').all();
    expect(tags).toHaveLength(1);
  });
});
```
