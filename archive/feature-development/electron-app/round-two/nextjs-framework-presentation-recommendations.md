---
id: nextjs-framework-presentation-recommendations
title: Next.js and Framework-Extensible UX Recommendations
phase: cross-cutting
---

# Next.js-Specific Metrics and Framework-Extensible UX Recommendations

## Executive Summary

This document provides UX recommendations for presenting Next.js-specific metrics within the Electron app while establishing extensible patterns for future framework analyzers (Vue, Angular, Svelte). The recommendations address four critical areas:

1. **Framework differentiation** - How to distinguish Next.js issues from React issues
2. **Multi-repository workspace** - Handling heterogeneous technology stacks
3. **Architectural anti-patterns** - Framework-specific anti-patterns
4. **Future extensibility** - Adapting UX to evolving metric availability

## Current State Analysis

### Available Next.js Metrics

Vipr's Next.js analyzer plugin (`@vipr/nextjs`) currently detects:

**Analysis Categories** (7 distinct analyses):

1. **Server-Client Analysis** - RSC boundary violations, serialization issues
2. **Data Fetching Analysis** - Server vs client data patterns, caching strategies
3. **Migration Analysis** - Pages Router to App Router migration issues
4. **Security Analysis** - Environment variable exposure, XSS in RSC
5. **Config Analysis** - next.config.js optimization opportunities
6. **Route Structure Analysis** - App Router conventions, route organization
7. **Rendering Analysis** - Static/Dynamic rendering, ISR, streaming patterns

**File Type Detection** (14 Next.js-specific types):

- App Router: `page`, `layout`, `loading`, `error`, `not-found`, `template`, `default`, `route`, `middleware`
- Server-side: `server-component`, `server-action`, `api-route`
- React fallback: `component`, `hook`, `context`, `hoc`, `provider`

**Technology Detection**:

- React: Base technology for component patterns
- Next.js: Detected via imports, directives, file conventions
- Proper hierarchy: Next.js files can have BOTH React and Next.js analysis

### Architecture Context

**Plugin Coordination Pattern**:

- React analyzer handles general React patterns
- Next.js analyzer handles framework-specific patterns
- Both can run on the same file (e.g., RSC with React hooks)
- `FileType` system provides fine-grained classification
- `FileTechnology` system provides high-level grouping

---

## Recommendation 1: Framework-Aware Visual Language

### Problem

Users need to quickly identify framework-specific issues without being overwhelmed by technical details. Next.js files have different constraints than React files, and this distinction must be immediately visible.

### Solution: Technology Badges and Context Indicators

#### 1.1 File Type Badges

Display framework context at every level of navigation.

**Visual Design**:

```
+-----------------+
| UserProfile.tsx |
+-----------------+
[Next.js]  [RSC]  [App Router]
```

**Badge Hierarchy** (left to right, most to least specific):

1. **Framework Badge** - Primary technology (Next.js, React, Vue, Angular)
2. **Execution Context Badge** - Where code runs (RSC, Client, Server Action)
3. **Architecture Badge** - Architectural layer (App Router, Pages Router, Route Handler)

**Badge Color System**:

```typescript
const BADGE_COLORS = {
  framework: {
    nextjs: '#000000', // Next.js black
    react: '#61DAFB', // React cyan
    vue: '#42B883', // Vue green
    angular: '#DD0031', // Angular red
  },
  execution: {
    server: '#10B981', // Green (server-side)
    client: '#3B82F6', // Blue (client-side)
    hybrid: '#8B5CF6', // Purple (both)
  },
  architecture: {
    appRouter: '#F59E0B', // Amber
    pagesRouter: '#6B7280', // Gray
  },
};
```

**Placement Strategy**:

- **File List View**: Inline badges after filename
- **File Detail View**: Header badges below filename
- **Treemap/Visualization**: Badge icon overlay on cells
- **Dependency Graph**: Badge icons on nodes

#### 1.2 Execution Context Indicators

For Next.js, the Server vs Client distinction is critical for understanding constraints.

**Server Component Indicator**:

```
┌────────────────────────────────┐
│ Dashboard.tsx                  │
│ [Next.js] [Server Component]   │
│ ────────────────────────────── │
│ ⚡ Can use async/await          │
│ ⚡ Direct database access OK    │
│ ⚠️  No useState/useEffect       │
│ ⚠️  No browser APIs             │
└────────────────────────────────┘
```

**Client Component Indicator**:

```
┌────────────────────────────────┐
│ InteractiveChart.tsx           │
│ [Next.js] [Client Component]   │
│ ────────────────────────────── │
│ ⚡ Can use hooks & state        │
│ ⚡ Browser APIs available       │
│ ⚠️  No async components         │
│ ⚠️  Props must be serializable  │
└────────────────────────────────┘
```

**Visual Pattern**:

- Green checkmarks (⚡) = Capabilities
- Orange warnings (⚠️) = Constraints
- Expandable "Why?" link to documentation

#### 1.3 Route Visualization

Next.js route structure deserves special visual treatment.

**App Router Hierarchy View**:

```
app/
├─ layout.tsx [Layout] [RSC]
├─ page.tsx [Page] [RSC]
├─ loading.tsx [Loading] [RSC]
├─ error.tsx [Error Boundary] [Client]
│
├─ dashboard/
│  ├─ layout.tsx [Layout] [RSC]
│  ├─ page.tsx [Page] [RSC]
│  └─ [userId]/
│     └─ page.tsx [Page] [RSC]
│
└─ api/
   └─ users/
      └─ route.ts [Route Handler] [Server]
```

