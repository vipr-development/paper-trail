---
id: usage
title: CLI Usage
sidebar_label: Usage
---

# CLI Usage

The Vipr CLI provides a powerful command-line interface for analyzing React code.

## Basic Usage

```bash
vipr analyze <path> [options]
```

## Common Examples

Analyze a directory:

```bash
vipr analyze ./src
```

Analyze with specific reports:

```bash
vipr analyze ./src --report security,accessibility
```

Output as JSON:

```bash
vipr analyze ./src --format json > output.json
```

Analyze with custom config:

```bash
vipr analyze ./src --config vipr.config.json
```

## Global Options

| Option      | Description         | Default            |
| ----------- | ------------------- | ------------------ |
| `--version` | Show version number | -                  |
| `--help`    | Show help           | -                  |
| `--config`  | Path to config file | `vipr.config.json` |

## Next Steps

- [Commands](./commands) - Learn about all available commands
- [Configuration](./configuration) - Configure Vipr for your needs
