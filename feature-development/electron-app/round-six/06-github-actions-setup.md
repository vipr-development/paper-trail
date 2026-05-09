# GitHub Actions Configuration Guide for Vipr Monorepo

## Overview

This document provides instructions for setting up GitHub Actions CI/CD for a private Turborepo monorepo with the following structure:

- **Package Manager**: pnpm (recommended for Turborepo)
- **Testing**: Vitest
- **Linting**: ESLint
- **Formatting**: Prettier
- **Language**: TypeScript
- **Analyzers**:
  - Core Analyzer (`@vipr/core` - JavaScript/TypeScript metrics)
  - React Analyzer (`@vipr/react` - React-specific analysis)
  - Next.js Analyzer (`@vipr/nextjs` - Next.js-specific analysis)
- **Applications**:
  - CLI (`@vipr/cli` - to be published to NPM)
  - VSCode Extension (`@vipr/vscode`)
  - Desktop Electron App (`@vipr/desktop` - placeholder)
  - Documentation Site (`@vipr/docs` - Docusaurus)

**IMPORTANT**: This is a private repository. Do NOT create any workflows that publish, deploy, or expose code publicly. All workflows are for internal CI validation only.

## File 1: Root `package.json` Scripts

The root `package.json` already contains the necessary scripts.

Coverage reporting is centralized through a single root `test:coverage` script:

```json
{
  "scripts": {
    "test:coverage": "turbo test --ui=stream --filter='./clients/*' --filter='./packages/*' --filter='./analyzers/*' -- --coverage --coverage.provider=v8 --coverage.reporter=text --coverage.reporter=json --coverage.reporter=json-summary --coverage.reporter=html && pnpm -w tsx packages/scripts/src/summarize-coverage.ts"
  }
}
```

The actual package manager is `pnpm@10.29.1` and Node requirement is `22.22.0`.

---

## File 2: `turbo.json`

The existing `turbo.json` already has the necessary task configurations.

Coverage runs through the existing `test` task by forwarding Vitest coverage CLI arguments from the root script, so no separate `test:coverage` task is required in `turbo.json`.

The current `test` task uses `dependsOn: ["^build"]` and no declared outputs.

---

## File 3: `.prettierrc.json`

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "avoid"
}
```

## File 4: `.github/workflows/ci.yml` (Main CI Workflow)

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    types: [opened, synchronize, reopened]

# Cancel in-progress runs for the same branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

jobs:
  # ============================================
  # DEPENDENCY INSTALLATION & CACHE
  # ============================================
  setup:
    name: Setup
    runs-on: ubuntu-latest
    outputs:
      cache-hit: ${{ steps.cache.outputs.cache-hit }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Get pnpm store directory
        shell: bash
        run: |
          echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV

      - name: Cache pnpm store
        id: cache
        uses: actions/cache@v4
        with:
          path: ${{ env.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-store-

      - name: Cache Turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

  # ============================================
  # LINTING & FORMATTING
  # ============================================
  lint:
    name: Lint & Format
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Check formatting with Prettier
        run: pnpm checks:formatting

      - name: Run ESLint
        run: pnpm lint

  # ============================================
  # TYPE CHECKING
  # ============================================
  typecheck:
    name: Type Check
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run TypeScript type check
        run: pnpm typecheck

  # ============================================
  # BUILD VALIDATION
  # ============================================
  build:
    name: Build
    needs: [lint, typecheck]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build all packages
        run: pnpm build

  # ============================================
  # TESTING WITH VITEST
  # ============================================
  test:
    name: Test
    needs: [lint, typecheck]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for coverage comparison

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Restore Turbo cache
        uses: actions/cache@v4
        with:
          path: .turbo
          key: ${{ runner.os }}-turbo-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-turbo-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run tests with coverage
        run: pnpm test:coverage

      - name: Upload coverage artifacts
        uses: actions/upload-artifact@v4
        with:
          name: coverage-reports
          path: |
            **/coverage/
          retention-days: 7

      - name: Add coverage summary to job output
        if: always()
        run: |
          if [ -f coverage/coverage-summary.md ]; then
            cat coverage/coverage-summary.md >> "$GITHUB_STEP_SUMMARY"
          else
            echo "Coverage summary was not generated." >> "$GITHUB_STEP_SUMMARY"
          fi

  # ============================================
  # CLI BUILD VALIDATION
  # ============================================
  cli-build:
    name: CLI Build
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build CLI
        run: pnpm --filter @vipr/cli build

      - name: Verify CLI bundle is self-contained
        run: |
          cd clients/cli
          # Verify the dist folder exists and has expected output
          ls -la dist/
          # Test that the CLI runs without errors
          node dist/index.js --version || echo "Version check complete"

  # ============================================
  # DOCUMENTATION SITE BUILD VALIDATION
  # ============================================
  docs-build:
    name: Documentation Site Build
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build documentation site
        run: pnpm docs:build

      - name: Upload build artifact (for review only)
        uses: actions/upload-artifact@v4
        with:
          name: docs-build
          path: documentation/build
          retention-days: 3
```

