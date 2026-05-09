# Interactive CLI Mode - User Guide

## Overview

Vipr now supports an interactive mode that guides you through configuration and analysis with visual prompts and menus. This guide covers all interactive features.

## Quick Start

### Interactive Init

Create a configuration file with guided prompts:

```bash
vipr init
```

You'll be asked:

1. Where to place the config file
2. Which preset to use (default, strict, lenient, or custom)
3. Custom settings (if you chose custom)

Skip interactive mode for scripting:

```bash
vipr init --quiet --preset strict
```

### Interactive Analysis

Run analysis with guided prompts:

```bash
vipr analyze -i
```

Or simply:

```bash
vipr analyze
```

When no files are specified, interactive mode starts automatically.

## Features

### 1. Interactive Init Flow

**Command:** `vipr init`

**What it does:**

- Displays the Vipr logo
- Guides you through configuration setup
- Validates your choices
- Creates a config file

**Options:**
| Option | Description | Default |
|--------|-------------|---------|
| `-q, --quiet` | Skip interactive mode | false |
| `-f, --force` | Overwrite existing config | false |
| `-p, --preset <type>` | Preset to use | none |

**Example Session:**

```
           d8,
          `8P

?88   d8P  88b?88,.d88b,  88bd88b
d88  d8P'  88P`?88'  ?88  88P'  `
?8b ,88'  d88   88b  d8P d88
`?888P'  d88'   888888P'd88'

┌  Welcome to Vipr configuration setup
│
◇  Where would you like to place the configuration file?
│  Current directory
│
◇  Which configuration preset would you like to use?
│  Default
│
◇  Configuration file created!
│
└  Setup complete!
```

### 2. Interactive Analysis Flow

**Command:** `vipr analyze -i [files...]`

**What it does:**

- Prompts for file path (if not provided)
- Lets you select which reports to generate
- Choose output format
- Decide where to save results

**Options:**
| Option | Description | Default |
|--------|-------------|---------|
| `-i, --interactive` | Enable interactive mode | false |
| `-q, --quiet` | Suppress output | false |
| `-d, --debug` | Enable debug logging | false |

**Report Types Available:**

- Overview - High-level summary and overall score
- React Overview - React-specific patterns
- Security - Security vulnerabilities
- Accessibility - A11y compliance
- Performance - Performance bottlenecks
- Reliability - Error handling
- Migration - Upgrade paths
- Dataflow - Data flow patterns
- Anti-patterns - Code anti-patterns
- Technical Debt - Refactoring opportunities
- Types - TypeScript type safety

**Output Formats:**

- **Interactive** - Section-by-section viewer (recommended)
- **CLI** - Standard terminal output
- **JSON (compact)** - Machine-readable compact JSON
- **JSON (full)** - Complete JSON with all details
- **Markdown** - Documentation-friendly format

**Output Destinations:**

- **Display only** - Show in terminal
- **Save to file** - Write to disk
- **Copy to clipboard** - Copy to system clipboard

**Example Session:**

```
◇  Enter the file or glob pattern to analyze:
│  src/components/**/*.tsx
│
◇  Which reports would you like to generate?
│  ◼ Overview
│  ◻ React Overview
│  ◼ Security
│  ◼ Performance
│  ◻ Accessibility
│
◇  How would you like to view the results?
│  Interactive Viewer
│
◆  Analyzing files...
│
└  Analysis complete!
```

### 3. Interactive Report Viewer

**Activated when:** You choose "Interactive Viewer" as output format

**Features:**

- Navigate through report sections one at a time
- Clear, focused view of each section
- Position indicator shows progress
- Export options from within viewer

**Navigation:**

```
[Section content displayed here]

  ○ ○ ○ ● ○ ○ ○  (4/7)

Section 4 of 7

? What would you like to do?
> Next section        (n)
  Previous section    (p)
  Jump to section...  (j)
  Export report       (e)
  Exit viewer         (q)
```

**Controls:**

- **Next section** - Move forward one section
- **Previous section** - Move back one section
- **Jump to section** - Choose from a list of all sections
- **Export report** - Save or copy the complete report
- **Exit viewer** - Return to command line

**Export Options:**
When you choose "Export report" from the viewer:

- **Save to file** - Write complete report to disk
- **Copy to clipboard** - Copy to system clipboard (macOS/Linux/Windows)

### 4. Clipboard Support

**Platform Support:**

- ✅ macOS (pbcopy)
- ✅ Linux (xclip/xsel required)
- ✅ Windows (clip)

**Usage:**
Clipboard is available as an output destination when:

1. Analyzing with non-interactive format (CLI, JSON, Markdown)
2. Exporting from the interactive viewer

**Example:**

```bash
vipr analyze src/app.tsx -i
# Select reports...
# Choose output format: JSON (compact)
# Where to save: Copy to clipboard
✔ Results copied to clipboard!
```

## Command Reference

### vipr init

Initialize a new configuration file.

**Interactive (default):**

```bash
vipr init
```

