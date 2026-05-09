---
id: 05-audit-results
---

# Next.js Analyzer Audit Results

**Date**: 2026-01-26
**Auditors**: nextjs-static-analysis-auditor, code-complexity-analyzer
**Scope**: Current implementation vs documented research requirements

## Executive Summary

The Next.js analyzer demonstrates a **solid architectural foundation** with sophisticated import graph infrastructure and data flow analysis for waterfall detection. However, critical gaps exist in **boundary propagation integration**, **type-level analysis**, and **router-specific pattern detection**.

**Overall Assessment**: 60% sophisticated, 40% naive or missing
**Sophistication Score**: 6.5/10

### Key Findings

**Strengths**:

- ✅ Excellent import graph infrastructure (`import-graph.ts`)
- ✅ Sophisticated waterfall detection with data flow analysis (`fetch-analyzer.ts`)
- ✅ Good boundary violation detection foundation (`boundary-analyzer.ts`)
- ✅ AST-based analysis throughout (not just regex)

**Critical Weaknesses**:

- ❌ Import chain analysis not integrated into directive checking
- ❌ Hook detection using naive regex instead of symbol resolution
- ❌ Missing dynamic rendering trigger analysis entirely
- ❌ No route segment config validation
- ❌ Many helpers still use regex instead of semantic AST analysis
- ❌ No caching strategy for expensive graph operations

## Detailed Findings by Analysis Area

### 1. Server/Client Boundary Analysis (CRITICAL GAPS)

#### Issue: Import Chain Analysis Not Used

**Status**: Import graph exists but not leveraged for directive analysis
**Impact**: HIGH - False positives on shared components
**File**: `analyzers/nextjs/src/analyses/server-client-analysis.ts:248-262`

**Problem**:

```typescript
// Current: Flags ANY file with hooks as needing 'use client'
private checkMissingClientDirective(sourceFile: SourceFile): ServerClientIssue | null {
  if (!needsUseClientDirective(sourceFile)) {
    return null;
  }
  // Doesn't check if imported by client file!
}
```

**False Positive Example**:

```tsx
// components/counter-display.tsx
// ❌ Flagged as missing directive, but this is FINE
export function CounterDisplay({ count }: { count: number }) {
  return <span>{count}</span>;
}

// components/counter.tsx
('use client');
import { CounterDisplay } from './counter-display';
// CounterDisplay inherits client context - no directive needed
```

**Fix Required**: Integrate import graph to check for client boundary inheritance
**Complexity**: MEDIUM (2-3 days) - Import graph exists, needs integration
**Priority**: P0 - Most common false positive

---

#### Issue: Hook Source Verification

**Status**: NAIVE - Regex-only hook detection
**Impact**: MEDIUM - False positives on custom hooks
**File**: `analyzers/nextjs/src/utils/directive-helpers.ts:13-26`

**Problem**:

```typescript
// Current: Matches ANY function named useState
const REACT_HOOK_PATTERNS = [
  /useState\s*\(/,
  /useEffect\s*\(/,
  // Just regex, no import verification
];
```

**False Positive**:

```typescript
// Custom useState, not from React
function useState() {
  /* custom */
}

export function MyComponent() {
  const [state] = useState(); // ❌ Flagged but not a React hook!
}
```

**Fix Required**: Use symbol resolution to verify hooks are from React
**Complexity**: LOW (4-6 hours) - AST-based import checking
**Priority**: P1 - Common pattern in large codebases

---

#### Issue: Non-Serializable Props Across Boundaries

**Status**: PARTIALLY IMPLEMENTED - Same-file only
**Impact**: HIGH - Misses serialization bugs
**File**: `analyzers/nextjs/src/analyses/server-client-analysis.ts:268-327`

**Problem**: Only checks props within the same file, doesn't use import graph to verify if component is actually a Client Component.

**Missing Detection**:

```tsx
// Server Component (no directive)
export default async function Page() {
  const handleClick = () => console.log('clicked'); // Function!

  // ❌ Should flag - passing function to Client Component
  return <ClientButton onClick={handleClick} />;
}
```

