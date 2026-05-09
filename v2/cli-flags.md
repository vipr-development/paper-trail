# CLI Flag Testing Guide

Comprehensive test commands for all Vipr CLI flags.

## Test File

All examples use this test file:

```bash
TEST_FILE="analyzers/react/src/fixtures/DataTable.tsx"
```

## Format Flags

### `-f, --format <type>`

Output format selection (cli, json, json-full)

```bash
# Default CLI format with colors and boxes
node clients/cli/dist/index.js analyze $TEST_FILE

# JSON format - pretty-printed with core metrics (cyclomatic, Halstead, LOC)
node clients/cli/dist/index.js analyze $TEST_FILE -f json

# JSON format - minified (no whitespace)
node clients/cli/dist/index.js analyze $TEST_FILE -f json --compact

# Full JSON format - complete data with all details and insights
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full

# Full JSON format - minified (no whitespace)
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full --compact
```

**Expected**: Different output formats with appropriate structure

**Note on JSON formats**:

- `-f json` includes core complexity metrics (cyclomatic complexity, Halstead measures, LOC)
- `-f json-full` includes complete detailed analysis data with all insights, locations, and analysis breakdowns
- Add `--compact` to any JSON format to minify output (removes whitespace):
  - Pretty: ~1.4KB (json), ~35KB (json-full)
  - Compact: ~790B (json), ~17KB (json-full)

**Verify core metrics are included**:

```bash
node clients/cli/dist/index.js analyze $TEST_FILE -f json -q | jq '.data.files[0].complexity'
```

Expected output:

```json
{
  "cyclomatic": 23,
  "decisionPoints": 22,
  "halstead": {
    "volume": 4086.27,
    "difficulty": 34.88,
    "effort": 142530.57,
    "bugs": 1.36,
    "time": 7918.37
  },
  "loc": {
    "total": 574,
    "operators": 203,
    "operands": 371
  }
}
```

## Output Flags

### `--compact`

Minify JSON output by removing whitespace (only works with JSON formats)

```bash
# Compact JSON (removes whitespace)
node clients/cli/dist/index.js analyze $TEST_FILE -f json --compact -q

# Compact full JSON (removes whitespace)
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full --compact -q

# Compare sizes
node clients/cli/dist/index.js analyze $TEST_FILE -f json -q | wc -c        # ~1400 bytes
node clients/cli/dist/index.js analyze $TEST_FILE -f json --compact -q | wc -c  # ~790 bytes

# Works with output files too
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full --compact -o output.json
```

**Expected**: Minified JSON with no whitespace or indentation

**Use cases**:

- Piping large JSON output to other tools
- Reducing network transfer size
- CI/CD pipelines where human readability isn't needed
- Storage optimization

**Note**: The `--compact` flag is ignored when using `-f cli` format.

### `-o, --output <file>`

Write output to file instead of stdout

```bash
# Write to file
node clients/cli/dist/index.js analyze $TEST_FILE -o /tmp/vipr-output.txt

# Verify file was created
cat /tmp/vipr-output.txt

# Write JSON to file
node clients/cli/dist/index.js analyze $TEST_FILE -f json -o /tmp/vipr-output.json

# Verify JSON structure
cat /tmp/vipr-output.json | jq '.format'
```

**Expected**: Output written to specified file, confirmation message shown

## Verbosity Flags

### `-q, --quiet`

Suppress all non-essential output (no logo, no progress messages)

```bash
# Quiet mode - only show final output
node clients/cli/dist/index.js analyze $TEST_FILE -q

# Combine with output file
node clients/cli/dist/index.js analyze $TEST_FILE -q -o /tmp/output.txt
```

**Expected**: No logo, no "Analyzing X file(s)..." or "Loaded X plugins" messages

### `--verbose`

Enable verbose diagnostic output

