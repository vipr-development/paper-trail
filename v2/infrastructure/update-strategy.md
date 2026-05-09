# Update Strategy

Vipr uses each distribution channel's built-in update mechanism. No custom update server is needed.

## Overview

| Client            | Update Mechanism                     | Distribution Channel             | CI Trigger                                 |
| ----------------- | ------------------------------------ | -------------------------------- | ------------------------------------------ |
| Desktop           | `electron-updater` + GitHub Releases | Direct download (.dmg/.exe/.deb) | `v*` tag → `electron-forge publish`        |
| VS Code Extension | Marketplace auto-update              | VS Code Marketplace + Open VSX   | `v*` tag → `vsce publish` + `ovsx publish` |
| CLI               | npm registry                         | `npm install -g @vipr/cli`       | `v*` tag → `npm publish`                   |

## Desktop (Electron)

The desktop app uses `electron-updater` with GitHub Releases as the update source.

### How it works

1. `electron-updater` checks GitHub Releases for newer versions on app launch
2. Downloads the update asset in the background
3. Prompts the user to restart and apply

### Prerequisites (not yet configured)

- **Code signing certificates**: Apple Developer ID (macOS) + Windows Authenticode
- **Forge config**: Uncomment `osxSign`/`osxNotarize` and publisher sections in `forge.config.ts`
- **GitHub secrets**:
  - `APPLE_ID` — Apple developer email
  - `APPLE_ID_PASSWORD` — App-specific password
  - `APPLE_TEAM_ID` — Apple team identifier
  - `CSC_LINK` — macOS code signing certificate (base64)
  - `CSC_KEY_PASSWORD` — Certificate password
  - `WIN_CSC_LINK` — Windows code signing certificate (base64)
  - `WIN_CSC_KEY_PASSWORD` — Windows certificate password
  - `GITHUB_TOKEN` — For publishing releases

### CI workflow

```yaml
# Triggered by v* tags
- name: Build and publish
  run: pnpm --filter @vipr/desktop make
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## VS Code Extension

The extension publishes to both VS Code Marketplace and Open VSX Registry.

### How it works

1. `vsce publish` pushes to VS Code Marketplace
2. `ovsx publish` pushes to Open VSX (for Cursor, Windsurf, etc.)
3. VS Code/Cursor auto-updates extensions in the background

### Prerequisites

- **CI secrets**:
  - `VSCE_PAT` — Azure DevOps Personal Access Token for VS Code Marketplace
  - `OVSX_PAT` — Open VSX access token

### CI workflow

```yaml
# Triggered by v* tags
- name: Publish to VS Code Marketplace
  run: pnpm --filter vipr-vscode-extension package && pnpm --filter vipr-vscode-extension publish:vscode
  env:
    VSCE_PAT: ${{ secrets.VSCE_PAT }}

- name: Publish to Open VSX
  run: pnpm --filter vipr-vscode-extension publish:openvsx
  env:
    OVSX_PAT: ${{ secrets.OVSX_PAT }}
```

## CLI

The CLI publishes as an npm package.

### How it works

1. Users install globally: `npm install -g @vipr/cli`
2. `update-notifier` checks for new versions and displays a banner
3. Users update manually: `npm update -g @vipr/cli`

### Prerequisites

- Set `"private": false` in `clients/cli/package.json`
- Add `publishConfig` with registry URL
- Add `update-notifier` dependency for version notifications
- **CI secrets**:
  - `NPM_TOKEN` — npm publish token

### CI workflow

```yaml
# Triggered by v* tags
- name: Publish to npm
  run: pnpm --filter @vipr/cli build && pnpm --filter @vipr/cli publish --no-git-checks
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## Version Coordination

All three clients share the monorepo version via `pnpm sync-versions`. When cutting a release:

1. Update root `package.json` version
2. Run `pnpm sync-versions` to propagate
3. Create and push a `v*` tag
4. CI publishes all three clients

## Rollback Strategy

| Client  | Rollback Method                                                                                              |
| ------- | ------------------------------------------------------------------------------------------------------------ |
| Desktop | Publish a patch release; `electron-updater` delivers it. Users can also reinstall from GitHub Releases.      |
| VS Code | Publish a patch version. Users can manually install a previous `.vsix` from the Marketplace version history. |
| CLI     | `npm install -g @vipr/cli@<version>` to pin a specific version.                                              |