**Fix Required**: Cross-file AST analysis with type-aware checking
**Complexity**: MEDIUM-HIGH (2-3 days) - Needs type system integration
**Priority**: P0 - Common bug, hard to debug

---

### 2. Missing Dynamic Rendering Analysis (CRITICAL)

**Status**: MISSING ENTIRELY
**Impact**: CRITICAL - Major performance implications
**Priority**: P0 - High user impact

**Required Detection**:

- `cookies()` and `headers()` usage from `next/headers`
- Check if usage is intentional (route config analysis)
- Identify conflicts (static config with dynamic triggers)
- Suggest optimizations (isolate dynamic parts, use PPR)

**False Negative Example**:

```tsx
// ❌ NOT DETECTED - Unintentional dynamic rendering
export default async function Page() {
  const theme = cookies().get('theme'); // Forces entire route dynamic!

  const posts = await getStaticPosts(); // Could be static
  return <PostList posts={posts} theme={theme} />;
}
```

**Fix Required**: New analysis module for dynamic rendering triggers
**Complexity**: MEDIUM (1-2 days) - AST-based function call detection
**Documentation**: See `01-problematic-patterns.md:670-686`

---

### 3. Missing Route Segment Config Analysis (HIGH PRIORITY)

**Status**: MISSING
**Impact**: MEDIUM-HIGH - Configuration errors break at runtime
**Priority**: P1

**Required Detection**:

1. Conflicting config: `dynamic = 'force-static'` with `revalidate = 0`
2. Invalid values: `dynamic = 'static'` (should be `'force-static'`)
3. Config-behavior mismatch: Static config with `no-store` fetches

**Fix Required**: Export declaration parsing for route segment config
**Complexity**: LOW-MEDIUM (1 day)
**Documentation**: See `02-ts-morph-analysis.md:963-1006`

---

### 4. Data Fetching Analysis (ENHANCEMENT NEEDED)

#### Issue: Fetch Cache Strategy Coherence

**Status**: PARTIALLY IMPLEMENTED - Basic detection
**Impact**: MEDIUM - Optimization opportunities missed
**File**: `analyzers/nextjs/src/analyses/data-fetching-analysis.ts:265-289`

**Current**: Uses naive regex to check for cache options existence
**Missing**:

- Parse actual cache option values
- Detect inconsistent strategies across multiple fetches
- Verify revalidation intervals are consistent

**Fix Required**: AST-based fetch options parsing
**Complexity**: MEDIUM (1-2 days)

---

#### Issue: Waterfall Detection Refinement

**Status**: SOPHISTICATED FOUNDATION - Needs enhancement
**Impact**: LOW - Already good, edge cases remain
**File**: `analyzers/nextjs/src/utils/fetch-analyzer.ts`

**Current Sophistication**: Excellent data flow analysis ✅
**Enhancement Opportunities**:

1. Property-access-aware data flow (e.g., `userData.id`)
2. Inter-procedural analysis (calls to helper functions)
3. Cycle detection with SCC algorithm

**Fix Required**: Enhance existing analysis
**Complexity**: LOW-MEDIUM (1 day)
**Priority**: P2 - Polish existing sophisticated implementation

---

### 5. Security Analysis (TRANSITIVE CHECKS NEEDED)

#### Issue: Transitive Import Detection Not Wired

**Status**: IMPLEMENTED but not used
**Impact**: MEDIUM - Security false negatives
**File**: `boundary-analyzer.ts:343-384` (exists but not called)

**Problem**: Code exists but isn't integrated into security analysis workflow

**Missing Detection**:

```tsx
'use client';
// ✅ Detected: Direct database import
import { db } from '@prisma/client';

// ❌ NOT DETECTED: Transitive server import
import { utils } from '@/lib/utils';
// utils.ts imports 'fs' internally - should be flagged!
```

**Fix Required**: Wire up existing functionality to security analysis
**Complexity**: LOW (2-4 hours)
**Priority**: P1 - Code already exists, just needs integration

---

#### Issue: Server Actions Control Flow

**Status**: NAIVE - Basic AST checking
**Impact**: MEDIUM - Security false negatives
**File**: `analyzers/nextjs/src/analyses/security-analysis.ts:68-92`

