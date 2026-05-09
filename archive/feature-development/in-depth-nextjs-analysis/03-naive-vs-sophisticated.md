---
id: 03-naive-vs-sophisticated
---

# Naive vs Sophisticated Static Analysis: Next.js Side-by-Side Comparisons

This document contrasts surface-level pattern matching with semantic analysis for Next.js anti-patterns across both Pages Router and App Router. Use this to evaluate existing implementations and identify improvement opportunities.

## Pattern 1: "use client" Directive Detection

### Naive Implementation

```typescript
// Detects: Any file using React hooks without "use client"
function detectMissingUseClient(sourceFile: ts.SourceFile) {
  const hasUseClient =
    sourceFile.getText().includes("'use client'") || sourceFile.getText().includes('"use client"');

  if (!hasUseClient) {
    ts.forEachChild(sourceFile, function visit(node) {
      if (ts.isCallExpression(node)) {
        const callee = node.expression.getText();
        if (['useState', 'useEffect', 'useRef'].includes(callee)) {
          report(node, 'missing-use-client');
        }
      }
      ts.forEachChild(node, visit);
    });
  }
}
```

**Problems**:

- Doesn't understand import chain inheritance
- Flags files that are imported by client files (unnecessary)
- Doesn't verify hooks are from React (could be custom)
- Text search is brittle

**False positives**:

```tsx
// components/counter-display.tsx
// ❌ Flagged, but this is FINE - imported by client file
export function CounterDisplay({ count }: { count: number }) {
  return <span>{count}</span>;
}

// components/counter.tsx
('use client');
import { CounterDisplay } from './counter-display';
// CounterDisplay becomes client through import chain
```

### Sophisticated Implementation

```typescript
interface UseClientAnalysis {
  needsDirective: boolean;
  confidence: 'high' | 'medium' | 'low';
  reason: string;
  clientAPIs: string[];
  importChain?: string[];
}

function analyzeUseClientNeed(sourceFile: SourceFile, project: Project): UseClientAnalysis {
  // 1. Check if already has directive
  if (hasUseClientDirective(sourceFile)) {
    return {
      needsDirective: false,
      confidence: 'high',
      reason: 'Already has "use client" directive',
      clientAPIs: [],
    };
  }

  // 2. Build import graph
  const importGraph = buildImportGraph(project);
  const filePath = sourceFile.getFilePath();

  // 3. Check if imported by a client file
  const clientImporter = findClientImporter(filePath, importGraph, new Set());
  if (clientImporter) {
    return {
      needsDirective: false,
      confidence: 'high',
      reason: 'Inherits client context through import chain',
      clientAPIs: [],
      importChain: getImportChain(filePath, clientImporter, importGraph),
    };
  }

  // 4. Check for client-only API usage
  const clientAPIs = findClientOnlyAPIUsage(sourceFile);

  if (clientAPIs.length === 0) {
    return {
      needsDirective: false,
      confidence: 'high',
      reason: 'No client-only APIs used',
      clientAPIs: [],
    };
  }

  // 5. Verify APIs are actually from React/browser
  const verifiedAPIs = clientAPIs.filter(api => {
    if (api.category === 'hook') {
      return verifyReactHookImport(sourceFile, api.api);
    }
    return true;
  });

  if (verifiedAPIs.length === 0) {
    return {
      needsDirective: false,
      confidence: 'medium',
      reason: 'Detected hooks may be custom implementations',
      clientAPIs: [],
    };
  }

  // 6. Check if this is an entry file (page/layout)
  const fileContext = determineFileContext(sourceFile);

  return {
    needsDirective: true,
    confidence: fileContext.isPage || fileContext.isLayout ? 'high' : 'medium',
    reason: `Uses client-only APIs: ${verifiedAPIs.map(a => a.api).join(', ')}`,
    clientAPIs: verifiedAPIs.map(a => a.api),
  };
}

function verifyReactHookImport(sourceFile: SourceFile, hookName: string): boolean {
  const imports = sourceFile.getImportDeclarations();

  for (const importDecl of imports) {
    const moduleSpecifier = importDecl.getModuleSpecifierValue();
    if (moduleSpecifier !== 'react') continue;

    const namedImports = importDecl.getNamedImports();
    for (const namedImport of namedImports) {
      if (namedImport.getName() === hookName) {
        return true;
      }
    }
  }

  // Check for React.useState style usage
  const reactImport = imports.find(
    i =>
      i.getDefaultImport()?.getText() === 'React' || i.getNamespaceImport()?.getText() === 'React'
  );

  return !!reactImport;
}
```

