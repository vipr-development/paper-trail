---
id: 02-ts-morph-analysis
---

# ts-morph Deep Analysis Techniques for Next.js Static Analysis

This document covers advanced ts-morph patterns for semantic analysis of Next.js applications, including both Pages Router and App Router specific techniques.

## Foundation: Next.js Project Setup

```typescript
import {
  Project,
  Node,
  SyntaxKind,
  SourceFile,
  Type,
  Symbol as TsSymbol,
  ImportDeclaration,
} from 'ts-morph';
import * as path from 'path';
import * as fs from 'fs';

interface NextJsProjectContext {
  project: Project;
  appDir: string | null;
  pagesDir: string | null;
  srcDir: string | null;
  routerType: 'app' | 'pages' | 'hybrid';
}

function createNextJsAnalysisProject(rootDir: string): NextJsProjectContext {
  const project = new Project({
    tsConfigFilePath: path.join(rootDir, 'tsconfig.json'),
    skipAddingFilesFromTsConfig: false,
  });

  // Detect directory structure
  const srcDir = fs.existsSync(path.join(rootDir, 'src')) ? path.join(rootDir, 'src') : null;

  const baseDir = srcDir || rootDir;
  const appDir = fs.existsSync(path.join(baseDir, 'app')) ? path.join(baseDir, 'app') : null;
  const pagesDir = fs.existsSync(path.join(baseDir, 'pages')) ? path.join(baseDir, 'pages') : null;

  const routerType = appDir && pagesDir ? 'hybrid' : appDir ? 'app' : 'pages';

  return { project, appDir, pagesDir, srcDir, routerType };
}
```

## Core Technique 1: "use client" Boundary Analysis

### Detecting Client Directive

```typescript
function hasUseClientDirective(sourceFile: SourceFile): boolean {
  const firstStatement = sourceFile.getStatements()[0];

  // Check for "use client" as expression statement
  if (Node.isExpressionStatement(firstStatement)) {
    const expression = firstStatement.getExpression();
    if (Node.isStringLiteral(expression)) {
      const text = expression.getLiteralText();
      return text === 'use client';
    }
  }

  // Also check first line comment pattern (some setups)
  const firstLine = sourceFile.getFullText().split('\n')[0];
  return firstLine.includes("'use client'") || firstLine.includes('"use client"');
}

function hasUseServerDirective(sourceFile: SourceFile): boolean {
  const firstStatement = sourceFile.getStatements()[0];

  if (Node.isExpressionStatement(firstStatement)) {
    const expression = firstStatement.getExpression();
    if (Node.isStringLiteral(expression)) {
      return expression.getLiteralText() === 'use server';
    }
  }

  return false;
}
```

### Building Import Graph for Boundary Propagation

```typescript
interface ImportGraphNode {
  filePath: string;
  isClientComponent: boolean;
  isServerComponent: boolean;
  imports: string[];
  importedBy: string[];
}

function buildImportGraph(project: Project): Map<string, ImportGraphNode> {
  const graph = new Map<string, ImportGraphNode>();

  // First pass: Initialize all nodes
  for (const sourceFile of project.getSourceFiles()) {
    const filePath = sourceFile.getFilePath();

    graph.set(filePath, {
      filePath,
      isClientComponent: hasUseClientDirective(sourceFile),
      isServerComponent: !hasUseClientDirective(sourceFile), // Default in App Router
      imports: [],
      importedBy: [],
    });
  }

  // Second pass: Build import relationships
  for (const sourceFile of project.getSourceFiles()) {
    const filePath = sourceFile.getFilePath();
    const node = graph.get(filePath)!;

    const importDecls = sourceFile.getImportDeclarations();

    for (const importDecl of importDecls) {
      const moduleSpecifier = importDecl.getModuleSpecifierValue();
      const resolvedPath = resolveImportPath(sourceFile, moduleSpecifier);

      if (resolvedPath && graph.has(resolvedPath)) {
        node.imports.push(resolvedPath);
        graph.get(resolvedPath)!.importedBy.push(filePath);
      }
    }
  }

  return graph;
}

function resolveImportPath(sourceFile: SourceFile, moduleSpecifier: string): string | null {
  // Skip node_modules imports
  if (!moduleSpecifier.startsWith('.') && !moduleSpecifier.startsWith('@/')) {
    return null;
  }

  const sourceDir = path.dirname(sourceFile.getFilePath());

  // Handle @ alias
  if (moduleSpecifier.startsWith('@/')) {
    const project = sourceFile.getProject();
    const compilerOptions = project.getCompilerOptions();
    const paths = compilerOptions.paths || {};

    for (const [alias, targets] of Object.entries(paths)) {
      if (moduleSpecifier.startsWith(alias.replace('*', ''))) {
        const baseUrl = compilerOptions.baseUrl || '.';
        const target = targets[0].replace('*', '');
        const resolved = moduleSpecifier.replace(alias.replace('*', ''), target);
        return resolveWithExtensions(path.resolve(baseUrl, resolved));
      }
    }
  }

  // Relative import
  const absolutePath = path.resolve(sourceDir, moduleSpecifier);
  return resolveWithExtensions(absolutePath);
}

function resolveWithExtensions(basePath: string): string | null {
  const extensions = ['.tsx', '.ts', '.jsx', '.js', '/index.tsx', '/index.ts'];

  for (const ext of extensions) {
    const fullPath = basePath + ext;
    if (fs.existsSync(fullPath)) {
      return fullPath;
    }
  }

  return null;
}
```