**Interactive Features**:

- Click file to open detail
- Hover to show route URL mapping
- Color-code by execution context
- Badge shows file convention type

---

## Recommendation 2: Multi-Repository Workspace Framework Handling

### Problem

The Multi-Repository Workspace Dashboard (US-NEW-10) must handle repositories with different technology stacks. Comparing Next.js metrics to React metrics (or Vue, Angular, etc.) requires careful UX design.

### Solution: Technology-Aware Comparison and Filtering

#### 2.1 Technology Detection and Grouping

**Workspace Dashboard Enhancement**:

```
Workspace Dashboard                              [Add Repository] [Analyze All]
================================================================================

Portfolio Health: 72/100 [====------]  Trend: Stable (0.0/week)
Total Issues: 47 Critical, 234 Warning    Repos: 6 active

Technology Filter: [All ▼] [Next.js: 2] [React: 3] [TypeScript: 1]
Sort: [Health Score ▼]    View: [Grid] [List]

┌─────────────────────────────┐  ┌─────────────────────────────┐
│ my-app                      │  │ client-portal               │
│ /Users/dev/my-app           │  │ /Users/dev/client-portal    │
│ [Next.js 15] [App Router]   │  │ [Next.js 14] [Pages]        │
│                             │  │                             │
│ Health: 85 [=========-]     │  │ Health: 71 [=======--]      │
│ Trend: +2.3/week            │  │ Trend: -1.2/week [!]        │
│                             │  │                             │
│ Framework Metrics:          │  │ Framework Metrics:          │
│ • RSC Issues: 2             │  │ • SSR Issues: 8             │
│ • Data Fetching: OK         │  │ • Data Fetching: 3 warnings │
│ • Route Structure: OK       │  │ • API Routes: 2 issues      │
│                             │  │                             │
│ [Open Repository]           │  │ [Open Repository]           │
└─────────────────────────────┘  └─────────────────────────────┘

┌─────────────────────────────┐  ┌─────────────────────────────┐
│ shared-components           │  │ legacy-dashboard            │
│ /Users/dev/shared-ui        │  │ /Users/dev/legacy-dash      │
│ [React 18] [No Framework]   │  │ [React 16] [No Framework]   │
│                             │  │                             │
│ Health: 92 [==========]     │  │ Health: 45 [====------]     │
│ Trend: +0.1/week            │  │ Trend: -3.1/week [!!!]      │
│                             │  │                             │
│ Framework Metrics:          │  │ Framework Metrics:          │
│ • Component Quality: High   │  │ • Component Quality: Low    │
│ • Hook Patterns: OK         │  │ • God Components: 12        │
│ • No Next.js features       │  │ • Needs Upgrade             │
│                             │  │                             │
│ [Open Repository]           │  │ [Open Repository]           │
└─────────────────────────────┘  └─────────────────────────────┘
```

**Key Features**:

1. **Technology Badge on Card** - Immediately visible framework
2. **Framework-Specific Metrics** - Different cards show different metric types
3. **Technology Filtering** - Click filter to show only Next.js repos
4. **Grouped Presentation** - Sort by technology creates visual grouping

#### 2.2 Cross-Technology Comparison Strategy

**Problem**: How do you compare Next.js-specific metrics (like RSC violations) with React-only repos that don't have RSC?

**Solution: Comparative Metric Normalization**

**Apples-to-Apples Comparison**:

```typescript
interface ComparableMetrics {
  // Universal metrics (all frameworks)
  healthScore: number; // 0-100
  complexityTrend: number; // delta per week
  criticalIssues: number; // count

  // Framework-agnostic categories
  structuralComplexity: number; // normalized 0-100
  couplingIssues: number; // count
  securityVulnerabilities: number; // count

  // Framework-specific (conditionally present)
  frameworkMetrics?: {
    nextjs?: NextJsMetrics;
    react?: ReactMetrics;
    vue?: VueMetrics;
  };
}
```

**Comparison View Example**:

```
Repository Comparison: my-app vs legacy-dashboard
================================================================================

                        | my-app (Next.js) | legacy-dashboard (React) |
------------------------|------------------|--------------------------|
Health Score           | 85               | 45                       |
Trend (30d)            | +2.3             | -3.1                     |
Critical Issues        | 3                | 24                       |
Avg Complexity         | 12.4             | 28.7                     |
God Components         | 1                | 12                       |
------------------------|------------------|--------------------------|
Next.js Specific:      |                  |                          |
  RSC Violations       | 2                | N/A                      |
  Data Fetching Issues | 0                | N/A                      |
  Route Structure      | OK               | N/A                      |
------------------------|------------------|--------------------------|
React Specific:        |                  |                          |
  Hook Issues          | 1                | 8                        |
  Prop Drilling        | 0                | 5                        |
  Context Overuse      | 0                | 3                        |

ANALYSIS:
legacy-dashboard requires immediate attention:
- Health score 40 points lower than my-app
- 8x more critical issues
- Actively declining (-3.1/week)
- Multiple architectural anti-patterns

Note: Direct Next.js metric comparison not applicable.
Use universal metrics (health score, complexity) for comparison.

[Generate Comparison Report]   [Open legacy-dashboard]
```

**Key Principles**:

1. **Always show universal metrics** - Health score, trend, complexity
2. **Framework-specific metrics in separate section** - Clearly labeled
3. **"N/A" for inapplicable metrics** - Don't hide, explain absence
4. **Guidance text** - Tell users which metrics are comparable

#### 2.3 Portfolio-Level Technology Insights

**Technology Distribution Visualization**:

