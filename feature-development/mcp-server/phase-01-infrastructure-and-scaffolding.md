# Phase 1: Infrastructure and Scaffolding

> **Note**: This document is retained for historical context. The old `servers/` workspace approach has been removed from the active monorepo architecture.

## Goal

Establish the `servers/` workspace, scaffold both MCP server packages with ESM configuration, consolidate style guide data into a canonical location, and verify basic MCP server functionality.

## Prerequisites

- Node.js >= 18
- pnpm >= 8
- Existing vipr monorepo at `/Users/jamesleebaker/Codespace/vipr`

## Steps

### Step 1: Historical workspace step

**File**: `pnpm-workspace.yaml`

**Action**: This step is no longer applicable in the current architecture.

**Before**:

```yaml
packages:
  - 'analyzers/*'
  - 'clients/*'
  - 'packages/*'
```

**After**:

```yaml
packages:
  - 'analyzers/*'
  - 'clients/*'
  - 'packages/*'
```

### Step 2: Create servers directory structure

**Command**:

```bash
mkdir -p servers/analyzer/src/{types,db,indexers,tools,resources,prompts,utils}
mkdir -p servers/analyzer/src
```

### Step 3: Scaffold @vipr/mcp-analyzer package

**File**: `servers/analyzer/package.json`

```json
{
  "name": "@vipr/mcp-analyzer",
  "version": "0.1.0",
  "description": "MCP server exposing vipr style guide knowledge for UI generation",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "bin": {
    "vipr-mcp-analyzer": "./dist/index.js"
  },
  "files": ["dist", "data"],
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist data",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src --ext .ts",
    "test": "vitest",
    "index": "node dist/indexers/index-all.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "better-sqlite3": "^11.0.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.0",
    "@types/node": "^22.0.0",
    "@vipr/eslint-config": "workspace:*",
    "@vipr/tsconfig": "workspace:*",
    "eslint": "^9.0.0",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**File**: `servers/analyzer/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022"],
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

**Note**: This tsconfig does NOT extend `@vipr/tsconfig/node.json` because that config uses CJS (`"module": "CommonJS"`), but MCP SDK requires ESM. This server gets its own independent ESM configuration.

**File**: `servers/analyzer/.eslintrc.json`

```json
{
  "extends": ["@vipr/eslint-config"],
  "parserOptions": {
    "project": "./tsconfig.json"
  }
}
```

**File**: `servers/analyzer/.gitignore`

```
# Build outputs
dist/
*.tsbuildinfo

# Database
data/
*.db
*.db-shm
*.db-wal

# Dependencies
node_modules/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# Logs
*.log
npm-debug.log*

# OS
.DS_Store
Thumbs.db
```

### Step 4: Create initial server implementation with ping tool

**File**: `servers/analyzer/src/types/index.ts`

```typescript
import { z } from 'zod';

/**
 * MCP server configuration
 */
export interface ServerConfig {
  name: string;
  version: string;
  dataDir: string;
  dbPath: string;
}

/**
 * Ping tool response schema
 */
export const PingResponseSchema = z.object({
  status: z.literal('ok'),
  timestamp: z.string(),
  server: z.string(),
  version: z.string(),
});

export type PingResponse = z.infer<typeof PingResponseSchema>;
```

**File**: `servers/analyzer/src/utils/paths.ts`

```typescript
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { existsSync } from 'node:fs';

/**
 * Get the directory of the current module
 */
export function getCurrentDir(importMetaUrl: string): string {
  return dirname(fileURLToPath(importMetaUrl));
}

/**
 * Find the monorepo root by walking up from the current directory
 * until we find pnpm-workspace.yaml
 */
export function findMonorepoRoot(startDir: string): string {
  let current = startDir;
  const root = '/';

  while (current !== root) {
    if (existsSync(join(current, 'pnpm-workspace.yaml'))) {
      return current;
    }
    current = dirname(current);
  }

  throw new Error('Could not find monorepo root (pnpm-workspace.yaml not found)');
}

/**
 * Resolve path to styleguide data directory
 */
export function getStyleguideDataDir(monorepoRoot: string): string {
  return join(monorepoRoot, 'clients', 'styleguide', 'data');
}

/**
 * Resolve path to server data directory (for SQLite DB)
 */
export function getServerDataDir(serverDir: string): string {
  return join(serverDir, 'data');
}
```