**Current**: Checks if auth is imported and called
**Missing**: Verify auth check happens **before** sensitive operations

**Example**:

```typescript
export async function deleteUser(userId: string) {
  const session = await auth(); // ✅ Imported and called

  // ❌ Doesn't verify this check exists!
  if (!session?.user?.isAdmin) {
    throw new Error('Unauthorized');
  }

  await db.users.delete({ where: { id: userId } });
}
```

**Fix Required**: Control flow analysis
**Complexity**: MEDIUM (1-2 days)
**Priority**: P1 - Security critical

---

### 6. Algorithmic Complexity Issues

#### Issue: Import Graph Rebuilt Per File

**Status**: PERFORMANCE BOTTLENECK
**Impact**: HIGH - 1000x slowdown on large projects
**File**: `analyzers/nextjs/src/analyses/server-client-analysis.ts:789-795`

**Problem**:

```typescript
// Rebuilds entire graph for EVERY file analyzed
const project = sourceFile.getProject();
const importGraph = new ImportGraphBuilder(project);
importGraph.build(); // O(V + E) per file!
```

**Fix Required**: Cache graph at plugin level with incremental updates
**Complexity**: MEDIUM (1 day)
**Priority**: P0 - Massive performance impact

---

#### Issue: Boundary Propagation Algorithm

**Status**: O(V × E × k) - Inefficient
**Impact**: MEDIUM - Slow on large graphs
**File**: `analyzers/nextjs/src/utils/import-graph.ts:236-268`

**Problem**: Checks ALL nodes on EVERY iteration

**Fix Required**: Worklist algorithm (only process changed nodes)
**Complexity**: LOW (4-6 hours)
**Priority**: P1 - Common performance bottleneck

**Expected Improvement**: 10-100x faster on large projects

---

## Implementation Priority Matrix

| Issue                             | Impact   | Complexity | Priority | Effort    |
| --------------------------------- | -------- | ---------- | -------- | --------- |
| **Import chain integration**      | HIGH     | MEDIUM     | P0       | 2-3 days  |
| **Cache import graph**            | HIGH     | MEDIUM     | P0       | 1 day     |
| **Non-serializable props**        | HIGH     | MED-HIGH   | P0       | 2-3 days  |
| **Dynamic rendering analysis**    | CRITICAL | MEDIUM     | P0       | 1-2 days  |
| **Hook source verification**      | MEDIUM   | LOW        | P1       | 4-6 hours |
| **Route config analysis**         | MED-HIGH | LOW-MED    | P1       | 1 day     |
| **Transitive import wiring**      | MEDIUM   | LOW        | P1       | 2-4 hours |
| **Server Actions control flow**   | MEDIUM   | MEDIUM     | P1       | 1-2 days  |
| **Boundary propagation worklist** | MEDIUM   | LOW        | P1       | 4-6 hours |
| **Fetch cache coherence**         | MEDIUM   | MEDIUM     | P2       | 1-2 days  |
| **Waterfall refinement**          | LOW      | LOW-MED    | P2       | 1 day     |

---

## Recommended Implementation Phases

### Phase 1: Fix Critical False Positives (Week 1)

**Goal**: Eliminate most common false positives

1. ✅ **Cache import graph at plugin level** (1 day)
   - Massive performance improvement
   - Enables all other import-graph-based fixes

2. ✅ **Integrate import chain analysis** (2-3 days)
   - Check client boundary inheritance
   - Eliminate false positives on shared components

3. ✅ **Hook source verification** (4-6 hours)
   - Symbol resolution for React hooks
   - Eliminate custom hook false positives

4. ✅ **Wire up transitive import detection** (2-4 hours)
   - Already implemented, just connect it
   - Quick security win

### Phase 2: Add Missing Critical Detections (Week 2)

**Goal**: Catch high-impact bugs

5. ✅ **Dynamic rendering trigger analysis** (1-2 days)
   - New analysis module
   - High performance impact for users

6. ✅ **Route segment config validation** (1 day)
   - Catch configuration errors
   - Prevent runtime failures