```
Technology Stack Overview
================================================================================

Frameworks Detected:
Next.js (33%):  2 repos  ████████████░░░░░░░░░░░░░░░░░░░░
React (50%):    3 repos  ██████████████████░░░░░░░░░░░░░░
TypeScript (17%): 1 repo ██████░░░░░░░░░░░░░░░░░░░░░░░░░░

Framework-Specific Health:
┌────────────────────────────────────────────┐
│ Next.js Repos      Avg: 78  [=======--]    │
│   Issues: 5 RSC, 3 Data Fetching          │
├────────────────────────────────────────────┤
│ React Repos        Avg: 65  [======---]    │
│   Issues: 25 God Components, 12 Hooks     │
├────────────────────────────────────────────┤
│ TypeScript Repos   Avg: 82  [========-]    │
│   Issues: 2 Type Safety                   │
└────────────────────────────────────────────┘

Recommendation:
Focus on React repos - they have lower average health
and more architectural anti-pattern issues than Next.js repos.
```

---

## Recommendation 3: Architectural AntiPatterns Framework Extension

### Problem

The Architectural AntiPatterns Detection feature (US-NEW-04) currently defines generic anti-patterns. Next.js introduces framework-specific anti-patterns that need dedicated detection and presentation.

### Solution: Framework-Aware Anti-Pattern Categories

#### 3.1 Next.js-Specific Architectural AntiPatterns

**Extended Anti-Pattern Categories**:

| Anti-Pattern                           | Applies To      | Detection Signal                           | Severity Threshold |
| -------------------------------------- | --------------- | ------------------------------------------ | ------------------ |
| **Server/Client Boundary Violation**   | Next.js         | Async component with 'use client'          | Critical           |
| **Serialization Violation**            | Next.js         | Non-serializable props to Client Component | Critical           |
| **Waterfall Data Fetching**            | Next.js         | Sequential awaits in RSC                   | Warning            |
| **Missing Suspense Boundary**          | Next.js         | Async component without Suspense           | Warning            |
| **Over-Dynamic Rendering**             | Next.js         | cookies()/headers() in static route intent | Warning            |
| **Route Segment Misconfiguration**     | Next.js         | Conflicting dynamic/revalidate settings    | Critical           |
| **Pages Router in App Router Project** | Next.js         | Mixed routing paradigms                    | Info               |
| **Client Component Prop Drilling**     | Next.js + React | Props through 3+ levels with 'use client'  | Warning            |

**Anti-Pattern Detection UI Enhancement**:

```
Architectural AntiPatterns Detection
================================================================================

Overview | Trends | Configuration

Filter: [All ▼] [Framework-Specific ▼] [Severity ▼]

┌──────────────────────────┐  ┌──────────────────────────┐
│ Server/Client Violations │  │ God Components           │
│ [Next.js Specific]       │  │ [Framework Agnostic]     │
│                          │  │                          │
│    [!!!] 4               │  │    [!!!] 7               │
│    Critical: 3           │  │    Critical: 3           │
│    Warning: 1            │  │    Warning: 4            │
│                          │  │                          │
│    Trend: +2 [!!!]       │  │    Trend: +1             │
│                          │  │                          │
│ Next.js App Router files │  │ Applies to all files     │
└──────────────────────────┘  └──────────────────────────┘

┌──────────────────────────┐  ┌──────────────────────────┐
│ Serialization Issues     │  │ Circular Dependencies    │
│ [Next.js Specific]       │  │ [Framework Agnostic]     │
│                          │  │                          │
│    [!] 6                 │  │    [!] 3                 │
│    Critical: 2           │  │    Critical: 1           │
│    Warning: 4            │  │    Warning: 2            │
│                          │  │                          │
│    Trend: 0              │  │    Trend: 0              │
│                          │  │                          │
│ RSC → Client Component   │  │ Applies to all files     │
└──────────────────────────┘  └──────────────────────────┘
```

**Key Features**:

1. **Framework Badge on Card** - "[Next.js Specific]" or "[Framework Agnostic]"
2. **Tooltip Context** - Explains where anti-pattern applies
3. **Filtering** - "Show only Next.js anti-patterns" or "Show only React anti-patterns"
4. **Scope Indicator** - Text below card explains applicability

#### 3.2 Anti-Pattern Detail Panel Enhancement

**Framework-Aware Recommendations**:

````
┌────────────────────────────────────────────────────────────┐
│ Server/Client Boundary Violation: UserProfile.tsx         │
│ [Next.js Specific] [App Router]                           │
├────────────────────────────────────────────────────────────┤
│ Severity: Critical                                         │
│                                                            │
│ This component violates Next.js Server/Client boundary:   │
│                                                            │
│ Problem:                                                   │
│ Component has 'use client' directive but is marked async. │
│ Async components must be Server Components.               │
│                                                            │
│ Location: Line 12                                          │
│ ```                                                        │
│ 'use client'                                               │
│                                                            │
│ export async function UserProfile() {  // ← Error here    │
│   const user = await fetchUser()                          │
│   return <div>{user.name}</div>                           │
│ }                                                          │
│ ```                                                        │
│                                                            │
│ Next.js Constraints:                                       │
│ ⚠️  Client Components cannot be async                      │
│ ⚠️  Data fetching must happen in Server Components         │
│                                                            │
│ RECOMMENDED REFACTORING:                                   │
│ Option 1: Remove 'use client' (make it Server Component)  │
│ Option 2: Move data fetching to parent Server Component   │
│                                                            │
│ Example (Option 2):                                        │
│ ```                                                        │
│ // UserProfileWrapper.tsx (Server Component)              │
│ export async function UserProfileWrapper() {              │
│   const user = await fetchUser()                          │
│   return <UserProfile user={user} />                      │
│ }                                                          │
│                                                            │
│ // UserProfile.tsx (Client Component)                     │
│ 'use client'                                               │
│ export function UserProfile({ user }) {                   │
│   return <div>{user.name}</div>                           │
│ }                                                          │
│ ```                                                        │
│                                                            │
│ Estimated Effort: 15-30 minutes                           │
│                                                            │
│ [Generate AI Refactoring Prompt]  [Open in IDE]  [Docs]  │
└────────────────────────────────────────────────────────────┘
````

