# Setting up automatic versioning with semver, husky and pre-commit hook

# Complete Setup Guide: Automated Versioning with semantic-release + Commitizen

## **Prerequisites**

- Git repository initialized
- npm project with monorepo structure (or single package)
- GitHub repository (for automated releases)

## **Step 1: Clean Up Existing Tools**

```bash
# Remove Changesets if previously installed
npm uninstall @changesets/cli
rm -rf .changeset

# Remove any existing version-related scripts from package.json
```

## **Step 2: Install Dependencies**

```bash
# Core tools
npm install -D commitizen cz-customizable
npm install -D @commitlint/cli @commitlint/config-conventional
npm install -D husky

# semantic-release and plugins
npm install -D semantic-release \
  @semantic-release/changelog \
  @semantic-release/git \
  @semantic-release/github \
  @semantic-release/npm \
  @semantic-release/exec
```

## **Step 3: Create Configuration Files**

### **3.1: Commitizen Config**

Create `scripts/cz-config.js`:

```javascript
export default {
  types: [
    {
      value: 'feat',
      name: 'feat:     New feature (triggers MINOR release)',
    },
    {
      value: 'fix',
      name: 'fix:      Bug fix (triggers PATCH release)',
    },
    {
      value: 'perf',
      name: 'perf:     Performance improvement (triggers PATCH release)',
    },
    {
      value: 'docs',
      name: 'docs:     Documentation only (no release)',
    },
    {
      value: 'style',
      name: 'style:    Code style changes (no release)',
    },
    {
      value: 'refactor',
      name: 'refactor: Code refactoring (no release)',
    },
    {
      value: 'test',
      name: 'test:     Adding or updating tests (no release)',
    },
    {
      value: 'build',
      name: 'build:    Build system changes (no release)',
    },
    {
      value: 'ci',
      name: 'ci:       CI configuration (no release)',
    },
    {
      value: 'chore',
      name: 'chore:    Other changes (no release)',
    },
    {
      value: 'revert',
      name: 'revert:   Revert a previous commit',
    },
  ],

  scopes: [
    { name: 'cli' },
    { name: 'core' },
    { name: 'vscode' },
    { name: 'web' },
    { name: 'docs' },
    { name: 'deps' },
  ],

  allowCustomScopes: true,
  allowBreakingChanges: ['feat', 'fix', 'perf', 'refactor'],

  messages: {
    type: "Select the type of change you're committing:",
    scope: 'Select the scope of this change (optional):',
    customScope: 'Enter a custom scope:',
    subject: 'Write a SHORT description:\n',
    body: 'Provide a LONGER description (optional). Use "|" for new lines:\n',
    breaking: 'List any BREAKING CHANGES (triggers MAJOR release):\n',
    footer: 'List any ISSUES CLOSED (e.g., #31, #34):\n',
    confirmCommit: 'Are you sure you want to proceed with the commit above?',
  },

  skipQuestions: ['body', 'footer'],
  subjectLimit: 72,
  breaklineChar: '|',
  footerPrefix: 'CLOSES:',
};
```

### **3.2: Commitlint Config**

Create `commitlint.config.js`:

```javascript
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'perf',
        'test',
        'build',
        'ci',
        'chore',
        'revert',
      ],
    ],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-case': [2, 'never', ['upper-case']],
    'subject-empty': [2, 'never'],
    'subject-full-stop': [2, 'never', '.'],
    'header-max-length': [2, 'always', 72],
  },
};
```

### **3.3: semantic-release Config**

Create `.releaserc.json`:

```json
{
  "branches": ["main"],
  "plugins": [
    [
      "@semantic-release/commit-analyzer",
      {
        "preset": "conventionalcommits",
        "releaseRules": [
          { "type": "feat", "release": "minor" },
          { "type": "fix", "release": "patch" },
          { "type": "perf", "release": "patch" },
          { "type": "revert", "release": "patch" },
          { "type": "docs", "release": false },
          { "type": "style", "release": false },
          { "type": "refactor", "release": false },
          { "type": "test", "release": false },
          { "type": "build", "release": false },
          { "type": "ci", "release": false },
          { "type": "chore", "release": false },
          { "type": "monorepo", "release": false }
        ]
      }
    ],
    [
      "@semantic-release/release-notes-generator",
      {
        "preset": "conventionalcommits",
        "presetConfig": {
          "types": [
            { "type": "feat", "section": "Features" },
            { "type": "fix", "section": "Bug Fixes" },
            { "type": "perf", "section": "Performance Improvements" },
            { "type": "revert", "section": "Reverts" },
            { "type": "docs", "section": "Documentation", "hidden": false },
            { "type": "style", "section": "Styles", "hidden": true },
            { "type": "chore", "section": "Miscellaneous Chores", "hidden": true },
            { "type": "refactor", "section": "Code Refactoring", "hidden": true },
            { "type": "test", "section": "Tests", "hidden": true },
            { "type": "build", "section": "Build System", "hidden": true },
            { "type": "ci", "section": "Continuous Integration", "hidden": true }
          ]
        }
      }
    ],
    [
      "@semantic-release/changelog",
      {
        "changelogFile": "CHANGELOG.md",
        "changelogTitle": "# Changelog\n\nAll notable changes to this project will be documented in this file."
      }
    ],
    [
      "@semantic-release/exec",
      {
        "prepareCmd": "npm run sync-versions"
      }
    ],
    [
      "@semantic-release/npm",
      {
        "npmPublish": true
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": [
          "CHANGELOG.md",
          "package.json",
          "package-lock.json",
          "packages/*/package.json",
          "packages/*/src/version.ts",
          "packages/*/src/schema.json"
        ],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    [
      "@semantic-release/github",
      {
        "assets": [{ "path": "dist/**/*" }]
      }
    ]
  ]
}
```

## **Step 4: Create Version Sync Script**

Create `scripts/sync-versions.js`:

```javascript
#!/usr/bin/env node
import { readFileSync, writeFileSync, existsSync } from 'fs';
import { join, dirname } from 'path';
import { fileURLToPath } from 'url';

const __dirname = dirname(fileURLToPath(import.meta.url));
const rootDir = join(__dirname, '..');

/**
 * Sync version from root package.json to all workspace packages,
 * JSON schemas, and version constant files
 */
function syncVersions() {
  // Read root package.json version
  const rootPkgPath = join(rootDir, 'package.json');
  const rootPkg = JSON.parse(readFileSync(rootPkgPath, 'utf8'));
  const version = rootPkg.version;

  console.log(`\n🔄 Syncing version ${version}...\n`);

  // Define your monorepo structure
  const workspaces = ['cli', 'core', 'vscode', 'web'];

  // 1. Update workspace package.json files
  workspaces.forEach(workspace => {
    const pkgPath = join(rootDir, 'packages', workspace, 'package.json');

    if (!existsSync(pkgPath)) {
      console.log(`⏭️  Skipping packages/${workspace} (doesn't exist)`);
      return;
    }

    try {
      const pkg = JSON.parse(readFileSync(pkgPath, 'utf8'));
      pkg.version = version;
      writeFileSync(pkgPath, JSON.stringify(pkg, null, 2) + '\n');
      console.log(`✅ Updated packages/${workspace}/package.json`);
    } catch (err) {
      console.error(`❌ Failed to update packages/${workspace}:`, err.message);
      process.exit(1);
    }
  });

  // 2. Update JSON schema files
  const schemaFiles = [
    'packages/core/src/schema.json',
    // Add more schema paths as needed
  ];

  schemaFiles.forEach(schemaPath => {
    const fullPath = join(rootDir, schemaPath);

    if (!existsSync(fullPath)) {
      console.log(`⏭️  Skipping ${schemaPath} (doesn't exist)`);
      return;
    }

    try {
      const schema = JSON.parse(readFileSync(fullPath, 'utf8'));
      schema.version = version;
      writeFileSync(fullPath, JSON.stringify(schema, null, 2) + '\n');
      console.log(`✅ Updated ${schemaPath}`);
    } catch (err) {
      console.error(`❌ Failed to update ${schemaPath}:`, err.message);
      process.exit(1);
    }
  });

  // 3. Update version constant files
  const versionFiles = [
    {
      path: 'packages/cli/src/version.ts',
      content: `/**
 * This file is auto-generated by scripts/sync-versions.js
 * DO NOT EDIT MANUALLY
 */
export const VERSION = '${version}';
`,
    },
    {
      path: 'packages/core/src/version.ts',
      content: `/**
 * This file is auto-generated by scripts/sync-versions.js
 * DO NOT EDIT MANUALLY
 */
export const VERSION = '${version}';
`,
    },
    // Add more version files as needed
  ];

  versionFiles.forEach(({ path, content }) => {
    const fullPath = join(rootDir, path);
    const dir = dirname(fullPath);

    if (!existsSync(dir)) {
      console.log(`⏭️  Skipping ${path} (directory doesn't exist)`);
      return;
    }

    try {
      writeFileSync(fullPath, content);
      console.log(`✅ Updated ${path}`);
    } catch (err) {
      console.error(`❌ Failed to update ${path}:`, err.message);
      process.exit(1);
    }
  });

  console.log(`\n✨ Version sync complete! All files updated to ${version}\n`);
}

