# ts-morph Deep Analysis Techniques for React Static Analysis

This document covers advanced ts-morph patterns for semantic analysis beyond simple AST matching.

## Foundation: Project Setup for Analysis

```typescript
import { Project, Node, SyntaxKind, Type, Symbol as TsSymbol } from 'ts-morph';

function createAnalysisProject(files: string[], tsConfigPath?: string): Project {
  const project = new Project({
    tsConfigFilePath: tsConfigPath,
    skipAddingFilesFromTsConfig: true,
  });

  project.addSourceFilesAtPaths(files);

  // Essential: resolve all types before analysis
  project.resolveSourceFileDependencies();

  return project;
}
```

## Core Technique 1: Type-Aware Analysis

### Getting Real Types (Not Just Syntax)

```typescript
function getResolvedType(node: Node): Type {
  return node.getType();
}

function isReactComponent(node: Node): boolean {
  const type = node.getType();
  const typeText = type.getText();

  // Check for React component signatures
  return (
    typeText.includes('React.FC') ||
    typeText.includes('React.Component') ||
    typeText.includes('JSX.Element') ||
    type.getCallSignatures().some(sig => {
      const returnType = sig.getReturnType();
      return (
        returnType.getText().includes('JSX.Element') || returnType.getText().includes('ReactNode')
      );
    })
  );
}

function isHookCall(node: Node): boolean {
  if (!Node.isCallExpression(node)) return false;

  const expression = node.getExpression();
  const name = expression.getText();

  // Hooks start with 'use' and are called at component level
  return /^use[A-Z]/.test(name);
}
```

### Distinguishing Stable vs Unstable References

```typescript
type Stability = 'stable' | 'unstable' | 'unknown';

function analyzeReferenceStability(identifier: Node): Stability {
  const symbol = identifier.getSymbol();
  if (!symbol) return 'unknown';

  const declarations = symbol.getDeclarations();
  if (declarations.length === 0) return 'unknown';

  for (const decl of declarations) {
    // Module-level const is stable
    if (Node.isVariableDeclaration(decl)) {
      const statement = decl.getVariableStatement();
      if (statement?.getDeclarationKind() === VariableDeclarationKind.Const) {
        const parent = statement.getParent();
        if (Node.isSourceFile(parent)) {
          return 'stable';
        }
      }
    }

    // useRef().current is stable
    if (isUseRefAccess(decl)) {
      return 'stable';
    }

    // useState dispatch function is stable
    if (isUseStateDispatch(decl)) {
      return 'stable';
    }

    // useCallback/useMemo result is stable (within deps)
    if (isMemoizedValue(decl)) {
      return 'stable';
    }

    // Object/array literal in render scope is unstable
    if (isRenderScopeLiteral(decl)) {
      return 'unstable';
    }
  }

  return 'unknown';
}

function isUseRefAccess(node: Node): boolean {
  // Pattern: const ref = useRef(...); ... ref.current
  if (!Node.isVariableDeclaration(node)) return false;

  const initializer = node.getInitializer();
  if (!Node.isCallExpression(initializer)) return false;

  const callee = initializer.getExpression().getText();
  return callee === 'useRef' || callee === 'React.useRef';
}

function isUseStateDispatch(node: Node): boolean {
  // Pattern: const [state, setState] = useState(...)
  // setState (second element) is stable
  if (!Node.isBindingElement(node)) return false;

  const parent = node.getParent();
  if (!Node.isArrayBindingPattern(parent)) return false;

  const elements = parent.getElements();
  if (elements.indexOf(node) !== 1) return false; // Must be second element

  const declaration = parent.getParent();
  if (!Node.isVariableDeclaration(declaration)) return false;

  const initializer = declaration.getInitializer();
  if (!Node.isCallExpression(initializer)) return false;

  const callee = initializer.getExpression().getText();
  return callee === 'useState' || callee === 'React.useState';
}
```

## Core Technique 2: Data Flow Analysis

### Tracing Value Origins