**Key Enhancements**:

1. **Framework Badge in Title** - Clear visual marker
2. **"Next.js Constraints" Section** - Explains framework rules
3. **Code Examples** - Show before/after with Next.js context
4. **Link to Docs** - Direct link to Next.js documentation

---

## Recommendation 4: Blast Radius Framework Sensitivity

### Problem

The Blast Radius Hotspot View (US-NEW-01) calculates impact based on dependency graph. Next.js introduces new impact dimensions: Server/Client boundary crossings amplify blast radius.

### Solution: Framework-Aware Blast Radius Calculation

#### 4.1 Next.js Blast Radius Formula Extension

**Enhanced Formula**:

```typescript
BlastRadius = ComplexityScore × DependencyFactor × FrameworkMultiplier

Where:
  ComplexityScore = (existing calculation)
  DependencyFactor = (existing calculation)

  FrameworkMultiplier = {
    1.0   // Default (no framework-specific risk)
    1.3   // Server Component with many dependents
    1.5   // Client Component imported by Server Component (serialization risk)
    1.8   // Route Handler with many consumers
    2.0   // Shared Server Action (high coupling)
  }
```

**Rationale**:

- **Server Components**: Changes affect all dependents immediately (no client bundle separation)
- **Client Components in RSC tree**: Breaking changes require serialization fixes across boundary
- **Route Handlers**: API contract changes ripple to all consumers
- **Server Actions**: Shared mutations create tight coupling

#### 4.2 Framework-Aware Hotspot Visualization

**Treemap Color Scale Extension**:

```
Standard Hotspot Colors:
  Green  (#22C55E) = Score 0-25   (Low Risk)
  Yellow (#EAB308) = Score 26-50  (Moderate Risk)
  Orange (#F97316) = Score 51-75  (High Risk)
  Red    (#EF4444) = Score 76-100 (Critical Risk)

Framework-Specific Overlays:
  [RSC] Badge     = Purple border (Server Component)
  [Client] Badge  = Blue border (Client Component)
  [Action] Badge  = Teal border (Server Action)
  [Route] Badge   = Gray border (Route Handler)
```

**Visual Example**:

```
┌────────────────────────────────────────────────────────────┐
│  src/                                                      │
│  ┌──────────────────────┐  ┌──────────────────────────────┤
│  │ app/                 │  │ components/                  │
│  │ ┌────────┬─────────┐ │  │ ┌──────────┬──────┬────────┐ │
│  │ │ page   │ layout  │ │  │ │ Button   │Avatar│Header  │ │
│  │ │ [RSC]  │ [RSC]   │ │  │ │ [Client] │[Cli] │[Cli]   │ │
│  │ │ (LOW)  │ (HIGH)  │ │  │ │ (LOW)    │(LOW) │(HIGH)  │ │
│  │ └────────┴─────────┘ │  │ └──────────┴──────┴────────┘ │
│  │ ┌────────────────┐   │  │ ┌──────────────────────────┐ │
│  │ │ route.ts       │   │  │ │ UserProfile              │ │
│  │ │ [Route Handler]│   │  │ │ [RSC with Client imports]│ │
│  │ │ (CRITICAL)     │   │  │ │ (CRITICAL)               │ │
│  │ └────────────────┘   │  │ └──────────────────────────┘ │
│  └──────────────────────┘  └──────────────────────────────┘
└────────────────────────────────────────────────────────────┘

Legend:
  [RSC]     = Server Component (purple border)
  [Client]  = Client Component (blue border)
  [Route]   = Route Handler (gray border)
```

#### 4.3 Framework-Specific Hotspot Patterns

**Extended Pattern Recognition**:

**1. RSC Boundary Hub** (Next.js-specific)

- Pattern: Server Component imports 10+ Client Components
- Problem: Serialization surface area is massive
- Risk Multiplier: 1.5x
- Recommendation: Extract composition logic to separate files

**2. Shared Server Action** (Next.js-specific)

- Pattern: Server Action called from 20+ places
- Problem: Changes break many consumers
- Risk Multiplier: 2.0x
- Recommendation: Versioning strategy, careful API design

**3. Dynamic Route Handler** (Next.js-specific)

- Pattern: Route handler with complex logic and many dependencies
- Problem: API contract changes ripple widely
- Risk Multiplier: 1.8x
- Recommendation: OpenAPI schema, versioned endpoints

---

## Recommendation 5: Future Framework Extensibility

### Problem

As Vue, Angular, and Svelte analyzers are added, the UX must adapt gracefully without requiring redesigns.

### Solution: Extensible Framework Metadata System

#### 5.1 Framework Registry Pattern

**Framework Metadata Structure**:

```typescript
interface FrameworkMetadata {
  id: string; // 'nextjs', 'react', 'vue', 'angular'
  name: string; // 'Next.js', 'React', 'Vue', 'Angular'
  version?: string; // Detected version
  color: string; // Badge color
  icon: string; // Icon identifier

  // Metric categories this framework provides
  metricCategories: {
    id: string; // 'rsc-violations', 'reactivity-issues'
    label: string; // 'Server/Client Violations'
    description: string;
    universalEquivalent?: string; // Maps to universal metric for comparison
  }[];

  // File types this framework detects
  fileTypes: {
    type: FileType;
    label: string;
    icon: string;
    color: string;
  }[];

  // Architectural anti-patterns this framework detects
  architecturalAntiPatterns: {
    id: string;
    name: string;
    severity: 'critical' | 'warning' | 'info';
    category: 'coupling' | 'responsibility' | 'data' | 'change';
  }[];
}
```

**Framework Registry Implementation**:

```typescript
class FrameworkRegistry {
  private frameworks = new Map<string, FrameworkMetadata>();

  register(metadata: FrameworkMetadata): void {
    this.frameworks.set(metadata.id, metadata);
  }

  getAvailableFrameworks(): FrameworkMetadata[] {
    return Array.from(this.frameworks.values());
  }

  getMetricCategories(frameworkId: string): MetricCategory[] {
    return this.frameworks.get(frameworkId)?.metricCategories || [];
  }

  getComparableMetrics(framework1: string, framework2: string): string[] {
    const f1Categories = this.getMetricCategories(framework1);
    const f2Categories = this.getMetricCategories(framework2);

    // Find metrics with same universalEquivalent
    const comparable = f1Categories
      .filter(c1 => c1.universalEquivalent)
      .filter(c1 => f2Categories.some(c2 => c2.universalEquivalent === c1.universalEquivalent))
      .map(c => c.universalEquivalent!);

    return [...new Set(comparable)];
  }
}
```

#### 5.2 Dynamic UI Component Generation

**Metric Card Component**:

```typescript
interface MetricCardProps {
  framework: FrameworkMetadata;
  category: MetricCategory;
  value: number;
  trend?: number;
}

function MetricCard({ framework, category, value, trend }: MetricCardProps) {
  return (
    <Card
      color={framework.color}
      icon={framework.icon}
      title={category.label}
      tooltip={category.description}
    >
      <Badge color={framework.color}>{framework.name}</Badge>
      <MetricValue value={value} trend={trend} />

      {category.universalEquivalent && (
        <Tooltip content="Comparable across frameworks">
          <CompareIcon />
        </Tooltip>
      )}
    </Card>
  );
}
```

**File Type Badge Component**:

```typescript
function FileTypeBadge({ fileType, framework }: Props) {
  const metadata = FrameworkRegistry.getFileTypeMetadata(framework, fileType);

  return (
    <Badge
      color={metadata.color}
      icon={metadata.icon}
      tooltip={`${framework.name}: ${metadata.label}`}
    >
      {metadata.label}
    </Badge>
  );
}
```

#### 5.3 Framework Filter Pattern

**Universal Filtering Component**:

```typescript
function FrameworkFilter({ repositories }: Props) {
  const frameworks = FrameworkRegistry.getAvailableFrameworks();
  const [selectedFrameworks, setSelectedFrameworks] = useState<string[]>([]);

  const reposByFramework = useMemo(() => {
    return repositories.filter(repo =>
      selectedFrameworks.length === 0 ||
      selectedFrameworks.includes(repo.framework)
    );
  }, [repositories, selectedFrameworks]);

  return (
    <FilterGroup label="Technology">
      {frameworks.map(fw => (
        <FilterOption
          key={fw.id}
          icon={fw.icon}
          color={fw.color}
          label={fw.name}
          count={countRepos(repositories, fw.id)}
          selected={selectedFrameworks.includes(fw.id)}
          onToggle={() => toggleFramework(fw.id)}
        />
      ))}
    </FilterGroup>
  );
}
```

#### 5.4 Comparison Matrix Extensibility

**Framework-Agnostic Comparison**:

```typescript
interface ComparisonRow {
  label: string;
  isUniversal: boolean;
  frameworkSpecific?: string; // Only for framework-specific rows
  values: Map<string, number | string>; // repoId -> value
}

function buildComparisonMatrix(repos: Repository[]): ComparisonRow[] {
  const rows: ComparisonRow[] = [];

  // Universal metrics (always included)
  rows.push({
    label: 'Health Score',
    isUniversal: true,
    values: new Map(repos.map(r => [r.id, r.healthScore])),
  });

  rows.push({
    label: 'Complexity Trend',
    isUniversal: true,
    values: new Map(repos.map(r => [r.id, r.complexityTrend])),
  });

  // Framework-specific metrics (conditionally included)
  const frameworks = [...new Set(repos.map(r => r.framework))];

  for (const framework of frameworks) {
    const categories = FrameworkRegistry.getMetricCategories(framework);

    for (const category of categories) {
      rows.push({
        label: category.label,
        isUniversal: false,
        frameworkSpecific: framework,
        values: new Map(
          repos.filter(r => r.framework === framework).map(r => [r.id, r.metrics[category.id]])
        ),
      });
    }
  }

  return rows;
}
```

---

## Recommendation 6: Technology-Specific AI Prompt Generation

### Problem

The One-Click AI Prompt Generation feature (US-NEW-19) should provide framework-aware context to AI assistants.

### Solution: Framework-Contextual Prompt Templates

#### 6.1 Next.js-Specific Prompt Enhancement

**Standard Prompt** (framework-agnostic):

```
Refactor this file to reduce complexity:

File: UserProfile.tsx
Complexity: 68
Issues:
- High cyclomatic complexity
- Many dependencies
- Long function

Current code:
[code snippet]
```

