---
id: 02-git-content-service
title: 'Git Content Retrieval Service'
phase: 4
dependencies:
  - 01-schema-extensions
status: planned
---

# Git Content Retrieval Service

## Problem Statement

No existing service retrieves historical file content via `git show <sha>:<path>`, lists tags, or walks commit ranges. `BranchDiffService` (`clients/desktop/src/main/git/branch-diff-service.ts`) is the closest pattern: it uses `execFile` (never `exec`), validates branch names implicitly through git, and wraps all operations in structured error handling. `GitContentService` follows the same conventions and becomes the single source of truth for all git data retrieval in Round Four.

Without this service, `HistoricalAnalysisEngine` (Phase 03) cannot retrieve file content at past commits, and `BackfillScheduler` (Phase 04) cannot enumerate which files existed at each historical SHA.

## New File

`clients/desktop/src/main/git/git-content-service.ts`

## Key Types

Define these in `git-content-service.ts` — they are not shared types, so no barrel export is needed.

```typescript
interface CommitWithFiles {
  sha: string;
  authorName: string;
  subject: string;
  timestamp: number;
  parentShas: string[];
  changedFiles: string[];
}

interface TagInfo {
  name: string;
  objectSha: string;
  commitSha: string; // dereferenced for annotated tags; same as objectSha for lightweight
  tagType: 'lightweight' | 'annotated';
  subject: string | null;
  taggerDate: number | null;
}
```

## Class Interface

```typescript
export class GitContentService {
  constructor(private readonly repoPath: string) {}

  /**
   * Retrieve the content of a file at a specific commit.
   * Returns null if the file did not exist at that commit (git exits non-zero).
   */
  getFileAtCommit(filePath: string, commitSha: string): Promise<string | null>;

  /**
   * Batch-retrieve multiple files at a single commit.
   * Internally batches in chunks of 10 parallel requests to avoid spawning
   * hundreds of child processes for large historical snapshots.
   */
  getFilesAtCommit(filePaths: string[], commitSha: string): Promise<Map<string, string>>;

  /**
   * Walk backwards from startSha, returning up to `limit` commits with their
   * changed file lists. Commits are in reverse chronological order (newest first).
   * Uses git-log + git-diff-tree; does not use git-show for efficiency.
   */
  getCommitRange(startSha: string, limit: number): Promise<CommitWithFiles[]>;

  /**
   * Get the set of files changed between two adjacent commits.
   * Delegates to `git diff-tree --no-commit-id -r --name-only`.
   */
  getChangedFilesBetweenCommits(parentSha: string, childSha: string): Promise<string[]>;

  /**
   * List all tags in the repository.
   * Results are cached in git_tags table by the caller (TagSnapshotService, Phase 05).
   */
  listTags(): Promise<TagInfo[]>;

  /**
   * Resolve a tag name to the commit SHA it points to.
   * Handles annotated tag dereferencing via `git rev-list -n 1`.
   * Returns null if the tag does not exist.
   */
  resolveTagToCommit(tagName: string): Promise<string | null>;

  /**
   * Get all files tracked by git at a specific commit.
   * Used by HistoricalAnalysisEngine for full snapshot construction.
   * Delegates to `git ls-tree -r --name-only <commitSha>`.
   */
  getTrackedFilesAtCommit(commitSha: string): Promise<string[]>;
}
```

## Security Design

All inputs that flow into `execFile` arguments must be validated before use. The patterns below match the philosophy used in `BranchDiffService` (no shell interpolation, `execFile` only).

### SHA Validation

```typescript
const SHA_REGEX = /^[0-9a-f]{7,40}$/i;

function validateSha(sha: string, label = 'SHA'): void {
  if (!SHA_REGEX.test(sha)) {
    throw new Error(`Invalid ${label}: "${sha}" — must be 7–40 hex characters`);
  }
}
```

Apply `validateSha` to every `commitSha`, `parentSha`, `childSha`, and `startSha` parameter before the `execFile` call.

### File Path Validation

```typescript
const PATH_UNSAFE = /[\0|;&$`<>]/;

function validateFilePath(filePath: string): void {
  if (filePath.includes('..') || PATH_UNSAFE.test(filePath)) {
    throw new Error(`Unsafe file path rejected: "${filePath}"`);
  }
}
```

Apply to every `filePath` before passing as a `git show` argument.

### Tag Name Validation

```typescript
const TAG_REGEX = /^[a-zA-Z0-9._\-\/]+$/;