```typescript
function traceValueOrigin(node: Node): ValueOrigin {
  // Direct literal
  if (Node.isLiteralExpression(node)) {
    return { type: 'literal', stable: true };
  }

  // Identifier - trace to declaration
  if (Node.isIdentifier(node)) {
    const symbol = node.getSymbol();
    const declarations = symbol?.getDeclarations() ?? [];

    for (const decl of declarations) {
      if (Node.isVariableDeclaration(decl)) {
        const init = decl.getInitializer();
        if (init) {
          return traceValueOrigin(init);
        }
      }
      if (Node.isParameter(decl)) {
        return { type: 'parameter', stable: false };
      }
    }
  }

  // Object/array literal
  if (Node.isObjectLiteralExpression(node) || Node.isArrayLiteralExpression(node)) {
    const scope = findContainingScope(node);
    return {
      type: 'literal',
      stable: scope === 'module', // Module scope = stable, render scope = unstable
    };
  }

  // Function call
  if (Node.isCallExpression(node)) {
    const callee = node.getExpression().getText();

    // Known stable factories
    if (['useRef', 'useCallback', 'useMemo'].includes(callee)) {
      return { type: 'hook-result', stable: true };
    }

    return { type: 'call', stable: false };
  }

  return { type: 'unknown', stable: false };
}

type ValueOrigin = {
  type: 'literal' | 'parameter' | 'hook-result' | 'call' | 'unknown';
  stable: boolean;
};
```

### Finding All References and Usages

```typescript
function findAllUsages(symbol: TsSymbol): Node[] {
  const usages: Node[] = [];

  for (const ref of symbol.findReferences()) {
    for (const refEntry of ref.getReferences()) {
      usages.push(refEntry.getNode());
    }
  }

  return usages;
}

function findPropConsumers(component: Node, propName: string): PropConsumer[] {
  const consumers: PropConsumer[] = [];

  // Find all JSX usages of this component
  const componentName = getComponentName(component);
  const project = component.getProject();

  for (const sourceFile of project.getSourceFiles()) {
    const jsxElements = sourceFile.getDescendantsOfKind(SyntaxKind.JsxElement);
    const jsxSelfClosing = sourceFile.getDescendantsOfKind(SyntaxKind.JsxSelfClosingElement);

    for (const jsx of [...jsxElements, ...jsxSelfClosing]) {
      const tagName = getJsxTagName(jsx);
      if (tagName !== componentName) continue;

      const propAttribute = findJsxAttribute(jsx, propName);
      if (propAttribute) {
        consumers.push({
          component: jsx,
          attribute: propAttribute,
          value: getAttributeValue(propAttribute),
        });
      }
    }
  }

  return consumers;
}
```

## Core Technique 3: Control Flow Analysis

### Analyzing Conditional Execution

```typescript
function isConditionallyExecuted(node: Node): boolean {
  let current: Node | undefined = node;

  while (current) {
    const parent = current.getParent();

    if (Node.isIfStatement(parent)) {
      // Check if we're in then/else branch
      if (current === parent.getThenStatement() || current === parent.getElseStatement()) {
        return true;
      }
    }

    if (Node.isConditionalExpression(parent)) {
      if (current === parent.getWhenTrue() || current === parent.getWhenFalse()) {
        return true;
      }
    }

    if (Node.isBinaryExpression(parent)) {
      const operator = parent.getOperatorToken().getText();
      if ((operator === '&&' || operator === '||') && current === parent.getRight()) {
        return true;
      }
    }

    current = parent;
  }

  return false;
}
```

### Detecting Loops and Complexity