### Key Differences

| Aspect                   | Naive | Sophisticated                 |
| ------------------------ | ----- | ----------------------------- |
| Import chain analysis    | None  | Full graph traversal          |
| Hook source verification | None  | Verifies React import         |
| Context awareness        | None  | Understands page/layout roles |
| False positive rate      | High  | Low                           |
| Handles inheritance      | No    | Yes                           |

---

## Pattern 2: Server-Only Imports in Client Components

### Naive Implementation

```typescript
// Detects: Database imports in any file
function detectServerImportsInClient(sourceFile: ts.SourceFile) {
  const serverPackages = ['prisma', 'pg', 'mysql2', 'mongoose'];

  ts.forEachChild(sourceFile, function visit(node) {
    if (ts.isImportDeclaration(node)) {
      const moduleSpecifier = node.moduleSpecifier.getText().slice(1, -1);

      if (serverPackages.some(pkg => moduleSpecifier.includes(pkg))) {
        if (sourceFile.getText().includes("'use client'")) {
          report(node, 'server-import-in-client');
        }
      }
    }
    ts.forEachChild(node, visit);
  });
}
```

**Problems**:

- Hard-coded package list
- Doesn't detect `server-only` package marker
- Doesn't handle transitive imports
- Misses environment variable exposure

**False negatives**:

```tsx
'use client';

// ❌ Not detected - custom database wrapper
import { db } from '@/lib/database'; // This wraps Prisma internally!

// ❌ Not detected - environment secrets
const apiKey = process.env.SECRET_API_KEY; // Server-only env var

// ❌ Not detected - transitive server import
import { utils } from '@/lib/utils'; // utils imports fs internally
```

### Sophisticated Implementation