### Determining Effective Component Type

```typescript
interface ComponentTypeAnalysis {
  explicitType: 'client' | 'server' | 'none';
  effectiveType: 'client' | 'server';
  reason: string;
  inheritedFrom?: string;
}

function analyzeComponentType(
  sourceFile: SourceFile,
  importGraph: Map<string, ImportGraphNode>
): ComponentTypeAnalysis {
  const filePath = sourceFile.getFilePath();

  // Check explicit directive
  if (hasUseClientDirective(sourceFile)) {
    return {
      explicitType: 'client',
      effectiveType: 'client',
      reason: 'Has "use client" directive',
    };
  }

  if (hasUseServerDirective(sourceFile)) {
    return {
      explicitType: 'server',
      effectiveType: 'server',
      reason: 'Has "use server" directive (Server Action)',
    };
  }

  // Check if imported by a client component
  const clientImporter = findClientImporter(filePath, importGraph, new Set());
  if (clientImporter) {
    return {
      explicitType: 'none',
      effectiveType: 'client',
      reason: 'Imported by client component',
      inheritedFrom: clientImporter,
    };
  }

  // Default: Server Component in App Router
  return {
    explicitType: 'none',
    effectiveType: 'server',
    reason: 'Default Server Component (no client boundary in import chain)',
  };
}

function findClientImporter(
  filePath: string,
  graph: Map<string, ImportGraphNode>,
  visited: Set<string>
): string | null {
  if (visited.has(filePath)) return null;
  visited.add(filePath);

  const node = graph.get(filePath);
  if (!node) return null;

  for (const importer of node.importedBy) {
    const importerNode = graph.get(importer);
    if (importerNode?.isClientComponent) {
      return importer;
    }

    const transitiveClient = findClientImporter(importer, graph, visited);
    if (transitiveClient) return transitiveClient;
  }

  return null;
}
```

## Core Technique 2: Server/Client API Detection

### Identifying Client-Only APIs

```typescript
const CLIENT_ONLY_APIS = {
  // React hooks (require client context)
  reactHooks: [
    'useState',
    'useEffect',
    'useRef',
    'useCallback',
    'useMemo',
    'useReducer',
    'useContext',
    'useLayoutEffect',
    'useImperativeHandle',
    'useDebugValue',
    'useDeferredValue',
    'useTransition',
    'useId',
    'useSyncExternalStore',
    'useInsertionEffect',
  ],

  // Browser APIs
  browserApis: [
    'window',
    'document',
    'localStorage',
    'sessionStorage',
    'navigator',
    'location',
    'history',
    'alert',
    'confirm',
    'prompt',
  ],

  // Event handlers (JSX props)
  eventHandlers: [
    'onClick',
    'onChange',
    'onSubmit',
    'onFocus',
    'onBlur',
    'onKeyDown',
    'onKeyUp',
    'onMouseEnter',
    'onMouseLeave',
    'onScroll',
    'onLoad',
    'onError',
  ],

  // Client-only Next.js imports
  nextClientImports: [
    'next/navigation:useRouter',
    'next/navigation:useSearchParams',
    'next/navigation:usePathname',
    'next/navigation:useParams',
  ],
};

interface ClientAPIUsage {
  api: string;
  category: 'hook' | 'browser' | 'event' | 'import';
  location: { line: number; column: number };
  node: Node;
}

function findClientOnlyAPIUsage(sourceFile: SourceFile): ClientAPIUsage[] {
  const usages: ClientAPIUsage[] = [];

  sourceFile.forEachDescendant(node => {
    // Check for hook calls
    if (Node.isCallExpression(node)) {
      const callee = node.getExpression();
      if (Node.isIdentifier(callee)) {
        const name = callee.getText();
        if (CLIENT_ONLY_APIS.reactHooks.includes(name)) {
          usages.push({
            api: name,
            category: 'hook',
            location: getLocation(node),
            node,
          });
        }
      }
    }

    // Check for browser API access
    if (Node.isIdentifier(node)) {
      const name = node.getText();
      if (CLIENT_ONLY_APIS.browserApis.includes(name)) {
        // Verify it's not a local variable
        const symbol = node.getSymbol();
        if (!symbol || isGlobalSymbol(symbol)) {
          usages.push({
            api: name,
            category: 'browser',
            location: getLocation(node),
            node,
          });
        }
      }
    }

    // Check for event handler props
    if (Node.isJsxAttribute(node)) {
      const name = node.getName();
      if (CLIENT_ONLY_APIS.eventHandlers.includes(name)) {
        usages.push({
          api: name,
          category: 'event',
          location: getLocation(node),
          node,
        });
      }
    }
  });

  // Check imports
  const clientImports = findClientOnlyImports(sourceFile);
  usages.push(
    ...clientImports.map(imp => ({
      api: imp.name,
      category: 'import' as const,
      location: getLocation(imp.node),
      node: imp.node,
    }))
  );

  return usages;
}

function isGlobalSymbol(symbol: TsSymbol): boolean {
  const declarations = symbol.getDeclarations();
  return (
    declarations.length === 0 ||
    declarations.some(d => {
      const sourceFile = d.getSourceFile();
      return sourceFile.getFilePath().includes('node_modules/typescript/lib');
    })
  );
}
```