```typescript
function analyzeComputationalComplexity(node: Node): ComplexityResult {
  let loopDepth = 0;
  let maxLoopDepth = 0;
  let hasRecursion = false;
  let arrayMethodChains = 0;

  node.forEachDescendant((descendant, traversal) => {
    // Loop detection
    if (
      Node.isForStatement(descendant) ||
      Node.isForInStatement(descendant) ||
      Node.isForOfStatement(descendant) ||
      Node.isWhileStatement(descendant) ||
      Node.isDoStatement(descendant)
    ) {
      loopDepth++;
      maxLoopDepth = Math.max(maxLoopDepth, loopDepth);
    }

    // Array method detection
    if (Node.isCallExpression(descendant)) {
      const expression = descendant.getExpression();
      if (Node.isPropertyAccessExpression(expression)) {
        const methodName = expression.getName();
        if (['map', 'filter', 'reduce', 'sort', 'find', 'forEach'].includes(methodName)) {
          arrayMethodChains++;
        }
      }

      // Recursion detection
      const callee = descendant.getExpression();
      if (Node.isIdentifier(callee)) {
        const containingFunction = descendant.getFirstAncestor(
          n =>
            Node.isFunctionDeclaration(n) || Node.isFunctionExpression(n) || Node.isArrowFunction(n)
        );
        if (containingFunction) {
          const funcName = getFunctionName(containingFunction);
          if (funcName && callee.getText() === funcName) {
            hasRecursion = true;
          }
        }
      }
    }
  });

  return {
    maxLoopDepth,
    hasRecursion,
    arrayMethodChains,
    estimatedComplexity: calculateComplexity(maxLoopDepth, hasRecursion, arrayMethodChains),
  };
}

function calculateComplexity(
  loopDepth: number,
  hasRecursion: boolean,
  arrayMethods: number
): 'trivial' | 'low' | 'medium' | 'high' {
  if (hasRecursion || loopDepth >= 2) return 'high';
  if (loopDepth === 1 || arrayMethods >= 2) return 'medium';
  if (arrayMethods === 1) return 'low';
  return 'trivial';
}
```

## Core Technique 4: React-Specific Analysis

### Identifying React Components

```typescript
function isReactFunctionComponent(node: Node): boolean {
  if (
    !Node.isFunctionDeclaration(node) &&
    !Node.isFunctionExpression(node) &&
    !Node.isArrowFunction(node)
  ) {
    return false;
  }

  // Check return type
  const returnType = node.getReturnType();
  if (returnsJSX(returnType)) return true;

  // Check body for JSX returns
  const body = node.getBody();
  if (!body) return false;

  if (Node.isBlock(body)) {
    return body.getStatements().some(stmt => {
      if (Node.isReturnStatement(stmt)) {
        const expr = stmt.getExpression();
        return (
          expr &&
          (Node.isJsxElement(expr) ||
            Node.isJsxSelfClosingElement(expr) ||
            Node.isJsxFragment(expr))
        );
      }
      return false;
    });
  }

  // Arrow function with expression body
  return Node.isJsxElement(body) || Node.isJsxSelfClosingElement(body) || Node.isJsxFragment(body);
}

function returnsJSX(type: Type): boolean {
  const typeText = type.getText();
  return (
    typeText.includes('JSX.Element') ||
    typeText.includes('ReactElement') ||
    typeText.includes('ReactNode')
  );
}
```

### Analyzing Hook Dependencies

