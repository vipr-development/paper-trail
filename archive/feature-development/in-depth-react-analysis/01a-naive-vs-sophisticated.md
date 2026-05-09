# Naive vs Sophisticated Static Analysis: Side-by-Side Comparisons

This document contrasts surface-level pattern matching with semantic analysis for each major React anti-pattern. Use this to evaluate existing implementations and identify improvement opportunities.

## Pattern 1: useMemo/useCallback Opportunities

### Naive Implementation

```typescript
// Detects: Any function passed as prop that could be wrapped in useCallback
function detectMissingCallback(node: ts.Node) {
  if (ts.isJsxAttribute(node)) {
    const initializer = node.initializer;
    if (ts.isJsxExpression(initializer)) {
      const expr = initializer.expression;
      if (ts.isArrowFunction(expr) || ts.isFunctionExpression(expr)) {
        // Flag: "Function passed as prop should use useCallback"
        report(node, 'missing-use-callback');
      }
    }
  }
}
```

**Problems**:

- Flags ALL inline functions, even when optimization is unnecessary
- No consideration of whether child component is memoized
- No analysis of render frequency or cost
- Creates noise that leads developers to ignore the rule

**False positives**:

```tsx
// Flagged but optimization unnecessary
<button onClick={() => setCount(c => c + 1)}>Click</button>
<input onChange={e => setName(e.target.value)} />
<Link href={() => buildUrl(id)}>View</Link>
```

### Sophisticated Implementation

```typescript
interface CallbackAnalysis {
  shouldMemoize: boolean;
  confidence: 'high' | 'medium' | 'low';
  reason: string;
}

function analyzeCallbackMemoizationNeed(jsxAttr: JsxAttribute, project: Project): CallbackAnalysis {
  const propName = jsxAttr.getName();
  const jsxElement = jsxAttr.getParent().getParent();
  const componentName = getJsxTagName(jsxElement);

  // 1. Check if target component is memoized
  const targetComponent = findComponentDefinition(project, componentName);
  if (!targetComponent) {
    return { shouldMemoize: false, confidence: 'low', reason: 'Cannot resolve component' };
  }

  const isMemoized = isComponentMemoized(targetComponent);
  if (!isMemoized) {
    return {
      shouldMemoize: false,
      confidence: 'high',
      reason: 'Target component is not memoized; callback optimization would have no effect',
    };
  }

  // 2. Analyze render cost of target component
  const renderCost = analyzeComponentRenderCost(targetComponent);
  if (renderCost.score < RENDER_COST_THRESHOLD) {
    return {
      shouldMemoize: false,
      confidence: 'medium',
      reason: `Target component render cost is low (${renderCost.score}); memoization overhead likely exceeds benefit`,
    };
  }

  // 3. Analyze parent re-render frequency
  const parentComponent = findContainingComponent(jsxAttr);
  const updateFrequency = estimateUpdateFrequency(parentComponent);
  if (updateFrequency === 'rare') {
    return {
      shouldMemoize: false,
      confidence: 'medium',
      reason: 'Parent component updates infrequently; memoization overhead likely exceeds benefit',
    };
  }

  // 4. Check for existing memoization chain
  const hasMemoChain = targetComponentDependsOnPropReference(targetComponent, propName);
  if (!hasMemoChain) {
    return {
      shouldMemoize: false,
      confidence: 'high',
      reason:
        'Target component does not use prop in dependency array; referential stability not required',
    };
  }

  // All conditions met: memoization would help
  return {
    shouldMemoize: true,
    confidence: 'high',
    reason:
      'Memoized component depends on callback reference; memoization would prevent unnecessary re-renders',
  };
}

function isComponentMemoized(component: Node): boolean {
  // Check for React.memo wrapper
  const parent = component.getParent();
  if (Node.isCallExpression(parent)) {
    const callee = parent.getExpression().getText();
    if (callee === 'memo' || callee === 'React.memo') return true;
  }

  // Check for export wrapped in memo
  const variableDecl = component.getParent();
  if (Node.isVariableDeclaration(variableDecl)) {
    const init = variableDecl.getInitializer();
    if (Node.isCallExpression(init)) {
      const callee = init.getExpression().getText();
      if (callee === 'memo' || callee === 'React.memo') return true;
    }
  }

  return false;
}

interface RenderCostAnalysis {
  score: number; // 0-100
  factors: string[];
}

function analyzeComponentRenderCost(component: Node): RenderCostAnalysis {
  const factors: string[] = [];
  let score = 0;

  // Factor 1: JSX tree depth
  const jsxDepth = measureJsxTreeDepth(component);
  if (jsxDepth > 5) {
    score += 20;
    factors.push(`Deep JSX tree (${jsxDepth} levels)`);
  }

  // Factor 2: Number of child components
  const childCount = countChildComponents(component);
  if (childCount > 10) {
    score += 25;
    factors.push(`Many child components (${childCount})`);
  }

  // Factor 3: Expensive computations in render
  const complexity = analyzeComputationalComplexity(component);
  if (complexity.estimatedComplexity === 'high') {
    score += 30;
    factors.push('Contains expensive computations');
  }

  // Factor 4: Context consumers
  const contextConsumers = countContextConsumers(component);
  if (contextConsumers > 2) {
    score += 15;
    factors.push(`Multiple context consumers (${contextConsumers})`);
  }

  // Factor 5: Dynamic list rendering
  if (hasMapWithJsx(component)) {
    score += 10;
    factors.push('Renders dynamic lists');
  }

  return { score, factors };
}
```

