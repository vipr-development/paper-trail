# debugging desktop app

This guide covers how to debug the Vipr Desktop Electron application using VSCode in the Turborepo monorepo.

## overview

The desktop app has multiple debugging targets:

- **Main Process**: Node.js process running Electron's main process (IPC handlers, database, file watching)
- **Renderer Process**: Chromium-based process running the React UI
- **Utility Worker**: Node.js utility process for background analysis tasks

All debugging configurations are set up to work from the workspace root directory.

## prerequisites

Before debugging, ensure dependencies are installed and built:

```bash
pnpm install
pnpm turbo build --filter=@vipr/desktop^...
```

This builds all workspace dependencies that the desktop app depends on.

## debugging configurations

### desktop: main + renderer

**Launch Configuration**: `Desktop: Main + Renderer`

Starts the entire Electron application with Electron Forge in development mode.

**Usage**:

1. Open VSCode in the workspace root
2. Select "Desktop: Main + Renderer" from the debug dropdown
3. Press F5 or click "Start Debugging"

**Features**:

- Starts both main and renderer processes
- Includes hot reload for renderer process
- Shows console output in integrated terminal

**Pre-launch Task**: `build:desktop-deps` - Builds all workspace dependencies using Turborepo filter `@vipr/desktop^...`

### desktop: main process

**Launch Configuration**: `Desktop: Main Process`

Launches and debugs only the Electron main process with full breakpoint support.

**Usage**:

1. Select "Desktop: Main Process" from the debug dropdown
2. Press F5 or click "Start Debugging"
3. Set breakpoints in main process code (`clients/desktop/src/main/**`)

**Source Maps**: Configured for `.vite/build/**/*.js` with proper resolution

**Environment**:

- `NODE_ENV=development`
- Source maps enabled
- Node internals and node_modules filtered from debugging

**Best For**:

- Debugging IPC handlers
- Database operations
- File system watching
- Application lifecycle

### desktop: attach to renderer

**Launch Configuration**: `Desktop: Attach to Renderer`

Attaches Chrome DevTools to the running Renderer process.

**Usage**:

1. First start the app using "Desktop: Main + Renderer"
2. Then select "Desktop: Attach to Renderer"
3. Press F5 to attach the debugger

**Port**: 9222 (Chrome remote debugging)

**Source Maps**: Configured with webpack protocol overrides

**Best For**:

- Debugging React components
- UI interactions
- Frontend state management (Zustand stores)
- Renderer-side IPC calls

### desktop: utility worker

**Launch Configuration**: `Desktop: Utility Worker`

Attaches to the utility worker process for background analysis tasks.

**Usage**:

1. Start the app normally
2. Trigger analysis that spawns utility worker
3. Select "Desktop: Utility Worker"
4. Press F5 to attach the debugger

**Port**: 5859 (Node.js inspector)

**Restart**: Automatically reconnects when worker restarts

**Best For**:

- Debugging background analysis
- Plugin loading in worker context
- Analysis coordinator operations

## vscode tasks

All tasks can be run from the workspace root using the Command Palette (Cmd/Ctrl+Shift+P > Tasks: Run Task).

### build tasks

#### build:desktop-deps

Builds all workspace dependencies required by the desktop app.

**Command**: `pnpm turbo run build --filter=@vipr/desktop^...`

**Turborepo Filter**: The `^...` syntax means "build all dependencies of @vipr/desktop, but not @vipr/desktop itself"

**What it builds**:

- `@vipr/common`
- `@vipr/engine`
- `@vipr/core`
- `@vipr/react`
- `@vipr/nextjs`
- `@vipr/logging`

**When to run**:

- Automatically runs before debug sessions (preLaunchTask)
- After pulling changes that affect workspace dependencies
- When switching branches

#### build:desktop

Builds the desktop app itself using Electron Forge.

**Command**: `pnpm --filter @vipr/desktop build`

**Dependencies**: Runs `build:desktop-deps` first

**Output**: `clients/desktop/.vite/` and `clients/desktop/out/`

### development tasks

#### dev:desktop

Starts the desktop app in development mode with hot reload.

**Command**: `pnpm --filter @vipr/desktop dev`

**Background**: Runs continuously until stopped