```typescript
interface HookDependencyAnalysis {
  hook: string;
  dependencies: DependencyInfo[];
  missingDeps: string[];
  unnecessaryDeps: string[];
  unstableDeps: string[];
}

interface DependencyInfo {
  name: string;
  node: Node;
  stability: Stability;
  usedInCallback: boolean;
}

function analyzeHookDependencies(callExpr: Node): HookDependencyAnalysis | null {
  if (!Node.isCallExpression(callExpr)) return null;

  const callee = callExpr.getExpression().getText();
  if (!['useEffect', 'useCallback', 'useMemo', 'useLayoutEffect'].includes(callee)) {
    return null;
  }

  const args = callExpr.getArguments();
  if (args.length < 1) return null;

  const callback = args[0];
  const depsArray = args[1];

  // Find all identifiers used in callback
  const usedIdentifiers = findReferencedIdentifiers(callback);

  // Parse dependency array
  const declaredDeps =
    depsArray && Node.isArrayLiteralExpression(depsArray)
      ? depsArray.getElements().map(e => e.getText())
      : [];

  // Analyze each dependency
  const dependencies: DependencyInfo[] = [];
  const missingDeps: string[] = [];
  const unnecessaryDeps: string[] = [];
  const unstableDeps: string[] = [];

  // Check for missing deps
  for (const identifier of usedIdentifiers) {
    const name = identifier.getText();
    const stability = analyzeReferenceStability(identifier);

    // Skip stable references (they don't need to be in deps)
    if (stability === 'stable') continue;

    if (!declaredDeps.includes(name)) {
      missingDeps.push(name);
    }

    dependencies.push({
      name,
      node: identifier,
      stability,
      usedInCallback: true,
    });

    if (stability === 'unstable') {
      unstableDeps.push(name);
    }
  }

  // Check for unnecessary deps
  for (const dep of declaredDeps) {
    const isUsed = usedIdentifiers.some(id => id.getText() === dep);
    if (!isUsed) {
      unnecessaryDeps.push(dep);
    }
  }

  return {
    hook: callee,
    dependencies,
    missingDeps,
    unnecessaryDeps,
    unstableDeps,
  };
}

function findReferencedIdentifiers(node: Node): Node[] {
  const identifiers: Node[] = [];
  const localDeclarations = new Set<string>();

  // First pass: collect local declarations
  node.forEachDescendant(desc => {
    if (Node.isVariableDeclaration(desc)) {
      localDeclarations.add(desc.getName());
    }
    if (Node.isParameter(desc)) {
      localDeclarations.add(desc.getName());
    }
  });

  // Second pass: collect external references
  node.forEachDescendant(desc => {
    if (Node.isIdentifier(desc)) {
      const name = desc.getText();

      // Skip if locally declared
      if (localDeclarations.has(name)) return;

      // Skip if it's a property access target
      const parent = desc.getParent();
      if (Node.isPropertyAccessExpression(parent) && parent.getNameNode() === desc) {
        return;
      }

      // Skip known globals
      if (['console', 'window', 'document', 'Math', 'JSON', 'Object', 'Array'].includes(name)) {
        return;
      }

      identifiers.push(desc);
    }
  });

  return identifiers;
}
```

### Detecting Component Definitions Inside Components

```typescript
function findNestedComponentDefinitions(component: Node): NestedComponent[] {
  const nested: NestedComponent[] = [];

  if (!isReactFunctionComponent(component)) return nested;

  const body = getFunctionBody(component);
  if (!body) return nested;

  body.forEachDescendant(desc => {
    // Skip callbacks to hooks (those are fine)
    if (isHookCallback(desc)) return;

    if (isReactFunctionComponent(desc)) {
      nested.push({
        node: desc,
        name: getFunctionName(desc) || '<anonymous>',
        hasState: containsStateHooks(desc),
      });
    }
  });

  return nested;
}

function isHookCallback(node: Node): boolean {
  const parent = node.getParent();
  if (!Node.isCallExpression(parent)) return false;

  const callee = parent.getExpression().getText();
  return ['useEffect', 'useCallback', 'useMemo', 'useLayoutEffect'].includes(callee);
}

function containsStateHooks(component: Node): boolean {
  let hasState = false;

  component.forEachDescendant(desc => {
    if (Node.isCallExpression(desc)) {
      const callee = desc.getExpression().getText();
      if (['useState', 'useReducer', 'useRef'].includes(callee)) {
        hasState = true;
      }
    }
  });

  return hasState;
}
```

## Core Technique 5: Cross-File Analysis

### Building Component Dependency Graph

```typescript
interface ComponentNode {
  name: string;
  file: string;
  children: string[];
  parents: string[];
  props: PropDefinition[];
  isMemoized: boolean;
}

function buildComponentGraph(project: Project): Map<string, ComponentNode> {
  const graph = new Map<string, ComponentNode>();

  for (const sourceFile of project.getSourceFiles()) {
    sourceFile.forEachDescendant(node => {
      if (isReactFunctionComponent(node)) {
        const name = getComponentName(node);
        if (!name) return;

        const componentNode: ComponentNode = {
          name,
          file: sourceFile.getFilePath(),
          children: findChildComponents(node),
          parents: [], // Filled in second pass
          props: extractPropDefinitions(node),
          isMemoized: isWrappedInMemo(node),
        };

        graph.set(name, componentNode);
      }
    });
  }

  // Second pass: fill in parent relationships
  for (const [name, node] of graph) {
    for (const childName of node.children) {
      const child = graph.get(childName);
      if (child) {
        child.parents.push(name);
      }
    }
  }

  return graph;
}

function findChildComponents(component: Node): string[] {
  const children: string[] = [];

  component.forEachDescendant(desc => {
    if (Node.isJsxElement(desc) || Node.isJsxSelfClosingElement(desc)) {
      const tagName = getJsxTagName(desc);
      // Custom components start with uppercase
      if (/^[A-Z]/.test(tagName)) {
        children.push(tagName);
      }
    }
  });

  return [...new Set(children)];
}

function isWrappedInMemo(component: Node): boolean {
  // Check for React.memo wrapper
  // Pattern: export const Foo = React.memo(function Foo() { ... })
  // Pattern: export const Foo = memo(FooComponent)

  const parent = component.getParent();
  if (Node.isCallExpression(parent)) {
    const callee = parent.getExpression().getText();
    return callee === 'memo' || callee === 'React.memo';
  }

  return false;
}
```