### Key Differences

| Aspect                    | Naive     | Sophisticated                                 |
| ------------------------- | --------- | --------------------------------------------- |
| Target analysis           | None      | Checks if consumer is memoized                |
| Cost-benefit              | Ignored   | Estimates render cost vs memoization overhead |
| Confidence                | Binary    | Graduated with reasoning                      |
| False positive rate       | High      | Low                                           |
| Implementation complexity | ~10 lines | ~100 lines                                    |

---

## Pattern 2: useEffect Dependencies

### Naive Implementation

```typescript
// Detects: Any identifier used in effect body not in dependency array
function checkEffectDeps(effectCall: ts.CallExpression) {
  const callback = effectCall.arguments[0];
  const depsArray = effectCall.arguments[1];

  const usedIdentifiers = findAllIdentifiers(callback);
  const declaredDeps = getDepsArrayElements(depsArray);

  for (const identifier of usedIdentifiers) {
    if (!declaredDeps.includes(identifier.text)) {
      report(identifier, 'missing-dependency');
    }
  }
}
```

**Problems**:

- Flags stable values (dispatch, refs, module constants)
- Doesn't understand intentional omissions
- No distinction between value dependencies and reference dependencies
- Generates false positives that lead to infinite loops when "fixed"

**False positives**:

```tsx
// dispatch is stable - should not be in deps
const [state, dispatch] = useReducer(reducer, initial);
useEffect(() => {
  dispatch({ type: 'INIT' }); // Flagged: "dispatch missing from dependencies"
}, []);

// ref.current access is intentional
const ref = useRef(callback);
useEffect(() => {
  ref.current = callback; // Don't want this to trigger effect
}, [callback]);

// Module constant is stable
import { API_URL } from './config';
useEffect(() => {
  fetch(API_URL + endpoint); // Flagged: "API_URL missing"
}, [endpoint]);
```

### Sophisticated Implementation