```typescript
interface ServerImportAnalysis {
  hasServerImports: boolean;
  violations: ServerImportViolation[];
  transitiveIssues: TransitiveImportIssue[];
}

interface ServerImportViolation {
  importPath: string;
  reason: 'database' | 'node-api' | 'server-only-marker' | 'next-server' | 'env-secret';
  location: { line: number; column: number };
}

interface TransitiveImportIssue {
  importPath: string;
  transitiveServerImport: string;
  importChain: string[];
}

function analyzeServerImportsInClient(
  sourceFile: SourceFile,
  project: Project
): ServerImportAnalysis {
  const violations: ServerImportViolation[] = [];
  const transitiveIssues: TransitiveImportIssue[] = [];

  if (!hasUseClientDirective(sourceFile)) {
    return { hasServerImports: false, violations: [], transitiveIssues: [] };
  }

  const imports = sourceFile.getImportDeclarations();

  for (const importDecl of imports) {
    const moduleSpecifier = importDecl.getModuleSpecifierValue();
    const resolvedPath = resolveImportPath(sourceFile, moduleSpecifier);

    // 1. Check direct server imports
    const directViolation = checkDirectServerImport(moduleSpecifier);
    if (directViolation) {
      violations.push({
        importPath: moduleSpecifier,
        reason: directViolation,
        location: getLocation(importDecl),
      });
      continue;
    }

    // 2. Check resolved file for server-only marker
    if (resolvedPath) {
      const importedFile = project.getSourceFile(resolvedPath);
      if (importedFile) {
        if (hasServerOnlyMarker(importedFile)) {
          violations.push({
            importPath: moduleSpecifier,
            reason: 'server-only-marker',
            location: getLocation(importDecl),
          });
          continue;
        }

        // 3. Check for transitive server imports
        const transitiveServer = findTransitiveServerImports(importedFile, project, new Set());

        if (transitiveServer.length > 0) {
          transitiveIssues.push({
            importPath: moduleSpecifier,
            transitiveServerImport: transitiveServer[0].import,
            importChain: transitiveServer[0].chain,
          });
        }
      }
    }
  }

  // 4. Check for server environment variable access
  const envViolations = findServerEnvAccess(sourceFile);
  violations.push(...envViolations);

  return {
    hasServerImports: violations.length > 0 || transitiveIssues.length > 0,
    violations,
    transitiveIssues,
  };
}

function checkDirectServerImport(moduleSpecifier: string): ServerImportViolation['reason'] | null {
  // Database packages
  const databasePackages = [
    '@prisma/client',
    'prisma',
    'drizzle-orm',
    'mongoose',
    'pg',
    'mysql2',
    'better-sqlite3',
    'redis',
    'ioredis',
    'typeorm',
    'sequelize',
    'knex',
  ];

  if (
    databasePackages.some(pkg => moduleSpecifier === pkg || moduleSpecifier.startsWith(pkg + '/'))
  ) {
    return 'database';
  }

  // Node.js APIs
  if (
    moduleSpecifier.startsWith('node:') ||
    ['fs', 'path', 'crypto', 'child_process', 'os', 'net', 'http', 'https'].includes(
      moduleSpecifier
    )
  ) {
    return 'node-api';
  }

  // Next.js server-only imports
  if (['next/headers', 'next/cache'].includes(moduleSpecifier)) {
    return 'next-server';
  }

  return null;
}

function hasServerOnlyMarker(sourceFile: SourceFile): boolean {
  const imports = sourceFile.getImportDeclarations();
  return imports.some(i => i.getModuleSpecifierValue() === 'server-only');
}

function findTransitiveServerImports(
  sourceFile: SourceFile,
  project: Project,
  visited: Set<string>,
  chain: string[] = []
): { import: string; chain: string[] }[] {
  const filePath = sourceFile.getFilePath();
  if (visited.has(filePath)) return [];
  visited.add(filePath);

  const results: { import: string; chain: string[] }[] = [];
  const imports = sourceFile.getImportDeclarations();

  for (const importDecl of imports) {
    const moduleSpecifier = importDecl.getModuleSpecifierValue();

    // Check direct server import
    if (checkDirectServerImport(moduleSpecifier)) {
      results.push({
        import: moduleSpecifier,
        chain: [...chain, filePath, moduleSpecifier],
      });
      continue;
    }

    // Recursively check
    const resolvedPath = resolveImportPath(sourceFile, moduleSpecifier);
    if (resolvedPath) {
      const importedFile = project.getSourceFile(resolvedPath);
      if (importedFile) {
        const transitive = findTransitiveServerImports(importedFile, project, visited, [
          ...chain,
          filePath,
        ]);
        results.push(...transitive);
      }
    }
  }

  return results;
}

function findServerEnvAccess(sourceFile: SourceFile): ServerImportViolation[] {
  const violations: ServerImportViolation[] = [];

  const serverEnvPatterns = [
    /^DATABASE/,
    /^DB_/,
    /SECRET/,
    /PRIVATE/,
    /API_KEY$/,
    /PASSWORD/,
    /TOKEN$/,
    /^AWS_/,
    /^STRIPE_SECRET/,
  ];

  sourceFile.forEachDescendant(node => {
    if (Node.isPropertyAccessExpression(node)) {
      const text = node.getText();
      if (text.startsWith('process.env.')) {
        const envVar = text.replace('process.env.', '');

        // Skip NEXT_PUBLIC_ vars (client-safe)
        if (envVar.startsWith('NEXT_PUBLIC_')) return;

        if (serverEnvPatterns.some(pattern => pattern.test(envVar))) {
          violations.push({
            importPath: `process.env.${envVar}`,
            reason: 'env-secret',
            location: getLocation(node),
          });
        }
      }
    }
  });

  return violations;
}
```

### Key Differences

| Aspect                | Naive           | Sophisticated           |
| --------------------- | --------------- | ----------------------- |
| Package detection     | Hard-coded list | Comprehensive + markers |
| Transitive imports    | Not checked     | Full chain analysis     |
| server-only marker    | Not detected    | Fully supported         |
| Environment variables | Not checked     | Detects secret patterns |
| False negative rate   | High            | Low                     |

---

## Pattern 3: Data Fetching Waterfall Detection

### Naive Implementation (App Router)

```typescript
// Detects: Multiple await statements in sequence
function detectWaterfall(func: ts.FunctionDeclaration) {
  let awaitCount = 0;

  ts.forEachChild(func, function visit(node) {
    if (ts.isAwaitExpression(node)) {
      awaitCount++;
      if (awaitCount > 1) {
        report(node, 'potential-waterfall');
      }
    }
    ts.forEachChild(node, visit);
  });
}
```