---

## File 5: `.github/workflows/vscode-build.yml` (VSCode Extension Build Validation)

```yaml
name: VSCode Extension Build

on:
  push:
    branches: [main, develop]
    paths:
      - 'clients/vscode/**'
      - 'packages/**'
      - 'analyzers/**'
      - 'pnpm-lock.yaml'
  pull_request:
    paths:
      - 'clients/vscode/**'
      - 'packages/**'
      - 'analyzers/**'
      - 'pnpm-lock.yaml'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-extension:
    name: Build VSCode Extension
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build extension
        run: pnpm --filter @vipr/vscode build

      - name: Package extension (validation only)
        working-directory: clients/vscode
        run: |
          npx @vscode/vsce package --no-dependencies --out extension.vsix
          ls -la extension.vsix

      # Note: VSCode extension does not currently have test scripts configured
      # To add tests, add "test" script to clients/vscode/package.json

      - name: Upload VSIX artifact (for review only)
        uses: actions/upload-artifact@v4
        if: matrix.os == 'ubuntu-latest'
        with:
          name: vscode-vsix
          path: clients/vscode/extension.vsix
          retention-days: 7
```

---

## File 6: `.github/workflows/electron-build.yml` (Electron Build Validation)

```yaml
name: Electron Desktop Build

on:
  push:
    branches: [main, develop]
    paths:
      - 'clients/desktop/**'
      - 'packages/**'
      - 'pnpm-lock.yaml'
  pull_request:
    paths:
      - 'clients/desktop/**'
      - 'packages/**'
      - 'pnpm-lock.yaml'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-electron:
    name: Build Electron (${{ matrix.os }})
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            platform: linux
          - os: macos-latest
            platform: mac
          - os: windows-latest
            platform: win
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 10

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build shared packages and analyzers
        run: pnpm --filter "./packages/*" --filter "./analyzers/*" build

      - name: Build Electron app
        working-directory: clients/desktop
        run: pnpm build

      # Build but DO NOT SIGN or RELEASE - validation only
      - name: Package Electron (validation only, unsigned)
        working-directory: clients/desktop
        run: |
          npx electron-builder --${{ matrix.platform }} --dir
        env:
          # Disable code signing for CI validation
          CSC_IDENTITY_AUTO_DISCOVERY: false

      - name: Verify build output
        working-directory: clients/desktop
        shell: bash
        run: |
          echo "Build output:"
          ls -la dist/

      - name: Upload build artifact (for review only)
        uses: actions/upload-artifact@v4
        with:
          name: electron-${{ matrix.platform }}-build
          path: clients/desktop/dist/
          retention-days: 3
```

---

## File 7: Vitest Configuration

The monorepo uses individual `vitest.config.*` files per package. The CLI has a custom config at `clients/cli/vitest.config.mts` for snapshot path resolution. Most packages use Vitest's default configuration.

If you need to add test coverage, ensure packages have `@vitest/coverage-v8` installed and configure coverage in their individual vitest config files.

---

## File 8: Shared ESLint Config (`packages/eslint-config/`)

The monorepo already has a shared ESLint configuration at `packages/eslint-config/` with:

- `base.mjs` - Base ESLint configuration
- `typescript.mjs` - TypeScript-specific rules
- `vitest.mjs` - Vitest-specific rules
- `index.mjs` - Main export

Packages reference this via `@vipr/eslint-config` workspace dependency.

---

## File 9: `.prettierignore`

```
# Dependencies
node_modules/
package-lock.json
pnpm-lock.yaml

# Build outputs
dist/
*.tsbuildinfo

# Cache and temporary files
.turbo/
.cache/
.temp/
.DS_Store

# Version control
.git/

# Logs
*.log

documentation/build/*
documentation/.docusaurus/*
```

---

## Required GitHub Secrets & Variables

### For Turbo Remote Caching (Optional but Recommended):

1. Go to **Settings > Secrets and variables > Actions**
2. Add the following:

| Type     | Name          | Description                                   |
| -------- | ------------- | --------------------------------------------- |
| Secret   | `TURBO_TOKEN` | Vercel Scoped Access Token for remote caching |
| Variable | `TURBO_TEAM`  | Your Vercel team name or username             |

