# Phase 1: Foundation and Scaffolding

## Status

Not Started

## Goals

Create the basic package structure and plugin scaffold for the Next.js analyzer plugin.

## Files Created

### Package Configuration

- `analyzers/nextjs/package.json` - Package manifest with dependencies
- `analyzers/nextjs/tsconfig.json` - TypeScript configuration

### Source Files

- `analyzers/nextjs/src/plugin.ts` - NextJsAnalyzerPlugin class implementing ITechnologyPlugin
- `analyzers/nextjs/src/index.ts` - Public API exports
- `analyzers/nextjs/src/internal.ts` - Internal exports for advanced use

### Directory Structure

```
analyzers/nextjs/
├── src/
│   ├── analyses/      # Analysis implementations (Phases 4-9)
│   │   └── index.ts
│   ├── presenters/    # Report presenters (Phases 10-11)
│   │   └── index.ts
│   ├── types/         # Type definitions (Phase 2)
│   │   └── index.ts
│   ├── constants/     # Constants and weights (Phase 2)
│   │   └── index.ts
│   ├── utils/         # Utility functions (Phase 3)
│   │   └── index.ts
│   ├── testing/       # Test fixtures (Phase 13)
│   ├── plugin.ts      # Main plugin class
│   ├── plugin.test.ts # Plugin tests
│   ├── index.ts       # Public exports
│   └── internal.ts    # Internal exports
├── package.json
└── tsconfig.json
```

## Implementation Details

### Plugin Class

The `NextJsAnalyzerPlugin` class implements `ITechnologyPlugin` with:

- `id: 'nextjs'` - Plugin identifier
- `name: 'Next.js Analyzer'` - Display name
- `version: '1.0.0'` - Semantic version
- `filePatterns` - Glob patterns for Next.js files
- `canHandle()` - Detects Next.js files by checking for:
  - next.config.\* files
  - 'use client' or 'use server' directives
  - Imports from next/\* packages
  - Files in app/ or pages/ directories
  - Next.js-specific functions (getServerSideProps, getStaticProps, etc.)

### Detection Logic

The `canHandle()` method uses multiple strategies to identify Next.js files:

1. **Config files**: Matches `next.config.(js|ts|mjs|cjs)`
2. **Directives**: Detects `'use client'` or `'use server'` at the top of files
3. **Imports**: Checks for imports from `next` or `next/*` packages
4. **Directory patterns**: Looks for `app/` or `pages/` directories
5. **Next.js functions**: Identifies getServerSideProps, getStaticProps, generateMetadata, etc.

## Acceptance Criteria

All acceptance criteria have been met:

- [x] `pnpm build` succeeds
- [x] Plugin can be instantiated
- [x] `canHandle()` returns true for Next.js files, false for others
- [x] Unit tests for `canHandle()` pass (24/24 tests passing)

## Test Coverage

Created comprehensive tests for `canHandle()`:

- Next.js config files (js, ts, mjs)
- Files with 'use client' directive (single/double quotes)
- Files with 'use server' directive
- Files with next/\* imports (next/image, next/link, next/navigation)
- App Router files (generateMetadata, route handlers, metadata export)
- Pages Router files (getServerSideProps, getStaticProps, getStaticPaths)
- Negative cases (regular React files, plain TypeScript files)
- Edge cases (directives after comments, directives not at top)

## Next Steps

Proceed to Phase 2: Core Types and Constants
