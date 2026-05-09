# VSCode Extension Development

This guide covers local development, testing, packaging, and installation of the Vipr VSCode Extension from the monorepo.

## Prerequisites

- Node.js >= 22.22.0
- pnpm >= 8
- VSCode >= 1.85.0
- VSCode Extension Manager CLI (`vsce`)

Install `vsce` globally if you haven't already:

```bash
npm install -g @vscode/vsce
```

## Local Development Setup

### 1. Install Dependencies

From the monorepo root:

```bash
pnpm install
```

This installs dependencies for all workspace packages including the VSCode extension.

### 2. Build the Extension

Build the extension and all its dependencies:

```bash
# Build all packages in the monorepo
pnpm build

# Or build just the extension
cd clients/vscode-extension
pnpm build
```

This compiles TypeScript to JavaScript in the `dist/` directory.

### 3. Watch Mode for Development

For active development with automatic rebuilds:

```bash
cd clients/vscode-extension
pnpm dev
```

This watches for file changes and rebuilds automatically.

## Testing the Extension

### Method 1: Run Extension Development Host

The fastest way to test during development:

1. Open the monorepo in VSCode
2. Navigate to `clients/vscode-extension/`
3. Press `F5` or go to Run and Debug > "Run Extension"
4. A new VSCode window opens with the extension loaded
5. Open a React/TypeScript project in this window
6. Test the extension features

**Debugging**: Set breakpoints in your TypeScript source files. They work automatically with source maps.

### Method 2: Install as VSIX (Local Package)

Test the extension as users would experience it:

```bash
cd clients/vscode-extension

# Package the extension into a .vsix file
pnpm package
```

This creates `vipr-vscode-{version}.vsix` in the extension directory.

Install the VSIX file:

```bash
# Install from command line
code --install-extension vipr-vscode-{version}.vsix

# Or in VSCode:
# 1. Open Extensions view (Cmd+Shift+X)
# 2. Click ... menu > Install from VSIX
# 3. Select the .vsix file
```

Reload VSCode after installation.

### Method 3: Symlink for Live Development

For testing with live updates without rebuilding:

```bash
cd clients/vscode-extension

# Create symlink in VSCode extensions directory
# macOS/Linux:
ln -s $(pwd) ~/.vscode/extensions/vipr-vscode-dev

# Windows (as Administrator):
mklink /D "%USERPROFILE%\.vscode\extensions\vipr-vscode-dev" "%CD%"

# Reload VSCode window
```

**Warning**: Run `pnpm build` or `pnpm dev` to ensure `dist/` folder is up to date.

## Running Extension Tests

The extension includes unit and integration tests:

```bash
cd clients/vscode-extension

# Run all tests
pnpm test

# Watch mode
pnpm test:watch

# Type checking
pnpm typecheck

# Linting
pnpm lint
```

## Testing Workflow

### 1. Prepare a Test Project

Create or use an existing React/TypeScript project:

```bash
# Example: Create a test React project
npx create-react-app test-project --template typescript
cd test-project
```

### 2. Test Core Features

Open the test project in the Extension Development Host and verify:

**Commands** (Cmd+Shift+P):

- `Vipr: Analyze Current File` - Analyzes the active file
- `Vipr: Analyze Workspace` - Analyzes all eligible files
- `Vipr: Show License Status` - Shows current license tier
- `Vipr: Fix with AI` - AI-assisted fixing (if diagnostic selected)

**Status Bar**:

- Shows Vipr icon in bottom status bar
- Displays "Ready" or file score after analysis
- Click to run analysis on current file

**Problems Panel** (Cmd+Shift+M):

- View diagnostics after running analysis
- Diagnostics grouped by file
- Severity levels: Error, Warning, Info, Hint

**CodeLens Hints**:

- Inline hints above React components showing scores
- Click hint to view details

**Editor Decorations**:

- Gutter icons for issues (🔴 error, 🟡 warning, 🔵 info)
- Hover over highlighted code for issue details

**Quick Fixes**:

- Light bulb appears on auto-fixable issues
- Click to apply automatic fix
- AI-powered fixes available via "Fix with AI" action

### 3. Test Configuration

Create `.vipr/config.json` in your test project:

```json
{
  "analyzers": {
    "react": {
      "enabled": true
    },
    "core": {
      "enabled": true
    }
  },
  "reporters": ["overview"]
}
```

Update VSCode settings (`.vscode/settings.json`):

```json
{
  "vipr.enableDiagnostics": true,
  "vipr.enableCodeLens": true,
  "vipr.enableDecorations": true,
  "vipr.analyzeOnSave": false,
  "vipr.licenseKey": ""
}
```

Test that changes take effect:

- Toggle settings
- Verify features enable/disable correctly
- Test license key validation

### 4. Test License Tiers

**Free Tier** (no license key):

- Core Overview reports available
- React Overview reports available
- Premium reports should be blocked

**Pro Tier**:

```json
{
  "vipr.licenseKey": "VIPR-PRO-XXXXXXXXXXXX"
}
```

- All reports available
- Test premium features unlock

**Enterprise Tier**:

```json
{
  "vipr.licenseKey": "VIPR-ENT-XXXXXXXXXXXX"
}
```

