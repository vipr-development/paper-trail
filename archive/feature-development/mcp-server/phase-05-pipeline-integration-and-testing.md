# Phase 5: Pipeline Integration and Testing

> **Note**: The style-guide MCP server has been removed. Style guide data is in `packages/ui/data/`. This document is retained for historical context.

## Goal

Rewire UI generation pipeline agents to use MCP tools instead of direct file reads, build comprehensive test suite (~25 test files), and create documentation with client configuration examples.

## Prerequisites

- Phase 4 completed: All 13 tools + 7 resources + 3 prompts operational
- MCP server fully functional
- Database indexed with complete style guide data

## Pipeline Agent Updates

### Agent 1: ui-gen-phase2-composer.md

**File**: `.claude/agents/ui-gen-phase2-composer.md`

**Current behavior**: Reads `component-catalog.json`, `patterns.md`, `ux-rules.md` directly from files.

**New behavior**: Use MCP tools for all style guide queries.

**Changes needed**:

1. **Update data paths** (from Phase 1):
   - Replace `.claude/skills/styleguide/component-catalog.json` with `packages/ui/data/component-catalog.json`
   - Replace `.claude/skills/styleguide/patterns.md` with `packages/ui/data/patterns.md`
   - Replace `.claude/skills/styleguide/ux-rules.md` with `packages/ui/data/ux-rules.md`

2. **Replace file reads with MCP tools**:

**Before** (direct file read):

```markdown
1. Read the component catalog from `packages/ui/data/component-catalog.json`
2. Filter components by category matching the view type
3. For each component, check if it's appropriate for the context
```

**After** (MCP tools):

```markdown
1. Use `suggest_components` tool with viewType and purpose to get ranked suggestions
2. For each suggested component, use `get_component` to review detailed information
3. Use `validate_component_usage` to check each component against the context
4. Use `search_components` for additional components if initial suggestions are insufficient
```

**Implementation notes**:

- Add prerequisite check: MCP server must be running
- Replace all file read instructions with MCP tool calls
- Use `search_components` instead of grep/text search
- Use `validate_component_usage` instead of manual anti-pattern checking

**Example diff**:

```diff
## Step 2: Component Selection

- Read component catalog from `packages/ui/data/component-catalog.json`
- For the view type, identify relevant component categories
- Select 3-5 primary components that match the purpose
+ Use the `suggest_components` MCP tool with viewType and purpose
+ Review suggestions using `get_component` to see full details
+ Validate selections with `validate_component_usage` against the context
```

### Agent 2: ui-gen-phase3-builder.md

**File**: `.claude/agents/ui-gen-phase3-builder.md`

**Current behavior**: Reads component JSX files directly, reads design token files.

**New behavior**: Use MCP tools for component details and tokens.

**Changes needed**:

1. **Replace component file reads**:

**Before**:

```markdown
1. Read the component file from `packages/ui/src/components/{Component}.tsx`
2. Extract props, variants, and usage examples from the file
```

**After**:

```markdown
1. Use `get_component` tool with `include: ['all']` to get complete details
2. Use `get_pattern` tool if the composition follows a known pattern
3. Use `get_design_tokens` to retrieve correct color/spacing/typography values
```

2. **Replace token queries**:

**Before**:

```markdown
1. Check `packages/ui/theme.ts` for color values
2. Use Tailwind classes matching the theme
```

**After**:

```markdown
1. Use `get_design_tokens` with section='colors' to get color tokens
2. Use the returned `tailwindClass` values directly
3. Verify dark mode support with `check_dark_mode_compliance`
```

**Example diff**:

```diff
## Step 3: Generate Component Code

- Read component from `packages/ui/src/components/{name}.tsx`
- Extract props interface from TypeScript definitions
- Review usage examples in the component file
+ Use `get_component` MCP tool with name and `include: ['props', 'variants', 'useCases']`
+ Use `get_design_tokens` with appropriate sections for colors, spacing, typography
+ Use `get_pattern` if implementing a known composition pattern
```

### Agent 3: ui-gen-phase4-reviewer.md

**File**: `.claude/agents/ui-gen-phase4-reviewer.md`

**Current behavior**: Reads files to audit composition, manually checks patterns.

**New behavior**: Use MCP validation tools.

**Changes needed**:

1. **Update data paths** (from Phase 1):
   - Same as phase 2 agent

2. **Replace manual validation with MCP tools**:

**Before**:

```markdown
1. Read component catalog to verify all components exist
2. Manually check for anti-pattern violations
3. Review dark mode classes in the generated code
4. Check accessibility attributes (ARIA labels, etc)
```

**After**:

```markdown
1. Use `validate_composition` with all components and their contexts
2. Use `check_dark_mode_compliance` to verify dark mode coverage
3. Use `check_accessibility` for each component
4. Use validation score (target: 80+) to determine if changes are needed
```

**Implementation notes**:

- Extract component list from generated code
- Map components to their usage contexts
- Use `validate_composition` as the primary audit tool
- Use detailed checks (dark mode, accessibility) for deeper validation

**Example diff**:

```diff
## Step 2: Composition Audit

- Read component catalog from `packages/ui/data/component-catalog.json`
- For each component in the generated code:
-   - Check if it exists in the catalog
-   - Check if context matches any `avoidWhen` patterns
-   - Verify dark mode support
-   - Check for accessibility notes
+ Extract all components from the generated code
+ Use `validate_composition` MCP tool with components and contexts
+ Review validation results:
+   - Score must be 80+ (target: 90+)
+   - Zero errors (blocking issues)
+   - Address warnings where feasible
+ Use `check_dark_mode_compliance` to verify 100% coverage
+ Use `check_accessibility` for critical interactive components
```

### Agent 4: generate-ui.md command

**File**: `.claude/commands/generate-ui.md`

**Changes needed**:

1. **Add MCP server prerequisite**:

```diff
## Prerequisites

+ - MCP server must be running: `pnpm mcp:analyzer`
+ - If not running, the pipeline will fall back to reading data files directly
  - Component catalog at `packages/ui/data/component-catalog.json`
  - Patterns at `packages/ui/data/patterns.md`
  - UX rules at `packages/ui/data/ux-rules.md`
```

2. **Add startup verification**:

```markdown
## Verification

Before starting the pipeline:

1. Check if MCP server is available by calling the `ping` tool
2. If ping succeeds, proceed with MCP tools
3. If ping fails, warn user and fall back to file reads (Phase 1 paths)
```

## Test Suite

### Test Organization

All tests use Vitest and follow the peer test file pattern per CLAUDE.md:

```
servers/analyzer/src/
├── db/
│   ├── schema.ts
│   ├── schema.test.ts          # Unit test for schema
│   ├── connection.ts
│   └── connection.test.ts      # Unit test for connection
├── indexers/
│   ├── component-indexer.ts
│   ├── component-indexer.test.ts
│   ├── pattern-indexer.ts
│   ├── pattern-indexer.test.ts
│   ├── rules-indexer.ts
│   ├── rules-indexer.test.ts
│   ├── token-indexer.ts
│   ├── token-indexer.test.ts
│   └── source-indexer.test.ts
├── tools/
│   ├── search-components.test.ts
│   ├── get-component.test.ts
│   ├── get-pattern.test.ts
│   ├── get-design-tokens.test.ts
│   ├── get-ux-rules.test.ts
│   ├── validate-component-usage.test.ts
│   ├── suggest-components.test.ts
│   ├── get-layout-recommendation.test.ts
│   ├── validate-composition.test.ts
│   ├── check-dark-mode-compliance.test.ts
│   ├── check-accessibility.test.ts
│   ├── get-electron-guidance.test.ts
│   └── generate-composition-spec.test.ts
├── resources/
│   └── resources.test.ts       # Integration test for all resources
├── prompts/
│   └── prompts.test.ts         # Integration test for all prompts
├── utils/
│   ├── hash.test.ts
│   ├── paths.test.ts
│   └── token-budget.test.ts
└── server.test.ts              # Integration test for full server
```

### Vitest Configuration

**File**: `servers/analyzer/vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['src/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['**/*.test.ts', '**/types/**', 'dist/**'],
    },
  },
});
```

### Example Test Files

**File**: `servers/analyzer/src/db/connection.test.ts`

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { mkdtempSync, rmSync } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import { createDatabase, closeDatabase } from './connection.js';