```typescript
interface DepAnalysis {
  identifier: string;
  category: 'required' | 'optional' | 'stable' | 'intentional-omission';
  reason: string;
  autoFixSafe: boolean;
}

function analyzeEffectDependencies(effectCall: Node): DepAnalysis[] {
  const callback = effectCall.getArguments()[0];
  const depsArray = effectCall.getArguments()[1];

  const usedIdentifiers = findReferencedIdentifiers(callback);
  const declaredDeps = parseDepsArray(depsArray);
  const analyses: DepAnalysis[] = [];

  for (const identifier of usedIdentifiers) {
    const name = identifier.getText();
    const analysis = categorizeIdentifier(identifier, callback);

    if (!declaredDeps.has(name)) {
      analyses.push({
        identifier: name,
        ...analysis,
        autoFixSafe: analysis.category === 'required',
      });
    }
  }

  return analyses;
}

function categorizeIdentifier(
  identifier: Node,
  effectCallback: Node
): Omit<DepAnalysis, 'identifier' | 'autoFixSafe'> {
  const symbol = identifier.getSymbol();
  if (!symbol) {
    return { category: 'required', reason: 'Cannot resolve symbol' };
  }

  const declarations = symbol.getDeclarations();

  for (const decl of declarations) {
    // 1. Check for module-level constants
    if (isModuleScopeConstant(decl)) {
      return {
        category: 'stable',
        reason: 'Module-level constant never changes',
      };
    }

    // 2. Check for useReducer dispatch
    if (isReducerDispatch(decl)) {
      return {
        category: 'stable',
        reason: 'useReducer dispatch function is referentially stable',
      };
    }

    // 3. Check for useRef
    if (isRefValue(decl)) {
      // Distinguish ref object vs ref.current
      if (isRefCurrentAccess(identifier)) {
        return {
          category: 'intentional-omission',
          reason: 'ref.current is intentionally excluded to avoid effect re-runs',
        };
      }
      return {
        category: 'stable',
        reason: 'useRef returns stable reference',
      };
    }

    // 4. Check for useState setter
    if (isStateSetterFunction(decl)) {
      return {
        category: 'stable',
        reason: 'useState setter function is referentially stable',
      };
    }

    // 5. Check for known stable imports
    if (isKnownStableImport(decl)) {
      return {
        category: 'stable',
        reason: 'Imported from library known to provide stable references',
      };
    }

    // 6. Check for function that captures the identifier
    if (isInClosureThatShouldCaptureStale(identifier, effectCallback)) {
      return {
        category: 'intentional-omission',
        reason: 'Closure intentionally captures value at effect creation time',
      };
    }
  }

  // Default: requires being in dependency array
  return {
    category: 'required',
    reason: 'Value may change between renders',
  };
}

function isReducerDispatch(decl: Node): boolean {
  // Pattern: const [state, dispatch] = useReducer(...)
  if (!Node.isBindingElement(decl)) return false;

  const parent = decl.getParent();
  if (!Node.isArrayBindingPattern(parent)) return false;

  const elements = parent.getElements();
  const index = elements.indexOf(decl);
  if (index !== 1) return false; // dispatch is second element

  const varDecl = parent.getParent();
  if (!Node.isVariableDeclaration(varDecl)) return false;

  const init = varDecl.getInitializer();
  if (!Node.isCallExpression(init)) return false;

  const callee = init.getExpression().getText();
  return callee === 'useReducer' || callee === 'React.useReducer';
}

function isRefCurrentAccess(identifier: Node): boolean {
  const parent = identifier.getParent();
  if (!Node.isPropertyAccessExpression(parent)) return false;

  return parent.getName() === 'current' && parent.getExpression() === identifier;
}

function isKnownStableImport(decl: Node): boolean {
  const sourceFile = decl.getSourceFile();
  const importDecl = decl.getFirstAncestorByKind(SyntaxKind.ImportDeclaration);

  if (!importDecl) return false;

  const moduleSpecifier = importDecl.getModuleSpecifierValue();

  // Known libraries with stable exports
  const stableModules = [
    'react-router',
    'react-router-dom', // useNavigate, useLocation
    'react-redux', // useDispatch
    '@reduxjs/toolkit', // useDispatch
    'zustand', // store hooks
    'jotai', // atom setters
  ];

  // Known stable named exports
  const stableExports: Record<string, string[]> = {
    'react-router-dom': ['useNavigate', 'useParams'],
    'react-redux': ['useDispatch'],
  };

  if (stableModules.some(m => moduleSpecifier.includes(m))) {
    const importName = getImportedName(decl);
    const moduleStables = stableExports[moduleSpecifier];
    if (moduleStables?.includes(importName)) {
      return true;
    }
  }

  return false;
}
```

### Key Differences

| Aspect              | Naive           | Sophisticated                    |
| ------------------- | --------------- | -------------------------------- |
| Stability analysis  | None            | Checks dispatch, refs, constants |
| Library awareness   | None            | Recognizes stable imports        |
| Intent recognition  | None            | Identifies intentional omissions |
| Auto-fix safety     | Always suggests | Only for truly missing deps      |
| False positive rate | Very high       | Low                              |

---

## Pattern 3: Detecting Expensive Renders

### Naive Implementation

```typescript
// Flags: Any component without React.memo
function suggestMemo(component: ts.FunctionDeclaration) {
  if (isReactComponent(component) && !isWrappedInMemo(component)) {
    report(component, 'consider-react-memo');
  }
}
```

**Problems**:

- React.memo has overhead; only helps when props are stable
- Most components are cheap to render
- Creates busywork without performance benefit

### Sophisticated Implementation