```bash
# Verbose mode
node clients/cli/dist/index.js analyze $TEST_FILE --verbose

# Verbose with timing
node clients/cli/dist/index.js analyze $TEST_FILE --verbose --timing
```

**Expected**: Shows debug-level messages like "Registered plugin: React Analyzer"

### `-d, --debug`

Enable debug logging

```bash
# Debug mode
node clients/cli/dist/index.js analyze $TEST_FILE -d

# Debug with specific report
node clients/cli/dist/index.js analyze $TEST_FILE -d -r security
```

**Expected**: Shows debug messages including:

- "Registered plugin: X"
- "Analyzing X with Y plugins"
- Plugin execution details

## Plugin Selection

### `-p, --plugins <list>`

Run only specified plugins (comma-separated)

```bash
# Run only React plugin
node clients/cli/dist/index.js analyze $TEST_FILE -p react -d

# Run only Core plugin
node clients/cli/dist/index.js analyze $TEST_FILE -p core -d

# Multiple plugins
node clients/cli/dist/index.js analyze $TEST_FILE -p react,core
```

**Expected**:

- `-p react`: Shows "Loaded 1 analyzer plugin(s)" with React-specific overview (includes core metrics)
- `-p core`: Shows "Loaded 1 analyzer plugin(s)" with Core overview (cyclomatic complexity, Halstead measures, LOC)
- Default: Shows "Loaded 2 analyzer plugin(s)" with React overview (React takes precedence, includes core metrics)

**Note**: When using `-p core`, the overview section displays core complexity metrics (cyclomatic complexity, Halstead measures, LOC) with bar charts. This is useful for analyzing framework-agnostic code complexity.

## Insight Filtering

### `-c, --categories <list>`

Filter insights by category (comma-separated)

```bash
# Show only hooks insights
node clients/cli/dist/index.js analyze $TEST_FILE -c hooks -q

# Show only performance insights
node clients/cli/dist/index.js analyze $TEST_FILE -c performance -q

# Multiple categories
node clients/cli/dist/index.js analyze $TEST_FILE -c hooks,performance -q

# With JSON output to see exact counts
node clients/cli/dist/index.js analyze $TEST_FILE -c hooks -f json-full -q | \
  jq '.data.files[0].insights | length'
```

**Expected**:

- No filter: "Top 10 Insights (out of 34)"
- `-c hooks`: "Top 10 Insights (out of 2)"
- `-c performance`: "Top 10 Insights (out of 13)"
- `-c hooks,performance`: "Top 10 Insights (out of 15)"

**Available categories**:

- `hooks` - Hook usage patterns
- `performance` - Performance optimizations
- `accessibility` - Accessibility issues
- `security` - Security vulnerabilities
- `reliability` - Code reliability
- `state-management` - State management patterns
- `dataflow` - Data flow analysis
- `structural` - Structural complexity
- `temporal` - Temporal coupling
- `identity` - Identity stability
- `type-complexity` - TypeScript complexity
- `technical-debt` - Technical debt

### `-s, --min-severity <level>`

Filter by minimum severity level (info, warning, critical)

```bash
# Show only critical insights
node clients/cli/dist/index.js analyze $TEST_FILE -s critical -f json-full -q | \
  jq '.data.files[0].insights[].severity' | sort -u

# Show warning and above
node clients/cli/dist/index.js analyze $TEST_FILE -s warning -q

# Show all (default)
node clients/cli/dist/index.js analyze $TEST_FILE -s info -q
```

**Expected**: Filters insights based on severity threshold

## Exit Code Control

### `-t, --fail-threshold <score>`

Exit with error if score falls below threshold (0-100)

```bash
# Low threshold - should pass
node clients/cli/dist/index.js analyze $TEST_FILE -t 50
echo "Exit code: $?"

# High threshold - should fail
node clients/cli/dist/index.js analyze $TEST_FILE -t 95
echo "Exit code: $?"
```

**Expected**:

- Score above threshold: Exit code 0
- Score below threshold: Exit code 1, warning message shown

