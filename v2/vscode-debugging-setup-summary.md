# vscode debugging setup summary

This document summarizes the VSCode debugging configuration for the Vipr Electron desktop app in the Turborepo monorepo.

## what was configured

### debug configurations (launch.json)

Four debug configurations were created for the desktop app:

1. **Desktop: Main + Renderer** - Launch the full Electron app with Electron Forge
2. **Desktop: Main Process** - Debug the main Node.js process with breakpoints
3. **Desktop: Attach to Renderer** - Attach Chrome DevTools to the renderer process
4. **Desktop: Utility Worker** - Attach to the background analysis worker

### tasks (tasks.json)

Eight tasks were created for desktop app development:

1. **build:desktop-deps** - Build workspace dependencies using Turborepo
2. **build:desktop** - Build the desktop app
3. **dev:desktop** - Run in development mode with hot reload
4. **clean:desktop** - Remove build artifacts
5. **typecheck:desktop** - TypeScript type checking
6. **lint:desktop** - ESLint linting
7. **rebuild:desktop-native** - Rebuild native modules for Electron

### documentation

Three documentation files were created:

1. **debugging-desktop-app.md** - Comprehensive debugging guide
2. **.vscode/README.md** - Quick reference for VSCode configurations
3. **vscode-debugging-setup-summary.md** - This file

## key design decisions

### workspace-relative paths

All paths use `${workspaceFolder}` to ensure debugging works from the root directory:

```json
{
  "cwd": "${workspaceFolder}/clients/desktop",
  "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron-forge"
}
```

This ensures developers can debug from the workspace root without switching directories.

### turborepo integration

The `build:desktop-deps` task uses Turborepo's filter syntax to build only dependencies:

```bash
pnpm turbo run build --filter=@vipr/desktop^...
```

**Filter breakdown**:

- `@vipr/desktop` - The desktop package
- `^` - Only dependencies (not the package itself)
- `...` - Include transitive dependencies

This builds:

- `@vipr/common`
- `@vipr/engine`
- `@vipr/core`
- `@vipr/react`
- `@vipr/nextjs`
- `@vipr/logging`
- `@vipr/tsconfig`
- `@vipr/eslint-config`

### pre-launch tasks

Debug configurations use `preLaunchTask` to ensure dependencies are built:

```json
{
  "name": "Desktop: Main + Renderer",
  "preLaunchTask": "build:desktop-deps"
}
```

Turborepo's caching ensures this is fast when nothing has changed.

### source map configuration

Source maps are configured for proper breakpoint resolution:

```json
{
  "sourceMaps": true,
  "outFiles": [
    "${workspaceFolder}/clients/desktop/.vite/build/**/*.js",
    "!${workspaceFolder}/clients/desktop/.vite/build/node_modules/**"
  ],
  "resolveSourceMapLocations": [
    "${workspaceFolder}/clients/desktop/**",
    "!${workspaceFolder}/clients/desktop/node_modules/**"
  ]
}
```

The `!` prefix excludes node_modules from source map resolution for performance.

### skip files configuration

Node internals and node_modules are excluded from debugging:

```json
{
  "skipFiles": [
    "<node_internals>/**",
    "${workspaceFolder}/node_modules/**",
    "${workspaceFolder}/clients/desktop/node_modules/**"
  ]
}
```

This prevents stepping into framework code when debugging.

## debugging workflow

### initial setup

```bash
# From workspace root
pnpm install
pnpm turbo build --filter=@vipr/desktop^...
```

### start debugging

1. Open VSCode at workspace root: `/Users/jamesleebaker/Codespace/vipr`
2. Open debug panel (Cmd/Ctrl+Shift+D)
3. Select "Desktop: Main + Renderer"
4. Press F5

The `build:desktop-deps` task runs automatically if needed.

### set breakpoints

**Main Process**:

- `/Users/jamesleebaker/Codespace/vipr/clients/desktop/src/main/**/*.ts`

**Renderer Process**:

- `/Users/jamesleebaker/Codespace/vipr/clients/desktop/src/renderer/**/*.tsx`

**Utility Worker**:

- `/Users/jamesleebaker/Codespace/vipr/clients/desktop/src/utility/**/*.ts`

### attach to processes

**Renderer** (Chrome DevTools):

1. Start app with "Desktop: Main + Renderer"
2. Select "Desktop: Attach to Renderer"
3. Press F5 to attach

**Worker** (Node.js):

1. Start app and trigger analysis
2. Select "Desktop: Utility Worker"
3. Press F5 to attach

## turborepo caching