7. ✅ **Non-serializable props across boundaries** (2-3 days)
   - Type-aware prop checking
   - Catch serialization bugs

### Phase 3: Enhance Existing Analyses (Week 3)

**Goal**: Improve accuracy and coverage

8. ✅ **Boundary propagation worklist** (4-6 hours)
   - 10-100x performance improvement
   - More predictable analysis time

9. ✅ **Fetch cache strategy coherence** (1-2 days)
   - Parse actual cache values
   - Detect inconsistencies

10. ✅ **Server Actions control flow** (1-2 days)
    - Verify auth placement
    - Reduce security false negatives

### Phase 4: Polish and Edge Cases (Week 4)

**Goal**: Handle sophisticated patterns

11. ✅ **Waterfall detection refinements** (1 day)
    - Property-access-aware data flow
    - Inter-procedural analysis

12. ✅ **Pages Router recommendations** (1-2 days)
    - Data fetching strategy guidance
    - Migration suggestions

---

## Quick Wins (High Value, Low Effort)

**Implement these first for immediate impact:**

1. **Hook import verification** (4-6 hours) → Eliminates false positives
2. **Wire up transitive imports** (2-4 hours) → Security improvement
3. **Boundary propagation worklist** (4-6 hours) → Performance boost

**Combined effort**: ~1 day
**Combined impact**: Major UX improvement

---

## Code Examples Roadmap

### Before/After: Hook Detection

**Before** (Naive):

```typescript
const REACT_HOOK_PATTERNS = [/useState\s*\(/];
for (const pattern of REACT_HOOK_PATTERNS) {
  if (pattern.test(text)) return true;
}
```

**After** (Sophisticated):

```typescript
const symbol = callee.getSymbol();
const declarations = symbol?.getDeclarations();
for (const decl of declarations) {
  const filePath = decl.getSourceFile().getFilePath();
  if (filePath.includes('node_modules/react')) {
    return true; // ✅ Verified React hook
  }
}
```

---

### Before/After: Import Graph Caching

**Before** (Rebuild per file):

```typescript
// Called for EVERY file
const importGraph = new ImportGraphBuilder(project);
importGraph.build(); // O(V + E) every time!
```

**After** (Cached):

```typescript
// Plugin-level singleton
private getOrUpdateImportGraph(project: Project): ImportGraphBuilder {
  if (this.cache.needsRebuild(project)) {
    this.cache.rebuild(project);
  }
  return this.cache.graph; // ✅ Reused across files
}
```

---

## Testing Strategy

### Unit Tests

Create tests for each analysis with:

- ✅ True positives (should detect)
- ✅ True negatives (should not flag)
- ✅ False positive cases from research docs
- ✅ Edge cases (circular imports, complex type hierarchies)

### Integration Tests

- Cross-file analysis (boundary propagation)
- Import graph consistency
- Performance benchmarks (graph caching)

### Regression Tests

Use documented examples from:

- `01-problematic-patterns.md`
- `03-naive-vs-sophisticated.md`

---

## Success Metrics

**Correctness**:

- False positive rate: Target < 5%
- False negative rate: Target < 10%
- Coverage of documented patterns: Target 95%

**Performance**:

- Analysis time per file: Target < 100ms (large projects)
- Import graph build: Target < 2s (1000 files)
- Memory usage: Target < 500MB (large projects)

**User Experience**:

- Actionable suggestions: 100% of findings
- Clear error messages: 100% of findings
- Auto-fixable issues: Target > 30%

---

## Next Steps

1. **Review and approve this audit report**
2. **Prioritize phases based on team capacity**
3. **Begin Phase 1 implementation** (Quick wins + critical fixes)
4. **Set up performance benchmarks** (Before implementing caching)
5. **Create test suite** (Based on documented patterns)
6. **Update metrics and presenters** (For new analyses)

---

## Appendix: Related Documentation

- **Research**: `00-research.md`
- **Problematic Patterns**: `01-problematic-patterns.md`
- **ts-morph Analysis**: `02-ts-morph-analysis.md`
- **Naive vs Sophisticated**: `03-naive-vs-sophisticated.md`
- **Audit Template**: `04-audit-nextjs-template.ts`
