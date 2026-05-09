# Phase 3: Core MCP Tools

> **Note**: The style-guide MCP server has been removed. Style guide data is in `packages/ui/data/`. This document is retained for historical context.

## Goal

Implement 8 primary MCP tools that expose style guide knowledge through structured queries with token budget management.

## Prerequisites

- Phase 2 completed: Database indexed with ~150 components, 39 patterns, 15 rules
- SQLite database accessible at `servers/analyzer/data/styleguide.db`
- FTS5 search operational

## Tool Specifications

### Tool 1: search_components

**Purpose**: Full-text search across components with optional category filtering.

**Input Schema**:

```typescript
{
  query: string;          // Search query (FTS5 syntax)
  category?: string;      // Optional category filter
  limit?: number;         // Max results (default 10, max 50)
}
```

**Output Schema**:

```typescript
{
  results: Array<{
    name: string;
    category: string;
    description: string;
    usageHint?: string;
    rank: number; // FTS5 relevance score
  }>;
  totalCount: number;
  query: string;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/search-components.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createSearchComponentsTool(db: Database.Database) {
  const inputSchema = z.object({
    query: z.string().min(1).describe('Search query (FTS5 syntax)'),
    category: z.string().optional().describe('Filter by category'),
    limit: z.number().int().min(1).max(50).default(10).describe('Maximum results'),
  });

  const outputSchema = z.object({
    results: z.array(
      z.object({
        name: z.string(),
        category: z.string(),
        description: z.string(),
        usageHint: z.string().optional(),
        rank: z.number(),
      })
    ),
    totalCount: z.number(),
    query: z.string(),
  });

  return {
    name: 'search_components',
    description: 'Search components using full-text search with optional category filtering',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      let sql = `
        SELECT
          c.name, c.category, c.description, c.usage_hint,
          rank as rank
        FROM components_fts
        JOIN components c ON components_fts.rowid = c.id
        WHERE components_fts MATCH ?
      `;

      const params: any[] = [args.query];

      if (args.category) {
        sql += ' AND c.category = ?';
        params.push(args.category);
      }

      sql += ' ORDER BY rank LIMIT ?';
      params.push(args.limit);

      const results = db.prepare(sql).all(...params) as Array<{
        name: string;
        category: string;
        description: string;
        usage_hint: string | null;
        rank: number;
      }>;

      return {
        results: results.map(r => ({
          name: r.name,
          category: r.category,
          description: r.description,
          usageHint: r.usage_hint ?? undefined,
          rank: r.rank,
        })),
        totalCount: results.length,
        query: args.query,
      };
    },
  };
}
```

### Tool 2: get_component

**Purpose**: Retrieve full component details including props, variants, use cases, and anti-patterns.

**Input Schema**:

```typescript
{
  name: string;           // Component name
  include?: string[];     // Optional fields to include: props, variants, useCases, avoidWhen, all
}
```

**Output Schema**:

```typescript
{
  name: string;
  category: string;
  description: string;
  usageHint?: string;
  filePath?: string;
  props?: Array<{ name: string; type: string; required: boolean }>;
  variants?: string[];
  useCases?: string[];
  avoidWhen?: string[];
  darkModeSupport: boolean;
  accessibilityNotes?: string;
  electronNotes?: string;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-component.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGetComponentTool(db: Database.Database) {
  const inputSchema = z.object({
    name: z.string().describe('Component name'),
    include: z
      .array(z.enum(['props', 'variants', 'useCases', 'avoidWhen', 'all']))
      .optional()
      .describe('Fields to include'),
  });

  const outputSchema = z.object({
    name: z.string(),
    category: z.string(),
    description: z.string(),
    usageHint: z.string().optional(),
    filePath: z.string().optional(),
    props: z
      .array(z.object({ name: z.string(), type: z.string(), required: z.boolean() }))
      .optional(),
    variants: z.array(z.string()).optional(),
    useCases: z.array(z.string()).optional(),
    avoidWhen: z.array(z.string()).optional(),
    darkModeSupport: z.boolean(),
    accessibilityNotes: z.string().optional(),
    electronNotes: z.string().optional(),
  });

  return {
    name: 'get_component',
    description: 'Get full component details including props, variants, and usage guidelines',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const row = db.prepare('SELECT * FROM components WHERE name = ?').get(args.name) as any;

      if (!row) {
        throw new Error(`Component not found: ${args.name}`);
      }

      const includeAll = args.include?.includes('all');
      const result: any = {
        name: row.name,
        category: row.category,
        description: row.description,
        usageHint: row.usage_hint ?? undefined,
        filePath: row.file_path ?? undefined,
        darkModeSupport: row.dark_mode_support === 1,
        accessibilityNotes: row.accessibility_notes ?? undefined,
        electronNotes: row.electron_notes ?? undefined,
      };

      if (includeAll || args.include?.includes('props')) {
        result.props = row.props ? JSON.parse(row.props) : undefined;
      }
      if (includeAll || args.include?.includes('variants')) {
        result.variants = row.variants ? JSON.parse(row.variants) : undefined;
      }
      if (includeAll || args.include?.includes('useCases')) {
        result.useCases = row.use_cases ? JSON.parse(row.use_cases) : undefined;
      }
      if (includeAll || args.include?.includes('avoidWhen')) {
        result.avoidWhen = row.avoid_when ? JSON.parse(row.avoid_when) : undefined;
      }

      return result;
    },
  };
}
```

### Tool 3: get_pattern

**Purpose**: Retrieve pattern details with JSX examples and component composition.

**Input Schema**:

```typescript
{
  name: string;           // Pattern name or slug
  group?: string;         // Optional group filter
}
```

**Output Schema**:

```typescript
{
  name: string;
  slug: string;
  groupName: string;
  description: string;
  whenToUse?: string;
  exampleCode?: string;
  componentsUsed?: string[];
  gridStructure?: string;
  responsiveNotes?: string;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-pattern.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGetPatternTool(db: Database.Database) {
  const inputSchema = z.object({
    name: z.string().describe('Pattern name or slug'),
    group: z.string().optional().describe('Filter by pattern group'),
  });

  const outputSchema = z.object({
    name: z.string(),
    slug: z.string(),
    groupName: z.string(),
    description: z.string(),
    whenToUse: z.string().optional(),
    exampleCode: z.string().optional(),
    componentsUsed: z.array(z.string()).optional(),
    gridStructure: z.string().optional(),
    responsiveNotes: z.string().optional(),
  });

  return {
    name: 'get_pattern',
    description: 'Get pattern details with examples and composition guidelines',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      let sql = 'SELECT * FROM patterns WHERE name = ? OR slug = ?';
      const params: any[] = [args.name, args.name];

      if (args.group) {
        sql += ' AND group_name = ?';
        params.push(args.group);
      }

      const row = db.prepare(sql).get(...params) as any;

      if (!row) {
        throw new Error(`Pattern not found: ${args.name}`);
      }

      return {
        name: row.name,
        slug: row.slug,
        groupName: row.group_name,
        description: row.description,
        whenToUse: row.when_to_use ?? undefined,
        exampleCode: row.example_code ?? undefined,
        componentsUsed: row.components_used ? JSON.parse(row.components_used) : undefined,
        gridStructure: row.grid_structure ?? undefined,
        responsiveNotes: row.responsive_notes ?? undefined,
      };
    },
  };
}
```

### Tool 4: get_design_tokens

**Purpose**: Query design tokens by section (colors, spacing, typography, etc).

**Input Schema**:

```typescript
{
  section: string;        // Token section: colors, spacing, typography, etc.
  key?: string;           // Optional specific token key
}
```

**Output Schema**:

```typescript
{
  section: string;
  tokens: Array<{
    key: string;
    value: string;
    description?: string;
    darkModeValue?: string;
    cssVariable?: string;
    tailwindClass?: string;
  }>;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-design-tokens.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGetDesignTokensTool(db: Database.Database) {
  const inputSchema = z.object({
    section: z.string().describe('Token section (colors, spacing, typography, etc)'),
    key: z.string().optional().describe('Specific token key'),
  });

  const outputSchema = z.object({
    section: z.string(),
    tokens: z.array(
      z.object({
        key: z.string(),
        value: z.string(),
        description: z.string().optional(),
        darkModeValue: z.string().optional(),
        cssVariable: z.string().optional(),
        tailwindClass: z.string().optional(),
      })
    ),
  });

  return {
    name: 'get_design_tokens',
    description: 'Get design tokens by section (colors, spacing, typography, etc)',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      let sql = 'SELECT * FROM design_tokens WHERE section = ?';
      const params: any[] = [args.section];

      if (args.key) {
        sql += ' AND key = ?';
        params.push(args.key);
      }

      const rows = db.prepare(sql).all(...params) as any[];

      return {
        section: args.section,
        tokens: rows.map(row => ({
          key: row.key,
          value: row.value,
          description: row.description ?? undefined,
          darkModeValue: row.dark_mode_value ?? undefined,
          cssVariable: row.css_variable ?? undefined,
          tailwindClass: row.tailwind_class ?? undefined,
        })),
      };
    },
  };
}
```

### Tool 5: get_ux_rules

**Purpose**: Lookup UX rules by category or full-text search.

**Input Schema**:

```typescript
{
  category?: string;      // Filter by category
  query?: string;         // FTS5 search query
}
```

**Output Schema**:

```typescript
{
  rules: Array<{
    title: string;
    category: string;
    ruleText: string;
    rationale?: string;
    examples?: string[];
    relatedComponents?: string[];
  }>;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-ux-rules.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGetUxRulesTool(db: Database.Database) {
  const inputSchema = z.object({
    category: z.string().optional().describe('Filter by rule category'),
    query: z.string().optional().describe('Full-text search query'),
  });

  const outputSchema = z.object({
    rules: z.array(
      z.object({
        title: z.string(),
        category: z.string(),
        ruleText: z.string(),
        rationale: z.string().optional(),
        examples: z.array(z.string()).optional(),
        relatedComponents: z.array(z.string()).optional(),
      })
    ),
  });

  return {
    name: 'get_ux_rules',
    description: 'Get UX rules by category or search query',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      let rows: any[];

      if (args.query) {
        // FTS search
        let sql = `
          SELECT r.* FROM ux_rules_fts
          JOIN ux_rules r ON ux_rules_fts.rowid = r.id
          WHERE ux_rules_fts MATCH ?
        `;
        const params: any[] = [args.query];

        if (args.category) {
          sql += ' AND r.category = ?';
          params.push(args.category);
        }

        rows = db.prepare(sql).all(...params);
      } else if (args.category) {
        // Category filter only
        rows = db.prepare('SELECT * FROM ux_rules WHERE category = ?').all(args.category);
      } else {
        // All rules
        rows = db.prepare('SELECT * FROM ux_rules').all();
      }

      return {
        rules: rows.map(row => ({
          title: row.title,
          category: row.category,
          ruleText: row.rule_text,
          rationale: row.rationale ?? undefined,
          examples: row.examples ? JSON.parse(row.examples) : undefined,
          relatedComponents: row.related_components
            ? JSON.parse(row.related_components)
            : undefined,
        })),
      };
    },
  };
}
```

### Tool 6: validate_component_usage

**Purpose**: Check if component usage violates anti-patterns for a given context.

**Input Schema**:

```typescript
{
  component: string;      // Component name
  context: string;        // Usage context description
  viewType?: string;      // Optional view type (dashboard, form, etc)
}
```

**Output Schema**:

```typescript
{
  valid: boolean;
  component: string;
  violations: Array<{
    antiPattern: string;
    severity: 'error' | 'warning';
    suggestion?: string;
  }>;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/validate-component-usage.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createValidateComponentUsageTool(db: Database.Database) {
  const inputSchema = z.object({
    component: z.string().describe('Component name'),
    context: z.string().describe('Usage context description'),
    viewType: z.string().optional().describe('View type (dashboard, form, etc)'),
  });

  const outputSchema = z.object({
    valid: z.boolean(),
    component: z.string(),
    violations: z.array(
      z.object({
        antiPattern: z.string(),
        severity: z.enum(['error', 'warning']),
        suggestion: z.string().optional(),
      })
    ),
  });

  return {
    name: 'validate_component_usage',
    description: 'Validate component usage against anti-patterns',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const row = db
        .prepare('SELECT avoid_when FROM components WHERE name = ?')
        .get(args.component) as { avoid_when: string | null } | undefined;

      if (!row) {
        throw new Error(`Component not found: ${args.component}`);
      }

      const avoidWhen: string[] = row.avoid_when ? JSON.parse(row.avoid_when) : [];
      const violations: any[] = [];

      // Check if context matches any anti-patterns
      for (const antiPattern of avoidWhen) {
        const contextLower = args.context.toLowerCase();
        const patternLower = antiPattern.toLowerCase();

        // Simple keyword matching (can be enhanced with NLP)
        if (contextLower.includes(patternLower) || patternLower.includes(contextLower)) {
          violations.push({
            antiPattern,
            severity: 'warning',
            suggestion: `Consider using an alternative component for this use case`,
          });
        }
      }

      return {
        valid: violations.length === 0,
        component: args.component,
        violations,
      };
    },
  };
}
```

### Tool 7: suggest_components

**Purpose**: Suggest ranked components for a specific view type and purpose.

**Input Schema**:

```typescript
{
  viewType: string;       // View type: dashboard, form, list, detail, etc
  purpose?: string;       // Optional purpose description
  limit?: number;         // Max suggestions (default 5, max 15)
}
```

**Output Schema**:

```typescript
{
  viewType: string;
  suggestions: Array<{
    component: string;
    category: string;
    reason: string;
    priority: number; // 1-10
  }>;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/suggest-components.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

// Mapping of view types to recommended component categories and names
const VIEW_TYPE_RECOMMENDATIONS: Record<
  string,
  Array<{ category: string; priority: number; reason: string }>
> = {
  dashboard: [
    {
      category: 'Data Display',
      priority: 10,
      reason: 'Dashboards require charts, tables, and stats cards',
    },
    { category: 'Layout', priority: 9, reason: 'Grid layouts organize dashboard widgets' },
    {
      category: 'Navigation',
      priority: 7,
      reason: 'Side navigation or tabs for dashboard sections',
    },
  ],
  form: [
    { category: 'Inputs', priority: 10, reason: 'Forms are built with input components' },
    { category: 'Feedback', priority: 8, reason: 'Validation feedback and error messages' },
    { category: 'Layout', priority: 7, reason: 'Form layout and grouping' },
  ],
  list: [
    { category: 'Data Display', priority: 10, reason: 'Tables and lists display collections' },
    { category: 'Navigation', priority: 8, reason: 'Pagination and filtering' },
  ],
  detail: [
    { category: 'Data Display', priority: 9, reason: 'Display detailed item information' },
    { category: 'Actions', priority: 8, reason: 'Actions on the item (edit, delete, etc)' },
  ],
};

export function createSuggestComponentsTool(db: Database.Database) {
  const inputSchema = z.object({
    viewType: z.string().describe('View type (dashboard, form, list, detail, etc)'),
    purpose: z.string().optional().describe('Optional purpose description'),
    limit: z.number().int().min(1).max(15).default(5).describe('Max suggestions'),
  });

  const outputSchema = z.object({
    viewType: z.string(),
    suggestions: z.array(
      z.object({
        component: z.string(),
        category: z.string(),
        reason: z.string(),
        priority: z.number(),
      })
    ),
  });

  return {
    name: 'suggest_components',
    description: 'Suggest ranked components for a specific view type',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const recommendations = VIEW_TYPE_RECOMMENDATIONS[args.viewType.toLowerCase()] || [];
      const suggestions: any[] = [];

      for (const rec of recommendations) {
        const components = db
          .prepare('SELECT name, category, description FROM components WHERE category = ?')
          .all(rec.category) as Array<{ name: string; category: string; description: string }>;

        for (const comp of components) {
          suggestions.push({
            component: comp.name,
            category: comp.category,
            reason: rec.reason,
            priority: rec.priority,
          });
        }
      }

      // Sort by priority and limit
      suggestions.sort((a, b) => b.priority - a.priority);
      const limited = suggestions.slice(0, args.limit);

      return {
        viewType: args.viewType,
        suggestions: limited,
      };
    },
  };
}
```