**Next.js-Enhanced Prompt**:

```
Refactor this Next.js Server Component to reduce complexity:

File: UserProfile.tsx
Framework: Next.js 15 (App Router)
File Type: Server Component
Execution Context: Server-side only
Complexity: 68

Framework-Specific Issues:
- High cyclomatic complexity (68)
- Server/Client boundary: Imports 12 Client Components
- Serialization surface: Passing complex objects as props
- Data fetching: Sequential awaits causing waterfalls

Next.js Constraints to Maintain:
✓ Must remain Server Component (async data fetching)
✓ Props passed to Client Components must be serializable
✓ Can use database/file system directly
✗ Cannot use useState, useEffect, or browser APIs

Current code:
[code snippet]

Suggested Focus Areas:
1. Extract Client Component composition to separate file
2. Parallelize data fetching with Promise.all()
3. Simplify props passed across Server/Client boundary
4. Consider Server Actions for mutations
```

#### 6.2 Framework-Aware Prompt Template System

**Template Structure**:

```typescript
interface PromptTemplate {
  id: string;
  name: string;
  frameworks: string[]; // ['nextjs', 'react'] or ['*'] for universal

  sections: {
    context: (file: AnalyzedFile) => string;
    constraints: (file: AnalyzedFile) => string[];
    suggestions: (file: AnalyzedFile) => string[];
  };
}

const NEXTJS_RSC_REFACTORING: PromptTemplate = {
  id: 'nextjs-rsc-refactoring',
  name: 'Next.js Server Component Refactoring',
  frameworks: ['nextjs'],

  sections: {
    context: file => `
      Framework: ${file.framework} ${file.frameworkVersion} (${file.routerType})
      File Type: ${file.fileType}
      Execution Context: ${file.executionContext}
      Complexity: ${file.complexity}
    `,

    constraints: file => {
      if (file.fileType === 'server-component') {
        return [
          '✓ Must remain Server Component (async data fetching)',
          '✓ Props to Client Components must be serializable',
          '✓ Can use database/file system directly',
          '✗ Cannot use useState, useEffect, or browser APIs',
        ];
      }
      // ... other file types
    },

    suggestions: file => {
      const suggestions = [];

      if (file.insights.some(i => i.category === 'data-fetching-waterfall')) {
        suggestions.push('Parallelize data fetching with Promise.all()');
      }

      if (file.insights.some(i => i.category === 'serialization-violation')) {
        suggestions.push('Simplify props passed across Server/Client boundary');
      }

      // ... more conditional suggestions

      return suggestions;
    },
  },
};
```

---

## Recommendation 7: Framework-Specific Dashboard Widgets

### Problem

The main Dashboard should surface framework-specific insights without overwhelming users with irrelevant information.

### Solution: Contextual Widget System

#### 7.1 Dynamic Widget Registry

**Widget Metadata**:

```typescript
interface DashboardWidget {
  id: string;
  title: string;
  frameworks: string[]; // Which frameworks this applies to
  priority: number;     // Display order

  shouldDisplay: (repo: Repository) => boolean;
  render: (data: WidgetData) => ReactNode;
}

const NEXTJS_RSC_ISSUES_WIDGET: DashboardWidget = {
  id: 'nextjs-rsc-issues',
  title: 'Server/Client Boundary Issues',
  frameworks: ['nextjs'],
  priority: 2,

  shouldDisplay: (repo) => {
    return repo.framework === 'nextjs' &&
           repo.metrics.nextjs?.serverClient?.violations > 0;
  },

  render: (data) => (
    <Card icon="server" color="purple">
      <CardHeader>
        <Badge>Next.js</Badge>
        <Title>Server/Client Boundary Issues</Title>
      </CardHeader>
      <MetricValue
        value={data.violations}
        severity={getSeverity(data.violations)}
      />
      <IssueList issues={data.issues} limit={3} />
      <CardAction>View All RSC Issues</CardAction>
    </Card>
  )
};
```

#### 7.2 Adaptive Dashboard Layout

**Repository-Specific Dashboard**:

```
Dashboard: my-app (Next.js 15 App Router)
================================================================================

┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────┐
│ Health Score           │  │ Complexity Trend       │  │ Critical Issues    │
│ [Universal]            │  │ [Universal]            │  │ [Universal]        │
│                        │  │                        │  │                    │
│        85              │  │     +2.3/week          │  │        3           │
│   [=========-]         │  │   ──────────────       │  │   RSC: 2           │
│                        │  │   Improving ↑          │  │   Hook: 1          │
└────────────────────────┘  └────────────────────────┘  └────────────────────┘

┌──────────────────────────────────────┐  ┌────────────────────────────────┐
│ Server/Client Boundary Issues        │  │ Data Fetching Performance      │
│ [Next.js Specific]                   │  │ [Next.js Specific]             │
│                                      │  │                                │
│    2 Critical                        │  │    3 Waterfalls Detected       │
│                                      │  │    12 Sequential Fetches       │
│    • UserProfile.tsx: Async with    │  │                                │
│      'use client'                    │  │    Recommendation:             │
│    • Dashboard.tsx: Non-serializable│  │    Parallelize with            │
│      props                           │  │    Promise.all()               │
│                                      │  │                                │
│    [View RSC Issues]                 │  │    [View Details]              │
└──────────────────────────────────────┘  └────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ Blast Radius Hotspots                                                    │
│ [Universal + Framework-Enhanced]                                         │
│                                                                          │
│  [Treemap visualization with Next.js badges]                           │
└──────────────────────────────────────────────────────────────────────────┘
```