**File**: `servers/analyzer/src/tools/ping.ts`

```typescript
import { z } from 'zod';
import type { ServerConfig } from '../types/index.js';
import { PingResponseSchema } from '../types/index.js';

/**
 * Ping tool - Verifies server is running and responsive
 */
export function createPingTool(config: ServerConfig) {
  return {
    name: 'ping',
    description: 'Check if the server is running and responsive',
    inputSchema: z.object({}),
    outputSchema: PingResponseSchema,
    handler: async () => {
      return {
        status: 'ok' as const,
        timestamp: new Date().toISOString(),
        server: config.name,
        version: config.version,
      };
    },
  };
}
```

**File**: `servers/analyzer/src/server.ts`

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import { mkdir } from 'node:fs/promises';
import { existsSync } from 'node:fs';
import { getCurrentDir, findMonorepoRoot, getServerDataDir } from './utils/paths.js';
import { createPingTool } from './tools/ping.js';
import type { ServerConfig } from './types/index.js';

/**
 * Initialize and start the MCP server
 */
export async function startServer() {
  // Resolve paths
  const serverDir = getCurrentDir(import.meta.url);
  const monorepoRoot = findMonorepoRoot(serverDir);
  const dataDir = getServerDataDir(serverDir);

  // Ensure data directory exists
  if (!existsSync(dataDir)) {
    await mkdir(dataDir, { recursive: true });
  }

  // Server configuration
  const config: ServerConfig = {
    name: '@vipr/mcp-analyzer',
    version: '0.1.0',
    dataDir,
    dbPath: `${dataDir}/styleguide.db`,
  };

  // Create MCP server
  const server = new Server(
    {
      name: config.name,
      version: config.version,
    },
    {
      capabilities: {
        tools: {},
      },
    }
  );

  // Register tools
  const tools = [createPingTool(config)];

  // Handle list_tools
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    return {
      tools: tools.map(tool => ({
        name: tool.name,
        description: tool.description,
        inputSchema: tool.inputSchema,
      })),
    };
  });

  // Handle call_tool
  server.setRequestHandler(CallToolRequestSchema, async request => {
    const tool = tools.find(t => t.name === request.params.name);

    if (!tool) {
      throw new Error(`Unknown tool: ${request.params.name}`);
    }

    try {
      const result = await tool.handler(request.params.arguments ?? {});
      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(result, null, 2),
          },
        ],
      };
    } catch (error) {
      throw new Error(
        `Tool ${request.params.name} failed: ${error instanceof Error ? error.message : String(error)}`
      );
    }
  });

  // Start server with stdio transport
  const transport = new StdioServerTransport();
  await server.connect(transport);

  console.error(`${config.name} v${config.version} started`);
  console.error(`Data directory: ${dataDir}`);
  console.error(`Monorepo root: ${monorepoRoot}`);
}
```

**File**: `servers/analyzer/src/index.ts`

```typescript
#!/usr/bin/env node

import { startServer } from './server.js';