### Identifying Server-Only APIs

```typescript
const SERVER_ONLY_APIS = {
  // Next.js server functions
  nextServerFunctions: ['cookies', 'headers', 'draftMode'],

  // Next.js server imports
  nextServerImports: ['next/headers', 'next/cache'],

  // Node.js APIs
  nodeApis: [
    'fs',
    'path',
    'crypto',
    'child_process',
    'os',
    'net',
    'http',
    'https',
    'stream',
    'buffer',
    'util',
  ],

  // Database packages
  databasePackages: [
    '@prisma/client',
    'prisma',
    'drizzle-orm',
    'mongoose',
    'pg',
    'mysql2',
    'better-sqlite3',
    'redis',
    'ioredis',
  ],

  // Server-only markers
  serverOnlyPackage: 'server-only',
};

interface ServerAPIUsage {
  api: string;
  category: 'next-server' | 'node' | 'database' | 'marker';
  location: { line: number; column: number };
  node: Node;
}

function findServerOnlyAPIUsage(sourceFile: SourceFile): ServerAPIUsage[] {
  const usages: ServerAPIUsage[] = [];

  // Check imports
  const importDecls = sourceFile.getImportDeclarations();

  for (const importDecl of importDecls) {
    const moduleSpecifier = importDecl.getModuleSpecifierValue();

    // Check for server-only package marker
    if (moduleSpecifier === 'server-only') {
      usages.push({
        api: 'server-only',
        category: 'marker',
        location: getLocation(importDecl),
        node: importDecl,
      });
    }

    // Check for Next.js server imports
    if (SERVER_ONLY_APIS.nextServerImports.includes(moduleSpecifier)) {
      usages.push({
        api: moduleSpecifier,
        category: 'next-server',
        location: getLocation(importDecl),
        node: importDecl,
      });
    }

    // Check for Node.js imports
    if (
      SERVER_ONLY_APIS.nodeApis.includes(moduleSpecifier) ||
      moduleSpecifier.startsWith('node:')
    ) {
      usages.push({
        api: moduleSpecifier,
        category: 'node',
        location: getLocation(importDecl),
        node: importDecl,
      });
    }

    // Check for database packages
    if (
      SERVER_ONLY_APIS.databasePackages.some(
        pkg => moduleSpecifier === pkg || moduleSpecifier.startsWith(pkg + '/')
      )
    ) {
      usages.push({
        api: moduleSpecifier,
        category: 'database',
        location: getLocation(importDecl),
        node: importDecl,
      });
    }
  }

  // Check for server function calls (cookies(), headers())
  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const callee = node.getExpression();
      if (Node.isIdentifier(callee)) {
        const name = callee.getText();
        if (SERVER_ONLY_APIS.nextServerFunctions.includes(name)) {
          // Verify it's from next/headers
          const symbol = callee.getSymbol();
          if (symbol && isNextHeadersImport(symbol)) {
            usages.push({
              api: name,
              category: 'next-server',
              location: getLocation(node),
              node,
            });
          }
        }
      }
    }
  });

  return usages;
}

function isNextHeadersImport(symbol: TsSymbol): boolean {
  const declarations = symbol.getDeclarations();
  for (const decl of declarations) {
    const sourceFile = decl.getSourceFile();
    if (
      sourceFile.getFilePath().includes('next/headers') ||
      sourceFile.getFilePath().includes('next/dist/client/components/headers')
    ) {
      return true;
    }
  }
  return false;
}
```

