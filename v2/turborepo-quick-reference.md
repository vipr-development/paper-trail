# Turborepo Quick Reference

Quick reference guide for common Turborepo commands and patterns in the Vipr monorepo.

## Common Commands

### Building

```bash
# Build everything
pnpm build

# Build specific package
pnpm build --filter=@vipr/common
pnpm build --filter=@vipr/desktop

# Build all packages in a directory
pnpm build:packages     # packages/* only
pnpm build:analyzers    # analyzers/* only
pnpm build:clients      # clients/* only

# Build a package and its dependencies
pnpm build --filter=@vipr/cli...

# Build packages that depend on a package
pnpm build --filter=...@vipr/common
```

### Development

```bash
# Start all dev servers
pnpm dev

# Start specific package
pnpm desktop:dev
pnpm docs:dev

# Dev mode for a package and its dependencies
turbo dev --filter=@vipr/desktop...
```

### Testing

```bash
# Run all tests
pnpm test

# Run tests for specific package
turbo test --filter=@vipr/common

# Run tests by layer
pnpm test:packages
pnpm test:analyzers
pnpm test:clients

# Watch mode
pnpm test:watch

# Test specific package in watch mode
turbo test:watch --filter=@vipr/engine
```

### Linting and Type Checking

```bash
# Lint everything
pnpm lint

# Lint specific package
turbo lint --filter=@vipr/react

# Type check everything
pnpm typecheck

# Type check specific package
pnpm desktop:typecheck
```

### Cleaning

```bash
# Clean all packages
pnpm clean

# Clean specific package
turbo clean --filter=@vipr/desktop

# Clean by layer
turbo clean --filter='./packages/*'
```

## Filter Patterns

### By Package Name

```bash
# Exact match
--filter=@vipr/common

# Multiple packages
--filter=@vipr/common --filter=@vipr/engine

# Pattern matching
--filter='@vipr/mcp-*'
```

### By Directory

```bash
# All packages in directory
--filter='./packages/*'
--filter='./analyzers/*'
--filter='./clients/*'

# Specific subdirectory
--filter='./clients/desktop'
```

### By Dependency

```bash
# Package and its dependencies
--filter=@vipr/cli...

# Package and dependents
--filter=...@vipr/common

# Package, its dependencies, and dependents
--filter=...@vipr/engine...
```

### By Git Changes

```bash
# Packages changed since main branch
--filter='[main]'

# Packages changed in last commit
--filter='[HEAD^1]'

# Packages changed in specific commit range
--filter='[origin/main...HEAD]'
```

### Exclusions

```bash
# Exclude specific package
--filter='!@vipr/docs'

# Exclude pattern
--filter='!@vipr/mcp-*'

# Multiple filters
pnpm test --filter='!@vipr/docs' --filter='!@vipr/fixtures'
```

## Dependency Chain Queries

### Show What Will Build

```bash
# Dry run to see what would build
turbo build --filter=@vipr/cli --dry-run

# Show dependency graph
turbo build --filter=@vipr/cli --graph
```

### Understanding Dependencies

```bash
# List dependencies of a package
pnpm list --filter=@vipr/desktop --depth=0

# Show workspace dependency tree
pnpm list --filter=@vipr/cli --depth=Infinity --workspace
```

## Common Workflows

### Making Changes to a Shared Package

```bash
# 1. Make changes to the package
cd packages/common
# ... edit files ...

# 2. Build the package
pnpm build --filter=@vipr/common

# 3. Test the package
turbo test --filter=@vipr/common

# 4. Test packages that depend on it
turbo test --filter=...@vipr/common

# 5. Build everything to ensure no breaking changes
pnpm build
```

### Developing a Client Package

```bash
# 1. Build all dependencies first
pnpm build:packages
pnpm build:analyzers

# 2. Start the client in dev mode
pnpm desktop:dev

# 3. If dependencies change, rebuild them
pnpm build --filter=@vipr/common
# Dev mode will pick up the changes (may need restart)
```

### Running Tests After Changes

```bash
# Test only changed packages and their dependents
turbo test --filter='[HEAD^1]...'

# Test everything if core packages changed
pnpm test
```

### CI/CD Pipeline

```bash
# Install dependencies
pnpm install --frozen-lockfile

# Build everything
turbo build

# Run all tests
turbo test --filter='!@vipr/docs'

# Lint and type check
turbo lint
turbo typecheck
```