function validateTagName(tagName: string): void {
  if (!TAG_REGEX.test(tagName)) {
    throw new Error(`Invalid tag name: "${tagName}"`);
  }
}
```

Apply in `resolveTagToCommit` before passing to `execFile`.

## Implementation Details

### `getFileAtCommit`

Git command:

```
git show <commitSha>:<filePath>
```

- Call `validateSha(commitSha)` and `validateFilePath(filePath)` first.
- Use `execFile('git', ['show', `${commitSha}:${filePath}`], { cwd: this.repoPath })`.
- Return `stdout` on success; catch the error and return `null` if git exits non-zero (file absent at that commit).
- Do not log at warn level for absent files — they are expected during historical traversal of growing codebases.

### `getFilesAtCommit`

```typescript
async getFilesAtCommit(filePaths: string[], commitSha: string): Promise<Map<string, string>> {
  validateSha(commitSha);
  const result = new Map<string, string>();
  const chunks = chunk(filePaths, 10);

  for (const batch of chunks) {
    const settled = await Promise.allSettled(
      batch.map(async p => {
        const content = await this.getFileAtCommit(p, commitSha);
        return { path: p, content };
      })
    );

    for (const outcome of settled) {
      if (outcome.status === 'fulfilled' && outcome.value.content !== null) {
        result.set(outcome.value.path, outcome.value.content);
      }
    }
  }

  return result;
}
```

The `chunk` helper is a local utility function — do not import lodash. Implement as:

```typescript
function chunk<T>(arr: T[], size: number): T[][] {
  const out: T[][] = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
```

### `listTags`

Git command:

```
git tag -l --format=%(refname:short)|%(objectname)|%(objecttype)|%(*objectname)|%(subject)|%(taggerdate:unix)
```

Parse each non-empty output line on `|`:

| Index | Field        | Notes                                                              |
| ----- | ------------ | ------------------------------------------------------------------ |
| 0     | `name`       | Short tag name                                                     |
| 1     | `objectSha`  | SHA of the tag object (or commit for lightweight)                  |
| 2     | `objectType` | `'tag'` for annotated, `'commit'` for lightweight                  |
| 3     | `derefSha`   | `%(*objectname)` — empty for lightweight, commit SHA for annotated |
| 4     | `subject`    | Tag message first line (empty string → `null`)                     |
| 5     | `taggerDate` | Unix timestamp string (empty string → `null`)                      |

Derivation logic:

```typescript
const tagType = objectType === 'tag' ? 'annotated' : 'lightweight';
const commitSha = derefSha.length > 0 ? derefSha : objectSha;
```

Return an empty array (not an error) if `git tag -l` produces no output.

### `resolveTagToCommit`

```
git rev-list -n 1 <tagName>
```

- Call `validateTagName(tagName)` first.
- `rev-list -n 1` dereferences annotated tags automatically — no separate `git tag -d` needed.
- Return the trimmed stdout, or `null` if the command exits non-zero.

### `getCommitRange`

Git command:

```
git log --format=%H|%an|%s|%at|%P -n <limit> <startSha>
```

Field mapping (pipe-separated):

| Format | Field                                                       |
| ------ | ----------------------------------------------------------- |
| `%H`   | `sha`                                                       |
| `%an`  | `authorName`                                                |
| `%s`   | `subject`                                                   |
| `%at`  | `timestamp` (Unix, parse to number)                         |
| `%P`   | Parent SHAs, space-separated (split on `' '`, filter empty) |

After collecting the commit list, call `getChangedFilesBetweenCommits(commit.parentShas[0], commit.sha)` for each commit. For the initial commit (no parents), use `git diff-tree --root -r --name-only <sha>` instead.

Batch the diff-tree calls in groups of 5 to avoid overwhelming the system:

```typescript
const diffChunks = chunk(commits, 5);
for (const batch of diffChunks) {
  await Promise.all(batch.map(c => this.populateChangedFiles(c)));
}
```

### `getChangedFilesBetweenCommits`

Git command:

```
git diff-tree --no-commit-id -r --name-only <parentSha> <childSha>
```

- Validate both SHAs.
- Split stdout on newline, trim, filter empty strings.
- Returns all changed paths regardless of extension — callers filter to analyzable extensions using `SUPPORTED_FILE_EXTENSIONS` from `@vipr/common/client-constants` (same pattern as `BranchDiffService`).

### `getTrackedFilesAtCommit`

Git command:

```
git ls-tree -r --name-only <commitSha>
```

- Validate the SHA.
- Returns all tracked file paths at that commit.
- Used by `HistoricalAnalysisEngine` to construct a full snapshot rather than an incremental one.

## Modification to BranchDiffService

File: `clients/desktop/src/main/git/branch-diff-service.ts`

Add one delegating method so existing callers can access commit-level diffs without a direct dependency on `GitContentService`:

```typescript
/**
 * Get files changed between two adjacent commits.
 * Delegates to GitContentService for consistent SHA validation and git execution.
 */
async getFilesChangedBetweenCommits(parentSha: string, childSha: string): Promise<string[]> {
  const service = new GitContentService(this.repoPath);
  return service.getChangedFilesBetweenCommits(parentSha, childSha);
}
```

`GitContentService` is imported at the top of `branch-diff-service.ts` — no barrel needed.

## Testing

File: `clients/desktop/src/main/git/git-content-service.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { execFile } from 'node:child_process';
import { GitContentService } from './git-content-service';

vi.mock('node:child_process');

const mockExecFile = vi.mocked(execFile);

// Helper: make execFile resolve with stdout
function resolveWith(stdout: string) {
  mockExecFile.mockImplementation((_cmd, _args, _opts, callback) => {
    (callback as Function)(null, stdout, '');
    return {} as ReturnType<typeof execFile>;
  });
}

describe('GitContentService', () => {
  let service: GitContentService;

  beforeEach(() => {
    vi.clearAllMocks();
    service = new GitContentService('/tmp/test-repo');
  });

  describe('SHA validation', () => {
    it('rejects non-hex strings', async () => {
      await expect(service.getFileAtCommit('src/index.ts', 'not-a-sha')).rejects.toThrow('Invalid');
    });

    it('rejects SHAs shorter than 7 chars', async () => {
      await expect(service.getFileAtCommit('src/index.ts', 'abc12')).rejects.toThrow('Invalid');
    });

    it('accepts 40-char full SHAs', async () => {
      resolveWith('content');
      await expect(service.getFileAtCommit('src/index.ts', 'a'.repeat(40))).resolves.toBe(
        'content'
      );
    });

    it('accepts 7-char short SHAs', async () => {
      resolveWith('content');
      await expect(service.getFileAtCommit('src/index.ts', 'abc1234')).resolves.toBe('content');
    });
  });

  describe('file path validation', () => {
    it('rejects paths with ..', async () => {
      await expect(service.getFileAtCommit('../etc/passwd', 'abc1234')).rejects.toThrow('Unsafe');
    });

    it('rejects paths with NUL byte', async () => {
      await expect(service.getFileAtCommit('src/\0index.ts', 'abc1234')).rejects.toThrow('Unsafe');
    });

    it('rejects paths with shell metacharacters', async () => {
      await expect(service.getFileAtCommit('src/index.ts; rm -rf /', 'abc1234')).rejects.toThrow(
        'Unsafe'
      );
    });
  });

  describe('listTags', () => {
    it('parses lightweight tags (empty %(*objectname))', async () => {
      resolveWith('v1.0|abc1234|commit||First release|');
      const tags = await service.listTags();
      expect(tags).toHaveLength(1);
      expect(tags[0]).toMatchObject({
        name: 'v1.0',
        objectSha: 'abc1234',
        commitSha: 'abc1234', // same as objectSha for lightweight
        tagType: 'lightweight',
        subject: 'First release',
        taggerDate: null,
      });
    });

    it('parses annotated tags (uses %(*objectname) as commitSha)', async () => {
      resolveWith('v2.0|tagobj999|tag|commitabc123|Annotated release|1700000000');
      const tags = await service.listTags();
      expect(tags[0]).toMatchObject({
        name: 'v2.0',
        objectSha: 'tagobj999',
        commitSha: 'commitabc123', // dereferenced
        tagType: 'annotated',
        subject: 'Annotated release',
        taggerDate: 1700000000,
      });
    });

    it('returns empty array when no tags exist', async () => {
      resolveWith('');
      const tags = await service.listTags();
      expect(tags).toEqual([]);
    });
  });

  describe('getFilesAtCommit', () => {
    it('batches requests in chunks of 10', async () => {
      resolveWith('content');
      const paths = Array.from({ length: 25 }, (_, i) => `src/file${i}.ts`);
      await service.getFilesAtCommit(paths, 'abc1234');
      // 25 files → ceil(25/10) = 3 batches → 25 execFile calls total
      expect(mockExecFile).toHaveBeenCalledTimes(25);
    });

    it('returns null for files not present at commit and excludes them from the Map', async () => {
      mockExecFile
        .mockImplementationOnce((_cmd, _args, _opts, cb) => {
          (cb as Function)(new Error('fatal: path not found'), '', '');
          return {} as ReturnType<typeof execFile>;
        })
        .mockImplementationOnce((_cmd, _args, _opts, cb) => {
          (cb as Function)(null, 'found', '');
          return {} as ReturnType<typeof execFile>;
        });
      const result = await service.getFilesAtCommit(['missing.ts', 'found.ts'], 'abc1234');
      expect(result.has('missing.ts')).toBe(false);
      expect(result.get('found.ts')).toBe('found');
    });
  });

  describe('getCommitRange', () => {
    it('returns commits in reverse chronological order', async () => {
      // git log output: newest first
      resolveWith(
        [`bbb2222|Bob|Fix bug|1700000100|aaa1111`, `aaa1111|Alice|Initial commit|1700000000|`].join(
          '\n'
        )
      );
      // Stub diff-tree calls
      mockExecFile.mockImplementation((_cmd, _args, _opts, cb) => {
        (cb as Function)(null, '', '');
        return {} as ReturnType<typeof execFile>;
      });
      const commits = await service.getCommitRange('bbb2222', 2);
      expect(commits[0].sha).toBe('bbb2222');
      expect(commits[1].sha).toBe('aaa1111');
    });

    it('respects limit parameter', async () => {
      resolveWith('bbb2222|Bob|Fix bug|1700000100|aaa1111');
      mockExecFile.mockImplementation((_cmd, _args, _opts, cb) => {
        (cb as Function)(null, '', '');
        return {} as ReturnType<typeof execFile>;
      });
      const commits = await service.getCommitRange('bbb2222', 1);
      expect(commits).toHaveLength(1);
    });

    it('parses multi-parent merge commits', async () => {
      resolveWith('merge999|Merger|Merge PR|1700000200|aaa1111 bbb2222');
      mockExecFile.mockImplementation((_cmd, _args, _opts, cb) => {
        (cb as Function)(null, '', '');
        return {} as ReturnType<typeof execFile>;
      });
      const commits = await service.getCommitRange('merge999', 1);
      expect(commits[0].parentShas).toEqual(['aaa1111', 'bbb2222']);
    });
  });
});
```

### Integration Test

File: `clients/desktop/src/main/git/git-content-service.integration.test.ts`

One integration test runs against the vipr repository itself to verify real git interop:

```typescript
import { describe, it, expect } from 'vitest';
import { resolve } from 'node:path';
import { GitContentService } from './git-content-service';

// This test requires a real git repo — skip in CI if not in a git context.
const REPO_ROOT = resolve(__dirname, '../../../../../');

describe('GitContentService integration (real git)', () => {
  it('listTags returns well-formed TagInfo objects for the vipr repo', async () => {
    const service = new GitContentService(REPO_ROOT);
    const tags = await service.listTags();
    // vipr may have zero tags in early development — just verify the shape.
    for (const tag of tags) {
      expect(tag.name).toBeTruthy();
      expect(tag.commitSha).toMatch(/^[0-9a-f]{7,40}$/i);
      expect(['lightweight', 'annotated']).toContain(tag.tagType);
    }
  });

  it('getTrackedFilesAtCommit returns a non-empty list at HEAD', async () => {
    const service = new GitContentService(REPO_ROOT);
    const { stdout } = await import('node:child_process').then(m =>
      m.execFileSync
        ? {
            stdout: m
              .execFileSync('git', ['rev-parse', 'HEAD'], { cwd: REPO_ROOT })
              .toString()
              .trim(),
          }
        : { stdout: '' }
    );
    if (!stdout) return; // skip if HEAD unavailable
    const files = await service.getTrackedFilesAtCommit(stdout);
    expect(files.length).toBeGreaterThan(0);
  });
});
```

Mark the integration test file with `@vitest-environment node` and ensure it is excluded from the default unit test run via the existing vitest project config — add a separate `integration` project config if one does not already exist.