## Core Technique 3: Props Serialization Analysis

### Detecting Non-Serializable Prop Values

```typescript
interface PropSerializationIssue {
  propName: string;
  propType: string;
  issue: 'function' | 'date' | 'regex' | 'map' | 'set' | 'class' | 'symbol' | 'promise';
  location: { line: number; column: number };
  node: Node;
}

function analyzePropsSerializability(
  jsxElement: Node,
  targetComponentType: 'client' | 'server'
): PropSerializationIssue[] {
  const issues: PropSerializationIssue[] = [];

  // Only check when passing from Server to Client
  if (targetComponentType !== 'client') return issues;

  const attributes = getJsxAttributes(jsxElement);

  for (const attr of attributes) {
    if (!Node.isJsxAttribute(attr)) continue;

    const propName = attr.getName();
    const initializer = attr.getInitializer();

    if (!initializer || !Node.isJsxExpression(initializer)) continue;

    const expression = initializer.getExpression();
    if (!expression) continue;

    const issue = checkSerializability(expression);
    if (issue) {
      issues.push({
        propName,
        propType: expression.getType().getText(),
        issue,
        location: getLocation(attr),
        node: attr,
      });
    }
  }

  return issues;
}

function checkSerializability(node: Node): PropSerializationIssue['issue'] | null {
  const type = node.getType();
  const typeText = type.getText();

  // Check for function types
  if (Node.isArrowFunction(node) || Node.isFunctionExpression(node)) {
    return 'function';
  }
  if (type.getCallSignatures().length > 0) {
    return 'function';
  }

  // Check for Date
  if (typeText === 'Date' || typeText.includes('Date')) {
    return 'date';
  }

  // Check for RegExp
  if (Node.isRegularExpressionLiteral(node) || typeText === 'RegExp') {
    return 'regex';
  }

  // Check for Map/Set
  if (typeText.startsWith('Map<') || typeText === 'Map') {
    return 'map';
  }
  if (typeText.startsWith('Set<') || typeText === 'Set') {
    return 'set';
  }

  // Check for Symbol
  if (typeText === 'symbol' || typeText.startsWith('unique symbol')) {
    return 'symbol';
  }

  // Check for Promise
  if (typeText.startsWith('Promise<')) {
    return 'promise';
  }

  // Check for class instances (non-plain objects)
  if (type.isClass()) {
    return 'class';
  }

  // Recursively check object properties
  if (Node.isObjectLiteralExpression(node)) {
    for (const prop of node.getProperties()) {
      if (Node.isPropertyAssignment(prop)) {
        const init = prop.getInitializer();
        if (init) {
          const nestedIssue = checkSerializability(init);
          if (nestedIssue) return nestedIssue;
        }
      }
    }
  }

  return null;
}
```

## Core Technique 4: Data Fetching Analysis

### Analyzing Fetch Patterns

