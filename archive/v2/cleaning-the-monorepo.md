# Cleaning the Monorepo

This guide explains how to clean build artifacts, caches, and temporary files from the Vipr monorepo.

## Quick Reference

```bash
# Clean everything (recommended for a fresh start)
pnpm clean

# Preview what will be deleted without actually deleting
pnpm clean --dry-run

# Clean specific workspace scope
pnpm desktop:clean    # Clean desktop app only
pnpm --filter @vipr/licensing clean   # Clean shared licensing package only
```

## What Gets Cleaned

The clean script removes the following artifacts across the workspace:

### 1. Package-Level Build Artifacts

Each package defines its own build outputs in its `clean` script. These typically include:

- `dist/` - Compiled JavaScript and TypeScript declaration files
- `coverage/` - Test coverage reports
- `.vite/` - Vite cache (desktop app, styleguide)
- `out/` - Electron build output (desktop app)
- `build/` - Documentation site output
- `.docusaurus/` - Docusaurus cache
- `storybook-static/` - Built Storybook files

### 2. Turborepo Cache Directories

- `.turbo/` in workspace root - Global Turborepo cache
- `.turbo/` in each package - Package-level Turborepo cache

### 3. TypeScript Build Info Files

- `tsconfig.tsbuildinfo` files used for incremental TypeScript compilation

### 4. Node Modules

- `node_modules/` in workspace root

## Clean Script Behavior

The clean script executes in four steps:

1. **Package-level cleaning** via `turbo clean` - Runs each package's clean script
2. **Turborepo cache removal** - Deletes all `.turbo` directories
3. **TypeScript cleanup** - Removes `tsconfig.tsbuildinfo` files
4. **Dependencies removal** - Deletes root `node_modules`

## Architecture Notes

### Why No Built-in `turbo clean`?

As of Turborepo 2.8.x, there is no official `turbo clean` command for removing build artifacts. The [feature has been proposed](https://github.com/vercel/turborepo/discussions/10552) but not yet implemented.

The Vipr monorepo implements cleaning through:

1. **Task-based cleaning** - Each package defines a `clean` script
2. **Centralized orchestration** - The root clean script coordinates all cleanup
3. **Manual cache cleanup** - Explicit removal of `.turbo` directories

This approach follows Turborepo best practices:

- Outputs are defined in `turbo.json` for each task
- Package-level clean scripts align with build outputs
- Cache directories are cleaned manually when needed

### Turborepo Cache Strategy

Turborepo caches task outputs in `.turbo/cache` directories based on:

- Source file hashes
- Environment variables
- Lock file contents
- Global dependencies

The cache provides:

- **Local caching** - Instant rebuilds when nothing changed
- **Task coordination** - Proper dependency ordering
- **Parallel execution** - Maximum build performance

For more information, see [Turborepo Caching Documentation](https://turborepo.dev/docs/crafting-your-repository/caching).

## Package-Level Clean Scripts

Each package defines its own clean script targeting its specific build outputs:

```json
{
  "scripts": {
    "clean": "rm -rf dist coverage .turbo && rm -f tsconfig.tsbuildinfo"
  }
}
```

Common patterns:

- **TypeScript packages** - Remove `dist/`, `coverage/`, `.turbo/`, `tsconfig.tsbuildinfo`
- **Desktop app** - Remove `.vite/`, `out/`, `dist/`, `storybook-static/`, `coverage/`, `.turbo/`, `tsconfig.tsbuildinfo`
- **Documentation** - Remove `build/`, `.docusaurus/`, `.turbo/`
- **Config packages** - Echo message (no artifacts to clean)

## When to Clean

Clean the workspace when:

- Starting fresh after pulling major changes
- Troubleshooting build issues
- Testing dependency changes
- Preparing for a fresh build verification
- Experiencing cache-related problems
- Before major version updates

## After Cleaning

After running `pnpm clean`, restore dependencies with:

```bash
pnpm install
```

Then rebuild packages:

```bash
pnpm build
```

## Scoped Cleaning

For targeted cleanup, use Turborepo filters:

```bash
# Clean only packages
turbo clean --filter='./packages/*'

# Clean only analyzers
turbo clean --filter='./analyzers/*'

# Clean only clients
turbo clean --filter='./clients/*'

# Clean a specific package and its dependencies
turbo clean --filter=@vipr/desktop
```

## Troubleshooting

### Clean Script Fails

If the clean script encounters errors:

1. Check if any processes are holding file locks (restart editor/terminal)
2. Manually remove problematic directories
3. Run clean with elevated permissions if needed (not recommended)

### Residual Artifacts

If artifacts remain after cleaning:

```bash
# Find remaining dist directories
find . -type d -name 'dist' -not -path '*/node_modules/*'

# Find remaining .turbo directories
find . -type d -name '.turbo' -not -path '*/node_modules/*'

# Find remaining build info files
find . -name 'tsconfig.tsbuildinfo' -type f
```

### Cache Not Clearing

Turborepo cache persists between runs by design. To force fresh execution:

```bash
# Force re-execution without cache
pnpm build --force

# Clean then rebuild
pnpm clean && pnpm install && pnpm build
```

## References

- [Turborepo Caching](https://turborepo.dev/docs/crafting-your-repository/caching)
- [Turborepo Cache Discussion](https://github.com/vercel/turborepo/discussions/5270)
- [turbo clean Feature Request](https://github.com/vercel/turborepo/discussions/10552)
- [pnpm Workspace](https://pnpm.io/workspaces)
