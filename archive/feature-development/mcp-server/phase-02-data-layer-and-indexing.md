# Phase 2: Data Layer and Indexing

> **Note**: The style-guide MCP server has been removed. Style guide data is in `packages/ui/data/` (e.g. component-catalog.json). This document is retained for historical context.

## Goal

Build SQLite persistence layer with FTS5 full-text search that indexes all style guide knowledge: components (~150), patterns (39), UX rules (15), design tokens (50+), and source code references.

## Prerequisites

- Phase 1 completed: Both servers built and running
- Style guide data files exist at `packages/ui/data/`
- `better-sqlite3` installed in `@vipr/mcp-analyzer`

## Database Schema

### Core Tables

#### components

Stores component metadata from `component-catalog.json`.

```sql
CREATE TABLE components (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  category TEXT NOT NULL,
  description TEXT NOT NULL,
  usage_hint TEXT,
  file_path TEXT,
  source_hash TEXT,
  props TEXT,               -- JSON array
  variants TEXT,            -- JSON array
  use_cases TEXT,           -- JSON array
  avoid_when TEXT,          -- JSON array
  dark_mode_support BOOLEAN DEFAULT 0,
  accessibility_notes TEXT,
  electron_notes TEXT,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_components_category ON components(category);
CREATE INDEX idx_components_source_hash ON components(source_hash);
```

#### patterns

Stores composition patterns from `patterns.md`.

```sql
CREATE TABLE patterns (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  slug TEXT NOT NULL UNIQUE,
  group_name TEXT NOT NULL,
  description TEXT NOT NULL,
  when_to_use TEXT,
  example_code TEXT,
  components_used TEXT,     -- JSON array
  grid_structure TEXT,
  responsive_notes TEXT,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_patterns_group ON patterns(group_name);
CREATE INDEX idx_patterns_slug ON patterns(slug);
```

#### ux_rules

Stores UX guidelines from `ux-rules.md`.

```sql
CREATE TABLE ux_rules (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  category TEXT NOT NULL,
  rule_text TEXT NOT NULL,
  rationale TEXT,
  examples TEXT,            -- JSON array
  related_components TEXT,  -- JSON array
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_ux_rules_category ON ux_rules(category);
```

#### design_tokens

Stores flattened design tokens (colors, spacing, typography, etc).

```sql
CREATE TABLE design_tokens (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  section TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  description TEXT,
  dark_mode_value TEXT,
  css_variable TEXT,
  tailwind_class TEXT,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_design_tokens_section_key ON design_tokens(section, key);
CREATE INDEX idx_design_tokens_section ON design_tokens(section);
```

#### composition_patterns

Stores common multi-component layouts.

```sql
CREATE TABLE composition_patterns (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  view_type TEXT NOT NULL,
  components TEXT NOT NULL,  -- JSON array
  grid_template TEXT,
  description TEXT,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_composition_patterns_view_type ON composition_patterns(view_type);
```

#### token_sections

Stores metadata about token categories.

