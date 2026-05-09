---
id: 01-problematic-patterns
---

# Problematic Next.js Patterns: Research-Backed Analysis Requirements

This document catalogs the most impactful anti-patterns in Next.js applications, organized by router architecture, with specific guidance on what static analysis must understand to detect them accurately.

## Category 1: Server/Client Boundary Anti-Patterns (App Router)

### 1.1 Missing "use client" Directive

**Pattern**: Using client-only APIs without the directive

```tsx
// ❌ Will fail at build time - no "use client"
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0); // Error: hooks in Server Component
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Why naive detection fails**:

```tsx
// Naive rule: "Flag useState without 'use client'"
// This flags BOTH cases below identically:

// Case A: Genuinely problematic - needs directive
export default function Counter() {
  const [count, setCount] = useState(0);
  return <button>{count}</button>;
}

// Case B: Fine - parent has directive, this is imported by client
// File: components/counter-display.tsx (no directive needed)
export function CounterDisplay({ count }: { count: number }) {
  return <span>{count}</span>;
}
// File: components/counter.tsx
('use client');
import { CounterDisplay } from './counter-display';
// CounterDisplay becomes client component through import
```

**What analysis must determine**:

1. **Import chain analysis**
   - Is this file imported (directly or transitively) by a `"use client"` file?
   - Build the import graph to trace client boundaries

2. **File role detection**
   - Is this a page/layout/route (entry point) or a shared component?
   - Entry points must declare their own boundary

3. **API usage classification**
   - Which APIs require client context (hooks, browser APIs)?
   - Which are server-only (`cookies()`, `headers()`)?

**ts-morph analysis approach**:

```typescript
function needsUseClientDirective(sourceFile: SourceFile, project: Project): boolean {
  // Check if already has directive
  if (hasUseClientDirective(sourceFile)) return false;

  // Check if imported by a client file (inherits client context)
  if (isImportedByClientFile(sourceFile, project)) return false;

  // Check for client-only API usage
  const clientAPIs = findClientOnlyAPIUsage(sourceFile);
  return clientAPIs.length > 0;
}

function isImportedByClientFile(sourceFile: SourceFile, project: Project): boolean {
  const importers = findAllImporters(sourceFile, project);

  for (const importer of importers) {
    if (hasUseClientDirective(importer)) return true;
    if (isImportedByClientFile(importer, project)) return true;
  }

  return false;
}
```

### 1.2 Unnecessary "use client" Directive

**Pattern**: Adding directive to components that don't need it

```tsx
// ❌ Unnecessary - no client APIs used
'use client';

export default function Header({ title }: { title: string }) {
  return (
    <header>
      <h1>{title}</h1>
    </header>
  );
}
```

**Why this matters**:

- Prevents server-side rendering benefits
- Increases client bundle size
- Loses ability to do async data fetching in component

**What analysis must determine**:

1. **Client API dependency**
   - Does the component use hooks, event handlers, browser APIs?
   - Does it import client-only libraries?

2. **Propagation necessity**
   - Does it need to pass client context to children?
   - Could children be refactored to be server components?

**Detection criteria**:

- Has `"use client"` directive
- No useState, useEffect, useRef, or other hooks
- No onClick, onChange, or other event handlers
- No browser API access (window, document, localStorage)
- No imports from known client-only packages

### 1.3 Server-Only Code in Client Components

**Pattern**: Importing server-only modules in client components

```tsx
"use client";
import { db } from '@/lib/database';  // ❌ Database client in browser!
import { headers } from 'next/headers';  // ❌ Server-only API