```typescript
interface FetchAnalysis {
  location: { line: number; column: number };
  url: string | 'dynamic';
  cacheOption: 'force-cache' | 'no-store' | 'default' | 'custom';
  revalidate: number | false | 'dynamic';
  inServerComponent: boolean;
  inRouteHandler: boolean;
  inServerAction: boolean;
  issues: FetchIssue[];
}

interface FetchIssue {
  type: 'missing-error-handling' | 'waterfall' | 'cache-mismatch' | 'dynamic-in-static';
  description: string;
  suggestion: string;
}

function analyzeFetchCalls(sourceFile: SourceFile): FetchAnalysis[] {
  const analyses: FetchAnalysis[] = [];
  const fileContext = determineFileContext(sourceFile);

  sourceFile.forEachDescendant(node => {
    if (!Node.isCallExpression(node)) return;

    const callee = node.getExpression();
    if (!Node.isIdentifier(callee) || callee.getText() !== 'fetch') return;

    const args = node.getArguments();
    if (args.length === 0) return;

    const urlArg = args[0];
    const optionsArg = args[1];

    const analysis: FetchAnalysis = {
      location: getLocation(node),
      url: extractUrlString(urlArg),
      cacheOption: extractCacheOption(optionsArg),
      revalidate: extractRevalidate(optionsArg),
      inServerComponent: fileContext.isServerComponent,
      inRouteHandler: fileContext.isRouteHandler,
      inServerAction: fileContext.isServerAction,
      issues: [],
    };

    // Check for issues
    analysis.issues = detectFetchIssues(node, analysis, fileContext);

    analyses.push(analysis);
  });

  return analyses;
}

function extractCacheOption(optionsArg: Node | undefined): FetchAnalysis['cacheOption'] {
  if (!optionsArg || !Node.isObjectLiteralExpression(optionsArg)) {
    return 'default'; // Next.js default is force-cache in Server Components
  }

  const cacheProperty = optionsArg.getProperty('cache');
  if (cacheProperty && Node.isPropertyAssignment(cacheProperty)) {
    const value = cacheProperty.getInitializer();
    if (value && Node.isStringLiteral(value)) {
      const text = value.getLiteralText();
      if (text === 'force-cache') return 'force-cache';
      if (text === 'no-store') return 'no-store';
    }
  }

  // Check next.revalidate option
  const nextProperty = optionsArg.getProperty('next');
  if (nextProperty && Node.isPropertyAssignment(nextProperty)) {
    return 'custom';
  }

  return 'default';
}

function extractRevalidate(optionsArg: Node | undefined): FetchAnalysis['revalidate'] {
  if (!optionsArg || !Node.isObjectLiteralExpression(optionsArg)) {
    return false;
  }

  const nextProperty = optionsArg.getProperty('next');
  if (!nextProperty || !Node.isPropertyAssignment(nextProperty)) {
    return false;
  }

  const nextValue = nextProperty.getInitializer();
  if (!nextValue || !Node.isObjectLiteralExpression(nextValue)) {
    return false;
  }

  const revalidateProperty = nextValue.getProperty('revalidate');
  if (revalidateProperty && Node.isPropertyAssignment(revalidateProperty)) {
    const value = revalidateProperty.getInitializer();
    if (value && Node.isNumericLiteral(value)) {
      return parseInt(value.getText(), 10);
    }
    if (value && value.getText() === 'false') {
      return false;
    }
  }

  return 'dynamic';
}

function detectFetchIssues(
  fetchCall: Node,
  analysis: FetchAnalysis,
  context: FileContext
): FetchIssue[] {
  const issues: FetchIssue[] = [];

  // Check for error handling
  if (!hasErrorHandling(fetchCall)) {
    issues.push({
      type: 'missing-error-handling',
      description: 'Fetch call without error handling',
      suggestion: 'Add try/catch or check response.ok before parsing',
    });
  }

  // Check for cache mismatch with route config
  if (context.routeConfig.dynamic === 'force-static' && analysis.cacheOption === 'no-store') {
    issues.push({
      type: 'cache-mismatch',
      description: 'no-store fetch in force-static route',
      suggestion: 'Remove no-store or change route to allow dynamic rendering',
    });
  }

  // Check for dynamic fetch in intended-static context
  if (
    context.routeConfig.dynamic === 'force-static' &&
    (analysis.cacheOption === 'no-store' || analysis.revalidate === 0)
  ) {
    issues.push({
      type: 'dynamic-in-static',
      description: 'Dynamic fetch options in statically configured route',
      suggestion: 'Use cache: "force-cache" or remove route config',
    });
  }

  return issues;
}

function hasErrorHandling(fetchCall: Node): boolean {
  // Check if wrapped in try-catch
  const tryStatement = fetchCall.getFirstAncestorByKind(SyntaxKind.TryStatement);
  if (tryStatement) return true;

  // Check if .catch() is chained
  const parent = fetchCall.getParent();
  if (Node.isPropertyAccessExpression(parent)) {
    if (parent.getName() === 'catch') return true;
  }

  // Check if response.ok is checked
  const func =
    fetchCall.getFirstAncestorByKind(SyntaxKind.FunctionDeclaration) ||
    fetchCall.getFirstAncestorByKind(SyntaxKind.ArrowFunction);
  if (func) {
    const hasOkCheck = func
      .getDescendantsOfKind(SyntaxKind.PropertyAccessExpression)
      .some(p => p.getName() === 'ok');
    if (hasOkCheck) return true;
  }

  return false;
}
```

### Detecting Data Fetching Waterfalls

```typescript
interface WaterfallAnalysis {
  fetches: FetchInSequence[];
  parallelizablePairs: [FetchInSequence, FetchInSequence][];
  totalSequentialTime: 'unknown' | number;
}

interface FetchInSequence {
  node: Node;
  awaitPosition: number;
  dependsOn: string[];
  providesVariables: string[];
}

function analyzeDataFetchingWaterfalls(func: Node): WaterfallAnalysis {
  const awaits = findAwaitExpressions(func);
  const fetches: FetchInSequence[] = [];

  for (let i = 0; i < awaits.length; i++) {
    const awaitExpr = awaits[i];
    const fetchCall = findFetchInExpression(awaitExpr);
    if (!fetchCall) continue;

    // Analyze dependencies
    const usedVars = findUsedIdentifiers(fetchCall);
    const assignedVar = getAssignedVariable(awaitExpr);

    fetches.push({
      node: fetchCall,
      awaitPosition: i,
      dependsOn: usedVars,
      providesVariables: assignedVar ? [assignedVar] : [],
    });
  }

  // Find parallelizable pairs
  const parallelizable: [FetchInSequence, FetchInSequence][] = [];

  for (let i = 0; i < fetches.length; i++) {
    for (let j = i + 1; j < fetches.length; j++) {
      const earlier = fetches[i];
      const later = fetches[j];

      // Check if later fetch depends on earlier fetch's result
      const hasDependency = later.dependsOn.some(dep => earlier.providesVariables.includes(dep));

      if (!hasDependency) {
        parallelizable.push([earlier, later]);
      }
    }
  }

  return {
    fetches,
    parallelizablePairs: parallelizable,
    totalSequentialTime: 'unknown',
  };
}

function findUsedIdentifiers(node: Node): string[] {
  const identifiers: string[] = [];

  node.forEachDescendant(desc => {
    if (Node.isIdentifier(desc)) {
      const name = desc.getText();
      // Skip if it's a property name
      const parent = desc.getParent();
      if (Node.isPropertyAccessExpression(parent) && parent.getNameNode() === desc) {
        return;
      }
      identifiers.push(name);
    }
  });

  return [...new Set(identifiers)];
}
```