```typescript
interface MemoRecommendation {
  recommend: boolean;
  confidence: 'high' | 'medium' | 'low';
  factors: MemoFactor[];
}

interface MemoFactor {
  type: 'positive' | 'negative';
  weight: number;
  description: string;
}

function analyzeMemoCandidate(component: Node): MemoRecommendation {
  const factors: MemoFactor[] = [];

  // Positive factors (suggest memoization)

  // 1. Parent re-renders frequently with unchanged props to this child
  const parentUpdatePattern = analyzeParentUpdatePattern(component);
  if (parentUpdatePattern.frequentUpdatesUnchangedProps) {
    factors.push({
      type: 'positive',
      weight: 30,
      description: 'Parent frequently re-renders with unchanged props to this component',
    });
  }

  // 2. Expensive render detected
  const renderCost = analyzeComponentRenderCost(component);
  if (renderCost.score > 50) {
    factors.push({
      type: 'positive',
      weight: renderCost.score * 0.5,
      description: `High render cost (${renderCost.factors.join(', ')})`,
    });
  }

  // 3. Pure component (no hooks that cause internal state changes)
  if (isPureComponent(component)) {
    factors.push({
      type: 'positive',
      weight: 15,
      description: 'Component is pure (output depends only on props)',
    });
  }

  // Negative factors (argue against memoization)

  // 1. Receives unstable props
  const propsStability = analyzePropsStability(component);
  if (propsStability.hasUnstableProps) {
    factors.push({
      type: 'negative',
      weight: 40,
      description: `Receives unstable props (${propsStability.unstableProps.join(', ')}); memo would rarely prevent re-renders`,
    });
  }

  // 2. Very simple component
  if (renderCost.score < 10) {
    factors.push({
      type: 'negative',
      weight: 25,
      description: 'Component is very simple; memo overhead may exceed render cost',
    });
  }

  // 3. Component always receives new props
  if (propsStability.alwaysNewProps) {
    factors.push({
      type: 'negative',
      weight: 50,
      description: 'Component always receives new prop values; memo would never help',
    });
  }

  // 4. Leaf component with no children
  if (isLeafComponent(component)) {
    factors.push({
      type: 'negative',
      weight: 10,
      description: 'Leaf component; no subtree to protect from re-renders',
    });
  }

  // Calculate recommendation
  const positiveScore = factors
    .filter(f => f.type === 'positive')
    .reduce((sum, f) => sum + f.weight, 0);
  const negativeScore = factors
    .filter(f => f.type === 'negative')
    .reduce((sum, f) => sum + f.weight, 0);

  const netScore = positiveScore - negativeScore;

  return {
    recommend: netScore > 20,
    confidence: Math.abs(netScore) > 40 ? 'high' : Math.abs(netScore) > 20 ? 'medium' : 'low',
    factors,
  };
}

function analyzePropsStability(component: Node): PropsStabilityAnalysis {
  const props = extractPropDefinitions(component);
  const unstableProps: string[] = [];
  let alwaysNewProps = false;

  for (const prop of props) {
    // Find all usages of this component
    const usages = findComponentUsages(component);

    for (const usage of usages) {
      const propValue = getPropValue(usage, prop.name);
      if (!propValue) continue;

      const stability = analyzeValueStability(propValue);
      if (stability === 'unstable') {
        unstableProps.push(prop.name);
      }
      if (stability === 'always-new') {
        alwaysNewProps = true;
      }
    }
  }

  return {
    hasUnstableProps: unstableProps.length > 0,
    unstableProps: [...new Set(unstableProps)],
    alwaysNewProps,
  };
}

function isPureComponent(component: Node): boolean {
  // Component is pure if:
  // - No useState, useReducer (internal state)
  // - No useRef that's mutated
  // - No useContext (external state)
  // - No side effects in render

  let hasSideEffects = false;

  component.forEachDescendant(desc => {
    if (Node.isCallExpression(desc)) {
      const callee = desc.getExpression().getText();

      // State hooks
      if (['useState', 'useReducer', 'useContext'].includes(callee)) {
        hasSideEffects = true;
      }

      // Note: useRef alone doesn't make it impure, only if mutated
      // This would require more sophisticated analysis
    }
  });

  return !hasSideEffects;
}
```

---

## Pattern 4: Index as Key Detection

### Naive Implementation

```typescript
// Flags: Any use of index as key in map
function checkIndexKey(node: ts.CallExpression) {
  if (isArrayMap(node)) {
    const callback = node.arguments[0];
    const params = getParameters(callback);
    const indexParam = params[1]; // Second param is index

    if (indexParam && isUsedAsKey(callback, indexParam)) {
      report(node, 'no-index-key');
    }
  }
}
```

**Problems**:

- Index keys are fine for static lists
- Index keys are fine for append-only lists
- Index keys only cause issues with stateful components

### Sophisticated Implementation