export default function Profile() {
  // This will fail or leak sensitive code
  const user = db.users.find(...);
}
```

**What analysis must determine**:

1. **Import classification**
   - Is the imported module server-only?
   - Does it use `server-only` package marker?
   - Does it access Node.js APIs, databases, or secrets?

2. **Transitive analysis**
   - Does importing this module pull in server code transitively?

**Server-only indicators**:

```typescript
const SERVER_ONLY_PATTERNS = {
  // Next.js server APIs
  nextServerImports: ['next/headers', 'next/cookies'],

  // Common server-only packages
  serverPackages: [
    'database',
    'prisma',
    '@prisma/client',
    'drizzle-orm',
    'mongoose',
    'pg',
    'mysql2',
    'fs',
    'path',
    'crypto',
    'child_process',
    'nodemailer',
    'bcrypt',
    'jsonwebtoken',
  ],

  // Environment patterns
  envPatterns: [/process\.env\.(DATABASE|SECRET|PRIVATE|API_KEY)/],

  // Explicit markers
  explicitMarkers: ['server-only'],
};
```

### 1.4 Non-Serializable Props Across Boundary

**Pattern**: Passing functions or non-serializable values from Server to Client Components

```tsx
// Server Component
export default async function Page() {
  const handleClick = () => console.log('clicked'); // ❌ Function!
  const data = {
    date: new Date(), // ❌ Date object!
    regex: /pattern/, // ❌ RegExp!
    fn: () => {}, // ❌ Function!
  };

  return <ClientComponent onClick={handleClick} data={data} />;
}

// Client Component
('use client');
export function ClientComponent({ onClick, data }) {
  // Props were serialized - functions become undefined!
}
```

**What analysis must determine**:

1. **Boundary crossing detection**
   - Is a Server Component rendering a Client Component?
   - What props are passed across the boundary?

2. **Serializability analysis**
   - Are prop values serializable (primitives, plain objects, arrays)?
   - Do they contain functions, Dates, RegExp, Map, Set, etc.?

**Serializable types**:

```typescript
const SERIALIZABLE_TYPES = [
  'string',
  'number',
  'boolean',
  'null',
  'undefined',
  'bigint', // Serialized as string
  'Array', // If contents are serializable
  'Object', // Plain objects with serializable values
];

const NON_SERIALIZABLE_TYPES = [
  'Function',
  'Date',
  'RegExp',
  'Map',
  'Set',
  'WeakMap',
  'WeakSet',
  'Symbol',
  'Error',
  'Promise', // Unless resolved
  'Buffer',
  'ArrayBuffer',
  'TypedArray',
];
```

### 1.5 Async Client Components

**Pattern**: Making Client Components async (not supported)

```tsx
'use client';

// ❌ Client Components cannot be async
export default async function ClientPage() {
  const data = await fetchData();
  return <div>{data}</div>;
}
```

**Detection**: Straightforward - async function with `"use client"` directive.

**What analysis should suggest**:

- Move data fetching to Server Component parent
- Use `useEffect` + `useState` for client-side fetching
- Use React Query/SWR for client data fetching

---

## Category 2: Data Fetching Anti-Patterns (App Router)

### 2.1 Data Fetching Waterfalls

**Pattern**: Sequential data fetches that could be parallel

```tsx
// ❌ Sequential - each awaits before next starts
export default async function Page() {
  const user = await getUser(); // 200ms
  const posts = await getPosts(user.id); // 300ms
  const comments = await getComments(); // 200ms
  // Total: 700ms

  return <Dashboard user={user} posts={posts} comments={comments} />;
}

// ✅ Parallel where possible
export default async function Page() {
  const userPromise = getUser();
  const commentsPromise = getComments(); // Independent!

  const user = await userPromise;
  const [posts, comments] = await Promise.all([
    getPosts(user.id), // Depends on user
    commentsPromise,
  ]);
  // Total: ~500ms (user + max(posts, comments))
}
```

**Why naive detection fails**:

```tsx
// Naive rule: "Multiple awaits in sequence = waterfall"
// But sometimes sequential IS required:

// Case A: Problematic waterfall (independent data)
const user = await getUser();
const settings = await getSettings(); // No dependency on user

// Case B: Required sequence (data dependency)
const user = await getUser();
const posts = await getPosts(user.id); // MUST wait for user.id
```

**What analysis must determine**:

1. **Data dependency analysis**
   - Does fetch B use values from fetch A?
   - Track variable flow from one await to another

2. **Independence detection**
   - Can fetches be parallelized with Promise.all?
   - Which fetches have no dependencies?

**ts-morph analysis approach**:

```typescript
interface FetchDependency {
  fetch: CallExpression;
  dependsOn: CallExpression[];
  canParallelize: CallExpression[];
}