**Problems**:

- Flags ALL sequential awaits, even when dependency requires it
- Doesn't understand data dependencies
- No differentiation between related and unrelated fetches

**False positives**:

```tsx
// ❌ Flagged, but this is REQUIRED (data dependency)
export default async function Page({ params }) {
  const user = await getUser(params.id);
  const posts = await getUserPosts(user.id); // MUST wait for user.id!
  return <Profile user={user} posts={posts} />;
}
```

### Sophisticated Implementation

```typescript
interface WaterfallAnalysis {
  hasWaterfall: boolean;
  confidence: 'high' | 'medium' | 'low';
  waterfalls: WaterfallInstance[];
  suggestions: WaterfallSuggestion[];
}

interface WaterfallInstance {
  fetches: FetchInfo[];
  canParallelize: boolean;
  reason: string;
}

interface WaterfallSuggestion {
  type: 'use-promise-all' | 'move-to-parallel-component' | 'use-suspense';
  description: string;
  codeExample?: string;
}

interface FetchInfo {
  node: Node;
  position: number;
  producesVariables: string[];
  consumesVariables: string[];
}

function analyzeDataFetchingWaterfalls(func: Node, sourceFile: SourceFile): WaterfallAnalysis {
  const awaits = findAwaitExpressions(func);
  const fetchInfos: FetchInfo[] = [];

  // 1. Build dependency graph for each fetch
  for (let i = 0; i < awaits.length; i++) {
    const awaitExpr = awaits[i];
    const fetchCall = findFetchInExpression(awaitExpr);
    if (!fetchCall) continue;

    fetchInfos.push({
      node: fetchCall,
      position: i,
      producesVariables: getProducedVariables(awaitExpr),
      consumesVariables: getConsumedVariables(fetchCall),
    });
  }

  // 2. Identify parallelizable fetches
  const waterfalls: WaterfallInstance[] = [];

  for (let i = 0; i < fetchInfos.length; i++) {
    for (let j = i + 1; j < fetchInfos.length; j++) {
      const earlier = fetchInfos[i];
      const later = fetchInfos[j];

      // Check if later depends on earlier's output
      const hasDependency = later.consumesVariables.some(consumed =>
        earlier.producesVariables.includes(consumed)
      );

      if (!hasDependency) {
        // These could be parallelized!
        waterfalls.push({
          fetches: [earlier, later],
          canParallelize: true,
          reason: `${getFetchName(later.node)} does not depend on ${getFetchName(earlier.node)} but awaits sequentially`,
        });
      }
    }
  }

  // 3. Generate suggestions
  const suggestions: WaterfallSuggestion[] = [];

  if (waterfalls.length > 0) {
    const parallelizable = waterfalls.filter(w => w.canParallelize);

    if (parallelizable.length > 0) {
      suggestions.push({
        type: 'use-promise-all',
        description: 'Use Promise.all() for independent fetches',
        codeExample: generatePromiseAllExample(parallelizable),
      });

      suggestions.push({
        type: 'use-suspense',
        description: 'Use Suspense boundaries to stream independent data',
        codeExample: generateSuspenseExample(parallelizable),
      });
    }
  }

  return {
    hasWaterfall: waterfalls.some(w => w.canParallelize),
    confidence: waterfalls.length > 0 ? 'high' : 'low',
    waterfalls,
    suggestions,
  };
}

function getProducedVariables(awaitExpr: Node): string[] {
  const parent = awaitExpr.getParent();

  if (Node.isVariableDeclaration(parent)) {
    return [parent.getName()];
  }

  // Destructuring: const { a, b } = await fetch()
  if (Node.isVariableDeclaration(parent)) {
    const nameNode = parent.getNameNode();
    if (Node.isObjectBindingPattern(nameNode)) {
      return nameNode.getElements().map(e => e.getName());
    }
    if (Node.isArrayBindingPattern(nameNode)) {
      return nameNode
        .getElements()
        .filter(e => Node.isBindingElement(e))
        .map(e => (e as any).getName());
    }
  }

  return [];
}

function getConsumedVariables(fetchCall: Node): string[] {
  const consumed: string[] = [];
  const localScope = getLocalScope(fetchCall);

  fetchCall.forEachDescendant(node => {
    if (Node.isIdentifier(node)) {
      const name = node.getText();

      // Skip property names
      const parent = node.getParent();
      if (Node.isPropertyAccessExpression(parent) && parent.getNameNode() === node) {
        return;
      }

      // Skip if it's a parameter of this function call
      if (isParameterOfAncestorCall(node)) return;

      // Check if defined in local scope above this point
      if (localScope.includes(name)) {
        consumed.push(name);
      }
    }
  });

  return [...new Set(consumed)];
}

function generatePromiseAllExample(waterfalls: WaterfallInstance[]): string {
  const fetchNames = waterfalls.flatMap(w => w.fetches).map(f => getFetchName(f.node));

  return `