// Run the sync
syncVersions();
```

Make it executable:

```bash
chmod +x scripts/sync-versions.js
```

## **Step 5: Update package.json**

Update your root `package.json`:

```json
{
  "name": "vipr",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "workspaces": ["packages/*"],
  "scripts": {
    "commit": "cz",
    "sync-versions": "node scripts/sync-versions.js",
    "prepare": "husky",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint"
  },
  "config": {
    "commitizen": {
      "path": "cz-customizable"
    },
    "cz-customizable": {
      "config": "scripts/cz-config.js"
    }
  },
  "devDependencies": {
    "@commitlint/cli": "^latest",
    "@commitlint/config-conventional": "^latest",
    "@semantic-release/changelog": "^latest",
    "@semantic-release/exec": "^latest",
    "@semantic-release/git": "^latest",
    "@semantic-release/github": "^latest",
    "@semantic-release/npm": "^latest",
    "commitizen": "^latest",
    "cz-customizable": "^latest",
    "husky": "^latest",
    "semantic-release": "^latest"
  }
}
```

## **Step 6: Set Up Git Hooks**

Initialize husky:

```bash
npx husky init
```

Create `.husky/pre-commit`:

```bash
#!/bin/sh

# Run version sync to ensure all files are up to date
npm run sync-versions

# Stage any updated version files
git add -u packages/**/src/version.ts 2>/dev/null || true
git add -u packages/**/src/schema.json 2>/dev/null || true
git add -u packages/**/package.json 2>/dev/null || true
```

Create `.husky/commit-msg`:

```bash
#!/bin/sh

# Validate commit message format
npx --no -- commitlint --edit $1
```

Create `.husky/prepare-commit-msg`:

```bash
#!/bin/sh

# Run commitizen for interactive commits
exec < /dev/tty && npx cz --hook || true
```

Make hooks executable:

```bash
chmod +x .husky/pre-commit
chmod +x .husky/commit-msg
chmod +x .husky/prepare-commit-msg
```

## **Step 7: Create Initial Version Files**

Create the version constant files that will be auto-updated:

```bash
# CLI version
mkdir -p packages/cli/src
cat > packages/cli/src/version.ts << 'EOF'
/**
 * This file is auto-generated by scripts/sync-versions.js
 * DO NOT EDIT MANUALLY
 */
export const VERSION = '0.0.0';
EOF

# Core version
mkdir -p packages/core/src
cat > packages/core/src/version.ts << 'EOF'
/**
 * This file is auto-generated by scripts/sync-versions.js
 * DO NOT EDIT MANUALLY
 */
export const VERSION = '0.0.0';
EOF
```

If you have JSON schemas, create them too:

```bash
cat > packages/core/src/schema.json << 'EOF'
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "version": "0.0.0",
  "type": "object",
  "properties": {}
}
EOF
```

## **Step 8: Use Version in Your CLI**

Update `packages/cli/src/index.ts`:

```typescript
#!/usr/bin/env node
import { Command } from 'commander';
import { VERSION } from './version.js';

const program = new Command();

program
  .name('vipr')
  .description('Technical debt analysis for AI-generated codebases')
  .version(VERSION, '-v, --version', 'Display version number');

program
  .command('analyze')
  .description('Analyze codebase for technical debt')
  .action(() => {
    console.log(`Vipr v${VERSION} - Starting analysis...`);
    // Your analysis logic
  });