function analyzeFetchDependencies(func: FunctionDeclaration): FetchDependency[] {
  const awaits = findAwaitExpressions(func);
  const dependencies: FetchDependency[] = [];

  for (const awaitExpr of awaits) {
    const fetchCall = awaitExpr.getExpression();
    const usedIdentifiers = findIdentifiersInCall(fetchCall);

    // Find which previous awaits provide these identifiers
    const dependsOn = awaits
      .filter(a => a.getStart() < awaitExpr.getStart())
      .filter(a => {
        const assignedVar = getAssignedVariable(a);
        return usedIdentifiers.some(id => id.getText() === assignedVar);
      });

    // Fetches we don't depend on could run in parallel
    const canParallelize = awaits
      .filter(a => a.getStart() < awaitExpr.getStart())
      .filter(a => !dependsOn.includes(a));

    dependencies.push({ fetch: fetchCall, dependsOn, canParallelize });
  }

  return dependencies;
}
```

### 2.2 Fetch Without Error Handling

**Pattern**: Uncaught fetch errors crashing the page

```tsx
// ❌ No error handling - will throw and show error.tsx
export default async function Page() {
  const data = await fetch('/api/data').then(r => r.json());
  return <Display data={data} />;
}

// ✅ With error handling
export default async function Page() {
  const res = await fetch('/api/data');

  if (!res.ok) {
    return <ErrorState message="Failed to load data" />;
  }

  const data = await res.json();
  return <Display data={data} />;
}
```

**What analysis must determine**:

1. **Error boundary presence**
   - Is there an error.tsx in the route segment?
   - Is the fetch wrapped in try/catch?

2. **Response validation**
   - Is `res.ok` checked?
   - Is response type validated?

### 2.3 Missing Suspense Boundaries

**Pattern**: Large async trees without streaming

```tsx
// ❌ Entire page waits for slowest fetch
export default async function Page() {
  const [fast, slow] = await Promise.all([
    getFastData(), // 100ms
    getSlowData(), // 3000ms
  ]);

  return (
    <div>
      <FastSection data={fast} /> {/* User waits 3s to see this */}
      <SlowSection data={slow} />
    </div>
  );
}

// ✅ Stream with Suspense
export default function Page() {
  return (
    <div>
      <Suspense fallback={<FastSkeleton />}>
        <FastSection /> {/* Shows after 100ms */}
      </Suspense>
      <Suspense fallback={<SlowSkeleton />}>
        <SlowSection /> {/* Streams in after 3s */}
      </Suspense>
    </div>
  );
}
```

**What analysis must determine**:

1. **Fetch duration estimation**
   - Are there multiple slow fetches?
   - Could user see partial content sooner?

2. **Component independence**
   - Can slow parts be isolated in separate components?
   - Is there a loading.tsx for the route?

### 2.4 Incorrect Cache Configuration

**Pattern**: Misunderstanding Next.js fetch cache behavior

```tsx
// ❌ Expects fresh data, gets cached
export default async function Page() {
  // Default in Server Components is cached!
  const data = await fetch('https://api.example.com/data');
  // This returns stale data
}

// ❌ Expects caching, forces dynamic
export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store', // Forces dynamic rendering for entire route
  });
}
```

**What analysis must determine**:

1. **Cache strategy coherence**
   - Is the cache option appropriate for the data type?
   - Does the route segment config match fetch cache options?

2. **Revalidation analysis**
   - Is `revalidate` option used appropriately?
   - Are related fetches revalidated together?

**Cache option semantics**:

```typescript
const CACHE_OPTIONS = {
  // force-cache (default in Server Components)
  'force-cache': {
    behavior: 'Cache indefinitely until revalidated',
    whenToUse: 'Static data that rarely changes',
    implications: 'Route can be statically rendered',
  },

  // no-store
  'no-store': {
    behavior: 'Never cache, always fetch fresh',
    whenToUse: 'User-specific or real-time data',
    implications: 'Forces dynamic rendering',
  },

  // revalidate
  revalidate: {
    behavior: 'Cache for N seconds, then refresh',
    whenToUse: 'Data that changes periodically',
    implications: 'ISR behavior',
  },
};
```

---

## Category 3: Data Fetching Anti-Patterns (Pages Router)

### 3.1 getStaticProps with Dynamic Data

**Pattern**: Using getStaticProps for data that changes frequently

```tsx
// ❌ Data is stale immediately after build
export async function getStaticProps() {
  const prices = await fetchStockPrices(); // Changes every second!
  return { props: { prices } };
}

// ✅ Use revalidate for ISR
export async function getStaticProps() {
  const prices = await fetchStockPrices();
  return {
    props: { prices },
    revalidate: 60, // Refresh every minute
  };
}

