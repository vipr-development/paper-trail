# Commit & Deploy

This guide covers the automated versioning and release workflow using semantic-release, Commitizen, and Husky.

## Prerequisites

- **Node.js 22.22.0**
- **pnpm 10.29.1**
- Git repository initialized
- GitHub repository (for automated releases)

If you're using an older Node.js version locally, you can:

- Use [nvm](https://github.com/nvm-sh/nvm) to manage multiple Node versions: `nvm install 22.22.0 && nvm use 22.22.0`
- Or use [fnm](https://github.com/Schniz/fnm): `fnm install 22.22.0 && fnm use 22.22.0`

## Overview

The project uses an automated release workflow that:

- Enforces conventional commit messages
- Automatically determines version bumps based on commit types
- Syncs versions across all workspace packages
- Generates changelogs
- Creates GitHub releases
- Publishes to npm (if configured)

## Daily Workflow

### Making Changes and Committing

1. **Make your changes** to the codebase

2. **Stage your changes:**

   ```bash
   git add .
   ```

3. **Commit using Commitizen** (interactive prompts):

   ```bash
   git commit
   # OR use the npm script
   pnpm commit
   ```

   You'll be prompted to:
   - Select commit type (feat, fix, docs, etc.)
   - Select scope (cli, core, react, etc.)
   - Enter a short description
   - Optionally add breaking changes

4. **Push to main:**

   ```bash
   git push origin main
   ```

5. **GitHub Actions automatically:**
   - Analyzes commits
   - Determines version bump (patch/minor/major)
   - Updates version in all package.json files
   - Updates CHANGELOG.md
   - Creates git tag
   - Creates GitHub release

## Commit Types

### Types That Trigger Releases

| Type              | Release | Semver | Description                |
| ----------------- | ------- | ------ | -------------------------- |
| `feat`            | Yes     | MINOR  | New features               |
| `fix`             | Yes     | PATCH  | Bug fixes                  |
| `perf`            | Yes     | PATCH  | Performance improvements   |
| `BREAKING CHANGE` | Yes     | MAJOR  | Any type + breaking footer |

### Types That Don't Trigger Releases

| Type       | Description          |
| ---------- | -------------------- |
| `docs`     | Documentation only   |
| `style`    | Code formatting      |
| `refactor` | Code restructuring   |
| `test`     | Adding tests         |
| `build`    | Build system changes |
| `ci`       | CI configuration     |
| `chore`    | Maintenance tasks    |

## Commit Message Format

All commits must follow the conventional commit format:

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Examples

**Feature (triggers MINOR release):**

```
feat(cli): add JSON output format
```

**Bug fix (triggers PATCH release):**

```
fix(react): resolve hook dependency array issue
```

**Breaking change (triggers MAJOR release):**

```
feat(core): redesign API for better performance

BREAKING CHANGE: The analyze() function signature has changed.
The first parameter is now a configuration object instead of a file path.
```

**Non-release commit:**

```
docs: update README with new examples
```

## Available Scopes

When committing, you can select from these scopes:

- `cli` - CLI client
- `core` - Core analyzer
- `react` - React analyzer
- `nextjs` - Next.js analyzer
- `vscode` - VSCode extension
- `web` - Web client
- `docs` - Documentation
- `deps` - Dependencies
- `engine` - Analysis engine
- `common` - Common package
- `plugin-loader` - Plugin loader

You can also enter a custom scope if needed.

## Testing the Setup Locally

### Test Commitizen

```bash
git add .
git commit
# Commitizen should prompt you interactively
```

### Test Commitlint

Try committing with an invalid message format:

```bash
git commit -m "bad message"
# Should fail validation with commitlint error
```

### Test Version Sync

Manually run the version sync script:

```bash
pnpm run sync-versions
```

This should show all package.json files and schema files being updated to match the root version.

### Test Semantic-Release (Dry Run)

See what semantic-release would do without actually releasing:

```bash
# Ensure you're using Node.js >= 22.22.0
node --version  # Should show v22.20.0 or higher

pnpm exec semantic-release --dry-run
```

This shows:

- What version would be released
- What commits would be included
- What files would be updated

**Note:** If you see a Node.js version error, ensure you're using Node.js `22.22.0`. Use `nvm use 22.22.0` or `fnm use 22.22.0` if you have multiple Node versions installed.

## Version Synchronization

The `sync-versions` script automatically updates:

- All workspace package.json files:
  - `packages/*/package.json`
  - `analyzers/*/package.json`
  - `clients/*/package.json`
- JSON schema files:
  - `packages/common/schemas/vipr-config.schema.json`
  - `packages/common/schemas/analyzer-config.schema.json`

This script runs:

- Automatically before each commit (via pre-commit hook)
- During the release process (via semantic-release)

## Git Hooks

Husky manages git hooks that enforce commit standards:

### Pre-commit Hook

- Runs `pnpm run sync-versions` to ensure all versions are in sync
- Stages updated version files automatically

### Commit-msg Hook

- Validates commit message format using commitlint
- Rejects commits that don't follow conventional format

### Prepare-commit-msg Hook

- Provides interactive Commitizen prompts when you run `git commit` directly
- Automatically skips if you're using `pnpm commit` (which already runs commitizen)
- Makes it easy to create properly formatted commits whether you use `git commit` or `pnpm commit`

## GitHub Actions Workflow

The release workflow (`.github/workflows/release.yml`) runs on every push to `main`:

1. Checks out code
2. Sets up Node.js and pnpm
3. Installs dependencies
4. Builds all packages
5. Runs tests
6. Executes semantic-release

### Requirements

- Must be on `main` branch
- Commits must follow conventional format
- GitHub Actions must have proper permissions
- `GITHUB_TOKEN` is automatically provided
- `NPM_TOKEN` must be set in GitHub Secrets (if publishing to npm)

## NPM Publishing (Optional)

If you want to publish packages to npm:

1. **Create an npm token:**

   ```bash
   npm login
   npm token create
   ```

2. **Add token to GitHub Secrets:**
   - Go to your repo → Settings → Secrets and variables → Actions
   - Add secret: `NPM_TOKEN` with your npm token

3. **Update `.releaserc.json`:**
   Change `npmPublish` to `true` for packages you want to publish:
   ```json
   {
     "plugins": [
       [
         "@semantic-release/npm",
         {
           "npmPublish": true
         }
       ]
     ]
   }
   ```

## Troubleshooting

### Commits Not Triggering Releases

Check that:

- You're on the `main` branch
- Commit follows conventional format (use `pnpm commit` to ensure)
- GitHub Actions workflow has proper permissions
- `NPM_TOKEN` is set in GitHub Secrets (if publishing)

### Version Sync Not Working

- Ensure file paths in `scripts/sync-versions.mjs` match your structure
- Check that directories exist
- Run `pnpm run sync-versions` manually to see errors
- Verify the script has execute permissions: `chmod +x scripts/sync-versions.mjs`

### Husky Hooks Not Running

If hooks aren't executing:

```bash
# Reinstall husky
rm -rf .husky
pnpm exec husky init

# Recreate hook files
# See .husky/pre-commit, .husky/commit-msg, .husky/prepare-commit-msg

# Make hooks executable
chmod +x .husky/pre-commit
chmod +x .husky/commit-msg
chmod +x .husky/prepare-commit-msg
```

### Commitlint Errors

If commitlint rejects valid commits:

- Check `commitlint.config.js` for rule configuration
- Verify commit message follows format: `<type>(<scope>): <subject>`
- Ensure subject is lowercase and doesn't end with a period
- Subject must be 72 characters or less

### Semantic-Release Not Releasing

Common issues:

- No commits since last release
- All commits are non-release types (docs, style, etc.)
- Not on `main` branch
- GitHub Actions workflow failed earlier steps (build/test)
- **Node.js version too old** - requires Node.js `22.22.0`

If you see a Node.js version error:

- Locally: Upgrade to Node.js `22.22.0` using `nvm install 22.22.0 && nvm use 22.22.0` or `fnm install 22.22.0 && fnm use 22.22.0`
- In CI: Ensure GitHub Actions workflow uses `node-version-file: '.nvmrc'` (already configured)

Check the GitHub Actions logs for detailed error messages.

## Manual Release (If Needed)

If you need to manually trigger a release:

1. **Ensure all changes are committed:**

   ```bash
   git status
   ```

2. **Create a release commit:**

   ```bash
   git commit -m "chore(release): trigger manual release"
   ```

3. **Push to main:**

   ```bash
   git push origin main
   ```

4. **GitHub Actions will handle the rest**

## Best Practices

1. **Always use `pnpm commit` or `git commit`** (not `git commit -m`) to ensure proper formatting
2. **Use descriptive commit messages** that explain what and why
3. **Group related changes** in a single commit
4. **Use appropriate commit types** - don't use `feat` for bug fixes
5. **Include breaking changes** in the footer when making API changes
6. **Test locally** before pushing to ensure hooks work correctly
7. **Check GitHub Actions** after pushing to verify release succeeded

## Quick Reference

### Common Commands

```bash
# Interactive commit
pnpm commit

# Manual version sync
pnpm run sync-versions

# Test semantic-release
pnpm exec semantic-release --dry-run

# Check commit message format
pnpm exec commitlint --from HEAD~1 --to HEAD --verbose
```

### Commit Type Quick Reference

- `feat`: New feature → MINOR version bump
- `fix`: Bug fix → PATCH version bump
- `perf`: Performance improvement → PATCH version bump
- `docs`: Documentation → No release
- `style`: Formatting → No release
- `refactor`: Code restructuring → No release
- `test`: Tests → No release
- `build`: Build system → No release
- `ci`: CI config → No release
- `chore`: Maintenance → No release