```typescript
interface IndexKeyAnalysis {
  problematic: boolean;
  confidence: 'high' | 'medium' | 'low';
  reason: string;
}

function analyzeIndexAsKey(mapExpression: Node): IndexKeyAnalysis {
  // 1. Get the array being mapped
  const arrayExpr = getMapTargetArray(mapExpression);

  // 2. Check if array is ever mutated in ways that reorder
  const mutations = findArrayMutations(arrayExpr);
  const hasReorderingMutations = mutations.some(
    m =>
      ['sort', 'reverse', 'splice', 'unshift', 'filter'].includes(m.method) ||
      m.type === 'reassignment-filtered' ||
      m.type === 'reassignment-sorted'
  );

  if (!hasReorderingMutations) {
    // Check if it's append-only
    const isAppendOnly = mutations.every(
      m => m.method === 'push' || m.type === 'reassignment-appended'
    );

    if (isAppendOnly || mutations.length === 0) {
      return {
        problematic: false,
        confidence: 'high',
        reason: 'Array is static or append-only; index keys are safe',
      };
    }
  }

  // 3. Check if rendered component has internal state
  const renderedComponent = getComponentFromMapCallback(mapExpression);
  const hasInternalState = componentHasState(renderedComponent);

  if (!hasInternalState) {
    return {
      problematic: false,
      confidence: 'medium',
      reason: 'Rendered component is stateless; index keys unlikely to cause issues',
    };
  }

  // 4. Check for uncontrolled inputs
  const hasUncontrolledInputs = componentHasUncontrolledInputs(renderedComponent);

  if (hasReorderingMutations && (hasInternalState || hasUncontrolledInputs)) {
    return {
      problematic: true,
      confidence: 'high',
      reason:
        'Array can be reordered and component has state; index keys will cause state mismatches',
    };
  }

  return {
    problematic: true,
    confidence: 'low',
    reason: 'Cannot fully determine if index keys are safe; consider using stable IDs',
  };
}

function findArrayMutations(arrayExpr: Node): ArrayMutation[] {
  const mutations: ArrayMutation[] = [];
  const symbol = arrayExpr.getSymbol();
  if (!symbol) return mutations;

  const references = symbol.findReferences();

  for (const ref of references) {
    for (const refEntry of ref.getReferences()) {
      const refNode = refEntry.getNode();
      const parent = refNode.getParent();

      // Method calls: arr.sort(), arr.splice(), etc.
      if (Node.isPropertyAccessExpression(parent)) {
        const grandparent = parent.getParent();
        if (Node.isCallExpression(grandparent)) {
          const method = parent.getName();
          mutations.push({ type: 'method-call', method });
        }
      }

      // Reassignments: setArr(arr.filter(...))
      if (isStateSetterCall(parent, refNode)) {
        const setterArg = getSetterArgument(parent);
        if (setterArg) {
          const transformation = detectArrayTransformation(setterArg);
          mutations.push({ type: `reassignment-${transformation}`, method: transformation });
        }
      }
    }
  }

  return mutations;
}

function componentHasState(component: Node | undefined): boolean {
  if (!component) return false;

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

function componentHasUncontrolledInputs(component: Node | undefined): boolean {
  if (!component) return false;

  let hasUncontrolled = false;
  component.forEachDescendant(desc => {
    if (Node.isJsxElement(desc) || Node.isJsxSelfClosingElement(desc)) {
      const tagName = getJsxTagName(desc);
      if (['input', 'textarea', 'select'].includes(tagName.toLowerCase())) {
        // Check if it has value prop (controlled) or not (uncontrolled)
        const valueAttr = findJsxAttribute(desc, 'value');
        const defaultValueAttr = findJsxAttribute(desc, 'defaultValue');

        if (!valueAttr && defaultValueAttr) {
          hasUncontrolled = true;
        }
      }
    }
  });

  return hasUncontrolled;
}
```

---

## Implementation Effort vs Value Matrix

| Pattern             | Naive Effort | Sophisticated Effort | Value Gain                     |
| ------------------- | ------------ | -------------------- | ------------------------------ |
| useMemo/useCallback | 1 hour       | 2-3 days             | High (eliminates noise)        |
| Effect dependencies | 2 hours      | 1 week               | Very High (prevents bugs)      |
| Expensive renders   | 30 min       | 2-3 days             | High (actionable advice)       |
| Index as key        | 30 min       | 1-2 days             | Medium (fewer false positives) |
| Nested components   | 1 hour       | 2 hours              | Low (naive works well)         |
| Effect cleanup      | 2 hours      | 4-6 hours            | Medium (better pairing)        |

## Recommendation

**Start sophisticated**: Effect dependencies, useMemo/useCallback analysis
**Keep naive**: Nested component detection (inherently problematic pattern)
**Invest later**: Expensive render detection (needs profiling integration)
