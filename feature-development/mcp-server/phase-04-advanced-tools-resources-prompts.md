# Phase 4: Advanced Tools, Resources, and Prompts

> **Note**: The style-guide MCP server has been removed. Style guide data is in `packages/ui/data/`. This document is retained for historical context.

## Goal

Add composition validation tools, MCP resources for browsable catalogs, prompt templates, and tiered response detail modes.

## Prerequisites

- Phase 3 completed: 8 core tools operational
- Database fully indexed
- Token budget utilities available

## Advanced Tools

### Tool 9: validate_composition

**Purpose**: Full composition audit checking component selection, dark mode, accessibility, and best practices.

**Input Schema**:

```typescript
{
  components: Array<{
    name: string;
    context: string;
    props?: Record<string, any>;
  }>;
  viewType?: string;
  checkDarkMode?: boolean;     // default true
  checkAccessibility?: boolean; // default true
}
```

**Output Schema**:

```typescript
{
  valid: boolean;
  score: number; // 0-100
  issues: Array<{
    type: 'error' | 'warning' | 'info';
    component?: string;
    category: string; // component-choice, dark-mode, accessibility, best-practice
    message: string;
    suggestion?: string;
  }>;
  summary: {
    totalComponents: number;
    errors: number;
    warnings: number;
    darkModeCompliance: number; // percentage
    accessibilityScore: number; // 0-100
  }
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/validate-composition.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createValidateCompositionTool(db: Database.Database) {
  const inputSchema = z.object({
    components: z.array(
      z.object({
        name: z.string(),
        context: z.string(),
        props: z.record(z.any()).optional(),
      })
    ),
    viewType: z.string().optional(),
    checkDarkMode: z.boolean().default(true),
    checkAccessibility: z.boolean().default(true),
  });

  const outputSchema = z.object({
    valid: z.boolean(),
    score: z.number(),
    issues: z.array(
      z.object({
        type: z.enum(['error', 'warning', 'info']),
        component: z.string().optional(),
        category: z.string(),
        message: z.string(),
        suggestion: z.string().optional(),
      })
    ),
    summary: z.object({
      totalComponents: z.number(),
      errors: z.number(),
      warnings: z.number(),
      darkModeCompliance: z.number(),
      accessibilityScore: z.number(),
    }),
  });

  return {
    name: 'validate_composition',
    description:
      'Perform full composition audit for component selection, dark mode, and accessibility',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const issues: any[] = [];
      let darkModeSupported = 0;
      let accessibilityScore = 100;

      // Check each component
      for (const comp of args.components) {
        const row = db.prepare('SELECT * FROM components WHERE name = ?').get(comp.name) as any;

        if (!row) {
          issues.push({
            type: 'error',
            component: comp.name,
            category: 'component-choice',
            message: `Component not found: ${comp.name}`,
            suggestion: 'Use search_components to find valid components',
          });
          continue;
        }

        // Check anti-patterns
        const avoidWhen: string[] = row.avoid_when ? JSON.parse(row.avoid_when) : [];
        for (const antiPattern of avoidWhen) {
          if (comp.context.toLowerCase().includes(antiPattern.toLowerCase())) {
            issues.push({
              type: 'warning',
              component: comp.name,
              category: 'component-choice',
              message: `Component ${comp.name} may not be suitable: ${antiPattern}`,
              suggestion: 'Consider alternative components for this use case',
            });
          }
        }

        // Check dark mode support
        if (args.checkDarkMode) {
          if (row.dark_mode_support === 1) {
            darkModeSupported++;
          } else {
            issues.push({
              type: 'warning',
              component: comp.name,
              category: 'dark-mode',
              message: `Component ${comp.name} may not support dark mode`,
              suggestion: 'Verify dark mode styling or add custom dark mode classes',
            });
          }
        }

        // Check accessibility
        if (args.checkAccessibility) {
          if (!row.accessibility_notes) {
            issues.push({
              type: 'info',
              component: comp.name,
              category: 'accessibility',
              message: `No accessibility notes for ${comp.name}`,
              suggestion: 'Ensure proper ARIA labels and keyboard navigation',
            });
            accessibilityScore -= 10;
          }
        }
      }

      // Calculate scores
      const errors = issues.filter(i => i.type === 'error').length;
      const warnings = issues.filter(i => i.type === 'warning').length;
      const darkModeCompliance = (darkModeSupported / args.components.length) * 100;
      const score = Math.max(0, 100 - errors * 20 - warnings * 5);

      return {
        valid: errors === 0,
        score,
        issues,
        summary: {
          totalComponents: args.components.length,
          errors,
          warnings,
          darkModeCompliance: Math.round(darkModeCompliance),
          accessibilityScore: Math.max(0, accessibilityScore),
        },
      };
    },
  };
}
```

