---
id: configuration
title: VSCode Extension Configuration
sidebar_label: Configuration
---

# VSCode Extension Configuration

Configure the Vipr VSCode extension to match your workflow.

## Settings

Access settings through:

- File > Preferences > Settings (Cmd+, or Ctrl+,)
- Search for "Vipr"

## Available Settings

### vipr.enabled

Enable or disable the extension.

```json
{
  "vipr.enabled": true
}
```

### vipr.autoAnalyze

Automatically analyze files on save.

```json
{
  "vipr.autoAnalyze": true
}
```

### vipr.reports

Configure which reports to show.

```json
{
  "vipr.reports": ["security", "accessibility", "performance"]
}
```

### vipr.severity

Minimum severity level to display.

```json
{
  "vipr.severity": "warning"
}
```

Options: `error`, `warning`, `info`, `hint`

### vipr.exclude

Glob patterns to exclude from analysis.

```json
{
  "vipr.exclude": ["**/*.test.tsx", "**/__tests__/**"]
}
```

## Workspace Settings

Create a `.vscode/settings.json` file in your project:

```json
{
  "vipr.enabled": true,
  "vipr.autoAnalyze": true,
  "vipr.reports": ["security", "accessibility"],
  "vipr.exclude": ["**/node_modules/**", "**/*.test.tsx"]
}
```

## Next Steps

- [Features](./features) - Learn about extension features
- [Installation](./installation) - Install the extension