**Non-interactive:**

```bash
vipr init --quiet --preset default
vipr init -q -p strict
vipr init --force --preset lenient
```

### vipr analyze

Analyze files for code quality issues.

**Interactive with file:**

```bash
vipr analyze src/components/Button.tsx -i
```

**Interactive with prompt for file:**

```bash
vipr analyze -i
```

**Auto-trigger interactive (no files):**

```bash
vipr analyze
```

**Non-interactive (classic mode):**

```bash
vipr analyze src/**/*.tsx
vipr analyze src/app.tsx -r overview,security -f json -o report.json
```

## Tips and Tricks

### 1. Skip Prompts in CI

Always use `--quiet` flag for non-interactive environments:

```bash
vipr init --quiet --preset strict
vipr analyze src/**/*.tsx -r overview -f json
```

### 2. Quick Analysis

For quick analysis without saving:

```bash
vipr analyze -i
# Select your file
# Choose reports
# Select "Interactive Viewer"
# Navigate and exit when done
```

### 3. Export for Documentation

Generate markdown reports for docs:

```bash
vipr analyze -i
# Select your file
# Choose reports
# Select "Markdown" format
# Save to file: "docs/code-analysis.md"
```

### 4. Share Results

Copy results to share in Slack/Teams:

```bash
vipr analyze -i
# Select your file
# Choose reports
# Select "CLI" format
# Copy to clipboard
# Paste in your chat app
```

## Troubleshooting

### Clipboard Not Working

**Linux:**
Install xclip:

```bash
sudo apt-get install xclip
```

Or xsel:

```bash
sudo apt-get install xsel
```

**macOS/Windows:**
Clipboard should work out of the box. If not, use "Save to file" instead.

### Interactive Mode Not Starting

**Check TTY:**

```bash
# This should output a path
tty
```

If you see "not a tty", you're in a non-interactive environment (CI, redirected output, etc.). Use `--quiet` mode instead.

**Force non-interactive:**

```bash
vipr init --quiet --preset default
vipr analyze src/**/*.tsx -f cli
```

### Prompts Not Appearing

Make sure you're not in quiet mode:

```bash
# Wrong (suppresses prompts)
vipr analyze -i --quiet

# Correct
vipr analyze -i
```

### Navigation Controls Not Working

The interactive viewer uses menu selection, not keyboard shortcuts. Use arrow keys or numbers to select options, then press Enter.

## Advanced Usage

### Custom Configuration with Interactive

Choose custom preset for full control:

```bash
vipr init
# Select "Custom" preset
# Configure caching
# Set grade thresholds
# Set fail conditions
```

### Selective Report Analysis

Analyze only specific aspects:

```bash
vipr analyze -i
# Enter file path
# Select only: Security, Performance
# View interactively
```

### Batch Analysis with Interactive Selection

```bash
vipr analyze src/**/*.tsx -i
# Reports will be selected interactively
# But files are already specified
```

## Non-TTY Behavior

When running in non-interactive environments (CI, piped output, etc.), Vipr automatically falls back to non-interactive mode. This ensures your scripts work everywhere:

```bash
# Works in both interactive and non-interactive environments
vipr init
vipr analyze src/**/*.tsx

# Explicitly non-interactive (recommended for CI)
vipr init --quiet --preset strict
vipr analyze src/**/*.tsx -f json -o report.json
```

## Examples

### Example 1: Quick Setup and Analysis

```bash
# Setup configuration
vipr init

# Analyze your code
vipr analyze

# Follow the prompts to:
# 1. Enter file path: src/**/*.tsx
# 2. Select reports: Overview, Security, Performance
# 3. Choose format: Interactive Viewer
# 4. Navigate through results
```

### Example 2: Generate Documentation

```bash
# Run interactive analysis
vipr analyze -i

# Select comprehensive reports:
# - Overview
# - React Overview
# - Security
# - Performance
# - Accessibility

# Choose format: Markdown
# Save to: docs/code-quality-report.md
```

### Example 3: CI Integration

```bash
#!/bin/bash
# ci-analyze.sh

# Non-interactive init
vipr init --quiet --preset strict

# Run analysis
vipr analyze src/**/*.tsx -r overview,security -f json -o report.json

# Check exit code
if [ $? -ne 0 ]; then
  echo "Analysis failed quality thresholds"
  exit 1
fi
```

## Keyboard Shortcuts (Coming Soon)

Future releases will include direct keyboard shortcuts in the report viewer:

- `n` - Next section
- `p` - Previous section
- `j` - Jump to section
- `e` - Export
- `q` - Quit

Currently, use menu selection with Enter key.

## Getting Help

### Built-in Help

```bash
vipr --help
vipr init --help
vipr analyze --help
```

### Documentation

- [Vipr Documentation](https://vipr.dev/docs)
- [Configuration Guide](https://vipr.dev/docs/configuration)
- [CLI Reference](https://vipr.dev/docs/cli)

### Issues

Report bugs or request features at:
https://github.com/vipr/vipr/issues