### Tool 10: check_dark_mode_compliance

**Purpose**: Verify dark mode coverage for a single component or composition.

**Input Schema**:

```typescript
{
  component?: string;        // Single component name
  components?: string[];     // Or list of components
}
```

**Output Schema**:

```typescript
{
  compliant: boolean;
  components: Array<{
    name: string;
    supported: boolean;
    notes?: string;
  }>;
  compliancePercentage: number;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/check-dark-mode-compliance.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createCheckDarkModeComplianceTool(db: Database.Database) {
  const inputSchema = z.object({
    component: z.string().optional(),
    components: z.array(z.string()).optional(),
  });

  const outputSchema = z.object({
    compliant: z.boolean(),
    components: z.array(
      z.object({
        name: z.string(),
        supported: z.boolean(),
        notes: z.string().optional(),
      })
    ),
    compliancePercentage: z.number(),
  });

  return {
    name: 'check_dark_mode_compliance',
    description: 'Check dark mode support for components',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const names = args.component ? [args.component] : args.components || [];

      if (names.length === 0) {
        throw new Error('Must provide either component or components');
      }

      const results: any[] = [];
      let supported = 0;

      for (const name of names) {
        const row = db
          .prepare('SELECT dark_mode_support, electron_notes FROM components WHERE name = ?')
          .get(name) as { dark_mode_support: number; electron_notes: string | null } | undefined;

        if (!row) {
          results.push({ name, supported: false, notes: 'Component not found' });
          continue;
        }

        const isSupported = row.dark_mode_support === 1;
        if (isSupported) supported++;

        results.push({
          name,
          supported: isSupported,
          notes: row.electron_notes ?? undefined,
        });
      }

      const percentage = (supported / names.length) * 100;

      return {
        compliant: supported === names.length,
        components: results,
        compliancePercentage: Math.round(percentage),
      };
    },
  };
}
```

### Tool 11: check_accessibility

**Purpose**: Accessibility checklist for components (ARIA, focus, touch targets, contrast).

**Input Schema**:

```typescript
{
  component: string;
  checks?: string[];        // Optional specific checks: aria, focus, touch, contrast
}
```

**Output Schema**:

```typescript
{
  component: string;
  accessible: boolean;
  checks: Array<{
    category: string;
    passed: boolean;
    details: string;
    recommendation?: string;
  }>;
  score: number; // 0-100
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/check-accessibility.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

const ACCESSIBILITY_CHECKS = {
  aria: 'Proper ARIA labels and roles',
  focus: 'Keyboard focus indicators',
  touch: 'Adequate touch target size (min 44x44px)',
  contrast: 'WCAG AA color contrast ratios',
};

export function createCheckAccessibilityTool(db: Database.Database) {
  const inputSchema = z.object({
    component: z.string(),
    checks: z.array(z.string()).optional(),
  });

  const outputSchema = z.object({
    component: z.string(),
    accessible: z.boolean(),
    checks: z.array(
      z.object({
        category: z.string(),
        passed: z.boolean(),
        details: z.string(),
        recommendation: z.string().optional(),
      })
    ),
    score: z.number(),
  });

  return {
    name: 'check_accessibility',
    description:
      'Run accessibility checklist for a component (ARIA, focus, touch targets, contrast)',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const row = db
        .prepare('SELECT accessibility_notes FROM components WHERE name = ?')
        .get(args.component) as { accessibility_notes: string | null } | undefined;

      if (!row) {
        throw new Error(`Component not found: ${args.component}`);
      }

      const checksToRun = args.checks || Object.keys(ACCESSIBILITY_CHECKS);
      const results: any[] = [];

      for (const check of checksToRun) {
        const description = ACCESSIBILITY_CHECKS[check as keyof typeof ACCESSIBILITY_CHECKS];
        const passed = row.accessibility_notes?.includes(check) ?? false;

        results.push({
          category: check,
          passed,
          details: description,
          recommendation: passed ? undefined : `Ensure ${description.toLowerCase()}`,
        });
      }

      const passedCount = results.filter(r => r.passed).length;
      const score = (passedCount / results.length) * 100;

      return {
        component: args.component,
        accessible: passedCount === results.length,
        checks: results,
        score: Math.round(score),
      };
    },
  };
}
```