```sql
CREATE TABLE token_sections (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  description TEXT,
  token_count INTEGER DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

#### source_files

Tracks content hashes of source data files for change detection.

```sql
CREATE TABLE source_files (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_path TEXT NOT NULL UNIQUE,
  content_hash TEXT NOT NULL,
  last_indexed TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_source_files_path ON source_files(file_path);
```

### FTS5 Virtual Tables

#### components_fts

Full-text search for components.

```sql
CREATE VIRTUAL TABLE components_fts USING fts5(
  name,
  category,
  description,
  usage_hint,
  content=components,
  content_rowid=id
);

-- Trigger to keep FTS in sync
CREATE TRIGGER components_ai AFTER INSERT ON components BEGIN
  INSERT INTO components_fts(rowid, name, category, description, usage_hint)
  VALUES (new.id, new.name, new.category, new.description, new.usage_hint);
END;

CREATE TRIGGER components_ad AFTER DELETE ON components BEGIN
  DELETE FROM components_fts WHERE rowid = old.id;
END;

CREATE TRIGGER components_au AFTER UPDATE ON components BEGIN
  UPDATE components_fts SET
    name = new.name,
    category = new.category,
    description = new.description,
    usage_hint = new.usage_hint
  WHERE rowid = new.id;
END;
```

#### patterns_fts

Full-text search for patterns.

```sql
CREATE VIRTUAL TABLE patterns_fts USING fts5(
  name,
  description,
  when_to_use,
  content=patterns,
  content_rowid=id
);

CREATE TRIGGER patterns_ai AFTER INSERT ON patterns BEGIN
  INSERT INTO patterns_fts(rowid, name, description, when_to_use)
  VALUES (new.id, new.name, new.description, new.when_to_use);
END;

CREATE TRIGGER patterns_ad AFTER DELETE ON patterns BEGIN
  DELETE FROM patterns_fts WHERE rowid = old.id;
END;

CREATE TRIGGER patterns_au AFTER UPDATE ON patterns BEGIN
  UPDATE patterns_fts SET
    name = new.name,
    description = new.description,
    when_to_use = new.when_to_use
  WHERE rowid = new.id;
END;
```

#### ux_rules_fts

Full-text search for UX rules.

```sql
CREATE VIRTUAL TABLE ux_rules_fts USING fts5(
  title,
  category,
  rule_text,
  content=ux_rules,
  content_rowid=id
);

CREATE TRIGGER ux_rules_ai AFTER INSERT ON ux_rules BEGIN
  INSERT INTO ux_rules_fts(rowid, title, category, rule_text)
  VALUES (new.id, new.title, new.category, new.rule_text);
END;

CREATE TRIGGER ux_rules_ad AFTER DELETE ON ux_rules BEGIN
  DELETE FROM ux_rules_fts WHERE rowid = old.id;
END;

CREATE TRIGGER ux_rules_au AFTER UPDATE ON ux_rules BEGIN
  UPDATE ux_rules_fts SET
    title = new.title,
    category = new.category,
    rule_text = new.rule_text
  WHERE rowid = new.id;
END;
```

## Implementation

### Step 1: Create database initialization module

**File**: `servers/analyzer/src/db/schema.ts`

```typescript
import type Database from 'better-sqlite3';

/**
 * Initialize database schema
 */
export function initializeSchema(db: Database.Database): void {
  // Create core tables
  db.exec(`
    CREATE TABLE IF NOT EXISTS components (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE,
      category TEXT NOT NULL,
      description TEXT NOT NULL,
      usage_hint TEXT,
      file_path TEXT,
      source_hash TEXT,
      props TEXT,
      variants TEXT,
      use_cases TEXT,
      avoid_when TEXT,
      dark_mode_support BOOLEAN DEFAULT 0,
      accessibility_notes TEXT,
      electron_notes TEXT,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
      updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_components_category ON components(category);
    CREATE INDEX IF NOT EXISTS idx_components_source_hash ON components(source_hash);

    CREATE TABLE IF NOT EXISTS patterns (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE,
      slug TEXT NOT NULL UNIQUE,
      group_name TEXT NOT NULL,
      description TEXT NOT NULL,
      when_to_use TEXT,
      example_code TEXT,
      components_used TEXT,
      grid_structure TEXT,
      responsive_notes TEXT,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
      updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_patterns_group ON patterns(group_name);
    CREATE INDEX IF NOT EXISTS idx_patterns_slug ON patterns(slug);

    CREATE TABLE IF NOT EXISTS ux_rules (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      category TEXT NOT NULL,
      rule_text TEXT NOT NULL,
      rationale TEXT,
      examples TEXT,
      related_components TEXT,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
      updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_ux_rules_category ON ux_rules(category);

    CREATE TABLE IF NOT EXISTS design_tokens (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      section TEXT NOT NULL,
      key TEXT NOT NULL,
      value TEXT NOT NULL,
      description TEXT,
      dark_mode_value TEXT,
      css_variable TEXT,
      tailwind_class TEXT,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
      updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE UNIQUE INDEX IF NOT EXISTS idx_design_tokens_section_key ON design_tokens(section, key);
    CREATE INDEX IF NOT EXISTS idx_design_tokens_section ON design_tokens(section);

    CREATE TABLE IF NOT EXISTS composition_patterns (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE,
      view_type TEXT NOT NULL,
      components TEXT NOT NULL,
      grid_template TEXT,
      description TEXT,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_composition_patterns_view_type ON composition_patterns(view_type);

    CREATE TABLE IF NOT EXISTS token_sections (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE,
      description TEXT,
      token_count INTEGER DEFAULT 0,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE IF NOT EXISTS source_files (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      file_path TEXT NOT NULL UNIQUE,
      content_hash TEXT NOT NULL,
      last_indexed TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX IF NOT EXISTS idx_source_files_path ON source_files(file_path);
  `);

  // Create FTS5 virtual tables
  db.exec(`
    CREATE VIRTUAL TABLE IF NOT EXISTS components_fts USING fts5(
      name,
      category,
      description,
      usage_hint,
      content=components,
      content_rowid=id
    );

    CREATE VIRTUAL TABLE IF NOT EXISTS patterns_fts USING fts5(
      name,
      description,
      when_to_use,
      content=patterns,
      content_rowid=id
    );

    CREATE VIRTUAL TABLE IF NOT EXISTS ux_rules_fts USING fts5(
      title,
      category,
      rule_text,
      content=ux_rules,
      content_rowid=id
    );
  `);

  // Create triggers for components_fts
  db.exec(`
    DROP TRIGGER IF EXISTS components_ai;
    CREATE TRIGGER components_ai AFTER INSERT ON components BEGIN
      INSERT INTO components_fts(rowid, name, category, description, usage_hint)
      VALUES (new.id, new.name, new.category, new.description, new.usage_hint);
    END;

    DROP TRIGGER IF EXISTS components_ad;
    CREATE TRIGGER components_ad AFTER DELETE ON components BEGIN
      DELETE FROM components_fts WHERE rowid = old.id;
    END;

    DROP TRIGGER IF EXISTS components_au;
    CREATE TRIGGER components_au AFTER UPDATE ON components BEGIN
      UPDATE components_fts SET
        name = new.name,
        category = new.category,
        description = new.description,
        usage_hint = new.usage_hint
      WHERE rowid = new.id;
    END;
  `);

  // Create triggers for patterns_fts
  db.exec(`
    DROP TRIGGER IF EXISTS patterns_ai;
    CREATE TRIGGER patterns_ai AFTER INSERT ON patterns BEGIN
      INSERT INTO patterns_fts(rowid, name, description, when_to_use)
      VALUES (new.id, new.name, new.description, new.when_to_use);
    END;

    DROP TRIGGER IF EXISTS patterns_ad;
    CREATE TRIGGER patterns_ad AFTER DELETE ON patterns BEGIN
      DELETE FROM patterns_fts WHERE rowid = old.id;
    END;

    DROP TRIGGER IF EXISTS patterns_au;
    CREATE TRIGGER patterns_au AFTER UPDATE ON patterns BEGIN
      UPDATE patterns_fts SET
        name = new.name,
        description = new.description,
        when_to_use = new.when_to_use
      WHERE rowid = new.id;
    END;
  `);

  // Create triggers for ux_rules_fts
  db.exec(`
    DROP TRIGGER IF EXISTS ux_rules_ai;
    CREATE TRIGGER ux_rules_ai AFTER INSERT ON ux_rules BEGIN
      INSERT INTO ux_rules_fts(rowid, title, category, rule_text)
      VALUES (new.id, new.title, new.category, new.rule_text);
    END;

    DROP TRIGGER IF EXISTS ux_rules_ad;
    CREATE TRIGGER ux_rules_ad AFTER DELETE ON ux_rules BEGIN
      DELETE FROM ux_rules_fts WHERE rowid = old.id;
    END;

    DROP TRIGGER IF EXISTS ux_rules_au;
    CREATE TRIGGER ux_rules_au AFTER UPDATE ON ux_rules BEGIN
      UPDATE ux_rules_fts SET
        title = new.title,
        category = new.category,
        rule_text = new.rule_text
      WHERE rowid = new.id;
    END;
  `);
}
```

**File**: `servers/analyzer/src/db/connection.ts`

```typescript
import Database from 'better-sqlite3';
import { initializeSchema } from './schema.js';

/**
 * Create and initialize database connection
 */
export function createDatabase(dbPath: string): Database.Database {
  const db = new Database(dbPath);

  // Enable WAL mode for better concurrency
  db.pragma('journal_mode = WAL');

  // Initialize schema
  initializeSchema(db);

  return db;
}

/**
 * Close database connection
 */
export function closeDatabase(db: Database.Database): void {
  db.close();
}
```

### Step 2: Create hash utility for change detection

**File**: `servers/analyzer/src/utils/hash.ts`

```typescript
import { createHash } from 'node:crypto';
import { readFile } from 'node:fs/promises';

/**
 * Calculate SHA-256 hash of file contents
 */
export async function hashFile(filePath: string): Promise<string> {
  const content = await readFile(filePath, 'utf-8');
  return hashString(content);
}

/**
 * Calculate SHA-256 hash of string
 */
export function hashString(content: string): Promise<string> {
  return createHash('sha256').update(content).digest('hex');
}

/**
 * Calculate hash of JSON object (deterministic)
 */
export function hashObject(obj: unknown): string {
  const normalized = JSON.stringify(obj, Object.keys(obj as object).sort());
  return createHash('sha256').update(normalized).digest('hex');
}
```

### Step 3: Create component indexer

**File**: `servers/analyzer/src/indexers/component-indexer.ts`

```typescript
import type Database from 'better-sqlite3';
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { hashFile, hashObject } from '../utils/hash.js';

interface ComponentEntry {
  name: string;
  category: string;
  description: string;
  usageHint?: string;
  filePath?: string;
  props?: Array<{ name: string; type: string; required: boolean }>;
  variants?: string[];
  useCases?: string[];
  avoidWhen?: string[];
  darkModeSupport?: boolean;
  accessibilityNotes?: string;
  electronNotes?: string;
}

interface ComponentCatalog {
  categories: Record<string, ComponentEntry[]>;
}

/**
 * Index component catalog into database
 */
export async function indexComponents(
  db: Database.Database,
  dataDir: string
): Promise<{ indexed: number; skipped: number }> {
  const catalogPath = join(dataDir, 'component-catalog.json');

  // Check if file has changed
  const currentHash = await hashFile(catalogPath);
  const existingHash = db
    .prepare('SELECT content_hash FROM source_files WHERE file_path = ?')
    .get(catalogPath) as { content_hash: string } | undefined;

  if (existingHash?.content_hash === currentHash) {
    const count = db.prepare('SELECT COUNT(*) as count FROM components').get() as {
      count: number;
    };
    return { indexed: 0, skipped: count.count };
  }

  // Load catalog
  const catalogJson = await readFile(catalogPath, 'utf-8');
  const catalog: ComponentCatalog = JSON.parse(catalogJson);

  // Clear existing components
  db.prepare('DELETE FROM components').run();

  // Insert components
  const insertStmt = db.prepare(`
    INSERT INTO components (
      name, category, description, usage_hint, file_path, source_hash,
      props, variants, use_cases, avoid_when,
      dark_mode_support, accessibility_notes, electron_notes
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `);

  let indexed = 0;
  const insertMany = db.transaction((components: ComponentEntry[], category: string) => {
    for (const component of components) {
      const componentHash = hashObject(component);
      insertStmt.run(
        component.name,
        category,
        component.description,
        component.usageHint ?? null,
        component.filePath ?? null,
        componentHash,
        component.props ? JSON.stringify(component.props) : null,
        component.variants ? JSON.stringify(component.variants) : null,
        component.useCases ? JSON.stringify(component.useCases) : null,
        component.avoidWhen ? JSON.stringify(component.avoidWhen) : null,
        component.darkModeSupport ? 1 : 0,
        component.accessibilityNotes ?? null,
        component.electronNotes ?? null
      );
      indexed++;
    }
  });

  for (const [category, components] of Object.entries(catalog.categories)) {
    insertMany(components, category);
  }

  // Update source file hash
  db.prepare(
    `INSERT OR REPLACE INTO source_files (file_path, content_hash, last_indexed)
     VALUES (?, ?, CURRENT_TIMESTAMP)`
  ).run(catalogPath, currentHash);

  return { indexed, skipped: 0 };
}
```

### Step 4: Create pattern indexer

**File**: `servers/analyzer/src/indexers/pattern-indexer.ts`

```typescript
import type Database from 'better-sqlite3';
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { hashFile } from '../utils/hash.js';

interface Pattern {
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

/**
 * Parse patterns from markdown file
 */
function parsePatterns(markdown: string): Pattern[] {
  const patterns: Pattern[] = [];
  const sections = markdown.split(/^##\s+/m).filter(Boolean);

  for (const section of sections) {
    const lines = section.split('\n');
    const name = lines[0].trim();
    const slug = name.toLowerCase().replace(/\s+/g, '-');

    // Extract description and other fields from markdown
    // This is a simplified parser - enhance as needed
    const description =
      lines
        .find(l => l.startsWith('**Description:**'))
        ?.replace('**Description:**', '')
        .trim() || name;
    const whenToUse = lines
      .find(l => l.startsWith('**When to use:**'))
      ?.replace('**When to use:**', '')
      .trim();

    patterns.push({
      name,
      slug,
      groupName: 'Composition Patterns', // Extract from file structure if needed
      description,
      whenToUse,
    });
  }

  return patterns;
}

/**
 * Index patterns into database
 */
export async function indexPatterns(
  db: Database.Database,
  dataDir: string
): Promise<{ indexed: number; skipped: number }> {
  const patternsPath = join(dataDir, 'patterns.md');

  // Check if file has changed
  const currentHash = await hashFile(patternsPath);
  const existingHash = db
    .prepare('SELECT content_hash FROM source_files WHERE file_path = ?')
    .get(patternsPath) as { content_hash: string } | undefined;

  if (existingHash?.content_hash === currentHash) {
    const count = db.prepare('SELECT COUNT(*) as count FROM patterns').get() as {
      count: number;
    };
    return { indexed: 0, skipped: count.count };
  }

  // Load and parse patterns
  const markdown = await readFile(patternsPath, 'utf-8');
  const patterns = parsePatterns(markdown);

  // Clear existing patterns
  db.prepare('DELETE FROM patterns').run();

  // Insert patterns
  const insertStmt = db.prepare(`
    INSERT INTO patterns (
      name, slug, group_name, description, when_to_use,
      example_code, components_used, grid_structure, responsive_notes
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
  `);

  const insertMany = db.transaction((patterns: Pattern[]) => {
    for (const pattern of patterns) {
      insertStmt.run(
        pattern.name,
        pattern.slug,
        pattern.groupName,
        pattern.description,
        pattern.whenToUse ?? null,
        pattern.exampleCode ?? null,
        pattern.componentsUsed ? JSON.stringify(pattern.componentsUsed) : null,
        pattern.gridStructure ?? null,
        pattern.responsiveNotes ?? null
      );
    }
  });

  insertMany(patterns);

  // Update source file hash
  db.prepare(
    `INSERT OR REPLACE INTO source_files (file_path, content_hash, last_indexed)
     VALUES (?, ?, CURRENT_TIMESTAMP)`
  ).run(patternsPath, currentHash);

  return { indexed: patterns.length, skipped: 0 };
}
```

### Step 5: Create rules indexer

**File**: `servers/analyzer/src/indexers/rules-indexer.ts`

```typescript
import type Database from 'better-sqlite3';
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';
import { hashFile } from '../utils/hash.js';

interface UxRule {
  title: string;
  category: string;
  ruleText: string;
  rationale?: string;
  examples?: string[];
  relatedComponents?: string[];
}

/**
 * Parse UX rules from markdown file
 */
function parseUxRules(markdown: string): UxRule[] {
  const rules: UxRule[] = [];
  const sections = markdown.split(/^##\s+/m).filter(Boolean);

  for (const section of sections) {
    const lines = section.split('\n');
    const title = lines[0].trim();

    // Extract category from title or context
    const category = 'General'; // Extract from markdown structure

    const ruleText = lines.find(l => !l.startsWith('#') && l.trim().length > 0)?.trim() || title;

    rules.push({
      title,
      category,
      ruleText,
    });
  }

  return rules;
}

/**
 * Index UX rules into database
 */
export async function indexUxRules(
  db: Database.Database,
  dataDir: string
): Promise<{ indexed: number; skipped: number }> {
  const rulesPath = join(dataDir, 'ux-rules.md');

  // Check if file has changed
  const currentHash = await hashFile(rulesPath);
  const existingHash = db
    .prepare('SELECT content_hash FROM source_files WHERE file_path = ?')
    .get(rulesPath) as { content_hash: string } | undefined;

  if (existingHash?.content_hash === currentHash) {
    const count = db.prepare('SELECT COUNT(*) as count FROM ux_rules').get() as {
      count: number;
    };
    return { indexed: 0, skipped: count.count };
  }

  // Load and parse rules
  const markdown = await readFile(rulesPath, 'utf-8');
  const rules = parseUxRules(markdown);

  // Clear existing rules
  db.prepare('DELETE FROM ux_rules').run();

  // Insert rules
  const insertStmt = db.prepare(`
    INSERT INTO ux_rules (
      title, category, rule_text, rationale, examples, related_components
    ) VALUES (?, ?, ?, ?, ?, ?)
  `);

  const insertMany = db.transaction((rules: UxRule[]) => {
    for (const rule of rules) {
      insertStmt.run(
        rule.title,
        rule.category,
        rule.ruleText,
        rule.rationale ?? null,
        rule.examples ? JSON.stringify(rule.examples) : null,
        rule.relatedComponents ? JSON.stringify(rule.relatedComponents) : null
      );
    }
  });

  insertMany(rules);

  // Update source file hash
  db.prepare(
    `INSERT OR REPLACE INTO source_files (file_path, content_hash, last_indexed)
     VALUES (?, ?, CURRENT_TIMESTAMP)`
  ).run(rulesPath, currentHash);

  return { indexed: rules.length, skipped: 0 };
}
```

### Step 6: Create token indexer

**File**: `servers/analyzer/src/indexers/token-indexer.ts`

```typescript
import type Database from 'better-sqlite3';

interface DesignToken {
  section: string;
  key: string;
  value: string;
  description?: string;
  darkModeValue?: string;
  cssVariable?: string;
  tailwindClass?: string;
}

/**
 * Hardcoded design tokens (extract from styleguide components or theme files)
 */
const DESIGN_TOKENS: DesignToken[] = [
  // Colors
  {
    section: 'colors',
    key: 'primary',
    value: '#3b82f6',
    cssVariable: '--color-primary',
    tailwindClass: 'bg-blue-500',
  },
  {
    section: 'colors',
    key: 'background',
    value: '#ffffff',
    darkModeValue: '#1e293b',
    cssVariable: '--color-bg',
    tailwindClass: 'bg-white dark:bg-slate-800',
  },
  // Spacing
  {
    section: 'spacing',
    key: 'xs',
    value: '0.25rem',
    cssVariable: '--space-xs',
    tailwindClass: 'space-x-1',
  },
  {
    section: 'spacing',
    key: 'sm',
    value: '0.5rem',
    cssVariable: '--space-sm',
    tailwindClass: 'space-x-2',
  },
  // Add more tokens as needed
];

/**
 * Index design tokens into database
 */
export async function indexDesignTokens(
  db: Database.Database
): Promise<{ indexed: number; skipped: number }> {
  // Clear existing tokens
  db.prepare('DELETE FROM design_tokens').run();
  db.prepare('DELETE FROM token_sections').run();

  // Insert tokens
  const insertTokenStmt = db.prepare(`
    INSERT INTO design_tokens (
      section, key, value, description, dark_mode_value, css_variable, tailwind_class
    ) VALUES (?, ?, ?, ?, ?, ?, ?)
  `);

  const insertMany = db.transaction((tokens: DesignToken[]) => {
    for (const token of tokens) {
      insertTokenStmt.run(
        token.section,
        token.key,
        token.value,
        token.description ?? null,
        token.darkModeValue ?? null,
        token.cssVariable ?? null,
        token.tailwindClass ?? null
      );
    }
  });

  insertMany(DESIGN_TOKENS);

  // Update token section counts
  const sections = [...new Set(DESIGN_TOKENS.map(t => t.section))];
  const insertSectionStmt = db.prepare(`
    INSERT INTO token_sections (name, token_count)
    VALUES (?, (SELECT COUNT(*) FROM design_tokens WHERE section = ?))
  `);

  for (const section of sections) {
    insertSectionStmt.run(section, section);
  }

  return { indexed: DESIGN_TOKENS.length, skipped: 0 };
}
```

### Step 7: Create source indexer (lightweight JSX analysis)

**File**: `servers/analyzer/src/indexers/source-indexer.ts`

```typescript
import type Database from 'better-sqlite3';
import { readdir, readFile } from 'node:fs/promises';
import { join, basename } from 'node:path';

/**
 * Lightweight JSX static analysis to extract component file paths
 */
async function findComponentFiles(styleguideSrcDir: string): Promise<Map<string, string>> {
  const componentMap = new Map<string, string>();
  const componentsDir = join(styleguideSrcDir, 'components');

  try {
    const files = await readdir(componentsDir, { recursive: true });

    for (const file of files) {
      if (file.endsWith('.tsx') || file.endsWith('.jsx')) {
        const componentName = basename(file, '.tsx').replace(/\.jsx$/, '');
        componentMap.set(componentName, join(componentsDir, file));
      }
    }
  } catch (error) {
    console.warn('Could not scan styleguide components directory:', error);
  }

  return componentMap;
}

/**
 * Update component file paths in database
 */
export async function indexComponentSources(
  db: Database.Database,
  styleguideSrcDir: string
): Promise<{ updated: number }> {
  const componentFiles = await findComponentFiles(styleguideSrcDir);

  const updateStmt = db.prepare('UPDATE components SET file_path = ? WHERE name = ?');

  let updated = 0;
  for (const [name, filePath] of componentFiles) {
    const result = updateStmt.run(filePath, name);
    if (result.changes > 0) {
      updated++;
    }
  }

  return { updated };
}
```

### Step 8: Create master indexer

**File**: `servers/analyzer/src/indexers/index-all.ts`

```typescript
#!/usr/bin/env node

import { createDatabase, closeDatabase } from '../db/connection.js';
import { indexComponents } from './component-indexer.js';
import { indexPatterns } from './pattern-indexer.js';
import { indexUxRules } from './rules-indexer.js';
import { indexDesignTokens } from './token-indexer.js';
import { indexComponentSources } from './source-indexer.js';
import {
  getCurrentDir,
  findMonorepoRoot,
  getStyleguideDataDir,
  getServerDataDir,
} from '../utils/paths.js';
import { join } from 'node:path';

async function main() {
  console.log('Starting indexing pipeline...\n');

  const serverDir = getCurrentDir(import.meta.url);
  const monorepoRoot = findMonorepoRoot(serverDir);
  const dataDir = getStyleguideDataDir(monorepoRoot);
  const dbPath = join(getServerDataDir(serverDir), 'styleguide.db');

  console.log(`Monorepo root: ${monorepoRoot}`);
  console.log(`Data directory: ${dataDir}`);
  console.log(`Database: ${dbPath}\n`);

  const db = createDatabase(dbPath);

  try {
    // Index components
    console.log('Indexing components...');
    const componentResult = await indexComponents(db, dataDir);
    console.log(`  Indexed: ${componentResult.indexed}, Skipped: ${componentResult.skipped}`);

    // Index patterns
    console.log('Indexing patterns...');
    const patternResult = await indexPatterns(db, dataDir);
    console.log(`  Indexed: ${patternResult.indexed}, Skipped: ${patternResult.skipped}`);

    // Index UX rules
    console.log('Indexing UX rules...');
    const rulesResult = await indexUxRules(db, dataDir);
    console.log(`  Indexed: ${rulesResult.indexed}, Skipped: ${rulesResult.skipped}`);

    // Index design tokens
    console.log('Indexing design tokens...');
    const tokensResult = await indexDesignTokens(db);
    console.log(`  Indexed: ${tokensResult.indexed}, Skipped: ${tokensResult.skipped}`);

    // Index component source files
    console.log('Indexing component sources...');
    const styleguideSrcDir = join(monorepoRoot, 'clients', 'styleguide', 'src');
    const sourcesResult = await indexComponentSources(db, styleguideSrcDir);
    console.log(`  Updated: ${sourcesResult.updated}`);

    console.log('\nIndexing complete!');
  } catch (error) {
    console.error('Indexing failed:', error);
    process.exit(1);
  } finally {
    closeDatabase(db);
  }
}

main();
```

### Step 9: Add auto-indexing to server startup

**File**: `servers/analyzer/src/server.ts`

**Modify** the `startServer()` function to call indexing on startup:

```typescript
import { indexComponents } from './indexers/component-indexer.js';
import { indexPatterns } from './indexers/pattern-indexer.js';
import { indexUxRules } from './indexers/rules-indexer.js';
import { indexDesignTokens } from './indexers/token-indexer.js';
import { createDatabase } from './db/connection.js';

export async function startServer() {
  // ... existing path resolution code ...

  const db = createDatabase(config.dbPath);

  // Auto-index on startup
  console.error('Checking for data updates...');
  try {
    const dataDir = getStyleguideDataDir(monorepoRoot);
    await indexComponents(db, dataDir);
    await indexPatterns(db, dataDir);
    await indexUxRules(db, dataDir);
    await indexDesignTokens(db);
  } catch (error) {
    console.error('Warning: Auto-indexing failed:', error);
  }

  // ... rest of server setup ...
}
```

## Verification

### Test 1: Build and index

**Commands**:

```bash
pnpm --filter @vipr/mcp-analyzer build
pnpm --filter @vipr/mcp-analyzer index
```

**Expected output**:

```
Starting indexing pipeline...

Monorepo root: /path/to/vipr
Data directory: /path/to/vipr/packages/ui/data
Database: /path/to/vipr/servers/analyzer/data/styleguide.db

Indexing components...
  Indexed: 152, Skipped: 0
Indexing patterns...
  Indexed: 39, Skipped: 0
Indexing UX rules...
  Indexed: 15, Skipped: 0
Indexing design tokens...
  Indexed: 50, Skipped: 0
Indexing component sources...
  Updated: 148

Indexing complete!
```

### Test 2: Verify database contents

**Command**:

```bash
sqlite3 servers/analyzer/data/styleguide.db "SELECT COUNT(*) FROM components;"
```

**Expected**: ~150

**Command**:

```bash
sqlite3 servers/analyzer/data/styleguide.db "SELECT name, category FROM components LIMIT 5;"
```

**Expected**: List of 5 components with names and categories

### Test 3: Test FTS5 search

**Command**:

```bash
sqlite3 servers/analyzer/data/styleguide.db "SELECT name, category FROM components_fts WHERE components_fts MATCH 'modal' LIMIT 5;"
```

**Expected**: Components containing "modal" in name, category, description, or usage_hint

### Test 4: Test change detection

**Command** (run indexer twice):

```bash
pnpm --filter @vipr/mcp-analyzer index
pnpm --filter @vipr/mcp-analyzer index
```

**Expected output** (second run):

```
Indexing components...
  Indexed: 0, Skipped: 152
Indexing patterns...
  Indexed: 0, Skipped: 39
...
```

All entries should be skipped on second run (no file changes).

### Test 5: Verify auto-indexing on server start

**Command**:

```bash
node servers/analyzer/dist/index.js
```

**Expected stderr output**:

```
Checking for data updates...
@vipr/mcp-analyzer v0.1.0 started
Data directory: /path/to/vipr/servers/analyzer/data
Monorepo root: /path/to/vipr
```

Server should check for updates on startup (no re-index if files unchanged).

## Troubleshooting

### SQLite FTS5 not available

**Symptom**: `no such module: fts5`

**Fix**: Ensure `better-sqlite3` is built with FTS5 support. Rebuild with `pnpm rebuild better-sqlite3`.

### Hash mismatch errors

**Symptom**: Indexer always re-indexes even when files unchanged

**Fix**: Check `hashFile()` implementation uses consistent encoding (UTF-8).

### JSON parsing errors

**Symptom**: Component indexer fails on malformed JSON

**Fix**: Validate `component-catalog.json` with `jq` or JSON linter before indexing.

## Acceptance Criteria

- ✅ Database schema created with all 7 core tables
- ✅ FTS5 virtual tables created with sync triggers
- ✅ Component indexer loads ~150 components
- ✅ Pattern indexer loads 39 patterns
- ✅ Rules indexer loads 15 UX rules
- ✅ Token indexer loads 50+ design tokens
- ✅ Source indexer updates component file paths
- ✅ Change detection prevents unnecessary re-indexing
- ✅ CLI indexing command works (`pnpm mcp:analyzer`)
- ✅ Auto-indexing runs on server start
- ✅ FTS5 search returns relevant results

## Next Phase

Proceed to [Phase 3: Core MCP Tools](phase-03-core-mcp-tools.md) to implement the 8 primary MCP tools.

## Agent Consultation

- **mcp-server-architect**: Database schema design, indexing pipeline architecture
- **typescript-engineer**: Type-safe database layer, async/await patterns
- **node-package-engineer**: better-sqlite3 configuration, build optimization