program.parse();
```

## **Step 9: Set Up GitHub Actions**

Create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  issues: write
  pull-requests: write
  id-token: write

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.22.0
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build packages
        run: npm run build

      - name: Run tests
        run: npm test

      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

## **Step 10: Configure NPM Publishing (Optional)**

If you want to publish to npm, add to each package's `package.json`:

```json
{
  "name": "@vipr/cli",
  "publishConfig": {
    "access": "public"
  },
  "files": ["dist", "README.md", "LICENSE"]
}
```

Create an npm token:

```bash
npm login
npm token create
```

Add the token to GitHub Secrets:

- Go to your repo → Settings → Secrets and variables → Actions
- Add secret: `NPM_TOKEN` with your npm token

## **Step 11: Test the Setup Locally**

```bash
# Test commitizen
git add .
git commit
# Commitizen should prompt you

# Test commitlint (try a bad commit message)
git commit -m "bad message"
# Should fail validation

# Test version sync
npm run sync-versions
# Should show all files being synced

# Test semantic-release dry run
npx semantic-release --dry-run
# Should show what would happen
```

## **Step 12: Add .gitignore Rules**

Add to `.gitignore`:

```gitignore
# Dependencies
node_modules/

# Build outputs
dist/
build/
*.tsbuildinfo

# Logs
npm-debug.log*

# OS
.DS_Store
```

## **Complete Workflow**

### **Daily Development:**

```bash
# 1. Make your changes
vim packages/cli/src/analyze.ts

# 2. Stage changes
git add .

# 3. Commit (Commitizen prompts automatically)
git commit
# ? Type: feat
# ? Scope: cli
# ? Description: add JSON output format
# ✓ Commit created with message: "feat(cli): add JSON output format"

# 4. Push to main
git push origin main

# 5. GitHub Actions automatically:
#    - Analyzes commits
#    - Determines this is a MINOR bump (feat = minor)
#    - Updates version: 1.0.0 → 1.1.0
#    - Runs sync-versions script
#    - Updates CHANGELOG.md
#    - Creates git tag v1.1.0
#    - Publishes to npm
#    - Creates GitHub release
```

### **Making a Breaking Change:**

```bash
git add .
git commit
# ? Type: feat
# ? Scope: core
# ? Description: redesign API for better performance
# ? Breaking changes: The analyze() function signature has changed
# ✓ Creates MAJOR version bump: 1.1.0 → 2.0.0
```

### **Non-Release Commits:**

```bash
# Documentation updates
git commit
# ? Type: docs
# ? Description: update README with new examples
# ✓ No release triggered

# Refactoring
git commit
# ? Type: refactor
# ? Description: extract utility functions
# ✓ No release triggered
```

## **Commit Type Reference**

| Type              | Release | Semver | Example                    |
| ----------------- | ------- | ------ | -------------------------- |
| `feat`            | ✅ Yes  | MINOR  | New features               |
| `fix`             | ✅ Yes  | PATCH  | Bug fixes                  |
| `perf`            | ✅ Yes  | PATCH  | Performance improvements   |
| `docs`            | ❌ No   | -      | Documentation only         |
| `style`           | ❌ No   | -      | Code formatting            |
| `refactor`        | ❌ No   | -      | Code restructuring         |
| `test`            | ❌ No   | -      | Adding tests               |
| `build`           | ❌ No   | -      | Build system changes       |
| `ci`              | ❌ No   | -      | CI configuration           |
| `chore`           | ❌ No   | -      | Maintenance tasks          |
| `BREAKING CHANGE` | ✅ Yes  | MAJOR  | Any type + breaking footer |

## **Troubleshooting**

### **Commits not triggering releases:**

Check that:

- You're on the `main` branch
- Commit follows conventional format
- GitHub Actions has proper permissions
- NPM_TOKEN is set in GitHub Secrets

### **Version sync not working:**

- Ensure file paths in `sync-versions.js` match your structure
- Check that directories exist
- Run `npm run sync-versions` manually to see errors

### **Husky hooks not running:**

```bash
# Reinstall husky
rm -rf .husky
npx husky init
# Recreate hook files from Step 6
```