### Tool 12: get_electron_guidance

**Purpose**: Desktop adaptation recommendations for Electron apps.

**Input Schema**:

```typescript
{
  component: string;
  platform?: string;       // darwin, win32, linux
}
```

**Output Schema**:

```typescript
{
  component: string;
  electronSupport: boolean;
  guidance: string;
  platformNotes?: Record<string, string>;
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/get-electron-guidance.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGetElectronGuidanceTool(db: Database.Database) {
  const inputSchema = z.object({
    component: z.string(),
    platform: z.enum(['darwin', 'win32', 'linux']).optional(),
  });

  const outputSchema = z.object({
    component: z.string(),
    electronSupport: z.boolean(),
    guidance: z.string(),
    platformNotes: z.record(z.string()).optional(),
  });

  return {
    name: 'get_electron_guidance',
    description: 'Get desktop adaptation recommendations for Electron apps',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      const row = db
        .prepare('SELECT electron_notes FROM components WHERE name = ?')
        .get(args.component) as { electron_notes: string | null } | undefined;

      if (!row) {
        throw new Error(`Component not found: ${args.component}`);
      }

      const hasGuidance = !!row.electron_notes;
      const guidance =
        row.electron_notes || 'No specific Electron guidance available for this component';

      return {
        component: args.component,
        electronSupport: hasGuidance,
        guidance,
        platformNotes: args.platform ? { [args.platform]: guidance } : undefined,
      };
    },
  };
}
```

### Tool 13: generate_composition_spec

**Purpose**: Synthesize a composition YAML specification from a view description.

**Input Schema**:

```typescript
{
  viewDescription: string;   // Natural language description
  viewType?: string;         // Optional view type hint
  includeLayout?: boolean;   // Include grid layout (default true)
}
```

**Output Schema**:

```typescript
{
  yaml: string;              // YAML composition spec
  components: string[];      // Suggested components
  explanation: string;       // Why these components were chosen
}
```

**Implementation**:

```typescript
// servers/analyzer/src/tools/generate-composition-spec.ts
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createGenerateCompositionSpecTool(db: Database.Database) {
  const inputSchema = z.object({
    viewDescription: z.string(),
    viewType: z.string().optional(),
    includeLayout: z.boolean().default(true),
  });

  const outputSchema = z.object({
    yaml: z.string(),
    components: z.array(z.string()),
    explanation: z.string(),
  });

  return {
    name: 'generate_composition_spec',
    description: 'Generate a composition YAML spec from a view description',
    inputSchema,
    outputSchema,
    handler: async (args: z.infer<typeof inputSchema>) => {
      // Simple keyword-based component suggestion (can be enhanced with embeddings)
      const keywords = args.viewDescription.toLowerCase().split(/\s+/);
      const componentNames: Set<string> = new Set();

      // Search for components matching keywords
      for (const keyword of keywords) {
        const results = db
          .prepare(`SELECT name FROM components_fts WHERE components_fts MATCH ? LIMIT 3`)
          .all(keyword) as Array<{ name: string }>;

        results.forEach(r => componentNames.add(r.name));
      }

      const components = Array.from(componentNames);

      // Generate YAML
      const yaml = `
view:
  type: ${args.viewType || 'custom'}
  description: ${args.viewDescription}

${
  args.includeLayout
    ? `layout:
  grid: "grid grid-cols-12 gap-4"
  responsive: true
`
    : ''
}
components:
${components.map(name => `  - name: ${name}`).join('\n')}
`.trim();

      return {
        yaml,
        components,
        explanation: `Suggested ${components.length} components based on keywords in the description`,
      };
    },
  };
}
```

## MCP Resources

MCP resources provide browsable catalogs and templates that clients can display in their UI.

### Resource 1-5: Static Catalogs

**File**: `servers/analyzer/src/resources/catalogs.ts`