### Tool 8: get_layout_recommendation

**Purpose**: Get Tailwind grid classes and responsive strategy for a view type.

**Input Schema**:

```typescript
{
  viewType: string;           // View type
  contentSections: string[];  // Content sections to layout
  responsive?: boolean;       // Include responsive breakpoints (default true)
}
```

**Output Schema**:

```typescript
{
  viewType: string;
  gridClasses: string;
  responsiveClasses?: Record<string, string>;  // Breakpoint -> classes
  explanation: string;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-layout-recommendation.ts
import { z } from 'zod';

const LAYOUT_TEMPLATES: Record<
  string,
  { grid: string; responsive: Record<string, string>; explanation: string }
> = {
  dashboard: {
    grid: 'grid grid-cols-12 gap-4',
    responsive: {
      sm: 'sm:grid-cols-1',
      md: 'md:grid-cols-2',
      lg: 'lg:grid-cols-3',
      xl: 'xl:grid-cols-4',
    },
    explanation: 'Dashboard uses 12-column grid with responsive breakpoints for widgets',
  },
  form: {
    grid: 'grid grid-cols-1 gap-6 md:grid-cols-2',
    responsive: {
      md: 'md:grid-cols-2',
    },
    explanation: 'Forms use single column on mobile, two columns on desktop',
  },
  list: {
    grid: 'flex flex-col gap-2',
    responsive: {},
    explanation: 'Lists use flex column with consistent spacing',
  },
};

export function createGetLayoutRecommendationTool() {
  const inputSchema = z.object({
    viewType: z.string().describe('View type'),
    contentSections: z.array(z.string()).describe('Content sections to layout'),
    responsive: z.boolean().default(true).describe('Include responsive breakpoints'),
  });

  const outputSchema = z.object({
    viewType: z.string(),
    gridClasses: z.string(),
    responsiveClasses: z.record(z.string()).optional(),
    explanation: z.string(),
  });

  return {
    name: 'get_layout_recommendation',
    description: 'Get Tailwind grid classes and responsive strategy for a view',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const template = LAYOUT_TEMPLATES[args.viewType.toLowerCase()];

      if (!template) {
        throw new Error(`No layout template for view type: ${args.viewType}`);
      }

      return {
        viewType: args.viewType,
        gridClasses: template.grid,
        responsiveClasses: args.responsive ? template.responsive : undefined,
        explanation: template.explanation,
      };
    },
  };
}
```

## Token Budget Utility

**File**: `servers/analyzer/src/utils/token-budget.ts`

```typescript
/**
 * Estimate token count for text (rough approximation: 1 token ≈ 4 chars)
 */
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

/**
 * Truncate text to fit within token budget
 */
export function truncateToTokenBudget(text: string, budget: number): string {
  const maxChars = budget * 4;
  if (text.length <= maxChars) {
    return text;
  }
  return text.slice(0, maxChars - 3) + '...';
}

/**
 * Truncate array of items to fit token budget (prioritize first items)
 */
export function truncateArrayToTokenBudget<T>(
  items: T[],
  budget: number,
  serializer: (item: T) => string
): T[] {
  const result: T[] = [];
  let usedTokens = 0;

  for (const item of items) {
    const serialized = serializer(item);
    const itemTokens = estimateTokens(serialized);

    if (usedTokens + itemTokens > budget) {
      break;
    }

    result.push(item);
    usedTokens += itemTokens;
  }

  return result;
}
```