## Performance Optimization

### Parallel Execution

```bash
# Limit concurrency
turbo build --concurrency=4

# Maximum parallelism
turbo build --concurrency=100
```

### Caching

```bash
# Force rebuild (ignore cache)
turbo build --force

# Clear cache and rebuild
turbo clean
pnpm build

# Only build, skip cache writes
turbo build --no-cache
```

### Selective Execution

```bash
# Build only packages that changed
turbo build --filter='[HEAD^1]'

# Build a subset during development
pnpm build:packages
turbo dev --filter=@vipr/desktop
```

## Debugging

### Verbose Output

```bash
# Show detailed logs
turbo build --verbosity=2

# Show full command output
turbo build --output-logs=full

# Show errors only
turbo build --output-logs=errors-only
```

### Understanding Task Execution

```bash
# Dry run with task graph
turbo build --dry-run --graph

# Show what would run
turbo test --filter=@vipr/cli --dry-run

# Summarize what ran
turbo build --summarize
```

### Cache Inspection

```bash
# Show cache status
turbo run build --dry-run=json

# Clear specific package cache
rm -rf node_modules/.cache/turbo
```

## Environment Variables

### Common Variables

```bash
# Disable Turborepo telemetry
export TURBO_TELEMETRY_DISABLED=1

# Set cache directory
export TURBO_CACHE_DIR=.turbo-cache

# Force color output
export FORCE_COLOR=1
```

### Using with Commands

```bash
# Set for single command
NODE_ENV=production turbo build

# Set globally
export NODE_ENV=production
turbo build
```

## Task-Specific Commands

### Storybook

```bash
# Start Storybook
pnpm desktop:storybook

# Build Storybook
pnpm desktop:build-storybook
```

### Electron

```bash
# Build desktop app
pnpm desktop:build

# Package desktop app
pnpm desktop:make

# Clean desktop build artifacts
pnpm desktop:clean
```

### Documentation

```bash
# Start docs dev server
pnpm docs:dev

# Build docs
pnpm docs:build

# Serve built docs
pnpm docs:serve
```

## Tips and Tricks

### 1. Fastest Development Workflow

```bash
# One-time full build
pnpm build

# Then only run what you're working on
pnpm desktop:dev
```

### 2. Incremental CI

```bash
# Only test changed packages
turbo test --filter='[origin/main...HEAD]'
```

### 3. Debug Build Issues

```bash
# Build with full output
turbo build --output-logs=full --verbosity=2

# Build specific package with verbose logs
turbo build --filter=@vipr/desktop --output-logs=full
```

### 4. Force Fresh Build

```bash
# Clear everything and rebuild
pnpm clean
pnpm install
pnpm build
```

### 5. Check What Changed

```bash
# See which packages have changes
git status

# See which packages would build
turbo build --filter='[HEAD^1]' --dry-run
```

## Package-Specific Shortcuts

These are defined in root `package.json` for convenience:

```bash
# Desktop app
pnpm desktop:dev
pnpm desktop:build
pnpm desktop:make
pnpm desktop:lint
pnpm desktop:typecheck
pnpm desktop:storybook

# Documentation
pnpm docs:dev
pnpm docs:build
pnpm docs:serve

# Styleguide
pnpm styleguide:dev
pnpm styleguide:build
pnpm styleguide:storybook

# MCP server (analyzer)
pnpm mcp:analyzer

# Security and quality
pnpm checks:security
pnpm checks:formatting
```

## Troubleshooting Common Issues

### Issue: Type Errors in IDE

```bash
# Solution: Rebuild type definitions
pnpm build:packages
pnpm typecheck
```

### Issue: Stale Build Artifacts

```bash
# Solution: Clean and rebuild
turbo clean --filter=@vipr/desktop
pnpm build --filter=@vipr/desktop
```

### Issue: Dependency Not Found

```bash
# Solution: Ensure dependency is built
pnpm build --filter=@vipr/common
pnpm build --filter=...@vipr/desktop
```

### Issue: Cache Corruption

```bash
# Solution: Clear Turbo cache
rm -rf node_modules/.cache/turbo
turbo build
```

### Issue: Out of Memory

```bash
# Solution: Reduce concurrency
turbo build --concurrency=2

# Or increase Node memory
NODE_OPTIONS=--max-old-space-size=4096 turbo build
```