**Plain React Repository Dashboard** (for comparison):

```
Dashboard: legacy-dashboard (React 16)
================================================================================

┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────┐
│ Health Score           │  │ Complexity Trend       │  │ Critical Issues    │
│ [Universal]            │  │ [Universal]            │  │ [Universal]        │
│                        │  │                        │  │                    │
│        45              │  │     -3.1/week          │  │        24          │
│   [====------]         │  │   ──────────────       │  │   God Comp: 12     │
│                        │  │   Declining ↓          │  │   Coupling: 8      │
└────────────────────────┘  └────────────────────────┘  └────────────────────┘

┌──────────────────────────────────────┐  ┌────────────────────────────────┐
│ God Components                        │  │ Prop Drilling Issues           │
│ [React Specific]                     │  │ [React Specific]               │
│                                      │  │                                │
│    12 Detected                       │  │    5 Deep Prop Chains          │
│                                      │  │    Avg Depth: 4 levels         │
│    Top Issues:                       │  │                                │
│    • Dashboard.tsx (CC: 72)          │  │    Recommendation:             │
│    • UserManager.tsx (CC: 68)        │  │    Use Context API or          │
│                                      │  │    state management            │
│    [View Components]                 │  │                                │
│                                      │  │    [View Details]              │
└──────────────────────────────────────┘  └────────────────────────────────┘

Note: Next.js-specific features not available for this repository.
Consider migrating to Next.js for enhanced capabilities.
```

**Key Principles**:

1. **Universal widgets always shown** - Health, trend, critical issues
2. **Framework-specific widgets conditionally shown** - Only if applicable
3. **Clear labeling** - Badge indicates universal vs framework-specific
4. **Educational hints** - React-only repos get hint about Next.js

---

## Implementation Checklist

### Phase 1: Visual Foundation (Week 1-2)

- [ ] Implement framework badge component system
- [ ] Create execution context indicator components
- [ ] Design and implement badge color system
- [ ] Add file type badges to file list views
- [ ] Update Blast Radius treemap with framework overlays

### Phase 2: Multi-Repository Workspace (Week 3-4)

- [ ] Add technology detection to workspace dashboard
- [ ] Implement technology filtering controls
- [ ] Create framework-aware comparison view
- [ ] Build comparative metrics normalization system
- [ ] Add portfolio-level technology insights

### Phase 3: Architectural AntiPatterns (Week 5-6)

- [ ] Extend anti-pattern detection for Next.js patterns
- [ ] Add framework badges to anti-pattern category cards
- [ ] Implement framework-specific anti-pattern detail panels
- [ ] Create framework filtering in anti-pattern view
- [ ] Add educational content for framework constraints

### Phase 4: Framework Extensibility (Week 7-8)

- [ ] Implement FrameworkRegistry system
- [ ] Create dynamic metric card components
- [ ] Build framework filter pattern
- [ ] Implement extensible comparison matrix
- [ ] Create framework-aware prompt template system

### Phase 5: Dashboard & Polish (Week 9-10)

- [ ] Implement dynamic widget registry
- [ ] Create adaptive dashboard layout system
- [ ] Add framework-specific dashboard widgets
- [ ] Polish all framework badge styling
- [ ] Add comprehensive tooltips and educational content

---

## Success Metrics

### User Experience Metrics

| Metric                                | Target      | Measurement                                   |
| ------------------------------------- | ----------- | --------------------------------------------- |
| Framework recognition time            | < 3 seconds | User identifies file's framework from badges  |
| Technology filtering usage            | > 60%       | Users filter by technology in multi-repo view |
| Framework-specific insight engagement | > 40%       | Users click into framework-specific issues    |
| Cross-framework comparison usage      | > 30%       | Users compare repos with different stacks     |

### Technical Metrics

| Metric                             | Target    | Measurement                               |
| ---------------------------------- | --------- | ----------------------------------------- |
| Framework badge render performance | < 16ms    | No layout shift from badge addition       |
| Dynamic widget load time           | < 100ms   | Framework-specific widgets appear quickly |
| Comparison matrix generation       | < 500ms   | Multi-repo comparison renders quickly     |
| Framework registry extensibility   | < 2 hours | Time to add new framework support         |

### Business Metrics

| Metric                               | Target | Measurement                                      |
| ------------------------------------ | ------ | ------------------------------------------------ |
| Framework migration decision support | > 50%  | Users report dashboard helped migration planning |
| Multi-framework team adoption        | > 70%  | Teams managing multiple frameworks adopt tool    |
| Framework-specific issue resolution  | +30%   | Faster resolution with framework context         |

---

## Open Questions for Design Review

1. **Badge Density**: With 3 types of badges (Framework, Execution, Architecture), do we risk visual clutter? Should we consolidate?

2. **Color System**: Should framework colors match official brand colors (Next.js black, React cyan), or use a unified color system for consistency?

3. **Comparison Defaults**: When comparing repos with different frameworks, which metrics should be shown by default?

4. **Framework Version Display**: Should we always show version (Next.js 15, React 18) or only when relevant?

5. **Educational Content**: How much Next.js-specific documentation should be embedded vs. linked externally?

6. **Mobile/Small Screen**: Badge strategy breaks down on small screens. What's the mobile fallback?

7. **Accessibility**: Are framework badges screen-reader friendly? Need alternative text strategy?

8. **Performance**: Framework badges add DOM nodes. At what file count does this become a problem?

---

## Appendix A: Framework Comparison Matrix

