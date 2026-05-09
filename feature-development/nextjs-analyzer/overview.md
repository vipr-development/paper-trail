# Next.js Code Analysis Issues and Anti-Patterns: Complete Reference Guide

A static/runtime code analyzer for Next.js must detect patterns spanning **eight critical categories** across both Pages Router and App Router paradigms. This comprehensive guide provides specific detection patterns, code examples of violations, and ESLint rule references for building robust analysis tooling.

## Server Components vs Client Components create the most common anti-patterns

The App Router's component model introduces unique analysis challenges. Components are Server Components by default, but improper "use client" placement creates **bundle bloat**, **hydration failures**, and **security vulnerabilities**.

### "use client" directive placement issues

The analyzer should flag these critical violations:

```javascript
// ❌ BAD: Directive not at top of file
import { useState } from 'react';
('use client'); // WRONG: Must be first line

// ❌ BAD: Both directives in same file
('use client');
('use server');

// ❌ BAD: "use server" inside client file (won't work)
('use client');
async function saveData() {
  'use server'; // Ineffective - file already client
  await db.save();
}
```

**Detection rule**: Flag `'use client'` or `'use server'` appearing after any import statement. Flag files containing both directives at module level.

### Unnecessary client components waste bundle size

```javascript
// ❌ BAD: Static component unnecessarily marked client
'use client';
export function StaticCard({ title }) {
  return <h2>{title}</h2>; // No hooks, no interactivity
}

// ❌ BAD: Client hooks used in Server Component (runtime error)
// app/page.tsx (Server Component by default)
import { useState } from 'react';
export default function Page() {
  const [count, setCount] = useState(0); // Will crash
}
```

**Detection rule**: Flag `'use client'` files that don't use: `useState`, `useEffect`, `useContext`, `useReducer`, `useCallback`, `useMemo`, `useTransition`, event handlers (`onClick`, `onChange`), or browser APIs. Conversely, flag files **without** `'use client'` that import/use these hooks.

### Non-serializable props crossing boundaries

Server Components can only pass serializable data to Client Components:

```javascript
// ❌ BAD: Passing function from Server to Client Component
export default function Page() {
  const handleClick = () => console.log('clicked');
  return <ClientButton onClick={handleClick} />; // Can't serialize!
}

// ✅ GOOD: Use Server Actions
import { handleAction } from './actions';
export default function Page() {
  return <ClientButton action={handleAction} />;
}
```

**Detection rule**: Flag Server Components passing inline functions, class instances, `Date`, `Map`, `Set`, `Symbol`, or `RegExp` to components imported from `'use client'` files.

---

## Data fetching anti-patterns differ by router paradigm

### Pages Router violations

| Pattern                                   | Issue                                | Detection                                                                     |
| ----------------------------------------- | ------------------------------------ | ----------------------------------------------------------------------------- |
| `getInitialProps` usage                   | Blocks Automatic Static Optimization | Flag any `Page.getInitialProps` or `getInitialProps` function                 |
| `getStaticProps` without `getStaticPaths` | Missing paths for dynamic routes     | Flag dynamic routes (`[param]`) with `getStaticProps` but no `getStaticPaths` |
| Client-side fetch for SSR data            | Unnecessary loading states           | Flag `useEffect` + `fetch` in pages without `getServerSideProps`              |

```javascript
// ❌ BAD: Dynamic route missing getStaticPaths
// pages/posts/[id].tsx
export async function getStaticProps({ params }) {
  const post = await getPost(params.id);
  return { props: { post } };
}
// Missing getStaticPaths! → Build will fail

// ❌ BAD: getStaticProps as component property
Page.getStaticProps = () => ({ props: {} }); // Wrong export method
```

### App Router violations

```javascript
// ❌ BAD: Fetching own API route from Server Component
export default async function Page() {
  const res = await fetch('http://localhost:3000/api/data'); // Wasteful!
  return <div>{res.data}</div>;
}

// ❌ BAD: fetch() without cache options (unpredictable in Next.js 15+)
const data = await fetch('https://api.example.com/data'); // No cache specified

// ❌ BAD: No revalidation after mutation
'use server';
export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: formData.get('title') } });
  // Missing: revalidatePath('/posts') or revalidateTag('posts')
}
```

**Detection rules**:

- Flag `fetch('http://localhost')` or relative API routes in Server Components
- Warn on `fetch()` calls without `cache` or `next.revalidate` options
- Flag Server Actions with database mutations without `revalidatePath`/`revalidateTag`

---

## Next.js component anti-patterns require specific detection

### next/image issues