- All features available
- Same as Pro for MVP

Run `Vipr: Show License Status` to verify tier detection.

## Debugging Tips

### Enable Extension Host Logging

View extension logs in Output panel:

1. Open Output panel (Cmd+Shift+U)
2. Select "Vipr" from dropdown
3. Logs appear as extension runs

### Debug Extension Code

Set breakpoints in TypeScript source:

1. Open `clients/vscode-extension/src/**/*.ts`
2. Click line number gutter to set breakpoint
3. Press `F5` to start debugging
4. Extension Development Host opens
5. Trigger feature to hit breakpoint
6. Inspect variables, step through code

### Common Issues

**Extension not activating**:

- Check `package.json` activation events
- Verify main entry point exists: `dist/extension.js`
- Check Extension Host logs for errors

**Diagnostics not appearing**:

- Verify file is eligible (React/TypeScript)
- Check if diagnostics are enabled in settings
- Run "Analyze Current File" command explicitly

**CodeLens not showing**:

- Verify `vipr.enableCodeLens` setting is true
- Check file contains React components
- Ensure analysis completed successfully

**Quick fixes not available**:

- Verify issue is auto-fixable (check presenter metadata)
- Ensure diagnostic is present
- Check code action provider registration

## Packaging for Distribution

### Create VSIX Package

Package the extension for distribution:

```bash
cd clients/vscode-extension

# Ensure clean build
pnpm clean
pnpm build

# Create package
pnpm package
```

This creates `vipr-vscode-{version}.vsix` file.

### Package Contents

The VSIX includes:

- Compiled JavaScript (`dist/`)
- `package.json` manifest
- README.md
- CHANGELOG.md
- LICENSE
- Icon files

Files excluded via `.vscodeignore`:

- TypeScript source (`src/`)
- Tests (`test/`)
- Development config files

### Test the Package

Before publishing, test the packaged VSIX:

```bash
# Install in clean VSCode instance
code --install-extension vipr-vscode-{version}.vsix

# Or create test profile
code --user-data-dir=/tmp/vscode-test --install-extension vipr-vscode-{version}.vsix
```

Verify all features work from the packaged version.

## Publishing to Marketplace

### Prerequisites

1. Create Azure DevOps account
2. Create Personal Access Token (PAT) with Marketplace scope
3. Create publisher account at https://marketplace.visualstudio.com/manage

### Publish Command

```bash
cd clients/vscode-extension

# Login with PAT
vsce login {publisher-name}

# Publish (increments patch version automatically)
vsce publish

# Or publish specific version
vsce publish minor
vsce publish major
vsce publish 1.2.3
```

### Pre-publish Checklist

- [ ] All tests pass (`pnpm test`)
- [ ] Type checking passes (`pnpm typecheck`)
- [ ] Linting passes (`pnpm lint`)
- [ ] CHANGELOG.md updated
- [ ] Version bumped in `package.json`
- [ ] README.md has clear installation instructions
- [ ] Extension icon is set
- [ ] Tested VSIX locally
- [ ] No hardcoded values or development artifacts

## Version Management

Follow semantic versioning:

- **Patch** (1.0.x): Bug fixes, minor improvements
- **Minor** (1.x.0): New features, backward compatible
- **Major** (x.0.0): Breaking changes

Update version in `package.json`:

```json
{
  "version": "1.2.3"
}
```

## Continuous Integration

For automated testing and packaging, add to monorepo CI:

```yaml
# Example GitHub Actions workflow
- name: Build VSCode Extension
  run: |
    cd clients/vscode-extension
    pnpm build
    pnpm test
    pnpm package

- name: Upload VSIX
  uses: actions/upload-artifact@v3
  with:
    name: vscode-extension
    path: clients/vscode-extension/*.vsix
```

## Troubleshooting

### Build Errors

**Module not found errors**:

```bash
# Ensure all dependencies installed
pnpm install

# Rebuild from clean state
pnpm clean
pnpm build
```

**Type errors**:

```bash
# Check TypeScript version consistency
pnpm why typescript

# Ensure @types/vscode matches engine version
```

### Runtime Errors

**Extension fails to activate**:

- Check Extension Host logs
- Verify `activationEvents` in `package.json`
- Ensure main entry point compiled: `dist/extension.js`

**Commands not registered**:

- Verify commands defined in `package.json` contributions
- Check command registration in `extension.ts`
- Ensure command handlers are async

**Providers not working**:

- Verify provider registration in `extension.ts`
- Check document selector matches file types
- Ensure analysis completes before provider queries

## Performance Monitoring

Monitor extension performance:

1. Open Command Palette > "Developer: Show Running Extensions"
2. View activation time, CPU usage, memory
3. Identify slow activation or high resource usage

Optimize if needed:

- Lazy load providers
- Debounce expensive operations
- Cache analysis results
- Use web workers for heavy computation

## Related Documentation

- [Installation](./installation) - User installation instructions
- [Features](./features) - Feature overview and usage
- [Configuration](./configuration) - Configuration options
- [Phase Documentation](../../feature-development/vscode-extension/) - Implementation phases

## Support

For issues or questions:

- GitHub Issues: https://github.com/yourusername/vipr/issues
- Documentation: https://vipr.dev/docs/vscode/development