### Universal Metrics (All Frameworks)

| Metric                | Next.js | React | Vue | Angular | Svelte |
| --------------------- | ------- | ----- | --- | ------- | ------ |
| Health Score          | ✓       | ✓     | ✓   | ✓       | ✓      |
| Complexity (CC)       | ✓       | ✓     | ✓   | ✓       | ✓      |
| Maintainability Index | ✓       | ✓     | ✓   | ✓       | ✓      |
| God Components        | ✓       | ✓     | ✓   | ✓       | ✓      |
| Circular Dependencies | ✓       | ✓     | ✓   | ✓       | ✓      |
| Prop Drilling         | ✓       | ✓     | ✓   | ✓       | ✓      |

### Framework-Specific Metrics

| Metric                 | Next.js | React | Vue | Angular | Svelte |
| ---------------------- | ------- | ----- | --- | ------- | ------ |
| RSC Violations         | ✓       | ✗     | ✗   | ✗       | ✗      |
| Data Fetching Patterns | ✓       | ✗     | ✗   | ✗       | ✗      |
| Route Structure        | ✓       | ✗     | ✗   | ✗       | ✗      |
| Hook Patterns          | ✓       | ✓     | ✗   | ✗       | ✗      |
| Reactivity Issues      | ✗       | ✗     | ✓   | ✗       | ✓      |
| Dependency Injection   | ✗       | ✗     | ✗   | ✓       | ✗      |
| RxJS Patterns          | ✗       | ✗     | ✗   | ✓       | ✗      |
| Store Patterns         | ✗       | ✗     | ✓   | ✓       | ✓      |

---

## Appendix B: Example Framework Metadata

### Next.js Framework Metadata

```typescript
const NEXTJS_METADATA: FrameworkMetadata = {
  id: 'nextjs',
  name: 'Next.js',
  color: '#000000',
  icon: 'nextjs-logo',

  metricCategories: [
    {
      id: 'rsc-violations',
      label: 'Server/Client Boundary Issues',
      description: 'Violations of Next.js Server Component constraints',
      universalEquivalent: 'boundary-violations',
    },
    {
      id: 'data-fetching',
      label: 'Data Fetching Patterns',
      description: 'Server-side data fetching optimization opportunities',
      universalEquivalent: 'performance-issues',
    },
    {
      id: 'route-structure',
      label: 'Route Organization',
      description: 'App Router file convention compliance',
      universalEquivalent: 'structural-organization',
    },
  ],

  fileTypes: [
    { type: 'page', label: 'Page', icon: 'file-code', color: '#F59E0B' },
    { type: 'layout', label: 'Layout', icon: 'layout', color: '#F59E0B' },
    { type: 'server-component', label: 'Server Component', icon: 'server', color: '#10B981' },
    { type: 'server-action', label: 'Server Action', icon: 'run', color: '#10B981' },
    { type: 'route', label: 'Route Handler', icon: 'globe', color: '#6B7280' },
  ],

  architecturalAntiPatterns: [
    {
      id: 'rsc-boundary-violation',
      name: 'Server/Client Boundary Violation',
      severity: 'critical',
      category: 'coupling',
    },
    {
      id: 'serialization-violation',
      name: 'Non-Serializable Props',
      severity: 'critical',
      category: 'data',
    },
    {
      id: 'data-fetching-waterfall',
      name: 'Data Fetching Waterfall',
      severity: 'warning',
      category: 'change',
    },
  ],
};
```

### React Framework Metadata

```typescript
const REACT_METADATA: FrameworkMetadata = {
  id: 'react',
  name: 'React',
  color: '#61DAFB',
  icon: 'react-logo',

  metricCategories: [
    {
      id: 'hook-violations',
      label: 'Hook Pattern Issues',
      description: 'Violations of React Hooks rules',
      universalEquivalent: 'pattern-violations',
    },
    {
      id: 'prop-drilling',
      label: 'Prop Drilling',
      description: 'Deep prop passing chains',
      universalEquivalent: 'coupling-issues',
    },
    {
      id: 'component-quality',
      label: 'Component Quality',
      description: 'Component design and reusability',
      universalEquivalent: 'code-quality',
    },
  ],

  fileTypes: [
    { type: 'component', label: 'Component', icon: 'symbol-class', color: '#61DAFB' },
    { type: 'hook', label: 'Hook', icon: 'symbol-method', color: '#61DAFB' },
    { type: 'context', label: 'Context', icon: 'symbol-variable', color: '#61DAFB' },
    { type: 'hoc', label: 'HOC', icon: 'symbol-function', color: '#61DAFB' },
  ],

  architecturalAntiPatterns: [
    {
      id: 'god-component',
      name: 'God Component',
      severity: 'critical',
      category: 'responsibility',
    },
    {
      id: 'prop-drilling-deep',
      name: 'Deep Prop Drilling',
      severity: 'warning',
      category: 'coupling',
    },
  ],
};
```

---

## Conclusion

These recommendations provide a comprehensive approach to presenting Next.js-specific metrics while establishing patterns that will scale to future framework analyzers. The key principles are:

1. **Framework-awareness without framework-specificity** - UI adapts to available metrics
2. **Clear visual hierarchy** - Badges and colors convey framework context instantly
3. **Graceful degradation** - Works equally well for single-framework and multi-framework teams
4. **Extensibility by design** - Adding Vue/Angular/Svelte requires minimal UX changes
5. **Educational scaffolding** - Framework constraints are explained, not just flagged

The recommendations balance sophistication with pragmatism, delivering immediate value for Next.js while future-proofing for framework diversity.