```javascript
// ❌ BAD: Missing dimensions (unless using fill)
<Image src="/hero.jpg" alt="Hero" /> // Needs width/height

// ❌ BAD: Deprecated layout prop (Next.js 13+)
<Image src="/photo.jpg" layout="fill" objectFit="cover" />

// ✅ GOOD: Modern API
<Image src="/photo.jpg" fill style={{ objectFit: 'cover' }} />

// ❌ BAD: fill without positioned parent
<div>
  <Image src="/bg.jpg" fill /> {/* Won't display correctly */}
</div>

// ❌ BAD: Hero image without priority (delays LCP)
<Image src="/hero.jpg" width={1200} height={600} alt="Hero" />
```

**ESLint rules**: `@next/next/no-img-element` enforces using `next/image` instead of `<img>`.

### next/link issues

```javascript
// ❌ BAD: Unnecessary anchor in Next.js 13+
<Link href="/about">
  <a>About</a> {/* Redundant since Next.js 13 */}
</Link>

// ❌ BAD: legacyBehavior still being used
<Link href="/about" legacyBehavior>
  <a>About</a>
</Link>

// ❌ BAD: Plain anchor for internal navigation
<a href="/about">About</a> // Loses client-side navigation
```

**ESLint rule**: `@next/next/no-html-link-for-pages` flags `<a>` tags for internal routes.

### next/script issues

```javascript
// ❌ BAD: Script inside next/head
import Head from 'next/head';
<Head>
  <script src="https://analytics.com/script.js" />
</Head>

// ❌ BAD: Inline script without id
<Script>{`console.log('Hello')`}</Script>

// ❌ BAD: beforeInteractive outside _document (Pages Router)
// pages/index.tsx
<Script src="/critical.js" strategy="beforeInteractive" />
```

**ESLint rules**: `@next/next/no-script-component-in-head`, `@next/next/inline-script-id`, `@next/next/no-before-interactive-script-outside-document`

---

## Migration patterns require version-specific detection

### Next.js 14 → 15 breaking changes (highest priority)

The **async Request APIs** change is the most impactful:

```javascript
// ❌ BAD: Synchronous access (broken in Next.js 15+)
import { cookies, headers, draftMode } from 'next/headers';
const cookieStore = cookies();
const token = cookieStore.get('token');

// ✅ GOOD: Async access required
const cookieStore = await cookies();
const token = cookieStore.get('token');

// ❌ BAD: Synchronous params/searchParams access
export default function Page({ params, searchParams }) {
  const { slug } = params; // Returns Promise now!
}

// ✅ GOOD: Await params
export default async function Page({ params, searchParams }) {
  const { slug } = await params;
  const { query } = await searchParams;
}
```

**Additional Next.js 15 changes to detect**:

- `useFormState` → `useActionState` (React 19)
- `request.geo` and `request.ip` removed from middleware
- `experimental.bundlePagesExternals` → `bundlePagesRouterDependencies`
- GET Route Handlers no longer cached by default
- `fetch()` no longer cached by default

### Next.js 12 → 13 migrations

```javascript
// ❌ FLAG: Old image import
import Image from 'next/legacy/image';

// ❌ FLAG: Removed image props
<Image layout="fill" objectFit="cover" />

// ❌ FLAG: Nested anchor in Link
<Link href="/about"><a>About</a></Link>

// ❌ FLAG: Deprecated font package
import { Inter } from '@next/font/google'; // Use next/font/google
```

### Router import misuse

```javascript
// ❌ FLAG: Using next/router in App Router (app/ directory)
import { useRouter } from 'next/router'; // Wrong!

// ✅ GOOD: Use next/navigation in App Router
import { useRouter, usePathname, useSearchParams } from 'next/navigation';
```

---

## Performance issues span bundle size, rendering, and caching

### Bundle size bloat patterns

```javascript
// ❌ BAD: Importing entire lodash (adds ~70KB)
import _ from 'lodash';
const result = _.get(obj, 'path');

// ✅ GOOD: Tree-shakeable import (~2KB)
import get from 'lodash/get';

// ❌ BAD: Barrel file imports
import { Button } from '@/components'; // Imports entire index.ts

// ✅ GOOD: Direct import
import { Button } from '@/components/Button';
```

**Detection**: Flag `import _ from 'lodash'`, `import * as` patterns, and imports from folder paths (implying index.ts barrel files).

### Hydration mismatch causes

```javascript
// ❌ BAD: Different values server vs client
function Component() {
  return <p>Time: {new Date().toLocaleTimeString()}</p>;
}

// ❌ BAD: Browser APIs during SSR
function Component() {
  const width = window.innerWidth; // Crashes on server
  const stored = localStorage.getItem('key');
}

// ❌ BAD: Conditional rendering on client state
function Component() {
  return typeof window !== 'undefined' ? <DesktopNav /> : <MobileNav />; // Mismatch!
}
```