### how it works

Turborepo caches build outputs based on:

- Source file contents
- Dependencies
- Environment variables
- Build command

When nothing changes, builds use cached results.

### cache location

```
/Users/jamesleebaker/Codespace/vipr/.turbo/cache/
```

### force rebuild

Ignore cache and rebuild everything:

```bash
pnpm turbo build --filter=@vipr/desktop --force
```

### verify cache behavior

Run with `--dry-run` to see what would be built:

```bash
pnpm turbo build --filter=@vipr/desktop^... --dry-run
```

## problem matchers

Tasks include problem matchers to show errors in VSCode's Problems panel:

**TypeScript** (`$tsc`):

- Used by: `build:desktop-deps`, `build:desktop`, `typecheck:desktop`
- Detects TypeScript compilation errors

**ESLint** (`$eslint-compact`, `$eslint-stylish`):

- Used by: `lint:desktop`
- Detects ESLint violations

**Electron Forge** (custom):

- Used by: `dev:desktop`
- Detects Electron Forge compilation errors

## background tasks

The `dev:desktop` task runs continuously with background detection:

```json
{
  "isBackground": true,
  "problemMatcher": {
    "background": {
      "beginsPattern": "^.*Compiling.*$",
      "endsPattern": "^.*Electron app started.*$"
    }
  }
}
```

This allows VSCode to know when the app is ready for debugging.

## monorepo considerations

### package hoisting

The monorepo uses pnpm workspaces with hoisting:

- Common dependencies are hoisted to root `node_modules`
- Package-specific deps stay in package `node_modules`

This is why binaries reference `${workspaceFolder}/node_modules/.bin/`.

### workspace filters

Task filters use package names, not file paths:

**Correct**:

```bash
pnpm --filter @vipr/desktop dev
pnpm turbo build --filter=@vipr/desktop^...
```

**Also valid** (path-based):

```bash
pnpm turbo build --filter=./clients/desktop...
```

### dependency graph

The desktop app depends on:

```
@vipr/desktop
├── @vipr/common
├── @vipr/engine
│   ├── @vipr/common
│   └── @vipr/logging
├── @vipr/core
│   ├── @vipr/common
│   ├── @vipr/engine
│   └── @vipr/logging
├── @vipr/react
│   ├── @vipr/common
│   └── @vipr/logging
└── @vipr/nextjs
    ├── @vipr/common
    └── @vipr/logging
```

Turborepo builds these in the correct order.

## native modules

The desktop app uses `better-sqlite3`, a native Node.js module.

### electron version compatibility

Native modules must be rebuilt for the Electron Node.js version:

```bash
pnpm --filter @vipr/desktop rebuild
```

This runs `electron-rebuild -f -w better-sqlite3`.

### when to rebuild

- After upgrading Electron version
- After installing/updating native dependencies
- When getting "module version mismatch" errors

### pnpm configuration

The root `package.json` includes:

```json
{
  "pnpm": {
    "ignoredBuiltDependencies": ["@swc/core", "better-sqlite3"]
  }
}
```

This prevents pnpm from auto-building, relying on the manual `rebuild` script instead.

## common issues and solutions

### issue: breakpoints not hitting

**Symptoms**:

- Breakpoints are gray/hollow
- Execution doesn't pause

**Solutions**:

1. Verify source maps are enabled in Electron Forge config
2. Rebuild with fresh source maps:
   ```bash
   pnpm turbo build --filter=@vipr/desktop --force
   ```
3. Check `outFiles` matches actual build output
4. Ensure `resolveSourceMapLocations` includes source directory

### issue: cannot find module

**Symptoms**:

```
Error: Cannot find module '@vipr/common'
```

**Solution**:
Dependencies need to be built first:

```bash
pnpm turbo build --filter=@vipr/desktop^...
```

This is normally automatic via `preLaunchTask`.

### issue: native module error

**Symptoms**:

```
Error: The module was compiled against a different Node.js version
```

**Solution**:
Rebuild native modules for current Electron version:

```bash
pnpm --filter @vipr/desktop rebuild
```

### issue: port already in use

**Symptoms**:

```
Error: listen EADDRINUSE: address already in use :::9222
```

**Solution**:
Kill existing Electron instances:

```bash
lsof -ti:9222 | xargs kill -9
lsof -ti:5859 | xargs kill -9
```

### issue: stale cache

**Symptoms**:

- Code changes don't appear after rebuild
- Old errors persist

**Solution**:
Force rebuild ignoring cache:

```bash
pnpm turbo build --filter=@vipr/desktop --force
```