const [${fetchNames.map(n => n.toLowerCase()).join(', ')}] = await Promise.all([
  ${fetchNames.join(',\n  ')},
]);`;
}

function generateSuspenseExample(waterfalls: WaterfallInstance[]): string {
  return `
// Split into parallel components with Suspense
<Suspense fallback={<Loading />}>
  <AsyncComponent1 />
</Suspense>
<Suspense fallback={<Loading />}>
  <AsyncComponent2 />
</Suspense>`;
}
```

### Key Differences

| Aspect                 | Naive     | Sophisticated          |
| ---------------------- | --------- | ---------------------- |
| Dependency tracking    | None      | Full variable flow     |
| False positive rate    | Very high | Low                    |
| Actionable suggestions | None      | Specific code examples |
| Handles destructuring  | No        | Yes                    |

---

## Pattern 4: Dynamic Rendering Detection (App Router)

### Naive Implementation

```typescript
// Detects: Any use of cookies() or headers()
function detectDynamicRendering(sourceFile: ts.SourceFile) {
  const dynamicFunctions = ['cookies', 'headers', 'searchParams'];

  ts.forEachChild(sourceFile, function visit(node) {
    if (ts.isCallExpression(node)) {
      const callee = node.expression.getText();
      if (dynamicFunctions.includes(callee)) {
        report(node, 'forces-dynamic-rendering');
      }
    }
    ts.forEachChild(node, visit);
  });
}
```

**Problems**:

- Doesn't check if dynamic is intentional (route config)
- Doesn't suggest optimization strategies
- No severity based on route type

**False positives**:

```tsx
// ❌ Flagged, but this is INTENTIONAL
export const dynamic = 'force-dynamic';

export default async function Dashboard() {
  const session = cookies().get('session'); // Expected to be dynamic!
  return <AuthenticatedView session={session} />;
}
```

### Sophisticated Implementation