**Problem Matcher**: Detects Electron Forge compilation errors

#### clean:desktop

Removes build artifacts and caches.

**Command**: `pnpm --filter @vipr/desktop clean`

**Removes**:

- `.vite/`
- `out/`
- `dist/`
- `tsconfig.tsbuildinfo`

### quality tasks

#### typecheck:desktop

Runs TypeScript type checking without emitting files.

**Command**: `pnpm --filter @vipr/desktop typecheck`

**Problem Matcher**: `$tsc` - Shows TypeScript errors in Problems panel

#### lint:desktop

Runs ESLint on the desktop app codebase.

**Command**: `pnpm --filter @vipr/desktop lint`

**Problem Matcher**: `$eslint-compact`, `$eslint-stylish`

#### rebuild:desktop-native

Rebuilds native modules (like better-sqlite3) for current Electron version.

**Command**: `pnpm --filter @vipr/desktop rebuild`

**When to run**:

- After upgrading Electron version
- After npm/pnpm install of native dependencies
- When getting native module errors

## turborepo integration

### workspace filters

The desktop app uses Turborepo filters to manage dependencies:

**Filter Syntax**:

- `@vipr/desktop` - The desktop package only
- `@vipr/desktop^...` - All dependencies of desktop (transitive)
- `@vipr/desktop...` - Desktop and all its dependencies
- `--filter=./clients/desktop...` - Path-based filter (equivalent to above)

**Why use `^...`**:
The `build:desktop-deps` task uses `@vipr/desktop^...` to build only dependencies without building the desktop app itself. This is faster when you only need to ensure dependencies are up to date.

### task dependencies

Turborepo automatically handles task dependencies defined in `turbo.json`:

```json
{
  "@vipr/desktop#build": {
    "dependsOn": ["^build"],
    "outputs": [".vite/**", "out/**"]
  }
}
```

The `^build` dependency means "run build on all workspace dependencies first".

### caching

Turborepo caches build outputs for faster rebuilds:

- **Cache hits**: When dependencies haven't changed, builds use cached results
- **Cache misses**: When source files change, full rebuilds occur
- **Cache location**: `.turbo/cache/` in workspace root

To force a rebuild ignoring cache:

```bash
pnpm turbo build --filter=@vipr/desktop --force
```

## debugging workflow

### typical debugging session

1. **Initial Setup** (first time or after dependency changes):

   ```bash
   pnpm install
   pnpm turbo build --filter=@vipr/desktop^...
   ```

2. **Start Debugging**:
   - Press F5 or select debug configuration
   - VSCode automatically runs `build:desktop-deps` (cached if nothing changed)
   - App launches with debugger attached

3. **Set Breakpoints**:
   - Main process: `clients/desktop/src/main/**/*.ts`
   - Renderer: `clients/desktop/src/renderer/**/*.tsx`
   - Worker: `clients/desktop/src/utility/**/*.ts`

4. **Debug**:
   - Step through code
   - Inspect variables
   - View call stacks
   - Check console output

### debugging main process ipc handlers

1. Set breakpoints in IPC handlers (`clients/desktop/src/main/ipc/handlers/**/*.ts`)
2. Launch with "Desktop: Main Process"
3. Trigger IPC calls from renderer (open repository, run analysis, etc.)
4. Debugger pauses at breakpoints

**Example**:

```typescript
// clients/desktop/src/main/ipc/handlers/repository.ts
export const repositoryHandlers = {
  'repo:open': async (event, path: string) => {
    debugger; // Or set breakpoint here
    // ... handler logic
  },
};
```

### debugging react components

1. Start app with "Desktop: Main + Renderer"
2. Open Chrome DevTools in Electron window (View > Toggle Developer Tools)
3. Or attach VSCode debugger using "Desktop: Attach to Renderer"
4. Set breakpoints in React components

**React DevTools**:
The app includes React DevTools extension for component inspection:

- Open DevTools in Electron window
- Switch to "Components" or "Profiler" tab

### debugging background analysis

The analysis runs in a utility worker process for performance isolation.

1. Start app normally
2. Open a repository to trigger analysis
3. Attach debugger: "Desktop: Utility Worker"
4. Set breakpoints in worker code or plugin analyzers