// Start the MCP server
startServer().catch(error => {
  console.error('Fatal error starting server:', error);
  process.exit(1);
});
```

### Step 5: Scaffold @vipr/mcp-analyzer package

**File**: `servers/analyzer/package.json`

```json
{
  "name": "@vipr/mcp-analyzer",
  "version": "0.1.0",
  "description": "MCP server exposing vipr code analysis capabilities (placeholder)",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "bin": {
    "vipr-mcp-analyzer": "./dist/index.js"
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src --ext .ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "@vipr/eslint-config": "workspace:*",
    "@vipr/tsconfig": "workspace:*",
    "eslint": "^9.0.0",
    "typescript": "^5.7.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**File**: `servers/analyzer/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022"],
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**File**: `servers/analyzer/.eslintrc.json`

```json
{
  "extends": ["@vipr/eslint-config"],
  "parserOptions": {
    "project": "./tsconfig.json"
  }
}
```

**File**: `servers/analyzer/.gitignore`

```
# Build outputs
dist/
*.tsbuildinfo

# Dependencies
node_modules/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# Logs
*.log
npm-debug.log*

# OS
.DS_Store
Thumbs.db
```

**File**: `servers/analyzer/src/index.ts`

```typescript
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';

/**
 * Placeholder MCP server for future code analysis capabilities
 */
async function startServer() {
  const server = new Server(
    {
      name: '@vipr/mcp-analyzer',
      version: '0.1.0',
    },
    {
      capabilities: {
        tools: {},
      },
    }
  );

  // Ping tool
  const pingTool = {
    name: 'ping',
    description: 'Check if the analyzer server is running',
    inputSchema: z.object({}),
  };

  server.setRequestHandler(ListToolsRequestSchema, async () => {
    return {
      tools: [
        {
          name: pingTool.name,
          description: pingTool.description,
          inputSchema: pingTool.inputSchema,
        },
      ],
    };
  });

  server.setRequestHandler(CallToolRequestSchema, async request => {
    if (request.params.name === 'ping') {
      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(
              {
                status: 'ok',
                timestamp: new Date().toISOString(),
                server: '@vipr/mcp-analyzer',
                version: '0.1.0',
              },
              null,
              2
            ),
          },
        ],
      };
    }

    throw new Error(`Unknown tool: ${request.params.name}`);
  });

  const transport = new StdioServerTransport();
  await server.connect(transport);

  console.error('@vipr/mcp-analyzer v0.1.0 started (placeholder)');
}

startServer().catch(error => {
  console.error('Fatal error starting analyzer server:', error);
  process.exit(1);
});
```

### Step 6: Consolidate style guide data

**Action**: Move style guide data from `.claude/skills/styleguide/` to `packages/ui/data/`.

**Commands**:

```bash
# Create data directory
mkdir -p clients/styleguide/data

# Move files (if they exist in .claude/skills/styleguide/)
# These files may need to be created/extracted from existing skill files
# Check .claude/skills/styleguide/SKILL.md for embedded data
```

**Files to create/move**:

- `packages/ui/data/component-catalog.json`
- `packages/ui/data/patterns.md`
- `packages/ui/data/ux-rules.md`

**Note**: If these files don't exist yet, they need to be extracted from `.claude/skills/styleguide/SKILL.md` or created based on existing styleguide components.

### Step 7: Update skill references to new data paths

**File**: `.claude/skills/styleguide/SKILL.md`

**Action**: Update any embedded data or file path references to point to `packages/ui/data/` instead of local skill directory.

**Example changes**:

- `./component-catalog.json` → `../../packages/ui/data/component-catalog.json`
- Any hardcoded component data → Reference to data files

### Step 8: Update pipeline agent references

**File**: `.claude/agents/ui-gen-phase2-composer.md`

**Action**: Update data path references (full MCP tool integration comes in Phase 5).

**File**: `.claude/agents/ui-gen-phase4-reviewer.md`

**Action**: Update data path references (full MCP tool integration comes in Phase 5).

### Step 9: Add convenience scripts to root package.json

**File**: `package.json` (root)

**Action**: Add MCP-related scripts to the `scripts` section.

```json
{
  "scripts": {
    "mcp:build": "pnpm --filter '@vipr/mcp-*' build",
    "mcp:dev": "pnpm --filter '@vipr/mcp-*' dev",
    "mcp:clean": "pnpm --filter '@vipr/mcp-*' clean",
    "mcp:analyzer": "node servers/analyzer/dist/index.js"
  }
}
```

### Step 10: Install dependencies

**Commands**:

```bash
# From repo root
pnpm install
```

This installs:

- `@modelcontextprotocol/sdk` in both servers
- `better-sqlite3` and `zod` in analyzer (if used)
- `zod` in analyzer

### Step 11: Build both servers

**Commands**:

```bash
pnpm mcp:build
```

Or individually:

```bash
pnpm --filter @vipr/mcp-analyzer build
pnpm --filter @vipr/mcp-analyzer build
```

**Expected output**:

```
servers/analyzer/dist/
├── index.js
├── index.d.ts
├── server.js
├── server.d.ts
├── types/
├── utils/
└── tools/

servers/analyzer/dist/
├── index.js
└── index.d.ts
```

## Verification

### Test 1: Start analyzer server

**Command**:

```bash
node servers/analyzer/dist/index.js
```

**Expected stderr output**:

```
@vipr/mcp-analyzer v0.1.0 started
Data directory: /path/to/vipr/servers/analyzer/data
Monorepo root: /path/to/vipr
```

Server should be running and waiting for JSON-RPC input on stdin.

### Test 2: Call ping tool on analyzer

**Command** (while server is running):

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node servers/analyzer/dist/index.js
```

**Expected stdout output** (JSON-RPC response):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "ping",
        "description": "Check if the server is running and responsive",
        "inputSchema": {
          "type": "object",
          "properties": {},
          "required": []
        }
      }
    ]
  }
}
```

**Command** (call ping tool):

```bash
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ping","arguments":{}}}' | node servers/analyzer/dist/index.js
```

**Expected stdout output**:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\n  \"status\": \"ok\",\n  \"timestamp\": \"2025-01-15T10:30:00.000Z\",\n  \"server\": \"@vipr/mcp-analyzer\",\n  \"version\": \"0.1.0\"\n}"
      }
    ]
  }
}
```

### Test 3: Start analyzer server

**Command**:

```bash
node servers/analyzer/dist/index.js
```

**Expected stderr output**:

```
@vipr/mcp-analyzer v0.1.0 started (placeholder)
```

### Test 4: Call ping tool on analyzer

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ping","arguments":{}}}' | node servers/analyzer/dist/index.js
```