```typescript
interface DynamicRenderingAnalysis {
  isDynamic: boolean;
  isIntentional: boolean;
  triggers: DynamicTrigger[];
  optimizationOpportunities: OptimizationOpportunity[];
  severity: 'info' | 'warning' | 'error';
}

interface DynamicTrigger {
  type: 'cookies' | 'headers' | 'searchParams' | 'fetch-no-store' | 'revalidate-0';
  location: { line: number; column: number };
  node: Node;
}

interface OptimizationOpportunity {
  type: 'isolate-dynamic' | 'use-client-cookie' | 'static-generation' | 'ppr';
  description: string;
  impact: 'high' | 'medium' | 'low';
}

function analyzeDynamicRendering(sourceFile: SourceFile): DynamicRenderingAnalysis {
  const fileContext = determineFileContext(sourceFile);
  const triggers: DynamicTrigger[] = [];

  // 1. Find all dynamic triggers
  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const callee = node.getExpression();
      if (Node.isIdentifier(callee)) {
        const name = callee.getText();

        if (name === 'cookies' && isFromNextHeaders(callee)) {
          triggers.push({
            type: 'cookies',
            location: getLocation(node),
            node,
          });
        }

        if (name === 'headers' && isFromNextHeaders(callee)) {
          triggers.push({
            type: 'headers',
            location: getLocation(node),
            node,
          });
        }
      }
    }

    // Check fetch options
    if (Node.isCallExpression(node)) {
      const fetchAnalysis = analyzeFetchCache(node);
      if (fetchAnalysis.forcesDynamic) {
        triggers.push({
          type: fetchAnalysis.reason as DynamicTrigger['type'],
          location: getLocation(node),
          node,
        });
      }
    }
  });

  // 2. Check route segment config
  const routeConfig = fileContext.routeConfig;
  const isIntentionallyDynamic =
    routeConfig.dynamic === 'force-dynamic' || routeConfig.revalidate === 0;

  const isIntentionallyStatic =
    routeConfig.dynamic === 'force-static' || routeConfig.dynamic === 'error';

  // 3. Determine severity
  let severity: DynamicRenderingAnalysis['severity'] = 'info';

  if (triggers.length > 0 && isIntentionallyStatic) {
    severity = 'error'; // Conflict!
  } else if (triggers.length > 0 && !isIntentionallyDynamic && fileContext.isPage) {
    severity = 'warning'; // Might be unintentional
  }

  // 4. Generate optimization opportunities
  const opportunities: OptimizationOpportunity[] = [];

  if (triggers.length > 0 && !isIntentionallyDynamic) {
    // Suggest isolating dynamic parts
    const cookieTriggers = triggers.filter(t => t.type === 'cookies');
    if (cookieTriggers.length > 0) {
      opportunities.push({
        type: 'isolate-dynamic',
        description: 'Move cookie access to a Client Component to keep the page static',
        impact: 'high',
      });

      opportunities.push({
        type: 'use-client-cookie',
        description: 'Use client-side cookie library (js-cookie) instead of next/headers',
        impact: 'medium',
      });
    }

    // Suggest PPR if available
    if (triggers.length > 0) {
      opportunities.push({
        type: 'ppr',
        description: 'Use Partial Prerendering to statically render most of the page',
        impact: 'high',
      });
    }
  }

  return {
    isDynamic: triggers.length > 0,
    isIntentional: isIntentionallyDynamic,
    triggers,
    optimizationOpportunities: opportunities,
    severity,
  };
}

function isFromNextHeaders(identifier: Node): boolean {
  const symbol = identifier.getSymbol();
  if (!symbol) return false;

  const declarations = symbol.getDeclarations();
  for (const decl of declarations) {
    const sourceFile = decl.getSourceFile();
    const filePath = sourceFile.getFilePath();
    if (
      filePath.includes('next/headers') ||
      filePath.includes('next/dist/client/components/headers')
    ) {
      return true;
    }
  }

  // Check import statement
  const sourceFile = identifier.getSourceFile();
  const imports = sourceFile.getImportDeclarations();

  for (const imp of imports) {
    if (imp.getModuleSpecifierValue() === 'next/headers') {
      const namedImports = imp.getNamedImports();
      if (namedImports.some(n => n.getName() === identifier.getText())) {
        return true;
      }
    }
  }

  return false;
}

function analyzeFetchCache(callExpr: Node): { forcesDynamic: boolean; reason?: string } {
  if (!Node.isCallExpression(callExpr)) return { forcesDynamic: false };

  const callee = callExpr.getExpression();
  if (!Node.isIdentifier(callee) || callee.getText() !== 'fetch') {
    return { forcesDynamic: false };
  }

  const args = callExpr.getArguments();
  if (args.length < 2) return { forcesDynamic: false };

  const options = args[1];
  if (!Node.isObjectLiteralExpression(options)) return { forcesDynamic: false };

  // Check cache: 'no-store'
  const cacheProperty = options.getProperty('cache');
  if (cacheProperty && Node.isPropertyAssignment(cacheProperty)) {
    const value = cacheProperty.getInitializer();
    if (value && Node.isStringLiteral(value) && value.getLiteralText() === 'no-store') {
      return { forcesDynamic: true, reason: 'fetch-no-store' };
    }
  }

  // Check next.revalidate: 0
  const nextProperty = options.getProperty('next');
  if (nextProperty && Node.isPropertyAssignment(nextProperty)) {
    const nextValue = nextProperty.getInitializer();
    if (nextValue && Node.isObjectLiteralExpression(nextValue)) {
      const revalidate = nextValue.getProperty('revalidate');
      if (revalidate && Node.isPropertyAssignment(revalidate)) {
        const revalValue = revalidate.getInitializer();
        if (revalValue && Node.isNumericLiteral(revalValue) && revalValue.getLiteralValue() === 0) {
          return { forcesDynamic: true, reason: 'revalidate-0' };
        }
      }
    }
  }

  return { forcesDynamic: false };
}
```