**Worker Location**: `clients/desktop/src/utility/worker.ts`

**Plugin Loading**: Plugins load dynamically via `DesktopPluginLoader`

## common issues

### breakpoints not hitting

**Symptom**: Breakpoints show gray/hollow or don't pause execution

**Solutions**:

1. Ensure source maps are enabled in build
2. Verify `outFiles` path matches build output
3. Check that `resolveSourceMapLocations` includes source directory
4. Rebuild with `pnpm turbo build --filter=@vipr/desktop --force`

### cannot find module errors

**Symptom**: `Cannot find module '@vipr/common'` or similar

**Solution**:

```bash
pnpm turbo build --filter=@vipr/desktop^...
```

Dependencies must be built before the desktop app can import them.

### native module errors

**Symptom**: `Error: The module was compiled against a different Node.js version`

**Solution**:

```bash
pnpm --filter @vipr/desktop rebuild
```

This rebuilds native modules (better-sqlite3) for the current Electron version.

### port already in use

**Symptom**: Debug port 9222 or 5859 already in use

**Solution**:

1. Stop all running Electron instances
2. Kill process using port: `lsof -ti:9222 | xargs kill -9`
3. Restart debug session

### turborepo cache issues

**Symptom**: Changes not reflected after rebuild

**Solution**:

```bash
pnpm turbo build --filter=@vipr/desktop --force
```

The `--force` flag ignores cache and forces full rebuild.

## advanced debugging

### debugging with inspect flag

For more control, manually start Electron with inspect flags:

```bash
cd clients/desktop
pnpm exec electron --inspect=5858 --inspect-brk .vite/build/main.js
```

Then attach VSCode debugger to port 5858.

### logging

The app uses `@vipr/logging` package for structured logging:

```typescript
import { Logger } from '@vipr/logging';

const logger = Logger.getLogger('my-module');
logger.debug('Debug message', { context: data });
logger.info('Info message');
logger.error('Error message', error);
```

**Log Levels**: Set via environment variable:

```bash
LOG_LEVEL=debug pnpm --filter @vipr/desktop dev
```

### profiling

1. Start app with "Desktop: Main + Renderer"
2. Open DevTools in Electron window
3. Go to "Profiler" tab
4. Record performance profile
5. Analyze component render times

## vscode workspace settings

The workspace includes recommended settings for debugging:

**.vscode/settings.json** (if not exists, create):

```json
{
  "debug.node.autoAttach": "on",
  "debug.javascript.autoAttachFilter": "smart",
  "typescript.tsdk": "node_modules/typescript/lib",
  "eslint.workingDirectories": [
    { "pattern": "clients/*/" },
    { "pattern": "packages/*/" },
    { "pattern": "analyzers/*/" }
  ]
}
```

## reference

### file locations

- **Launch configs**: `.vscode/launch.json`
- **Tasks**: `.vscode/tasks.json`
- **Turborepo config**: `turbo.json`
- **Desktop app**: `clients/desktop/`
- **Main process**: `clients/desktop/src/main/`
- **Renderer**: `clients/desktop/src/renderer/`
- **Worker**: `clients/desktop/src/utility/`

### useful commands

```bash
# Build dependencies only
pnpm turbo build --filter=@vipr/desktop^...

# Build desktop app with dependencies
pnpm turbo build --filter=@vipr/desktop...

# Force rebuild ignoring cache
pnpm turbo build --filter=@vipr/desktop --force

# Run desktop app in dev mode
pnpm --filter @vipr/desktop dev

# Type check
pnpm --filter @vipr/desktop typecheck

# Lint
pnpm --filter @vipr/desktop lint

# Clean build artifacts
pnpm --filter @vipr/desktop clean

# Rebuild native modules
pnpm --filter @vipr/desktop rebuild
```

### documentation

- [Electron Debugging](https://www.electronjs.org/docs/latest/tutorial/debugging-main-process)
- [VSCode Debugging](https://code.visualstudio.com/docs/editor/debugging)
- [Turborepo Filtering](https://turbo.build/repo/docs/core-concepts/monorepos/filtering)
- [Electron Forge](https://www.electronforge.io/)