```typescript
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createCatalogResources(db: Database.Database) {
  return [
    {
      uri: 'styleguide://catalog/components',
      name: 'Component Catalog',
      description: 'Browse all available components by category',
      mimeType: 'application/json',
      handler: async () => {
        const components = db
          .prepare('SELECT name, category, description FROM components ORDER BY category, name')
          .all();
        return JSON.stringify({ components }, null, 2);
      },
    },
    {
      uri: 'styleguide://catalog/patterns',
      name: 'Pattern Catalog',
      description: 'Browse all composition patterns',
      mimeType: 'application/json',
      handler: async () => {
        const patterns = db
          .prepare('SELECT name, group_name, description FROM patterns ORDER BY group_name, name')
          .all();
        return JSON.stringify({ patterns }, null, 2);
      },
    },
    {
      uri: 'styleguide://catalog/rules',
      name: 'UX Rules Catalog',
      description: 'Browse all UX guidelines and rules',
      mimeType: 'application/json',
      handler: async () => {
        const rules = db
          .prepare('SELECT title, category, rule_text FROM ux_rules ORDER BY category, title')
          .all();
        return JSON.stringify({ rules }, null, 2);
      },
    },
    {
      uri: 'styleguide://catalog/tokens',
      name: 'Design Tokens Catalog',
      description: 'Browse all design tokens by section',
      mimeType: 'application/json',
      handler: async () => {
        const tokens = db
          .prepare(
            'SELECT section, key, value, tailwind_class FROM design_tokens ORDER BY section, key'
          )
          .all();
        return JSON.stringify({ tokens }, null, 2);
      },
    },
    {
      uri: 'styleguide://catalog/electron',
      name: 'Electron Guidance Catalog',
      description: 'Browse desktop adaptation guidance',
      mimeType: 'application/json',
      handler: async () => {
        const guidance = db
          .prepare('SELECT name, electron_notes FROM components WHERE electron_notes IS NOT NULL')
          .all();
        return JSON.stringify({ guidance }, null, 2);
      },
    },
  ];
}
```

### Resource 6-7: Dynamic Templates

**File**: `servers/analyzer/src/resources/templates.ts`

```typescript
import { z } from 'zod';
import type Database from 'better-sqlite3';

export function createTemplateResources(db: Database.Database) {
  return [
    {
      uriTemplate: 'styleguide://components/{name}',
      name: 'Component Detail',
      description: 'Get detailed information for a specific component',
      mimeType: 'application/json',
      handler: async (name: string) => {
        const component = db.prepare('SELECT * FROM components WHERE name = ?').get(name);
        if (!component) {
          throw new Error(`Component not found: ${name}`);
        }
        return JSON.stringify(component, null, 2);
      },
    },
    {
      uriTemplate: 'styleguide://patterns/{slug}',
      name: 'Pattern Detail',
      description: 'Get detailed information for a specific pattern',
      mimeType: 'application/json',
      handler: async (slug: string) => {
        const pattern = db.prepare('SELECT * FROM patterns WHERE slug = ?').get(slug);
        if (!pattern) {
          throw new Error(`Pattern not found: ${slug}`);
        }
        return JSON.stringify(pattern, null, 2);
      },
    },
  ];
}
```

### Register Resources in Server

**File**: `servers/analyzer/src/server.ts`

```typescript
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { createCatalogResources } from './resources/catalogs.js';
import { createTemplateResources } from './resources/templates.js';

export async function startServer() {
  // ... existing setup ...

  const catalogResources = createCatalogResources(db);
  const templateResources = createTemplateResources(db);
  const allResources = [...catalogResources, ...templateResources];

  // Handle list_resources
  server.setRequestHandler(ListResourcesRequestSchema, async () => {
    return {
      resources: allResources.map(r => ({
        uri: r.uri || r.uriTemplate!,
        name: r.name,
        description: r.description,
        mimeType: r.mimeType,
      })),
    };
  });

  // Handle read_resource
  server.setRequestHandler(ReadResourceRequestSchema, async request => {
    const resource = allResources.find(r => {
      if (r.uri) {
        return r.uri === request.params.uri;
      }
      // Match URI template
      const template = r.uriTemplate!;
      const pattern = template.replace(/\{[^}]+\}/g, '([^/]+)');
      const match = new RegExp(`^${pattern}$`).exec(request.params.uri);
      return !!match;
    });

    if (!resource) {
      throw new Error(`Resource not found: ${request.params.uri}`);
    }

    // Extract template parameters
    let content: string;
    if (resource.uriTemplate) {
      const template = resource.uriTemplate;
      const pattern = template.replace(/\{([^}]+)\}/g, '(?<$1>[^/]+)');
      const match = new RegExp(`^${pattern}$`).exec(request.params.uri);
      const params = match?.groups || {};
      content = await resource.handler(Object.values(params)[0]);
    } else {
      content = await resource.handler();
    }

    return {
      contents: [
        {
          uri: request.params.uri,
          mimeType: resource.mimeType,
          text: content,
        },
      ],
    };
  });

  // ... rest of setup ...
}
```