// ✅ Or use getServerSideProps for real-time
export async function getServerSideProps() {
  const prices = await fetchStockPrices();
  return { props: { prices } };
}
```

**What analysis must determine**:

1. **Data volatility classification**
   - Is the fetched data time-sensitive?
   - How often does the underlying data change?

2. **Revalidation presence**
   - Is `revalidate` option set?
   - Is the interval appropriate for the data type?

### 3.2 getServerSideProps for Static Data

**Pattern**: Using SSR for data that could be static

```tsx
// ❌ Unnecessarily slow - runs on every request
export async function getServerSideProps() {
  const navigation = await fetchNavigation(); // Same for all users!
  const footer = await fetchFooterLinks(); // Never changes!
  return { props: { navigation, footer } };
}

// ✅ Use getStaticProps for static data
export async function getStaticProps() {
  const navigation = await fetchNavigation();
  const footer = await fetchFooterLinks();
  return {
    props: { navigation, footer },
    revalidate: 3600, // Rebuild hourly
  };
}
```

**What analysis must determine**:

1. **Data classification**
   - Is data user-specific? → SSR
   - Is data request-specific? → SSR
   - Is data global/static? → SSG/ISR

2. **Performance impact**
   - Is SSR necessary, or would SSG suffice?
   - What's the cache hit potential?

### 3.3 Missing getStaticPaths for Dynamic Routes

**Pattern**: Dynamic routes without path generation

```tsx
// pages/posts/[id].tsx

// ❌ Missing getStaticPaths - will fail build
export async function getStaticProps({ params }) {
  const post = await getPost(params.id);
  return { props: { post } };
}

// ✅ With getStaticPaths
export async function getStaticPaths() {
  const posts = await getAllPosts();
  return {
    paths: posts.map(p => ({ params: { id: p.id } })),
    fallback: 'blocking',
  };
}

export async function getStaticProps({ params }) {
  const post = await getPost(params.id);
  return { props: { post } };
}
```

**Detection**: Check for dynamic route files (`[param].tsx`) with `getStaticProps` but no `getStaticPaths`.

### 3.4 Client-Side Fetch in getServerSideProps/getStaticProps

**Pattern**: Using browser fetch APIs in server functions

```tsx
// ❌ fetch to own API route is unnecessary round-trip
export async function getServerSideProps() {
  const res = await fetch('http://localhost:3000/api/users');
  const users = await res.json();
  return { props: { users } };
}

// ✅ Call the function directly
import { getUsers } from '@/lib/users';

export async function getServerSideProps() {
  const users = await getUsers();
  return { props: { users } };
}
```

**What analysis must determine**:

1. **Internal API detection**
   - Is fetch calling the same Next.js app's API routes?
   - Could the underlying function be called directly?

2. **Absolute URL requirement**
   - Does the code handle the requirement for absolute URLs in server context?

---

## Category 4: Rendering Strategy Anti-Patterns

### 4.1 Unintentional Dynamic Rendering (App Router)

**Pattern**: Accidentally forcing dynamic rendering

```tsx
// ❌ cookies() forces entire route to be dynamic
export default async function Page() {
  const theme = cookies().get('theme'); // Innocent looking...

  // Even though the rest could be static:
  const posts = await getStaticPosts();
  return <PostList posts={posts} theme={theme} />;
}

// ✅ Isolate dynamic parts
export default async function Page() {
  const posts = await getStaticPosts(); // Can be static
  return (
    <Suspense fallback={<PostListSkeleton />}>
      <DynamicThemeWrapper>
        {' '}
        {/* Client component reads cookie */}
        <PostList posts={posts} />
      </DynamicThemeWrapper>
    </Suspense>
  );
}
```

**Dynamic rendering triggers**:

```typescript
const DYNAMIC_TRIGGERS = {
  // Next.js functions
  functions: ['cookies', 'headers', 'searchParams', 'useSearchParams'],

  // Fetch options
  fetchOptions: [{ cache: 'no-store' }, { next: { revalidate: 0 } }],

  // Route segment config
  segmentConfig: {
    dynamic: ['force-dynamic', 'error'],
    revalidate: 0,
  },
};
```

**What analysis must determine**:

1. **Intentionality**
   - Does the route segment config indicate dynamic is intended?
   - Is there `export const dynamic = 'force-dynamic'`?

2. **Scope minimization**
   - Can dynamic parts be isolated to smaller components?
   - Would PPR (Partial Prerendering) help?

### 4.2 Static Route with Request-Time Data

**Pattern**: Expecting fresh data in statically rendered routes

```tsx
// Route segment config says static
export const dynamic = 'force-static';