### Caching strategy issues

```javascript
// ❌ BAD: SSR when SSG would work (Pages Router)
export async function getServerSideProps() {
  const posts = await getBlogPosts(); // Same for all users!
  return { props: { posts } };
}

// ✅ GOOD: Use ISR
export async function getStaticProps() {
  const posts = await getBlogPosts();
  return { props: { posts }, revalidate: 3600 };
}

// ❌ BAD: force-dynamic when not needed (App Router)
export const dynamic = 'force-dynamic';
export default function About() {
  return <div>Static content...</div>; // Doesn't need dynamic
}

// ❌ BAD: Database queries without caching
async function getUsers() {
  return await prisma.user.findMany(); // Runs on every request
}

// ✅ GOOD: Use unstable_cache
import { unstable_cache } from 'next/cache';
const getUsers = unstable_cache(async () => prisma.user.findMany(), ['users'], {
  revalidate: 3600,
  tags: ['users'],
});
```

---

## Security vulnerabilities specific to Next.js

### CVE-2025-29927: Critical middleware bypass

Versions **11.1.4 through 15.2.2** are vulnerable to complete authentication bypass via the `x-middleware-subrequest` header. Self-hosted deployments must upgrade to patched versions (**15.2.3+**, **14.2.25+**, **13.5.9+**, **12.3.5+**).

### Server Actions security

```javascript
// ❌ BAD: No authentication in Server Action
'use server';
export async function deleteUser(userId: string) {
  await db.user.delete({ where: { id: userId } }); // Anyone can call!
}

// ✅ GOOD: Always verify authentication
'use server';
import { auth } from '@/lib/auth';
export async function deleteUser(userId: string) {
  const session = await auth();
  if (!session?.user?.isAdmin) throw new Error('Unauthorized');
  await db.user.delete({ where: { id: userId } });
}

// ❌ BAD: Non-async Server Action
'use server';
export function saveData(formData: FormData) { // Must be async
  return { success: true };
}
```

**Detection rules**:

- Flag `'use server'` files without auth library imports
- Flag Server Actions without async keyword
- Flag Server Actions without input validation (e.g., Zod, Valibot imports)

### Environment variable exposure

```javascript
// ❌ BAD: NEXT_PUBLIC_ for sensitive data
NEXT_PUBLIC_DATABASE_URL=postgres://... // Exposed to browser!
NEXT_PUBLIC_API_SECRET=sk_live_...

// ❌ BAD: Dynamic access to NEXT_PUBLIC_ (won't work)
const varName = 'NEXT_PUBLIC_ANALYTICS_ID';
const value = process.env[varName]; // Returns undefined!

// ❌ BAD: Non-public env in client component
'use client';
export function Client() {
  const secret = process.env.API_SECRET; // Always undefined!
}
```

### Middleware security

```javascript
// ❌ BAD: Node.js modules in middleware (Edge Runtime)
// middleware.ts
import fs from 'fs'; // Not available!
import { PrismaClient } from '@prisma/client'; // Requires Node

// ❌ BAD: No matcher (runs on everything including static assets)
export function middleware(request) {}
// Missing: export const config = { matcher: [...] }

// ❌ BAD: Heavy operations in middleware
export async function middleware(request) {
  const result = await expensiveDatabaseQuery(); // Runs on EVERY request
}
```

---

## Configuration issues in next.config.js

### Deprecated and removed options

```javascript
// ❌ REMOVED in Next.js 15+
module.exports = {
  swcMinify: true, // Now always true
  target: 'serverless', // Removed
}

// ❌ DEPRECATED (Next.js 14+)
module.exports = {
  images: {
    domains: ['example.com'], // Use remotePatterns
  },
  experimental: {
    appDir: true, // No longer needed
  }
}

// ❌ REMOVED in Next.js 16
module.exports = {
  amp: { ... }, // AMP support removed
  eslint: { dirs: [...] }, // Use ESLint CLI
  serverRuntimeConfig: { ... }, // Use env vars
  publicRuntimeConfig: { ... }, // Use NEXT_PUBLIC_
}

// ✅ GOOD: Modern configuration
/** @type {import('next').NextConfig} */
module.exports = {
  images: {
    remotePatterns: [{ hostname: 'example.com' }]
  },
  serverExternalPackages: ['@prisma/client'],
}
```

### Route configuration issues

```javascript
// ❌ BAD: Invalid matcher pattern
export const config = {
  matcher: 'about', // Must start with /
};

// ❌ BAD: Invalid HTTP method exports
export async function get() {} // Should be GET (uppercase)
export async function Post() {} // Should be POST

// ❌ BAD: generateStaticParams wrong return type
export async function generateStaticParams() {
  return ['post-1', 'post-2']; // Should return array of objects!
}

// ✅ GOOD
export async function generateStaticParams() {
  return [{ slug: 'post-1' }, { slug: 'post-2' }];
}
```