describe('Database Connection', () => {
  let tempDir: string;
  let dbPath: string;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'vipr-test-'));
    dbPath = join(tempDir, 'test.db');
  });

  afterEach(() => {
    rmSync(tempDir, { recursive: true, force: true });
  });

  it('should create database with schema', () => {
    const db = createDatabase(dbPath);

    // Check tables exist
    const tables = db.prepare(`SELECT name FROM sqlite_master WHERE type='table'`).all() as Array<{
      name: string;
    }>;

    const tableNames = tables.map(t => t.name);
    expect(tableNames).toContain('components');
    expect(tableNames).toContain('patterns');
    expect(tableNames).toContain('ux_rules');

    closeDatabase(db);
  });

  it('should enable WAL mode', () => {
    const db = createDatabase(dbPath);

    const result = db.pragma('journal_mode', { simple: true });
    expect(result).toBe('wal');

    closeDatabase(db);
  });
});
```

**File**: `servers/analyzer/src/tools/search-components.test.ts`

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { mkdtempSync, rmSync } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import { createDatabase, closeDatabase } from '../db/connection.js';
import { createSearchComponentsTool } from './search-components.js';

describe('search_components tool', () => {
  let tempDir: string;
  let db: any;
  let tool: any;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'vipr-test-'));
    const dbPath = join(tempDir, 'test.db');
    db = createDatabase(dbPath);

    // Insert test data
    db.prepare(
      `INSERT INTO components (name, category, description, usage_hint)
       VALUES (?, ?, ?, ?)`
    ).run('Button', 'Actions', 'A clickable button component', 'Use for primary actions');

    db.prepare(
      `INSERT INTO components (name, category, description, usage_hint)
       VALUES (?, ?, ?, ?)`
    ).run('Modal', 'Overlays', 'A dialog modal overlay', 'Use for important confirmations');

    tool = createSearchComponentsTool(db);
  });

  afterEach(() => {
    closeDatabase(db);
    rmSync(tempDir, { recursive: true, force: true });
  });

  it('should search components by query', async () => {
    const result = await tool.handler({ query: 'button', limit: 10 });

    expect(result.results).toHaveLength(1);
    expect(result.results[0].name).toBe('Button');
    expect(result.totalCount).toBe(1);
  });

  it('should filter by category', async () => {
    const result = await tool.handler({ query: 'button modal', category: 'Actions', limit: 10 });

    expect(result.results).toHaveLength(1);
    expect(result.results[0].category).toBe('Actions');
  });

  it('should respect limit', async () => {
    // Insert more test components
    for (let i = 0; i < 20; i++) {
      db.prepare(
        `INSERT INTO components (name, category, description)
         VALUES (?, ?, ?)`
      ).run(`Component${i}`, 'Test', 'test description action button');
    }

    const result = await tool.handler({ query: 'action', limit: 5 });

    expect(result.results.length).toBeLessThanOrEqual(5);
  });
});
```

**File**: `servers/analyzer/src/server.test.ts` (Integration)

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { spawn } from 'node:child_process';
import { join } from 'node:path';