export default async function Page() {
  // ❌ This data will be from build time only!
  const currentUser = await getCurrentUser();
  return <Dashboard user={currentUser} />;
}
```

**What analysis must determine**:

1. **Config consistency**
   - Does the code behavior match the route config?
   - Are there conflicting signals?

2. **Data freshness requirements**
   - Does the data need to be request-time fresh?
   - Is caching acceptable?

### 4.3 Heavy Client Components

**Pattern**: Components that should be server-rendered marked as client

```tsx
'use client';

// ❌ Large component tree unnecessarily in client bundle
export default function ProductCatalog({ products }) {
  return (
    <div>
      {products.map(p => (
        <ProductCard
          key={p.id}
          product={p}
          // Hundreds of nested components...
        />
      ))}
    </div>
  );
}

// ✅ Keep most as Server, isolate interactivity
export default function ProductCatalog({ products }) {
  return (
    <div>
      {products.map(p => (
        <ProductCard key={p.id} product={p}>
          <AddToCartButton productId={p.id} /> {/* Only this is client */}
        </ProductCard>
      ))}
    </div>
  );
}
```

**What analysis must determine**:

1. **Bundle size impact**
   - How large is the Client Component tree?
   - What percentage actually needs client features?

2. **Interactivity isolation**
   - Can interactive parts be extracted?
   - Would composition pattern help?

---

## Category 5: Route Configuration Anti-Patterns

### 5.1 Missing Loading States

**Pattern**: No loading.tsx for slow routes

```
app/
  dashboard/
    page.tsx          # Slow data fetching
    # ❌ No loading.tsx - user sees nothing during load
```

**What analysis must determine**:

1. **Page fetch analysis**
   - Does page.tsx do async data fetching?
   - Is the fetch potentially slow?

2. **Loading boundary presence**
   - Is there a loading.tsx in this or parent segment?
   - Are there Suspense boundaries in the page?

### 5.2 Missing Error Boundaries

**Pattern**: No error.tsx for routes that can fail

```tsx
// app/api-data/page.tsx
export default async function Page() {
  const data = await fetch('https://unreliable-api.com/data').then(r => r.json()); // ❌ Can throw, no error.tsx

  return <Display data={data} />;
}
```

**What analysis must determine**:

1. **Error possibility**
   - Does the page do external fetches?
   - Are there operations that can throw?

2. **Error boundary coverage**
   - Is there an error.tsx in the route hierarchy?
   - Is the boundary at the right granularity?

### 5.3 Incorrect Route Segment Config

**Pattern**: Conflicting or incorrect config exports

```tsx
// ❌ Conflicting options
export const dynamic = 'force-static';
export const revalidate = 0; // 0 = always revalidate = dynamic!

// ❌ Invalid values
export const dynamic = 'static'; // Wrong! Should be 'force-static'
export const revalidate = -1; // Invalid negative value
```

**Valid configurations**:

```typescript
const VALID_SEGMENT_CONFIG = {
  dynamic: ['auto', 'force-dynamic', 'error', 'force-static'],
  revalidate: 'false | number (>= 0)',
  fetchCache: ['auto', 'default-cache', 'only-cache', 'force-cache', 'force-no-store', 'default-no-store', 'only-no-store'],
  runtime: ['edge', 'nodejs'],
  preferredRegion: ['auto', 'global', 'home', string[]],
};
```

### 5.4 Middleware Overreach

**Pattern**: Middleware doing too much, slowing all routes

```tsx
// middleware.ts
export async function middleware(request: NextRequest) {
  // ❌ Runs on EVERY request including static assets
  const user = await validateSession(request);
  const permissions = await fetchUserPermissions(user.id);
  const settings = await fetchAppSettings();
  // ...
}

