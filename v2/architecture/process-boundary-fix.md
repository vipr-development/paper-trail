# Process Boundary Fix: BaseReportPresenter

## Problem

The desktop application build was failing with:

```
RollupError: "isAbsolute" is not exported by "__vite-browser-external",
imported by "../../packages/common/dist/utils/base-presenter.js"
```

## Root Cause

`BaseReportPresenter` in `@vipr/common/utils/base-presenter.ts` imports Node.js `path` module:

```typescript
import { isAbsolute, resolve } from 'path';
```

This class was exported from the main entry point (`@vipr/common`), which meant when the Electron renderer process imported ANY type or constant from `@vipr/common`, Vite attempted to bundle the entire package including Node.js modules that don't exist in the browser context.

## Solution

### 1. Remove from Main Export

Removed `BaseReportPresenter` from browser-safe exports:

- `packages/common/src/index.ts` - Removed export with documentation
- `packages/common/src/utils/index.ts` - Commented out export with explanation

### 2. Use Subpath Export

`BaseReportPresenter` remains available via the existing subpath export:

```typescript
// Node.js-only import (analyzers, CLI, main process)
import { BaseReportPresenter } from '@vipr/common/utils/base-presenter';
```

This subpath was already defined in `package.json` exports, so no changes needed there.

### 3. Update Analyzer Imports

Updated all analyzer base presenters to use subpath import:

- `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/presenters/core-overview-presenter.ts`
- `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/base-presenter.ts`
- `/Users/jamesleebaker/Codespace/vipr/analyzers/nextjs/src/presenters/base-presenter.ts`

Before:

```typescript
import { BaseReportPresenter, createAnalysisId } from '@vipr/common';
```

After:

```typescript
import { createAnalysisId } from '@vipr/common';
import { BaseReportPresenter } from '@vipr/common/utils/base-presenter';
```

### 4. Documentation

Created `/Users/jamesleebaker/Codespace/vipr/packages/common/README.md` documenting:

- Browser-safe vs Node.js-only exports
- Import rules for different process contexts
- Architecture notes on process boundaries

## Architecture Pattern

This follows the same pattern as the config loader fix:

1. **Main export** (`@vipr/common`) - Browser-safe only
2. **Subpath exports** - Can include Node.js APIs if documented

### Process Context Rules

| Context                                 | Can Import Main | Can Import Subpaths          |
| --------------------------------------- | --------------- | ---------------------------- |
| Analyzer plugins (Node.js)              | Yes             | All                          |
| CLI (Node.js)                           | Yes             | All                          |
| Desktop main process (Electron/Node.js) | Yes             | All                          |
| Desktop renderer (Electron/Browser)     | Yes             | None with Node.js APIs       |
| VSCode extension (Mixed)                | Yes             | Depends on execution context |

## Node.js-Only Exports in @vipr/common

| Export Path                         | Uses Node.js API | Reason              |
| ----------------------------------- | ---------------- | ------------------- |
| `@vipr/common/config`               | `fs`, `path`     | Config file loading |
| `@vipr/common/utils/base-presenter` | `path`           | File URL generation |

## Testing

### Verified Builds

1. Analyzer packages build successfully

   ```bash
   pnpm --filter @vipr/core build      # ✓
   pnpm --filter @vipr/react build     # ✓
   pnpm --filter @vipr/nextjs build    # ✓
   ```

2. CLI builds successfully

   ```bash
   pnpm --filter @vipr/cli build       # ✓
   # base-presenter bundled as separate chunk
   ```

3. Desktop app builds successfully
   ```bash
   pnpm --filter @vipr/desktop build   # ✓
   # No Node.js modules in renderer bundle
   ```

### Smoke Tests

- CLI can still load presenters from analyzer plugins
- Desktop renderer doesn't import presenter classes (verified via grep)
- Full workspace build passes

## Impact

### Breaking Changes

None. The subpath export already existed, so this is purely an internal refactor.

### Performance

No impact. Same code, different import path.

### Maintenance

Improved separation of concerns. Clear documentation of which exports are safe in which contexts.

## Related Issues

Similar pattern used for config loader in previous fix. This establishes a consistent approach for handling Node.js dependencies in a mixed-context monorepo.

## Future Considerations

If more Node.js-only utilities are needed:

1. Add to existing subpath exports with clear naming
2. Document in `packages/common/README.md`
3. Update this architecture doc

Consider creating a `@vipr/node-utils` package if Node.js-only utilities grow significantly, but current approach is cleaner for the small number of cases.