## MCP Prompts

**File**: `servers/analyzer/src/prompts/templates.ts`

```typescript
export function createPromptTemplates() {
  return [
    {
      name: 'component-selection',
      description: 'Guide component selection for a specific view type',
      arguments: [
        {
          name: 'viewType',
          description: 'Type of view (dashboard, form, list, etc)',
          required: true,
        },
        { name: 'purpose', description: 'Purpose of the view', required: false },
      ],
      handler: (args: { viewType: string; purpose?: string }) => {
        return `
You are selecting components for a ${args.viewType} view${args.purpose ? ` with purpose: ${args.purpose}` : ''}.

Follow these steps:
1. Use the 'suggest_components' tool with viewType="${args.viewType}" to get ranked suggestions
2. For each suggested component, use 'get_component' to review detailed information
3. Use 'validate_component_usage' to check each component against the context
4. Consider dark mode support using 'check_dark_mode_compliance'
5. Verify accessibility with 'check_accessibility'

Select components that:
- Match the view type and purpose
- Have no anti-pattern violations for this use case
- Support dark mode (or can be adapted)
- Meet accessibility standards
- Follow the composition patterns for this view type
`.trim();
      },
    },
    {
      name: 'composition-review',
      description: 'Review a composition for compliance with style guide',
      arguments: [
        {
          name: 'components',
          description: 'Comma-separated list of component names',
          required: true,
        },
        { name: 'viewType', description: 'Type of view', required: false },
      ],
      handler: (args: { components: string; viewType?: string }) => {
        const componentList = args.components.split(',').map(c => c.trim());
        return `
You are reviewing a composition with components: ${componentList.join(', ')}${args.viewType ? ` for a ${args.viewType} view` : ''}.

Follow these steps:
1. Use 'validate_composition' with all components to get a comprehensive audit
2. Review each issue in the audit results:
   - Errors (blocking): Must be fixed before proceeding
   - Warnings (non-blocking): Should be addressed if possible
   - Info (suggestions): Consider for quality improvement
3. Check dark mode compliance percentage (target: 100%)
4. Check accessibility score (target: 90+)
5. If score < 80, suggest alternative components or improvements

Provide a summary with:
- Overall composition score
- Critical issues to fix
- Recommendations for improvement
- Alternative component suggestions if needed
`.trim();
      },
    },
    {
      name: 'validate-generated-code',
      description: 'Validate generated code against style guide rules',
      arguments: [
        { name: 'code', description: 'Generated code snippet', required: true },
        { name: 'viewType', description: 'Type of view', required: false },
      ],
      handler: (args: { code: string; viewType?: string }) => {
        return `
You are validating generated code${args.viewType ? ` for a ${args.viewType} view` : ''}.