## environment variables

### node_env

The main process debug config sets:

```json
{
  "env": {
    "NODE_ENV": "development"
  }
}
```

This enables development features in the app.

### log_level

To enable debug logging, set in terminal before launching:

```bash
LOG_LEVEL=debug code .
```

Or add to task args:

```json
{
  "args": ["dev"],
  "env": {
    "LOG_LEVEL": "debug"
  }
}
```

## performance tips

### incremental builds

Turborepo's incremental builds mean only changed packages rebuild:

- First build: ~30-60 seconds
- Subsequent builds with cache: ~2-5 seconds
- Changed package only: ~5-10 seconds

### parallel tasks

Turborepo builds packages in parallel when possible:

```bash
pnpm turbo build --filter=@vipr/desktop^...
# Builds @vipr/tsconfig, @vipr/eslint-config, @vipr/logging in parallel
# Then @vipr/common (depends on above)
# Then @vipr/engine, @vipr/core, @vipr/react, @vipr/nextjs in parallel
```

### skip type checking during dev

For faster iteration, use dev mode which skips full type checking:

```bash
pnpm --filter @vipr/desktop dev
```

Run type checking separately when needed:

```bash
pnpm --filter @vipr/desktop typecheck
```

## file structure

```
/Users/jamesleebaker/Codespace/vipr/
├── .vscode/
│   ├── launch.json          # Debug configurations
│   ├── tasks.json           # Build and dev tasks
│   └── README.md            # Quick reference
├── clients/desktop/
│   ├── src/
│   │   ├── main/            # Electron main process
│   │   ├── renderer/        # React UI
│   │   ├── preload/         # Preload scripts
│   │   └── utility/         # Background worker
│   ├── .vite/               # Build output
│   │   └── build/
│   │       ├── main.js      # Main process entry
│   │       └── **/*.js      # Compiled code
│   ├── forge.config.ts      # Electron Forge config
│   └── package.json         # Desktop app package
├── documentation/docs/
│   ├── debugging-desktop-app.md
│   └── vscode-debugging-setup-summary.md
└── turbo.json               # Turborepo config
```

## vscode extensions recommended

While not required, these extensions enhance the debugging experience:

- **ESLint** - `dbaeumer.vscode-eslint`
- **Prettier** - `esbenp.prettier-vscode`
- **TypeScript** - Built-in
- **Chrome Debugger** - Built-in (for renderer debugging)

## next steps

### verify setup works

1. Open workspace in VSCode
2. Run initial build:
   ```bash
   pnpm turbo build --filter=@vipr/desktop^...
   ```
3. Select "Desktop: Main + Renderer" in debug panel
4. Press F5
5. Verify app launches successfully

### test breakpoint debugging

1. Open `/Users/jamesleebaker/Codespace/vipr/clients/desktop/src/main/index.ts`
2. Set breakpoint on app ready event
3. Launch "Desktop: Main Process"
4. Verify debugger pauses at breakpoint

### test renderer debugging

1. Launch "Desktop: Main + Renderer"
2. Wait for app to start
3. Attach with "Desktop: Attach to Renderer"
4. Set breakpoint in React component
5. Trigger component render
6. Verify breakpoint hits

## reference

### key files modified

- `/Users/jamesleebaker/Codespace/vipr/.vscode/launch.json`
- `/Users/jamesleebaker/Codespace/vipr/.vscode/tasks.json`
- `/Users/jamesleebaker/Codespace/vipr/.vscode/README.md`
- `/Users/jamesleebaker/Codespace/vipr/documentation/docs/debugging-desktop-app.md`
- `/Users/jamesleebaker/Codespace/vipr/documentation/docs/vscode-debugging-setup-summary.md`

### turborepo documentation

- [Filtering Workspaces](https://turbo.build/repo/docs/core-concepts/monorepos/filtering)
- [Caching](https://turbo.build/repo/docs/core-concepts/caching)
- [Running Tasks](https://turbo.build/repo/docs/core-concepts/monorepos/running-tasks)

### electron documentation

- [Debugging Main Process](https://www.electronjs.org/docs/latest/tutorial/debugging-main-process)
- [Debugging Renderer Process](https://www.electronjs.org/docs/latest/tutorial/debugging-vscode)

### vscode documentation

- [Debugging](https://code.visualstudio.com/docs/editor/debugging)
- [Tasks](https://code.visualstudio.com/docs/editor/tasks)
- [Launch Configurations](https://code.visualstudio.com/docs/editor/debugging#_launch-configurations)