## Tool Registration

**File**: `servers/analyzer/src/server.ts`

**Modify** to register all 8 tools:

```typescript
import { createSearchComponentsTool } from './tools/search-components.js';
import { createGetComponentTool } from './tools/get-component.js';
import { createGetPatternTool } from './tools/get-pattern.js';
import { createGetDesignTokensTool } from './tools/get-design-tokens.js';
import { createGetUxRulesTool } from './tools/get-ux-rules.js';
import { createValidateComponentUsageTool } from './tools/validate-component-usage.js';
import { createSuggestComponentsTool } from './tools/suggest-components.js';
import { createGetLayoutRecommendationTool } from './tools/get-layout-recommendation.js';

export async function startServer() {
  // ... existing setup ...

  const db = createDatabase(config.dbPath);

  // Register all tools
  const tools = [
    createPingTool(config),
    createSearchComponentsTool(db),
    createGetComponentTool(db),
    createGetPatternTool(db),
    createGetDesignTokensTool(db),
    createGetUxRulesTool(db),
    createValidateComponentUsageTool(db),
    createSuggestComponentsTool(db),
    createGetLayoutRecommendationTool(),
  ];

  // ... rest of server setup ...
}
```

## Verification

### Test 1: List all tools

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node servers/analyzer/dist/index.js
```

**Expected**: List of 9 tools (ping + 8 core tools).

### Test 2: Search components

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_components","arguments":{"query":"modal","limit":5}}}' | node servers/analyzer/dist/index.js
```

**Expected**: 5 components matching "modal".

### Test 3: Get component details

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_component","arguments":{"name":"Button","include":["all"]}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Full Button component details with props, variants, etc.

### Test 4: Get pattern

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_pattern","arguments":{"name":"master-detail"}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Master-detail pattern with example code.

### Test 5: Get design tokens

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_design_tokens","arguments":{"section":"colors"}}}' | node servers/analyzer/dist/index.js
```

**Expected**: All color tokens with values and CSS variables.

### Test 6: Validate component usage

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"validate_component_usage","arguments":{"component":"Modal","context":"displaying a small alert message"}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Validation warning if Modal is in avoidWhen for alerts.

### Test 7: Suggest components

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"suggest_components","arguments":{"viewType":"dashboard","limit":5}}}' | node servers/analyzer/dist/index.js
```

**Expected**: 5 component suggestions ranked by priority.

### Test 8: Get layout recommendation

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_layout_recommendation","arguments":{"viewType":"dashboard","contentSections":["stats","chart","table"]}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Grid classes and responsive breakpoints.

## Acceptance Criteria

- ✅ All 8 core tools implemented
- ✅ Each tool has Zod input/output schemas
- ✅ Tools query SQLite database correctly
- ✅ FTS5 search tools return relevant results
- ✅ Token budget utility prevents oversized responses
- ✅ Dependency injection pattern (db instance passed to tools)
- ✅ Tools return structured, type-safe data
- ✅ Error handling for missing components/patterns
- ✅ All tools respond correctly via JSON-RPC

## Next Phase

Proceed to [Phase 4: Advanced Tools, Resources, and Prompts](phase-04-advanced-tools-resources-prompts.md) to add composition validation, resources, and prompts.

## Agent Consultation

- **mcp-server-architect**: Tool implementation, MCP protocol patterns
- **tailwind-ux-engineer**: Layout recommendation logic, responsive strategies
- **react-engineer**: Component composition knowledge, usage validation
- **typescript-engineer**: Type-safe tool schemas, Zod validation