### `--fail-on-critical`

Exit with error if any critical severity insights found

```bash
# Check for critical insights
node clients/cli/dist/index.js analyze $TEST_FILE --fail-on-critical
echo "Exit code: $?"

# Combine with quiet mode
node clients/cli/dist/index.js analyze $TEST_FILE --fail-on-critical -q
echo "Exit code: $?"
```

**Expected**: Exit code 1 if critical insights exist, 0 otherwise

## Performance Flags

### `--no-cache`

Disable result caching (in-memory only, effective for multi-file analysis)

**Note**: The cache is in-memory and only persists within a single CLI invocation. It helps when analyzing multiple files in one run, not between separate CLI runs.

```bash
# Analyze multiple files WITH cache (faster for duplicate analysis patterns)
time node clients/cli/dist/index.js analyze analyzers/react/src/fixtures/*.tsx --timing -q

# Analyze multiple files WITHOUT cache (slower, re-runs all analyses)
time node clients/cli/dist/index.js analyze analyzers/react/src/fixtures/*.tsx --no-cache --timing -q

# View cache statistics in debug mode
node clients/cli/dist/index.js analyze analyzers/react/src/fixtures/*.tsx -d 2>&1 | grep "cache hits"
```

**Expected**:

- With cache: May show cache hits for repeated analysis patterns
- Without cache: Shows "0 cache hits", slightly slower for large file batches

### `--timing`

Show execution timing

```bash
# With timing
node clients/cli/dist/index.js analyze $TEST_FILE --timing

# Timing with quiet mode
node clients/cli/dist/index.js analyze $TEST_FILE --timing -q
```

**Expected**: Shows "Analysis completed in Xms" message

## Report Selection

### `-r, --report <type>`

Display specific report type

```bash
# Overview report (default)
node clients/cli/dist/index.js analyze $TEST_FILE -r overview

# Security report
node clients/cli/dist/index.js analyze $TEST_FILE -r security

# Accessibility report
node clients/cli/dist/index.js analyze $TEST_FILE -r accessibility

# Performance report
node clients/cli/dist/index.js analyze $TEST_FILE -r performance

# Reliability report
node clients/cli/dist/index.js analyze $TEST_FILE -r reliability

# Migration report
node clients/cli/dist/index.js analyze $TEST_FILE -r migration

# Dataflow report
node clients/cli/dist/index.js analyze $TEST_FILE -r dataflow

# Anti-patterns report
node clients/cli/dist/index.js analyze $TEST_FILE -r anti-patterns

# Invalid report type (error handling)
node clients/cli/dist/index.js analyze $TEST_FILE -r invalid
```

**Expected**:

- Valid types: Shows only that specific report
- Invalid type: Error message listing available report types

## Combined Flag Tests

### Quiet + JSON Output

```bash
# Perfect for CI pipelines
node clients/cli/dist/index.js analyze $TEST_FILE -q -f json > output.json
cat output.json | jq '.format'
```

### Debug + Specific Plugin + Category

```bash
# Detailed debugging of specific analysis
node clients/cli/dist/index.js analyze $TEST_FILE -d -p react -c hooks
```

### Quiet + Fail Threshold + JSON

```bash
# CI check with machine-readable output
node clients/cli/dist/index.js analyze $TEST_FILE -q -f json -t 70 -o results.json
echo "Exit code: $?"
```

### Multiple Categories + Specific Report

```bash
# Focus on specific concerns
node clients/cli/dist/index.js analyze $TEST_FILE -c security,performance -r security
```

### Full Debug Analysis

```bash
# Maximum diagnostics
node clients/cli/dist/index.js analyze $TEST_FILE -d --verbose --timing
```

## Batch Analysis

All flags work with multiple files:

```bash
# Analyze multiple files
node clients/cli/dist/index.js analyze \
  analyzers/react/src/fixtures/*.tsx \
  -f json -q

# With filtering
node clients/cli/dist/index.js analyze \
  analyzers/react/src/fixtures/*.tsx \
  -c performance,security \
  -t 60

# Full batch with output
node clients/cli/dist/index.js analyze \
  analyzers/react/src/fixtures/*.tsx \
  -f json-full \
  -o batch-results.json \
  -q
```

## Help

### `-h, --help`

Display help information

```bash
# General help
node clients/cli/dist/index.js --help

# Analyze command help
node clients/cli/dist/index.js analyze --help
```

**Expected**: Displays usage information and all available flags

## Verification Checklist

Run this comprehensive test suite:

```bash
# 1. Format flags
echo "=== Format Flags ==="
node clients/cli/dist/index.js analyze $TEST_FILE -f cli -q | head -1
node clients/cli/dist/index.js analyze $TEST_FILE -f json -q | jq '.format'
node clients/cli/dist/index.js analyze $TEST_FILE -f json --compact -q | jq '.format'
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full -q | jq '.format'
node clients/cli/dist/index.js analyze $TEST_FILE -f json-full --compact -q | jq '.format'

# 2. Core metrics in JSON
echo "=== Core Metrics ==="
node clients/cli/dist/index.js analyze $TEST_FILE -f json -q | \
  jq '.data.files[0].complexity | keys'

# 3. Verbosity
echo "=== Quiet (should show no logs) ==="
node clients/cli/dist/index.js analyze $TEST_FILE -q 2>&1 | grep -i "loaded\|analyzing" && echo "FAIL" || echo "PASS"

echo "=== Debug (should show debug logs) ==="
node clients/cli/dist/index.js analyze $TEST_FILE -d 2>&1 | grep -i "registered plugin" && echo "PASS" || echo "FAIL"

# 4. Plugin filtering
echo "=== All plugins (2) ==="
node clients/cli/dist/index.js analyze $TEST_FILE -d 2>&1 | grep "Loaded.*plugin(s)"

echo "=== React only (1) ==="
node clients/cli/dist/index.js analyze $TEST_FILE -p react -d 2>&1 | grep "Loaded.*plugin(s)"

# 5. Category filtering
echo "=== No filter ==="
node clients/cli/dist/index.js analyze $TEST_FILE -q 2>&1 | grep "Top 10 Insights"

echo "=== Hooks only ==="
node clients/cli/dist/index.js analyze $TEST_FILE -c hooks -q 2>&1 | grep "Top 10 Insights"

echo "=== Performance only ==="
node clients/cli/dist/index.js analyze $TEST_FILE -c performance -q 2>&1 | grep "Top 10 Insights"

# 6. Exit codes
echo "=== Threshold pass ==="
node clients/cli/dist/index.js analyze $TEST_FILE -t 50 -q && echo "PASS (exit 0)" || echo "FAIL (exit 1)"

echo "=== Threshold fail ==="
node clients/cli/dist/index.js analyze $TEST_FILE -t 95 -q && echo "FAIL (exit 0)" || echo "PASS (exit 1)"

# 7. File size comparison
echo "=== File size comparison ==="
node clients/cli/dist/index.js analyze $TEST_FILE -f json -q -o /tmp/json-pretty.json
node clients/cli/dist/index.js analyze $TEST_FILE -f json --compact -q -o /tmp/json-compact.json
ls -lh /tmp/json-pretty.json /tmp/json-compact.json
echo "Compact should be ~50% smaller than pretty"
```

## Notes

- All flags can be combined as needed
- Short forms (`-f`, `-o`, etc.) and long forms (`--format`, `--output`) work identically
- Boolean flags (`-q`, `-d`, `--verbose`, `--compact`) don't take arguments
- List flags (`-p`, `-c`) accept comma-separated values with no spaces
- The CLI automatically detects the appropriate analyzer plugins based on file content
- Use `--compact` with JSON formats to minify output (removes whitespace)