describe('MCP Server Integration', () => {
  let serverProcess: any;
  let serverReady = false;

  beforeAll(async () => {
    // Start server as child process
    const serverPath = join(__dirname, '../dist/index.js');
    serverProcess = spawn('node', [serverPath], {
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    // Wait for server to be ready
    await new Promise<void>(resolve => {
      serverProcess.stderr.on('data', (data: Buffer) => {
        if (data.toString().includes('started')) {
          serverReady = true;
          resolve();
        }
      });
    });
  }, 10000);

  afterAll(() => {
    serverProcess?.kill();
  });

  it('should respond to ping tool', async () => {
    const request = JSON.stringify({
      jsonrpc: '2.0',
      id: 1,
      method: 'tools/call',
      params: { name: 'ping', arguments: {} },
    });

    serverProcess.stdin.write(request + '\n');

    const response = await new Promise<any>(resolve => {
      serverProcess.stdout.once('data', (data: Buffer) => {
        resolve(JSON.parse(data.toString()));
      });
    });

    expect(response.result.content[0].text).toContain('ok');
  });

  it('should list all tools', async () => {
    const request = JSON.stringify({
      jsonrpc: '2.0',
      id: 2,
      method: 'tools/list',
    });

    serverProcess.stdin.write(request + '\n');

    const response = await new Promise<any>(resolve => {
      serverProcess.stdout.once('data', (data: Buffer) => {
        resolve(JSON.parse(data.toString()));
      });
    });

    expect(response.result.tools.length).toBeGreaterThanOrEqual(9);
  });
});
```

### E2E Test Script

**File**: `servers/analyzer/test/e2e.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { spawn } from 'node:child_process';
import { join } from 'node:path';

describe('E2E: MCP Server', () => {
  const runServerCommand = (request: any): Promise<any> => {
    return new Promise((resolve, reject) => {
      const serverPath = join(__dirname, '../dist/index.js');
      const proc = spawn('node', [serverPath], {
        stdio: ['pipe', 'pipe', 'pipe'],
      });

      let output = '';

      proc.stdout.on('data', data => {
        output += data.toString();
      });

      proc.stderr.on('data', data => {
        // Ignore stderr (server logs)
      });

      proc.on('close', code => {
        if (code === 0) {
          try {
            const lines = output.trim().split('\n');
            const lastLine = lines[lines.length - 1];
            resolve(JSON.parse(lastLine));
          } catch (error) {
            reject(error);
          }
        } else {
          reject(new Error(`Process exited with code ${code}`));
        }
      });

      // Write request and close stdin
      proc.stdin.write(JSON.stringify(request) + '\n');
      proc.stdin.end();

      // Timeout after 5 seconds
      setTimeout(() => {
        proc.kill();
        reject(new Error('Timeout'));
      }, 5000);
    });
  };

  it('should handle full workflow: search -> get -> validate', async () => {
    // 1. Search for components
    const searchResult = await runServerCommand({
      jsonrpc: '2.0',
      id: 1,
      method: 'tools/call',
      params: { name: 'search_components', arguments: { query: 'button', limit: 5 } },
    });

    expect(searchResult.result.content[0].text).toContain('Button');

    // 2. Get component details
    const getResult = await runServerCommand({
      jsonrpc: '2.0',
      id: 2,
      method: 'tools/call',
      params: { name: 'get_component', arguments: { name: 'Button', include: ['all'] } },
    });

    expect(getResult.result.content[0].text).toContain('Button');

    // 3. Validate component usage
    const validateResult = await runServerCommand({
      jsonrpc: '2.0',
      id: 3,
      method: 'tools/call',
      params: {
        name: 'validate_component_usage',
        arguments: { component: 'Button', context: 'form submission' },
      },
    });

    expect(validateResult.result.content[0].text).toContain('valid');
  }, 30000);
});
```

### Test Execution

**Commands**:

```bash
# Run all tests
pnpm --filter @vipr/mcp-analyzer test

# Run with coverage
pnpm --filter @vipr/mcp-analyzer test --coverage

# Run in watch mode
pnpm --filter @vipr/mcp-analyzer test --watch

# Run E2E tests only
pnpm --filter @vipr/mcp-analyzer test test/e2e
```

## Documentation

### Server README

**File**: `servers/analyzer/README.md`

````markdown
# @vipr/mcp-analyzer

MCP server exposing vipr style guide knowledge for UI generation.

## Features

- **13 MCP Tools**: Search, retrieve, validate, and suggest components
- **7 Resources**: Browsable catalogs and component templates
- **3 Prompts**: Guided workflows for component selection, composition review, and code validation
- **SQLite + FTS5**: Fast full-text search across 150+ components, 39 patterns, 15 UX rules
- **Token Budget Management**: Automatic response truncation to fit context windows

## Installation

From the vipr monorepo root:

```bash
pnpm install
pnpm --filter @vipr/mcp-analyzer build
```
````

## Usage

### Start Server (stdio mode)

```bash
node servers/analyzer/dist/index.js
```

Or use the convenience script:

```bash
pnpm mcp:analyzer
```

### Index Data

First-time setup or after updating style guide data:

```bash
pnpm --filter @vipr/mcp-analyzer index
```

Or:

```bash
pnpm mcp:analyzer
```

The server auto-indexes on startup if data files have changed.

## Configuration

### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "vipr-analyzer": {
      "command": "node",
      "args": ["/absolute/path/to/vipr/servers/analyzer/dist/index.js"],
      "env": {}
    }
  }
}
```

Restart Claude Desktop after editing.

### Cursor IDE

Create `.cursor/mcp.json` in your project:

```json
{
  "mcpServers": {
    "vipr-analyzer": {
      "command": "node",
      "args": ["../../servers/analyzer/dist/index.js"]
    }
  }
}
```

Restart Cursor after editing.

### Environment Variables

The server uses the following paths:

- **Data directory**: `packages/ui/data/` (relative to monorepo root)
- **Database**: `servers/analyzer/data/styleguide.db` (auto-created)

No additional configuration required.

## Tools Reference

### Core Tools

| Tool                        | Purpose            | Input                      | Output                       |
| --------------------------- | ------------------ | -------------------------- | ---------------------------- |
| `search_components`         | Full-text search   | query, category?, limit?   | Array of matching components |
| `get_component`             | Component details  | name, include?             | Full component data          |
| `get_pattern`               | Pattern details    | name, group?               | Pattern with examples        |
| `get_design_tokens`         | Token query        | section, key?              | Token values and classes     |
| `get_ux_rules`              | Rule lookup        | category?, query?          | Array of rules               |
| `validate_component_usage`  | Anti-pattern check | component, context         | Validation result            |
| `suggest_components`        | Ranked suggestions | viewType, purpose?, limit? | Suggested components         |
| `get_layout_recommendation` | Grid/responsive    | viewType, contentSections  | Grid classes                 |

### Advanced Tools

| Tool                         | Purpose                 |
| ---------------------------- | ----------------------- |
| `validate_composition`       | Full composition audit  |
| `check_dark_mode_compliance` | Dark mode coverage      |
| `check_accessibility`        | Accessibility checklist |
| `get_electron_guidance`      | Desktop adaptation      |
| `generate_composition_spec`  | Synthesize YAML spec    |

## Resources

- `styleguide://catalog/components` - All components by category
- `styleguide://catalog/patterns` - All patterns by group
- `styleguide://catalog/rules` - All UX rules by category
- `styleguide://catalog/tokens` - All design tokens by section
- `styleguide://catalog/electron` - Desktop guidance
- `styleguide://components/{name}` - Single component (template)
- `styleguide://patterns/{slug}` - Single pattern (template)

## Prompts

- `component-selection` - Guide for selecting components for a view
- `composition-review` - Review composition for compliance
- `validate-generated-code` - Validate code against style guide

## Development

### Build

```bash
pnpm --filter @vipr/mcp-analyzer build
```

### Test

```bash
pnpm --filter @vipr/mcp-analyzer test
```

### Watch Mode

```bash
pnpm --filter @vipr/mcp-analyzer dev
```

## Troubleshooting

### Server won't start

**Check**: Node version >= 18, dependencies installed (`pnpm install`)

### Database not found

**Fix**: Run `pnpm mcp:analyzer` to create and populate the database

### FTS5 search errors

**Fix**: Rebuild better-sqlite3 with `pnpm rebuild better-sqlite3`

### MCP client can't connect

**Check**: Server path in client config is absolute, not relative

## License

MIT

````

## Verification

### Test 1: Run full test suite

**Command**:
```bash
pnpm --filter @vipr/mcp-analyzer test
````

**Expected**: All ~25 tests pass.

### Test 2: Run E2E test

**Command**:

```bash
pnpm --filter @vipr/mcp-analyzer test test/e2e
```

**Expected**: Full workflow test completes successfully.

### Test 3: Test agent rewiring

**Command**:

```bash
# Start MCP server
pnpm mcp:analyzer &

# Run /generate-ui command
# Verify agents use MCP tools instead of file reads
```

**Expected**: Agents call MCP tools (visible in server stderr logs).

### Test 4: Verify documentation

**Command**:

```bash
cat servers/analyzer/README.md
```

**Expected**: Complete README with setup, tools reference, client configs.

## Acceptance Criteria

- ✅ 3 pipeline agents updated to use MCP tools
- ✅ Data path references updated to `packages/ui/data/`
- ✅ `/generate-ui` command includes MCP prerequisites
- ✅ ~25 test files created (unit, integration, E2E)
- ✅ All tests pass with `pnpm test`
- ✅ E2E test validates full server workflow
- ✅ Server README complete with client configuration examples
- ✅ All 6 phase documents created in `documentation/docs/feature-development/mcp-server/`

## Implementation Complete

All 5 phases documented:

1. ✅ **Phase 1**: Infrastructure and scaffolding
2. ✅ **Phase 2**: Data layer and indexing
3. ✅ **Phase 3**: Core MCP tools
4. ✅ **Phase 4**: Advanced tools, resources, prompts
5. ✅ **Phase 5**: Pipeline integration and testing

## Next Steps

1. **Review phase documents**: Ensure all details are accurate and actionable
2. **Begin implementation**: Start with Phase 1, use `mcp-server-architect` agent
3. **Iterate through phases**: Complete each phase before moving to next
4. **Test incrementally**: Run verification steps at end of each phase
5. **Document learnings**: Update phase docs if implementation reveals issues

## Agent Consultation

- **vitest-engineer**: Test suite implementation, coverage configuration
- **mcp-server-architect**: Agent rewiring patterns, MCP integration
- **technical-writer**: README documentation, client configuration guides
- **typescript-engineer**: E2E test patterns, child process management