## Core Technique 5: Route and File Convention Analysis

### Determining File Context

```typescript
interface FileContext {
  isPage: boolean;
  isLayout: boolean;
  isLoading: boolean;
  isError: boolean;
  isRouteHandler: boolean;
  isServerAction: boolean;
  isServerComponent: boolean;
  isClientComponent: boolean;
  isMiddleware: boolean;
  routeSegment: string;
  routeConfig: RouteSegmentConfig;
}

interface RouteSegmentConfig {
  dynamic?: 'auto' | 'force-dynamic' | 'error' | 'force-static';
  dynamicParams?: boolean;
  revalidate?: number | false;
  fetchCache?: string;
  runtime?: 'edge' | 'nodejs';
  preferredRegion?: string | string[];
}

function determineFileContext(sourceFile: SourceFile): FileContext {
  const filePath = sourceFile.getFilePath();
  const fileName = path.basename(filePath, path.extname(filePath));

  const context: FileContext = {
    isPage: fileName === 'page',
    isLayout: fileName === 'layout',
    isLoading: fileName === 'loading',
    isError: fileName === 'error',
    isRouteHandler: fileName === 'route',
    isServerAction: hasUseServerDirective(sourceFile),
    isServerComponent: !hasUseClientDirective(sourceFile),
    isClientComponent: hasUseClientDirective(sourceFile),
    isMiddleware: fileName === 'middleware',
    routeSegment: extractRouteSegment(filePath),
    routeConfig: extractRouteSegmentConfig(sourceFile),
  };

  return context;
}

function extractRouteSegmentConfig(sourceFile: SourceFile): RouteSegmentConfig {
  const config: RouteSegmentConfig = {};

  // Find exported const declarations
  const exportedDecls = sourceFile.getExportedDeclarations();

  for (const [name, decls] of exportedDecls) {
    for (const decl of decls) {
      if (!Node.isVariableDeclaration(decl)) continue;

      const initializer = decl.getInitializer();
      if (!initializer) continue;

      switch (name) {
        case 'dynamic':
          if (Node.isStringLiteral(initializer)) {
            config.dynamic = initializer.getLiteralText() as RouteSegmentConfig['dynamic'];
          }
          break;

        case 'revalidate':
          if (Node.isNumericLiteral(initializer)) {
            config.revalidate = parseInt(initializer.getText(), 10);
          } else if (initializer.getText() === 'false') {
            config.revalidate = false;
          }
          break;

        case 'fetchCache':
          if (Node.isStringLiteral(initializer)) {
            config.fetchCache = initializer.getLiteralText();
          }
          break;

        case 'runtime':
          if (Node.isStringLiteral(initializer)) {
            config.runtime = initializer.getLiteralText() as 'edge' | 'nodejs';
          }
          break;
      }
    }
  }

  return config;
}

function extractRouteSegment(filePath: string): string {
  // Extract route segment from file path
  // e.g., /app/dashboard/settings/page.tsx -> /dashboard/settings

  const appMatch = filePath.match(/\/app(.+)\/(?:page|layout|route|loading|error)\.[jt]sx?$/);
  if (appMatch) {
    return appMatch[1] || '/';
  }

  const pagesMatch = filePath.match(/\/pages(.+)\.[jt]sx?$/);
  if (pagesMatch) {
    return pagesMatch[1].replace(/\/index$/, '') || '/';
  }

  return '/';
}
```

### Checking Route Structure