### Key Differences

| Aspect                   | Naive  | Sophisticated            |
| ------------------------ | ------ | ------------------------ |
| Route config awareness   | None   | Full config parsing      |
| Intentionality detection | None   | Checks for force-dynamic |
| Optimization suggestions | None   | Multiple strategies      |
| Severity levels          | Binary | Contextual               |

---

## Pattern 5: Pages Router Data Fetching

### Naive Implementation

```typescript
// Detects: Missing getStaticProps or getServerSideProps
function detectMissingDataFetching(sourceFile: ts.SourceFile) {
  const hasGSP = sourceFile.getText().includes('getStaticProps');
  const hasGSSP = sourceFile.getText().includes('getServerSideProps');

  if (!hasGSP && !hasGSSP) {
    report(sourceFile, 'missing-data-fetching');
  }
}
```

**Problems**:

- Not all pages need server data fetching
- Client-only pages are valid
- Doesn't check if data is actually needed

**False positives**:

```tsx
// pages/about.tsx
// ❌ Flagged, but this is FINE - static content only
export default function AboutPage() {
  return (
    <div>
      <h1>About Us</h1>
      <p>We are a company that does things.</p>
    </div>
  );
}

// pages/dashboard.tsx
// ❌ Flagged, but this is FINE - client-only data fetching
export default function Dashboard() {
  const { data } = useSWR('/api/dashboard', fetcher);
  return <DashboardView data={data} />;
}
```

### Sophisticated Implementation