export const config = {
  // ❌ No matcher - runs on everything
};
```

**What analysis must determine**:

1. **Matcher configuration**
   - Is there a matcher to limit middleware scope?
   - Are static assets excluded?

2. **Performance impact**
   - How much work does middleware do?
   - Are there async operations that could be avoided?

**Recommended matcher pattern**:

```typescript
export const config = {
  matcher: [
    // Match all request paths except for:
    // - _next/static (static files)
    // - _next/image (image optimization)
    // - favicon.ico (favicon)
    // - public files (images, etc.)
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

---

## Category 6: Server Actions Anti-Patterns (App Router)

### 6.1 Server Actions Without Validation

**Pattern**: Trusting client input in Server Actions

```tsx
'use server';

// ❌ No validation - SQL injection, type errors possible
export async function updateUser(formData: FormData) {
  const name = formData.get('name');
  const email = formData.get('email');

  await db.users.update({
    where: { email },
    data: { name },
  });
}

// ✅ With validation
import { z } from 'zod';

const UpdateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

export async function updateUser(formData: FormData) {
  const result = UpdateUserSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });

  if (!result.success) {
    return { error: result.error.flatten() };
  }

  await db.users.update({
    where: { email: result.data.email },
    data: { name: result.data.name },
  });
}
```

**What analysis must determine**:

1. **Input validation presence**
   - Is FormData validated before use?
   - Is there schema validation (zod, yup, etc.)?

2. **Database query safety**
   - Are values directly interpolated into queries?
   - Is the ORM providing parameterization?

### 6.2 Server Actions Without Authentication

**Pattern**: Sensitive actions without auth checks

```tsx
'use server';

// ❌ No auth check - anyone can delete!
export async function deleteUser(userId: string) {
  await db.users.delete({ where: { id: userId } });
}

// ✅ With auth
import { auth } from '@/lib/auth';

export async function deleteUser(userId: string) {
  const session = await auth();

  if (!session?.user?.isAdmin) {
    throw new Error('Unauthorized');
  }

  await db.users.delete({ where: { id: userId } });
}
```

**What analysis must determine**:

1. **Auth check presence**
   - Is there authentication validation?
   - Is there authorization for the specific action?

2. **Sensitive operation detection**
   - Does the action modify data?
   - Does it access sensitive resources?

### 6.3 Server Actions Returning Sensitive Data

**Pattern**: Leaking internal data through action returns

```tsx
'use server';

// ❌ Returns entire user object including sensitive fields
export async function getUser(id: string) {
  return await db.users.findUnique({ where: { id } });
  // Returns: { id, email, passwordHash, ssn, ... }
}

// ✅ Select only needed fields
export async function getUser(id: string) {
  return await db.users.findUnique({
    where: { id },
    select: { id: true, name: true, email: true },
  });
}
```

**What analysis must determine**:

1. **Return value analysis**
   - What data is returned from the action?
   - Could sensitive fields be exposed?

2. **Field selection**
   - Does the query use `select` to limit fields?
   - Are sensitive fields explicitly excluded?

---

## Analysis Complexity Ratings

| Pattern                    | Router | Detection Complexity | Accuracy Without Types | Accuracy With Types |
| -------------------------- | ------ | -------------------- | ---------------------- | ------------------- |
| Missing "use client"       | App    | High                 | Low                    | Medium              |
| Unnecessary "use client"   | App    | Medium               | Medium                 | High                |
| Server code in client      | App    | High                 | Low                    | High                |
| Non-serializable props     | App    | Very High            | Very Low               | Medium              |
| Data fetching waterfalls   | Both   | High                 | Low                    | Medium              |
| Incorrect cache config     | App    | Medium               | Low                    | Medium              |
| getStaticProps misuse      | Pages  | Medium               | Medium                 | High                |
| Dynamic rendering triggers | App    | Medium               | Medium                 | High                |
| Missing loading/error      | App    | Low                  | High                   | High                |
| Server Action security     | App    | High                 | Low                    | Medium              |

## Implementation Priority

Based on impact and feasibility:

**High Priority (High impact, achievable)**:

1. Missing "use client" directive detection
2. Server-only imports in client components
3. Missing loading.tsx/error.tsx
4. Server Actions without validation

**Medium Priority (High impact, complex)**:

1. Data fetching waterfall detection
2. Non-serializable props across boundary
3. Unnecessary "use client" detection
4. Dynamic rendering trigger analysis

**Lower Priority (Needs runtime data)**:

1. Cache configuration optimization
2. Bundle size impact analysis
3. Performance-based SSG vs SSR recommendations