```typescript
interface RouteStructureAnalysis {
  hasPage: boolean;
  hasLayout: boolean;
  hasLoading: boolean;
  hasError: boolean;
  hasTemplate: boolean;
  hasRouteHandler: boolean;
  children: string[];
  issues: RouteStructureIssue[];
}

interface RouteStructureIssue {
  type: 'missing-loading' | 'missing-error' | 'conflicting-files' | 'missing-layout';
  description: string;
  suggestion: string;
}

function analyzeRouteStructure(routeDir: string, project: Project): RouteStructureAnalysis {
  const files = fs.readdirSync(routeDir);

  const analysis: RouteStructureAnalysis = {
    hasPage: files.some(f => f.startsWith('page.')),
    hasLayout: files.some(f => f.startsWith('layout.')),
    hasLoading: files.some(f => f.startsWith('loading.')),
    hasError: files.some(f => f.startsWith('error.')),
    hasTemplate: files.some(f => f.startsWith('template.')),
    hasRouteHandler: files.some(f => f.startsWith('route.')),
    children: files.filter(f => {
      const fullPath = path.join(routeDir, f);
      return fs.statSync(fullPath).isDirectory();
    }),
    issues: [],
  };

  // Check for conflicting files
  if (analysis.hasPage && analysis.hasRouteHandler) {
    analysis.issues.push({
      type: 'conflicting-files',
      description: 'Route has both page.tsx and route.ts',
      suggestion: 'Remove one - a route cannot be both a page and an API route',
    });
  }

  // Check for async page without loading
  if (analysis.hasPage && !analysis.hasLoading) {
    const pageFile = files.find(f => f.startsWith('page.'));
    if (pageFile) {
      const pagePath = path.join(routeDir, pageFile);
      const sourceFile = project.getSourceFile(pagePath);
      if (sourceFile && hasAsyncDataFetching(sourceFile)) {
        analysis.issues.push({
          type: 'missing-loading',
          description: 'Async page without loading.tsx',
          suggestion: 'Add loading.tsx for better UX during data fetching',
        });
      }
    }
  }

  // Check for data fetching without error boundary
  if (analysis.hasPage && !analysis.hasError) {
    const pageFile = files.find(f => f.startsWith('page.'));
    if (pageFile) {
      const pagePath = path.join(routeDir, pageFile);
      const sourceFile = project.getSourceFile(pagePath);
      if (sourceFile && hasExternalDataFetching(sourceFile)) {
        analysis.issues.push({
          type: 'missing-error',
          description: 'Page with external data fetching without error.tsx',
          suggestion: 'Add error.tsx to handle fetch failures gracefully',
        });
      }
    }
  }

  return analysis;
}

function hasAsyncDataFetching(sourceFile: SourceFile): boolean {
  const defaultExport = sourceFile.getDefaultExportSymbol();
  if (!defaultExport) return false;

  const declarations = defaultExport.getDeclarations();
  for (const decl of declarations) {
    // Check if function is async
    if (Node.isFunctionDeclaration(decl) || Node.isArrowFunction(decl)) {
      if (decl.isAsync()) return true;
    }

    // Check if variable assigned async function
    if (Node.isVariableDeclaration(decl)) {
      const init = decl.getInitializer();
      if (init && (Node.isArrowFunction(init) || Node.isFunctionExpression(init))) {
        if (init.isAsync()) return true;
      }
    }
  }

  return false;
}

function hasExternalDataFetching(sourceFile: SourceFile): boolean {
  let hasFetch = false;

  sourceFile.forEachDescendant(node => {
    if (Node.isCallExpression(node)) {
      const callee = node.getExpression();
      if (Node.isIdentifier(callee) && callee.getText() === 'fetch') {
        hasFetch = true;
      }
    }
  });

  return hasFetch;
}
```

## Core Technique 6: Pages Router Specific Analysis

### Analyzing Data Fetching Functions

