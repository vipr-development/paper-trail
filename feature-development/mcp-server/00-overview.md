---
id: 00-overview
---

# MCP Server Implementation - Overview

## Purpose

This document provides a high-level overview of the MCP (Model Context Protocol) server implementation for the vipr monorepo. The implementation consists of:

1. **@vipr/mcp-analyzer** - MCP server for code analysis capabilities (in `servers/analyzer/`)

## Goals

- Expose analysis and tooling via MCP for IDE and agent integration
- Establish patterns for the analyzer MCP server in `servers/analyzer/`
- Style guide data remains in `packages/ui/data/` (component-catalog.json, design-tokens.json, etc.) for direct use; no separate MCP server for style guide

## Architecture Decisions

| Decision                | Choice                      | Rationale                                                        |
| ----------------------- | --------------------------- | ---------------------------------------------------------------- |
| **Server directory**    | `servers/` at repo root     | New pnpm workspace; clean separation from `clients/`             |
| **Data canonical home** | `packages/ui/data/`         | Styleguide owns metadata; MCP server depends on it               |
| **Phase count**         | 5 phases                    | Scaffolding → Data → Core Tools → Advanced → Integration         |
| **Persistence**         | SQLite + FTS5               | ~150 components don't need vector search; FTS5 handles full-text |
| **MCP SDK**             | `@modelcontextprotocol/sdk` | Official TypeScript SDK, v1.x stable                             |
| **Module system**       | ESM (`"type": "module"`)    | MCP SDK requires ESM; independent tsconfig                       |
| **Validation**          | Zod                         | MCP SDK's native schema system                                   |

## Implementation Phases

### Phase 1: Infrastructure and Scaffolding

**Duration estimate**: Foundation work

**Goal**: Establish workspace structure, scaffold both servers, consolidate data.

**Key deliverables**:

- Historical note: an earlier draft used a `servers/` workspace layout, which is now removed
- Scaffold `@vipr/mcp-analyzer` package in `servers/analyzer/`
- ESM configuration for MCP server
- Basic ping tool verification

**Outcome**: Server starts and responds to ping tool calls. Style guide data lives in `packages/ui/data/`.

### Phase 2–5 (historical)

Phases 2–5 described a separate style-guide MCP server with SQLite indexing, tools, resources, and prompts. That server has been removed. Style guide data is consumed directly from `packages/ui/data/` (e.g. `component-catalog.json`, `design-tokens.json`). The remaining MCP server is **@vipr/mcp-analyzer** in `servers/analyzer/`.

## Data Flow Architecture

```mermaid
graph TB
    subgraph "Data Sources"
        CJ[component-catalog.json]
        PM[patterns.md]
        UR[ux-rules.md]
        SRC[styleguide components/*.tsx]
    end

    subgraph "Indexing Pipeline"
        CI[component-indexer]
        PI[pattern-indexer]
        RI[rules-indexer]
        TI[token-indexer]
        SI[source-indexer]
    end

    subgraph "Persistence Layer"
        DB[(SQLite DB)]
        FTS[FTS5 Virtual Tables]
    end

    subgraph "MCP Server"
        TOOLS[8 Core Tools]
        ADV[5 Advanced Tools]
        RES[7 Resources]
        PROM[3 Prompts]
    end

    subgraph "Clients"
        CLAUDE[Claude Desktop]
        CURSOR[Cursor IDE]
        AGENTS[Pipeline Agents]
    end

    CJ --> CI --> DB
    PM --> PI --> DB
    UR --> RI --> DB
    SRC --> SI --> DB
    SRC --> TI --> DB

    DB --> FTS
    FTS --> TOOLS
    DB --> TOOLS
    TOOLS --> RES
    TOOLS --> PROM
    ADV --> DB

    RES --> CLAUDE
    TOOLS --> CURSOR
    PROM --> AGENTS
</antmlaid>

## File Structure

### New Files Created

```

servers/
└── analyzer/ # @vipr/mcp-analyzer
├── package.json
├── tsconfig.json
├── .eslintrc.json
├── .gitignore
└── src/
├── index.ts
└── server.ts

packages/ui/data/ # Style guide data (used directly; no MCP server)
├── component-catalog.json
├── design-tokens.json
└── (other catalog data)

documentation/docs/feature-development/mcp-server/
├── 00-overview.md # This file
├── phase-01-infrastructure-and-scaffolding.md
├── phase-02-data-layer-and-indexing.md
├── phase-03-core-mcp-tools.md
├── phase-04-advanced-tools-resources-prompts.md
└── phase-05-pipeline-integration-and-testing.md

```

### Modified Files

```

pnpm-workspace.yaml # Add servers/_ glob
package.json (root) # Add mcp:_ scripts
.claude/skills/styleguide/SKILL.md # Update data paths

```

## MCP Server Capabilities

### @vipr/mcp-analyzer

MCP server in `servers/analyzer/` for code analysis integration. Exposes tools for IDE and agent use (e.g. ping for verification). Style guide data is consumed directly from `packages/ui/data/` (component-catalog.json, design-tokens.json, etc.); no separate style-guide MCP server.

## Technology Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **MCP SDK** | `@modelcontextprotocol/sdk` | ^1.0.0 | Protocol implementation |
| **Database** | `better-sqlite3` | ^11.0.0 | SQLite persistence |
| **Validation** | `zod` | ^3.24.0 | Schema validation |
| **Module system** | ESM | - | MCP SDK requirement |
| **TypeScript** | ^5.7.0 | - | Type safety |
| **Test framework** | Vitest | ^2.1.0 | Unit/integration/E2E tests |

## Integration Points

### Client Configuration

To use the analyzer MCP server, point your MCP client at the built server entry:

- Start server: `pnpm mcp:analyzer` (runs `node servers/analyzer/dist/index.js`)

Configure your IDE or Claude Desktop to run that command for the vipr analyzer MCP server.

## Development Workflow

1. **Initial setup**: `pnpm install`
2. **Build server**: `turbo build --filter=@vipr/mcp-analyzer` (or build from root)
3. **Start server**: `pnpm mcp:analyzer` or `node servers/analyzer/dist/index.js` (stdio mode)

## Success Criteria

- ✅ Analyzer MCP server starts and responds to tool calls
- ✅ Style guide data remains in `packages/ui/data/` (component-catalog.json, design-tokens.json, etc.) for direct use

## Next Steps

Proceed to [Phase 1: Infrastructure and Scaffolding](phase-01-infrastructure-and-scaffolding.md) to begin implementation.

## References

- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [SQLite FTS5 Documentation](https://www.sqlite.org/fts5.html)
```