### Tracking Prop Flow Through Component Tree

```typescript
interface PropFlowAnalysis {
  prop: string;
  source: string; // Component where prop originates
  path: string[]; // Components it passes through
  consumers: string[]; // Components that actually use it
  depth: number;
}

function analyzePropDrilling(
  graph: Map<string, ComponentNode>,
  propName: string,
  sourceComponent: string
): PropFlowAnalysis {
  const visited = new Set<string>();
  const path: string[] = [];
  const consumers: string[] = [];

  function traverse(componentName: string, depth: number): number {
    if (visited.has(componentName)) return depth;
    visited.add(componentName);

    const component = graph.get(componentName);
    if (!component) return depth;

    const hasProp = component.props.some(p => p.name === propName);
    if (!hasProp) return depth;

    path.push(componentName);

    // Check if this component uses the prop or just passes it
    const usesProp = componentUsesProp(component, propName);
    if (usesProp) {
      consumers.push(componentName);
    }

    // Traverse children
    let maxChildDepth = depth;
    for (const childName of component.children) {
      const childComponent = graph.get(childName);
      if (childComponent?.props.some(p => p.name === propName)) {
        maxChildDepth = Math.max(maxChildDepth, traverse(childName, depth + 1));
      }
    }

    return maxChildDepth;
  }

  const maxDepth = traverse(sourceComponent, 0);

  return {
    prop: propName,
    source: sourceComponent,
    path,
    consumers,
    depth: maxDepth,
  };
}
```

## Utility Functions

```typescript
function getComponentName(node: Node): string | undefined {
  if (Node.isFunctionDeclaration(node)) {
    return node.getName();
  }

  // const Foo = () => { ... }
  const parent = node.getParent();
  if (Node.isVariableDeclaration(parent)) {
    return parent.getName();
  }

  return undefined;
}

function getFunctionBody(node: Node): Node | undefined {
  if (
    Node.isFunctionDeclaration(node) ||
    Node.isFunctionExpression(node) ||
    Node.isArrowFunction(node)
  ) {
    return node.getBody();
  }
  return undefined;
}

function getFunctionName(node: Node): string | undefined {
  if (Node.isFunctionDeclaration(node)) {
    return node.getName();
  }
  if (Node.isFunctionExpression(node)) {
    return node.getName();
  }
  // For arrow functions, check parent variable declaration
  const parent = node.getParent();
  if (Node.isVariableDeclaration(parent)) {
    return parent.getName();
  }
  return undefined;
}

function getJsxTagName(jsx: Node): string {
  if (Node.isJsxElement(jsx)) {
    const opening = jsx.getOpeningElement();
    return opening.getTagNameNode().getText();
  }
  if (Node.isJsxSelfClosingElement(jsx)) {
    return jsx.getTagNameNode().getText();
  }
  return '';
}

function findContainingScope(node: Node): 'module' | 'function' | 'block' {
  let current: Node | undefined = node.getParent();

  while (current) {
    if (Node.isSourceFile(current)) return 'module';
    if (
      Node.isFunctionDeclaration(current) ||
      Node.isFunctionExpression(current) ||
      Node.isArrowFunction(current)
    ) {
      return 'function';
    }
    if (Node.isBlock(current)) return 'block';
    current = current.getParent();
  }

  return 'module';
}
```