---

## Accessibility issues specific to Next.js

### Missing lang attribute

```javascript
// ❌ BAD: Custom _document without lang
// pages/_document.tsx
export default function Document() {
  return (
    <Html> {/* Missing lang="en" */}
      <Head />
      <body><Main /><NextScript /></body>
    </Html>
  );
}

// ✅ GOOD: Include lang attribute
<Html lang="en">

// Alternative: Set in next.config.js
module.exports = {
  i18n: { locales: ['en'], defaultLocale: 'en' }
}
```

### next/image alt text

The `alt` prop is technically optional but critical for accessibility:

```javascript
// ❌ BAD: Missing alt (screen readers get no context)
<Image src="/photo.jpg" width={500} height={300} />

// ❌ BAD: Redundant alt text
<Image src="/photo.jpg" alt="image of a photo" /> // Avoid "image", "picture", "photo"

// ✅ GOOD: Descriptive alt
<Image src="/team-photo.jpg" alt="Engineering team at annual retreat" width={800} height={600} />

// ✅ GOOD: Decorative image with empty alt
<Image src="/decorative-border.png" alt="" width={100} height={10} />
```

**ESLint rule**: `jsx-a11y/alt-text` flags missing or incorrect alt attributes.

---

## TypeScript patterns for proper typing

### Pages Router typing

```typescript
// ❌ BAD: Untyped page
export const getServerSideProps = async (context) => { // 'any' type
  return { props: { data: 'hello' } }
}

// ✅ GOOD: Fully typed
import type { GetServerSideProps, InferGetServerSidePropsType } from 'next';

interface Props { repo: { name: string; stars: number } }

export const getServerSideProps: GetServerSideProps<Props> = async () => {
  const repo = await fetch('...').then(r => r.json());
  return { props: { repo } };
}

export default function Page({
  repo
}: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <p>{repo.stars}</p>;
}
```

### App Router typing (Next.js 15+)

```typescript
// ❌ BAD: Synchronous params type (broken in Next.js 15+)
export default function Page({ params }: { params: { slug: string } }) {
  return <div>{params.slug}</div>; // Type error!
}

// ✅ GOOD: Async params type
type Props = {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}

export default async function Page({ params, searchParams }: Props) {
  const { slug } = await params;
  return <div>{slug}</div>;
}

// Route handlers
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
): Promise<Response> {
  const { id } = await params;
  return NextResponse.json({ id });
}

// Server Actions
'use server';
export async function createUser(
  prevState: { error?: string } | null,
  formData: FormData
): Promise<{ success: boolean; error?: string }> {
  // Validate with Zod
  return { success: true };
}
```

---

## ESLint plugin rules reference

The `@next/eslint-plugin-next` provides these critical rules:

| Rule                                            | Severity | Category         |
| ----------------------------------------------- | -------- | ---------------- |
| `no-img-element`                                | warn     | Performance/A11y |
| `no-html-link-for-pages`                        | error    | Performance      |
| `no-script-component-in-head`                   | error    | Correctness      |
| `inline-script-id`                              | error    | Correctness      |
| `no-before-interactive-script-outside-document` | error    | Correctness      |
| `no-async-client-component`                     | warn     | Correctness      |
| `no-document-import-in-page`                    | error    | Correctness      |
| `no-head-import-in-document`                    | error    | Correctness      |
| `no-duplicate-head`                             | error    | Correctness      |
| `no-typos`                                      | warn     | Correctness      |
| `google-font-display`                           | warn     | Performance      |
| `google-font-preconnect`                        | warn     | Performance      |
| `no-sync-scripts`                               | error    | Performance      |
| `no-page-custom-font`                           | warn     | Performance      |

Next.js also includes `eslint-plugin-jsx-a11y` by default for accessibility rules including `alt-text`, `html-has-lang`, `aria-*` validation, and interactive element focus management.

---

## Conclusion

A comprehensive Next.js analyzer must detect issues across **migrations** (version-specific breaking changes), **anti-patterns** (Server/Client Component misuse, directive errors), **performance** (bundle size, caching, hydration), **security** (Server Actions, env vars, middleware bypass), **accessibility** (lang, alt, focus), **configuration** (deprecated options, invalid values), and **TypeScript** (async params, proper generics). Priority should be given to **breaking changes** (runtime errors) and **security vulnerabilities** (CVE-2025-29927, exposed secrets), followed by performance anti-patterns that measurably impact Core Web Vitals.