**Expected stdout output**:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\n  \"status\": \"ok\",\n  \"timestamp\": \"2025-01-15T10:31:00.000Z\",\n  \"server\": \"@vipr/mcp-analyzer\",\n  \"version\": \"0.1.0\"\n}"
      }
    ]
  }
}
```

### Test 5: Verify data directory creation

**Command**:

```bash
ls -la servers/analyzer/data/
```

**Expected output**:

```
drwxr-xr-x  2 user  staff   64 Jan 15 10:30 .
```

Directory should exist (created by server on first run).

## Troubleshooting

### ESM module errors

**Symptom**: `ERR_REQUIRE_ESM` or `Cannot use import statement outside a module`

**Fix**: Ensure `"type": "module"` is in `package.json` and tsconfig uses `"module": "ESNext"`.

### MCP SDK import errors

**Symptom**: Cannot find module `@modelcontextprotocol/sdk/server/index.js`

**Fix**: Check MCP SDK version is ^1.0.0 and imports use full subpath: `@modelcontextprotocol/sdk/server/index.js` (not `/server`).

### Path resolution errors

**Symptom**: Cannot find monorepo root

**Fix**: Verify `pnpm-workspace.yaml` exists at repo root. Check `findMonorepoRoot()` logic.

### TypeScript compilation errors

**Symptom**: Cannot compile ESM imports

**Fix**: Use `moduleResolution: "bundler"` (not "node") and ensure `.js` extensions in all imports.

## Acceptance Criteria

- ✅ Historical step documented (no longer part of current workspace wiring)
- ✅ Both servers have complete package.json with ESM configuration
- ✅ Both servers have independent tsconfig.json (not extending CJS config)
- ✅ `pnpm install` succeeds
- ✅ `pnpm mcp:build` compiles both servers without errors
- ✅ Analyzer server starts and responds to ping tool
- ✅ `analyzer` server starts and responds to ping tool
- ✅ Data directory is created at `servers/analyzer/data/`
- ✅ Style guide data moved to `packages/ui/data/`
- ✅ Skill and agent references updated to new data paths

## Next Phase

Proceed to [Phase 2: Data Layer and Indexing](phase-02-data-layer-and-indexing.md) to build SQLite persistence and FTS5 search indexing.

## Agent Consultation

- **turborepo-architect**: Workspace configuration, build pipeline integration
- **typescript-engineer**: ESM/CJS interop, tsconfig setup, import resolution
- **node-package-engineer**: pnpm workspace configuration, package.json dependencies
