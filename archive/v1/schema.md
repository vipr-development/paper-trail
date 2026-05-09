# JSON Schema Documentation

## Overview

The React Complexity Analyzer produces JSON output that conforms to the JSON Schema Draft 07 specification defined in `schema.json`.

## Schema URL

```
https://github.com/vipr/react-complexity/schema/v2.json
```

## Top-Level Structure

```json
{
  "$schema": "https://github.com/vipr/react-complexity/schema/v2.json",
  "version": "2.0.0",
  "metadata": { ... },
  "summary": { ... },
  "dimensions": { ... },
  "traditional": { ... },
  "insights": [ ... ],
  "refactorings": [ ... ]  // Optional
}
```

## Metadata

Information about the analyzed file and analysis context:

| Field             | Type     | Description                          |
| ----------------- | -------- | ------------------------------------ |
| `file`            | string   | Relative or absolute path to file    |
| `analyzedAt`      | datetime | ISO 8601 timestamp                   |
| `analyzerVersion` | string   | Semantic version of analyzer         |
| `analysisLayers`  | array    | Tools used (babel, ts-morph, eslint) |
| `lineCount`       | integer  | Total lines in file (optional)       |
| `componentName`   | string   | Primary component name (optional)    |

## Summary

High-level analysis results:

| Field            | Type   | Values                              |
| ---------------- | ------ | ----------------------------------- |
| `totalScore`     | number | Composite complexity score (0-100+) |
| `grade`          | string | A, B, C, D, F                       |
| `complexity`     | string | low, moderate, high, very-high      |
| `recommendation` | string | Human-readable recommendation       |
| `riskLevel`      | string | low, medium, high, critical         |

## Dimensions

Detailed breakdown of complexity across multiple dimensions:

### Required Dimensions

All analyses include these five core dimensions:

1. **structural** - Control flow complexity
2. **hooks** - React hook usage patterns
3. **temporal** - Effect and lifecycle complexity
4. **coupling** - Component dependencies
5. **identity** - Memoization and referential stability

### Optional Dimensions

Advanced analyses may include:

- **typeComplexity** - TypeScript type complexity
- **dataFlow** - State and prop flow analysis
- **reliability** - Error handling and fault tolerance

## Traditional Metrics

Classical complexity metrics for comparison:

- **cyclomaticComplexity** - McCabe's metric (M = E - N + 2P)
- **halstead** - Complete Halstead measures:
  - `uniqOperators` (n1) - Distinct operators
  - `uniqOperands` (n2) - Distinct operands
  - `totalOperators` (N1) - Total operators
  - `totalOperands` (N2) - Total operands
  - `programLength` (N = N1 + N2)
  - `vocabularySize` (n = n1 + n2)
  - `volume` (V = N × log₂ n)
  - `difficulty` (D = (n1/2) × (N2/n2))
  - `effort` (E = D × V)
  - `time` (T = E / 18 seconds)
  - `bugs` (B = V / 3000)

## Insights

Array of actionable suggestions for improvement:

```typescript
{
  severity: "info" | "warning" | "critical",
  category: string,  // hooks, temporal, coupling, etc.
  message: string,
  line?: number,
  suggestion?: string
}
```

## Refactorings (Optional)

Specific refactoring suggestions with code examples:

```typescript
{
  id: string,
  type: "extract-hook" | "split-component" | "add-memo" | "lift-state" | "add-error-boundary" | "fix-deps",
  severity: "optional" | "recommended" | "critical",
  confidence: "low" | "medium" | "high",
  description: string,
  location?: { startLine: number, endLine: number },
  estimatedImpact: {
    complexityReduction: number,
    linesAdded?: number,
    linesRemoved?: number
  },
  beforeCode?: string,
  afterCode?: string,
  diffPreview?: string
}
```

## Validation

Validate your JSON output against the schema:

### Using AJV CLI

```bash
npm install -g ajv-cli
npm run analyze -- Component.tsx --json | ajv validate -s schema.json -d -
```

### Using Node.js

```javascript
const Ajv = require('ajv');
const schema = require('./schema.json');
const result = require('./analysis-result.json');

const ajv = new Ajv();
const validate = ajv.compile(schema);
const valid = validate(result);

if (!valid) {
  console.error(validate.errors);
}
```

### Using TypeScript

The analyzer exports TypeScript types that match the schema:

```typescript
import { ReactComplexityResult } from './src/analyzer';

const result: ReactComplexityResult = {
  // TypeScript ensures type safety
};
```

## Example Output

See `src/sample-components/` for example components and their analysis results:

```bash
# Generate example
npm run analyze -- src/sample-components/DataTable.tsx --json --pretty > example-output.json
```

## Version History

- **v2.0.0** (2026-01-01)
  - Complete rewrite with React 19 support
  - Added new hooks: `useOptimistic`, `useActionState`, `useFormState`, `useFormStatus`, `use()`
  - Enhanced Halstead metrics with comprehensive operator detection
  - Improved schema with detailed documentation
  - Added optional refactoring suggestions

- **v1.0.0** - Initial release

## Schema Compliance

The schema follows [JSON Schema Draft 07](http://json-schema.org/draft-07/schema#) specification:

- ✅ Semantic versioning (`version` field)
- ✅ Required vs optional fields clearly marked
- ✅ Type constraints and enums
- ✅ Minimum/maximum values for numeric fields
- ✅ Descriptive documentation for all fields
- ✅ Backward compatibility via legacy fields

## Integration

The schema enables integration with various tools:

- **VS Code**: JSON validation and autocomplete
- **CI/CD**: Automated quality gates
- **Dashboards**: Complexity tracking over time
- **LLMs**: Structured input for code analysis
- **Documentation**: Auto-generated API docs

## Support

For schema-related questions or issues:

- Review `schema.json` for complete field definitions
- Check TypeScript types in `src/types.ts`
- See examples in `src/sample-components/`
- File issues on GitHub