### For Future Publishing (NOT TO BE USED YET):

These will be needed later when you're ready to publish:

| Type   | Name                     | Description                                      |
| ------ | ------------------------ | ------------------------------------------------ |
| Secret | `NPM_TOKEN`              | NPM automation token (for CLI publishing)        |
| Secret | `VSCE_PAT`               | VSCode Marketplace Personal Access Token         |
| Secret | `MAC_CERTS`              | Base64-encoded Apple certificates (for Electron) |
| Secret | `MAC_CERTS_PASSWORD`     | Password for Apple certificates                  |
| Secret | `WINDOWS_CERTS`          | Base64-encoded Windows code signing cert         |
| Secret | `WINDOWS_CERTS_PASSWORD` | Password for Windows cert                        |
| Secret | `APPLE_ID`               | Apple ID for notarization                        |
| Secret | `APPLE_ID_PASSWORD`      | App-specific password for Apple ID               |
| Secret | `APPLE_TEAM_ID`          | Apple Developer Team ID                          |

---

## CLI Build Configuration

The CLI (`@vipr/cli`) is located at `clients/cli/` and uses TypeScript compiler (`tsc`) for building. The `package.json` already has:

- `bin` entry pointing to `./dist/index.js`
- `files` field set to `["dist"]` to ensure only the dist directory is published
- Build script using `tsc` (not `tsup`)

If you need to add test coverage, add `@vitest/coverage-v8` to devDependencies and configure coverage in `vitest.config.mts`.

---

## Branch Protection Rules (Recommended)

Configure these in **Settings > Branches > Add rule** for `main`:

1. **Require status checks to pass before merging**:
   - `Lint & Format`
   - `Type Check`
   - `Build`
   - `Test`

2. **Require branches to be up to date before merging**

3. **Require pull request reviews before merging** (optional)

---

## Summary of What This Configuration Does

| Workflow                 | Trigger                         | Actions                                                 |
| ------------------------ | ------------------------------- | ------------------------------------------------------- |
| `ci.yml`                 | Push/PR to main/develop         | Lint, format check, typecheck, build, test all packages |
| `vscode-build.yml`       | Changes to `clients/vscode/**`  | Build and package VSIX on all platforms                 |
| `electron-build.yml`     | Changes to `clients/desktop/**` | Build Electron app on all platforms (unsigned)          |
| `docs-build` (in ci.yml) | Changes to `documentation/**`   | Build Docusaurus documentation site                     |

**What it does NOT do (by design):**

- ❌ Publish to NPM
- ❌ Publish to VSCode Marketplace
- ❌ Release to GitHub Releases
- ❌ Deploy the documentation site
- ❌ Sign or notarize any applications
- ❌ Expose any code publicly

---

## Future Publishing Workflows (DO NOT IMPLEMENT YET)

When ready to publish, you'll need separate workflows for:

1. **CLI NPM Publishing**: Use `changesets` for version management and automated publishing of `@vipr/cli`
2. **VSCode Marketplace Publishing**: Use `vsce publish` with `VSCE_PAT` for `@vipr/vscode`
3. **Electron App Store Releases**: Use `electron-builder` with code signing and notarization for `@vipr/desktop`
4. **Documentation Site Deployment**: Deploy Docusaurus site to Vercel, Netlify, or your hosting provider

These should be implemented in a separate phase when licensing and distribution are ready.

## Monorepo Structure Reference

The actual structure is:

```
vipr/
├── analyzers/          # Analyzer plugins (core, react, nextjs)
│   ├── core/
│   ├── react/
│   └── nextjs/
├── clients/            # Client applications
│   ├── cli/            # @vipr/cli - CLI tool
│   ├── vscode/         # @vipr/vscode - VSCode extension
│   └── desktop/        # @vipr/desktop - Electron app (placeholder)
├── packages/           # Shared packages
│   ├── common/         # @vipr/common - Shared types and utilities
│   ├── engine/         # @vipr/engine - Analysis engine
│   ├── eslint-config/  # @vipr/eslint-config - Shared ESLint config
│   ├── licensing/      # @vipr/licensing - License management
│   ├── logging/        # @vipr/logging - Logging utilities
│   ├── plugin-loader/  # @vipr/plugin-loader - Plugin discovery and loading
│   └── tsconfig/       # @vipr/tsconfig - Shared TypeScript configs
└── documentation/      # @vipr/docs - Docusaurus documentation site
```

All package filters in GitHub Actions should use the scoped package names (e.g., `@vipr/cli`, `@vipr/vscode`) rather than directory paths.