```typescript
interface PagesDataFetchingAnalysis {
  hasGetStaticProps: boolean;
  hasGetServerSideProps: boolean;
  hasGetStaticPaths: boolean;
  hasGetInitialProps: boolean;
  isDynamicRoute: boolean;
  issues: PagesDataFetchingIssue[];
}

interface PagesDataFetchingIssue {
  type: 'missing-paths' | 'conflicting-methods' | 'unnecessary-ssr' | 'missing-revalidate';
  description: string;
  suggestion: string;
}

function analyzePagesDataFetching(sourceFile: SourceFile): PagesDataFetchingAnalysis {
  const filePath = sourceFile.getFilePath();
  const isDynamicRoute = /\[.+\]/.test(path.basename(filePath));

  const analysis: PagesDataFetchingAnalysis = {
    hasGetStaticProps: hasExportedFunction(sourceFile, 'getStaticProps'),
    hasGetServerSideProps: hasExportedFunction(sourceFile, 'getServerSideProps'),
    hasGetStaticPaths: hasExportedFunction(sourceFile, 'getStaticPaths'),
    hasGetInitialProps: hasGetInitialProps(sourceFile),
    isDynamicRoute,
    issues: [],
  };

  // Check for conflicting data methods
  if (analysis.hasGetStaticProps && analysis.hasGetServerSideProps) {
    analysis.issues.push({
      type: 'conflicting-methods',
      description: 'Both getStaticProps and getServerSideProps defined',
      suggestion: 'Use only one data fetching method per page',
    });
  }

  // Check for missing getStaticPaths on dynamic routes
  if (isDynamicRoute && analysis.hasGetStaticProps && !analysis.hasGetStaticPaths) {
    analysis.issues.push({
      type: 'missing-paths',
      description: 'Dynamic route with getStaticProps but no getStaticPaths',
      suggestion: 'Add getStaticPaths to define which paths to pre-render',
    });
  }

  // Check for getStaticProps without revalidate
  if (analysis.hasGetStaticProps) {
    const gspFunc = findExportedFunction(sourceFile, 'getStaticProps');
    if (gspFunc && !hasRevalidateReturn(gspFunc)) {
      analysis.issues.push({
        type: 'missing-revalidate',
        description: 'getStaticProps without revalidate option',
        suggestion: 'Consider adding revalidate for ISR if data changes',
      });
    }
  }

  return analysis;
}

function hasExportedFunction(sourceFile: SourceFile, name: string): boolean {
  const exportedDecls = sourceFile.getExportedDeclarations();
  return exportedDecls.has(name);
}

function findExportedFunction(sourceFile: SourceFile, name: string): Node | undefined {
  const exportedDecls = sourceFile.getExportedDeclarations();
  const decls = exportedDecls.get(name);
  return decls?.[0];
}

function hasGetInitialProps(sourceFile: SourceFile): boolean {
  const defaultExport = sourceFile.getDefaultExportSymbol();
  if (!defaultExport) return false;

  const declarations = defaultExport.getDeclarations();
  for (const decl of declarations) {
    // Check for Component.getInitialProps
    const symbol = decl.getSymbol();
    if (symbol) {
      const type = symbol.getTypeAtLocation(decl);
      const property = type.getProperty('getInitialProps');
      if (property) return true;
    }
  }

  return false;
}

function hasRevalidateReturn(func: Node): boolean {
  let hasRevalidate = false;

  func.forEachDescendant(node => {
    if (Node.isReturnStatement(node)) {
      const expr = node.getExpression();
      if (expr && Node.isObjectLiteralExpression(expr)) {
        const revalidateProp = expr.getProperty('revalidate');
        if (revalidateProp) {
          hasRevalidate = true;
        }
      }
    }
  });

  return hasRevalidate;
}
```

## Utility Functions

```typescript
function getLocation(node: Node): { line: number; column: number } {
  const start = node.getStart();
  const sourceFile = node.getSourceFile();
  const { line, column } = sourceFile.getLineAndColumnAtPos(start);
  return { line, column };
}

function getJsxAttributes(jsxElement: Node): Node[] {
  if (Node.isJsxElement(jsxElement)) {
    return jsxElement.getOpeningElement().getAttributes();
  }
  if (Node.isJsxSelfClosingElement(jsxElement)) {
    return jsxElement.getAttributes();
  }
  return [];
}

function findAwaitExpressions(func: Node): Node[] {
  const awaits: Node[] = [];

  func.forEachDescendant(node => {
    if (Node.isAwaitExpression(node)) {
      awaits.push(node);
    }
  });

  return awaits;
}

function findFetchInExpression(node: Node): Node | null {
  if (Node.isCallExpression(node)) {
    const callee = node.getExpression();
    if (Node.isIdentifier(callee) && callee.getText() === 'fetch') {
      return node;
    }
  }

  let fetchNode: Node | null = null;
  node.forEachDescendant(desc => {
    if (Node.isCallExpression(desc)) {
      const callee = desc.getExpression();
      if (Node.isIdentifier(callee) && callee.getText() === 'fetch') {
        fetchNode = desc;
      }
    }
  });

  return fetchNode;
}

function getAssignedVariable(awaitExpr: Node): string | null {
  const parent = awaitExpr.getParent();

  if (Node.isVariableDeclaration(parent)) {
    return parent.getName();
  }

  if (Node.isPropertyAssignment(parent)) {
    return parent.getName();
  }

  return null;
}

function extractUrlString(urlArg: Node): string | 'dynamic' {
  if (Node.isStringLiteral(urlArg)) {
    return urlArg.getLiteralText();
  }
  if (Node.isTemplateExpression(urlArg)) {
    return 'dynamic';
  }
  if (Node.isNoSubstitutionTemplateLiteral(urlArg)) {
    return urlArg.getLiteralText();
  }
  return 'dynamic';
}
```