Code to validate:
\`\`\`
${args.code}
\`\`\`

Follow these steps:
1. Extract all component names from the code
2. Use 'get_component' to verify each component exists and is used correctly
3. Check for proper prop usage (match component prop schemas)
4. Verify Tailwind classes match design tokens (use 'get_design_tokens')
5. Check composition patterns (use 'get_pattern' for the view type)
6. Run 'validate_composition' with all components and their contexts
7. Verify accessibility (proper ARIA labels, semantic HTML)
8. Check dark mode classes (use 'dark:' prefix consistently)

Report:
- ✅ Valid components and props
- ⚠️ Warnings (style/best practice issues)
- ❌ Errors (invalid components, missing props, broken patterns)
- Suggested fixes with corrected code snippets
`.trim();
      },
    },
  ];
}
```

### Register Prompts in Server

**File**: `servers/analyzer/src/server.ts`

```typescript
import {
  GetPromptRequestSchema,
  ListPromptsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { createPromptTemplates } from './prompts/templates.js';

export async function startServer() {
  // ... existing setup ...

  const prompts = createPromptTemplates();

  // Handle list_prompts
  server.setRequestHandler(ListPromptsRequestSchema, async () => {
    return {
      prompts: prompts.map(p => ({
        name: p.name,
        description: p.description,
        arguments: p.arguments,
      })),
    };
  });

  // Handle get_prompt
  server.setRequestHandler(GetPromptRequestSchema, async request => {
    const prompt = prompts.find(p => p.name === request.params.name);

    if (!prompt) {
      throw new Error(`Prompt not found: ${request.params.name}`);
    }

    const args = request.params.arguments || {};
    const message = prompt.handler(args);

    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: message,
          },
        },
      ],
    };
  });

  // ... rest of setup ...
}
```

## Tiered Response Detail

Add a `detail` parameter to tools that produce large output:

**File**: `servers/analyzer/src/types/index.ts`

```typescript
export type DetailLevel = 'compact' | 'standard' | 'full';

export interface TieredOutput<T> {
  compact: (data: T) => Partial<T>;
  standard: (data: T) => Partial<T>;
  full: (data: T) => T;
}
```

**Example**: Update `get_component` to support detail levels:

```typescript
// In get-component.ts
const inputSchema = z.object({
  name: z.string(),
  include: z.array(z.enum(['props', 'variants', 'useCases', 'avoidWhen', 'all'])).optional(),
  detail: z.enum(['compact', 'standard', 'full']).default('standard'),
});

// Handler logic:
if (args.detail === 'compact') {
  return {
    name: row.name,
    category: row.category,
    description: row.description.slice(0, 100) + '...',
  };
} else if (args.detail === 'standard') {
  // ... return standard fields ...
} else {
  // ... return all fields ...
}
```

## Verification

### Test 1: Validate composition

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"validate_composition","arguments":{"components":[{"name":"Button","context":"submit form"},{"name":"Input","context":"user email"}]}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Composition validation report with score, issues, and summary.

### Test 2: List resources

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"resources/list"}' | node servers/analyzer/dist/index.js
```

**Expected**: List of 7 resources (5 catalogs + 2 templates).

### Test 3: Read catalog resource

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"resources/read","params":{"uri":"styleguide://catalog/components"}}' | node servers/analyzer/dist/index.js
```

**Expected**: JSON catalog of all components.

### Test 4: Read template resource

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"resources/read","params":{"uri":"styleguide://components/Button"}}' | node servers/analyzer/dist/index.js
```

**Expected**: Full Button component JSON.

### Test 5: List prompts

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"prompts/list"}' | node servers/analyzer/dist/index.js
```

**Expected**: List of 3 prompts.

### Test 6: Get prompt

**Command**:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"prompts/get","params":{"name":"component-selection","arguments":{"viewType":"dashboard"}}}' | node servers/analyzer/dist/index.js
```

**Expected**: Prompt message with component selection guidance.

## Acceptance Criteria

- ✅ 5 advanced tools implemented (validate_composition, check_dark_mode_compliance, check_accessibility, get_electron_guidance, generate_composition_spec)
- ✅ 7 MCP resources available (5 static catalogs + 2 dynamic templates)
- ✅ 3 MCP prompts available (component-selection, composition-review, validate-generated-code)
- ✅ Tiered response detail mode (compact/standard/full) implemented
- ✅ Resources return browsable JSON catalogs
- ✅ Prompts generate well-structured instructions
- ✅ Composition validation catches anti-patterns and compliance issues
- ✅ All resources/prompts accessible via MCP protocol

## Next Phase

Proceed to [Phase 5: Pipeline Integration and Testing](phase-05-pipeline-integration-and-testing.md) to rewire UI generation agents and build test suite.

## Agent Consultation

- **mcp-server-architect**: Resource/prompt patterns, MCP protocol compliance
- **react-engineer**: Composition validation logic, accessibility checks
- **tailwind-ux-engineer**: Dark mode validation, layout patterns
- **typescript-engineer**: Type-safe resource/prompt handlers