```typescript
interface PagesDataFetchingAnalysis {
  recommendation: 'getStaticProps' | 'getServerSideProps' | 'client-side' | 'none';
  confidence: 'high' | 'medium' | 'low';
  reasons: string[];
  currentSetup: 'gsp' | 'gssp' | 'gip' | 'client' | 'none';
  issues: PageDataIssue[];
}

interface PageDataIssue {
  type: 'unnecessary-ssr' | 'missing-revalidate' | 'static-for-dynamic' | 'missing-paths';
  description: string;
  suggestion: string;
}

function analyzePagesDataFetching(sourceFile: SourceFile): PagesDataFetchingAnalysis {
  const hasGSP = hasExportedFunction(sourceFile, 'getStaticProps');
  const hasGSSP = hasExportedFunction(sourceFile, 'getServerSideProps');
  const hasGIP = hasGetInitialProps(sourceFile);
  const hasClientFetching = hasClientSideDataFetching(sourceFile);

  const currentSetup: PagesDataFetchingAnalysis['currentSetup'] = hasGSP
    ? 'gsp'
    : hasGSSP
      ? 'gssp'
      : hasGIP
        ? 'gip'
        : hasClientFetching
          ? 'client'
          : 'none';

  const issues: PageDataIssue[] = [];
  const reasons: string[] = [];

  // Analyze what data the page actually needs
  const dataNeeds = analyzePageDataNeeds(sourceFile);

  // 1. Check if page uses props from data fetching
  if (!dataNeeds.usesProps && (hasGSP || hasGSSP)) {
    issues.push({
      type: 'unnecessary-ssr',
      description: 'Page has data fetching but component does not use props',
      suggestion: 'Remove unused data fetching or use the fetched data',
    });
  }

  // 2. Check if static data fetching is used for dynamic data
  if (hasGSP && !hasRevalidate(sourceFile) && dataNeeds.dataVolatility === 'high') {
    issues.push({
      type: 'static-for-dynamic',
      description: 'Using getStaticProps for frequently changing data without revalidate',
      suggestion: 'Add revalidate option or switch to getServerSideProps',
    });
  }

  // 3. Check for missing getStaticPaths
  const filePath = sourceFile.getFilePath();
  const isDynamicRoute = /\[.+\]/.test(path.basename(filePath));

  if (isDynamicRoute && hasGSP && !hasExportedFunction(sourceFile, 'getStaticPaths')) {
    issues.push({
      type: 'missing-paths',
      description: 'Dynamic route with getStaticProps requires getStaticPaths',
      suggestion: 'Add getStaticPaths to define which paths to pre-render',
    });
  }

  // 4. Determine recommendation
  let recommendation: PagesDataFetchingAnalysis['recommendation'] = 'none';
  let confidence: 'high' | 'medium' | 'low' = 'low';

  if (!dataNeeds.needsServerData) {
    recommendation = 'none';
    confidence = 'high';
    reasons.push('Page does not require server-side data');
  } else if (dataNeeds.isUserSpecific) {
    recommendation = 'getServerSideProps';
    confidence = 'high';
    reasons.push('Page requires user-specific data');
  } else if (dataNeeds.dataVolatility === 'low') {
    recommendation = 'getStaticProps';
    confidence = 'high';
    reasons.push('Page data is static or changes infrequently');
  } else if (dataNeeds.dataVolatility === 'medium') {
    recommendation = 'getStaticProps';
    confidence = 'medium';
    reasons.push('Page data changes periodically - use revalidate');
  } else {
    recommendation = 'getServerSideProps';
    confidence = 'medium';
    reasons.push('Page data changes frequently');
  }

  return {
    recommendation,
    confidence,
    reasons,
    currentSetup,
    issues,
  };
}

interface PageDataNeeds {
  needsServerData: boolean;
  usesProps: boolean;
  isUserSpecific: boolean;
  dataVolatility: 'low' | 'medium' | 'high' | 'unknown';
}

function analyzePageDataNeeds(sourceFile: SourceFile): PageDataNeeds {
  const defaultExport = findDefaultExport(sourceFile);
  if (!defaultExport) {
    return {
      needsServerData: false,
      usesProps: false,
      isUserSpecific: false,
      dataVolatility: 'unknown',
    };
  }

  // Check if component accepts and uses props
  const params = getFunctionParameters(defaultExport);
  const usesProps = params.length > 0;

  // Check for user-specific indicators
  const text = sourceFile.getFullText();
  const isUserSpecific =
    text.includes('session') ||
    text.includes('user') ||
    text.includes('authenticated') ||
    text.includes('cookies');

  // Check for API calls or data patterns
  const hasApiUrls = /\/api\/|https?:\/\//.test(text);
  const hasClientFetching = hasClientSideDataFetching(sourceFile);

  const needsServerData = usesProps || (hasApiUrls && !hasClientFetching);

  // Estimate data volatility based on patterns
  let dataVolatility: PageDataNeeds['dataVolatility'] = 'unknown';

  if (text.includes('static') || text.includes('content') || text.includes('blog')) {
    dataVolatility = 'low';
  } else if (text.includes('real-time') || text.includes('live') || text.includes('stock')) {
    dataVolatility = 'high';
  } else if (isUserSpecific) {
    dataVolatility = 'high';
  }

  return { needsServerData, usesProps, isUserSpecific, dataVolatility };
}

function hasClientSideDataFetching(sourceFile: SourceFile): boolean {
  const patterns = ['useSWR', 'useQuery', 'useEffect.*fetch', 'useState.*fetch'];
  const text = sourceFile.getFullText();

  return patterns.some(pattern => new RegExp(pattern).test(text));
}
```

---

## Implementation Effort vs Value Matrix

| Pattern                  | Router | Naive Effort | Sophisticated Effort | Value Gain |
| ------------------------ | ------ | ------------ | -------------------- | ---------- |
| "use client" detection   | App    | 1 hour       | 2-3 days             | Very High  |
| Server imports in client | App    | 2 hours      | 3-5 days             | Very High  |
| Data waterfall detection | App    | 1 hour       | 2-3 days             | High       |
| Dynamic rendering        | App    | 1 hour       | 1-2 days             | High       |
| Pages data fetching      | Pages  | 1 hour       | 1-2 days             | Medium     |
| Route structure          | App    | 30 min       | 4-6 hours            | Medium     |
| Server Actions           | App    | 2 hours      | 1-2 days             | High       |

## Recommendation

**Start sophisticated (App Router)**:

- "use client" directive detection (most common error)
- Server imports in client components (security critical)
- Non-serializable props (subtle but impactful bugs)

**Start sophisticated (Pages Router)**:

- getStaticProps vs getServerSideProps guidance
- Dynamic route path generation

**Keep naive initially**:

- Route structure validation (file-based, simpler)
- Basic config validation

**Invest later**:

- Performance-based recommendations (needs runtime data)
- Automatic code fixes (high complexity, high value)
