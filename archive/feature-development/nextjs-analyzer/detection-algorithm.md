# Detecting Next.js Code

```pseudo
// =============================================================================
// NEXT.JS DETECTION STRATEGY
// =============================================================================
//
// Three-layer approach with increasing confidence and cost:
//   Layer 1: Quick Heuristics (fast, may have false positives)
//   Layer 2: Structural Analysis (medium cost, high confidence)
//   Layer 3: Deep Content Analysis (expensive, definitive)
//
// =============================================================================

// -----------------------------------------------------------------------------
// TYPES & CONSTANTS
// -----------------------------------------------------------------------------

enum NextRouterType {
  PAGES_ROUTER,
  APP_ROUTER,
  HYBRID,        // Both routers present
  UNKNOWN
}

enum NextFileType {
  PAGE,
  LAYOUT,
  LOADING,
  ERROR,
  NOT_FOUND,
  TEMPLATE,
  DEFAULT,       // Parallel route default
  ROUTE_HANDLER, // API route (app router)
  API_ROUTE,     // API route (pages router)
  MIDDLEWARE,
  CONFIG,
  CUSTOM_APP,    // _app.tsx
  CUSTOM_DOCUMENT, // _document.tsx
  METADATA_FILE, // opengraph-image, icon, sitemap, robots, manifest
  INSTRUMENTATION,
  REGULAR_COMPONENT, // Not a special Next.js file
  NOT_NEXTJS
}

struct DetectionResult {
  isNextJs: boolean
  confidence: float         // 0.0 - 1.0
  routerType: NextRouterType
  fileType: NextFileType
  reasons: string[]
}

const NEXT_CONFIG_FILES = [
  "next.config.js",
  "next.config.mjs",
  "next.config.ts"
]

const APP_ROUTER_SPECIAL_FILES = {
  "page":           NextFileType.PAGE,
  "layout":         NextFileType.LAYOUT,
  "loading":        NextFileType.LOADING,
  "error":          NextFileType.ERROR,
  "not-found":      NextFileType.NOT_FOUND,
  "template":       NextFileType.TEMPLATE,
  "default":        NextFileType.DEFAULT,
  "route":          NextFileType.ROUTE_HANDLER,
  "opengraph-image": NextFileType.METADATA_FILE,
  "twitter-image":  NextFileType.METADATA_FILE,
  "icon":           NextFileType.METADATA_FILE,
  "apple-icon":     NextFileType.METADATA_FILE,
  "sitemap":        NextFileType.METADATA_FILE,
  "robots":         NextFileType.METADATA_FILE,
  "manifest":       NextFileType.METADATA_FILE
}

const PAGES_ROUTER_SPECIAL_FILES = {
  "_app":      NextFileType.CUSTOM_APP,
  "_document": NextFileType.CUSTOM_DOCUMENT,
  "_error":    NextFileType.ERROR,
  "404":       NextFileType.NOT_FOUND,
  "500":       NextFileType.ERROR
}

const VALID_EXTENSIONS = [".tsx", ".ts", ".jsx", ".js"]


// =============================================================================
// LAYER 1: QUICK HEURISTICS (Cost: O(1) - O(n) path operations)
// =============================================================================

function quickDetectFolder(folderPath: Path) -> DetectionResult {
  reasons = []
  confidence = 0.0

  // Check 1: Config file exists (strongest quick signal)
  for configFile in NEXT_CONFIG_FILES {
    if exists(folderPath / configFile) {
      confidence = 0.85
      reasons.push("Found " + configFile)
      break
    }
  }

  // Check 2: .next build directory exists
  if exists(folderPath / ".next") {
    confidence = max(confidence, 0.80)
    reasons.push("Found .next build directory")
  }

  // Check 3: Standard directory structure
  hasAppDir = exists(folderPath / "app")
  hasPagesDir = exists(folderPath / "pages")
  hasSrcApp = exists(folderPath / "src" / "app")
  hasSrcPages = exists(folderPath / "src" / "pages")

  routerType = determineRouterType(hasAppDir, hasPagesDir, hasSrcApp, hasSrcPages)

  if hasAppDir or hasSrcApp {
    confidence = max(confidence, 0.60)
    reasons.push("Found app/ directory")
  }

  if hasPagesDir or hasSrcPages {
    confidence = max(confidence, 0.55)
    reasons.push("Found pages/ directory")
  }

  // Check 4: Middleware at root
  if exists(folderPath / "middleware.ts") or exists(folderPath / "middleware.js") {
    confidence = max(confidence, 0.70)
    reasons.push("Found middleware file at root")
  }

  return DetectionResult {
    isNextJs: confidence > 0.5,
    confidence: confidence,
    routerType: routerType,
    fileType: NextFileType.NOT_NEXTJS,
    reasons: reasons
  }
}


function quickDetectFile(filePath: Path) -> DetectionResult {
  fileName = getFileName(filePath)          // "page.tsx"
  baseName = getBaseName(filePath)          // "page"
  extension = getExtension(filePath)        // ".tsx"
  pathSegments = getPathSegments(filePath)  // ["src", "app", "dashboard", "page.tsx"]

  // Early exit: Not a JS/TS file
  if extension not in VALID_EXTENSIONS {
    return DetectionResult { isNextJs: false, confidence: 1.0, ... }
  }

  // Check 1: Is this a config file?
  if fileName in NEXT_CONFIG_FILES {
    return DetectionResult {
      isNextJs: true,
      confidence: 0.95,
      routerType: NextRouterType.UNKNOWN,
      fileType: NextFileType.CONFIG,
      reasons: ["Next.js config file"]
    }
  }

  // Check 2: Middleware or instrumentation at expected location
  if baseName == "middleware" and isAtProjectRoot(filePath) {
    return DetectionResult {
      isNextJs: true,
      confidence: 0.90,
      fileType: NextFileType.MIDDLEWARE,
      ...
    }
  }

  if baseName == "instrumentation" and isAtProjectRoot(filePath) {
    return DetectionResult {
      isNextJs: true,
      confidence: 0.85,
      fileType: NextFileType.INSTRUMENTATION,
      ...
    }
  }

  // Check 3: App Router patterns
  if pathContains(pathSegments, "app") {
    appIndex = indexOf(pathSegments, "app")

    // Validate it's app/ or src/app/ (not node_modules/some-app/)
    if isValidAppDirectory(pathSegments, appIndex) {
      if baseName in APP_ROUTER_SPECIAL_FILES {
        return DetectionResult {
          isNextJs: true,
          confidence: 0.90,
          routerType: NextRouterType.APP_ROUTER,
          fileType: APP_ROUTER_SPECIAL_FILES[baseName],
          reasons: ["App Router special file: " + baseName]
        }
      }

      // Regular component inside app/ - might be a component, might not
      return DetectionResult {
        isNextJs: true,
        confidence: 0.50,
        routerType: NextRouterType.APP_ROUTER,
        fileType: NextFileType.REGULAR_COMPONENT,
        reasons: ["File inside app/ directory"]
      }
    }
  }

  // Check 4: Pages Router patterns
  if pathContains(pathSegments, "pages") {
    pagesIndex = indexOf(pathSegments, "pages")

    if isValidPagesDirectory(pathSegments, pagesIndex) {
      // Check for special underscore files
      if baseName in PAGES_ROUTER_SPECIAL_FILES {
        return DetectionResult {
          isNextJs: true,
          confidence: 0.90,
          routerType: NextRouterType.PAGES_ROUTER,
          fileType: PAGES_ROUTER_SPECIAL_FILES[baseName],
          reasons: ["Pages Router special file: " + baseName]
        }
      }

      // Check for API routes
      if pathContains(pathSegments, "api") and indexOf(pathSegments, "api") > pagesIndex {
        return DetectionResult {
          isNextJs: true,
          confidence: 0.85,
          routerType: NextRouterType.PAGES_ROUTER,
          fileType: NextFileType.API_ROUTE,
          reasons: ["Pages Router API route"]
        }
      }

      // Any other file in pages/ is likely a page
      return DetectionResult {
        isNextJs: true,
        confidence: 0.75,
        routerType: NextRouterType.PAGES_ROUTER,
        fileType: NextFileType.PAGE,
        reasons: ["File in pages/ directory"]
      }
    }
  }

  // No strong signals
  return DetectionResult {
    isNextJs: false,
    confidence: 0.30,
    fileType: NextFileType.NOT_NEXTJS,
    reasons: ["No Next.js patterns detected in path"]
  }
}


// =============================================================================
// LAYER 2: STRUCTURAL ANALYSIS (Cost: O(n) file system + JSON parsing)
// =============================================================================

function structuralDetectFolder(folderPath: Path) -> DetectionResult {
  quickResult = quickDetectFolder(folderPath)
  reasons = quickResult.reasons.copy()
  confidence = quickResult.confidence

  // Check 5: package.json analysis
  packageJsonPath = folderPath / "package.json"
  if exists(packageJsonPath) {
    packageJson = parseJSON(readFile(packageJsonPath))

    // Check dependencies
    allDeps = merge(
      packageJson.dependencies ?? {},
      packageJson.devDependencies ?? {}
    )

    if "next" in allDeps {
      confidence = max(confidence, 0.95)
      reasons.push("'next' found in package.json dependencies")

      // Extract version for additional context
      nextVersion = allDeps["next"]
      reasons.push("Next.js version: " + nextVersion)
    }

    // Check scripts for next commands
    scripts = packageJson.scripts ?? {}
    for scriptName, scriptValue in scripts {
      if contains(scriptValue, "next ") or contains(scriptValue, "next-") {
        confidence = max(confidence, 0.90)
        reasons.push("Found 'next' in scripts." + scriptName)
        break
      }
    }
  }

  // Check 6: Validate directory structure depth
  if quickResult.routerType in [NextRouterType.APP_ROUTER, NextRouterType.HYBRID] {
    appDir = findAppDirectory(folderPath)
    if appDir {
      hasRootLayout = exists(appDir / "layout.tsx") or exists(appDir / "layout.js")
      hasAnyPage = findFilesRecursive(appDir, "page.*").length > 0

      if hasRootLayout {
        confidence = max(confidence, 0.92)
        reasons.push("Found root layout in app/")
      }

      if hasAnyPage {
        confidence = max(confidence, 0.88)
        reasons.push("Found page files in app/")
      }
    }
  }

  if quickResult.routerType in [NextRouterType.PAGES_ROUTER, NextRouterType.HYBRID] {
    pagesDir = findPagesDirectory(folderPath)
    if pagesDir {
      hasIndexPage = exists(pagesDir / "index.tsx") or
                     exists(pagesDir / "index.jsx") or
                     exists(pagesDir / "index.js")

      if hasIndexPage {
        confidence = max(confidence, 0.88)
        reasons.push("Found index page in pages/")
      }
    }
  }

  return DetectionResult {
    isNextJs: confidence > 0.5,
    confidence: confidence,
    routerType: quickResult.routerType,
    fileType: NextFileType.NOT_NEXTJS,
    reasons: reasons
  }
}


function structuralDetectFile(filePath: Path) -> DetectionResult {
  quickResult = quickDetectFile(filePath)

  // If quick detection is confident, return early
  if quickResult.confidence >= 0.85 {
    return quickResult
  }

  reasons = quickResult.reasons.copy()
  confidence = quickResult.confidence

  // Walk up to find project root
  projectRoot = findProjectRoot(filePath)

  if projectRoot == null {
    return quickResult // Can't determine more without project context
  }

  // Verify this is actually a Next.js project
  folderResult = structuralDetectFolder(projectRoot)

  if not folderResult.isNextJs {
    return DetectionResult {
      isNextJs: false,
      confidence: 0.90,
      fileType: NextFileType.NOT_NEXTJS,
      reasons: ["Parent project is not a Next.js application"]
    }
  }

  // Adjust confidence based on project context
  if quickResult.isNextJs {
    confidence = min(1.0, confidence + 0.15)
    reasons.push("Confirmed: parent is Next.js project")
  }

  // Refine router type if we determined it at folder level
  routerType = quickResult.routerType
  if routerType == NextRouterType.UNKNOWN {
    routerType = folderResult.routerType
  }

  return DetectionResult {
    isNextJs: quickResult.isNextJs,
    confidence: confidence,
    routerType: routerType,
    fileType: quickResult.fileType,
    reasons: reasons
  }
}


// =============================================================================
// LAYER 3: DEEP CONTENT ANALYSIS (Cost: O(n) file reading + AST parsing)
// =============================================================================

function deepDetectFile(filePath: Path) -> DetectionResult {
  structuralResult = structuralDetectFile(filePath)

  // If already highly confident, skip expensive parsing
  if structuralResult.confidence >= 0.95 {
    return structuralResult
  }

  reasons = structuralResult.reasons.copy()
  confidence = structuralResult.confidence
  fileType = structuralResult.fileType

  // Read and parse file content
  content = readFile(filePath)
  ast = parseTypeScriptOrJSX(content)

  // Analysis 1: Check for Next.js specific imports
  imports = extractImports(ast)

  nextImports = {
    "next/link":           { boost: 0.10, reason: "Uses next/link" },
    "next/image":          { boost: 0.10, reason: "Uses next/image" },
    "next/router":         { boost: 0.15, reason: "Uses next/router (pages)" },
    "next/navigation":     { boost: 0.15, reason: "Uses next/navigation (app)" },
    "next/head":           { boost: 0.12, reason: "Uses next/head" },
    "next/script":         { boost: 0.10, reason: "Uses next/script" },
    "next/font":           { boost: 0.12, reason: "Uses next/font" },
    "next/font/google":    { boost: 0.12, reason: "Uses next/font/google" },
    "next/font/local":     { boost: 0.12, reason: "Uses next/font/local" },
    "next/dynamic":        { boost: 0.10, reason: "Uses next/dynamic" },
    "next/headers":        { boost: 0.15, reason: "Uses next/headers (app router)" },
    "next/cookies":        { boost: 0.15, reason: "Uses next/cookies (app router)" },
    "next/server":         { boost: 0.20, reason: "Uses next/server" },
    "next/og":             { boost: 0.15, reason: "Uses next/og (image generation)" },
    "@next/third-parties": { boost: 0.10, reason: "Uses @next/third-parties" }
  }

  for importPath in imports {
    if importPath in nextImports {
      confidence = min(1.0, confidence + nextImports[importPath].boost)
      reasons.push(nextImports[importPath].reason)
    }

    // Partial match for next/* pattern
    if startsWith(importPath, "next/") and importPath not in nextImports {
      confidence = min(1.0, confidence + 0.08)
      reasons.push("Uses " + importPath)
    }
  }

  // Analysis 2: Check for Next.js specific exports
  exports = extractExports(ast)

  // App Router: Check for metadata exports
  if structuralResult.routerType == NextRouterType.APP_ROUTER {
    appRouterExports = {
      "metadata":         { type: NextFileType.PAGE, reason: "Exports metadata object" },
      "generateMetadata": { type: NextFileType.PAGE, reason: "Exports generateMetadata function" },
      "generateStaticParams": { type: NextFileType.PAGE, reason: "Exports generateStaticParams" },
      "revalidate":       { type: NextFileType.PAGE, reason: "Exports revalidate config" },
      "dynamic":          { type: NextFileType.PAGE, reason: "Exports dynamic config" },
      "dynamicParams":    { type: NextFileType.PAGE, reason: "Exports dynamicParams config" },
      "fetchCache":       { type: NextFileType.PAGE, reason: "Exports fetchCache config" },
      "runtime":          { type: NextFileType.PAGE, reason: "Exports runtime config ('edge' | 'nodejs')" },
      "preferredRegion":  { type: NextFileType.PAGE, reason: "Exports preferredRegion config" },
      "GET":              { type: NextFileType.ROUTE_HANDLER, reason: "Exports GET handler" },
      "POST":             { type: NextFileType.ROUTE_HANDLER, reason: "Exports POST handler" },
      "PUT":              { type: NextFileType.ROUTE_HANDLER, reason: "Exports PUT handler" },
      "PATCH":            { type: NextFileType.ROUTE_HANDLER, reason: "Exports PATCH handler" },
      "DELETE":           { type: NextFileType.ROUTE_HANDLER, reason: "Exports DELETE handler" },
      "HEAD":             { type: NextFileType.ROUTE_HANDLER, reason: "Exports HEAD handler" },
      "OPTIONS":          { type: NextFileType.ROUTE_HANDLER, reason: "Exports OPTIONS handler" }
    }

    for exportName in exports {
      if exportName in appRouterExports {
        confidence = min(1.0, confidence + 0.20)
        fileType = appRouterExports[exportName].type
        reasons.push(appRouterExports[exportName].reason)
      }
    }
  }

  // Pages Router: Check for data fetching exports
  if structuralResult.routerType == NextRouterType.PAGES_ROUTER {
    pagesRouterExports = {
      "getStaticProps":    { boost: 0.25, reason: "Exports getStaticProps" },
      "getStaticPaths":    { boost: 0.25, reason: "Exports getStaticPaths" },
      "getServerSideProps": { boost: 0.25, reason: "Exports getServerSideProps" },
      "getInitialProps":   { boost: 0.20, reason: "Exports getInitialProps" }
    }

    for exportName in exports {
      if exportName in pagesRouterExports {
        confidence = min(1.0, confidence + pagesRouterExports[exportName].boost)
        fileType = NextFileType.PAGE
        reasons.push(pagesRouterExports[exportName].reason)
      }
    }
  }

  // Analysis 3: Check for 'use client' / 'use server' directives
  directives = extractDirectives(ast)

  if "use client" in directives {
    confidence = min(1.0, confidence + 0.15)
    reasons.push("Has 'use client' directive (App Router)")
    // Refine: this is likely App Router
    if structuralResult.routerType == NextRouterType.UNKNOWN {
      structuralResult.routerType = NextRouterType.APP_ROUTER
    }
  }

  if "use server" in directives {
    confidence = min(1.0, confidence + 0.15)
    reasons.push("Has 'use server' directive (Server Actions)")
    if structuralResult.routerType == NextRouterType.UNKNOWN {
      structuralResult.routerType = NextRouterType.APP_ROUTER
    }
  }

  // Analysis 4: Check for Next.js config patterns
  if fileType == NextFileType.CONFIG {
    // Verify it's actually Next.js config by checking content
    hasNextConfig = ast.containsCallExpression("defineConfig") or
                    ast.containsIdentifier("nextConfig") or
                    ast.exportsObject() or
                    ast.exportsFunction()

    if hasNextConfig {
      confidence = 1.0
      reasons.push("Valid Next.js configuration structure")
    }
  }

  // Analysis 5: Check for middleware patterns
  if fileType == NextFileType.MIDDLEWARE {
    hasMiddlewareExport = "middleware" in exports
    hasConfigExport = "config" in exports
    hasNextRequest = importsFrom(imports, "next/server", ["NextRequest", "NextResponse"])

    if hasMiddlewareExport and hasNextRequest {
      confidence = 1.0
      reasons.push("Valid Next.js middleware structure")
    }
  }

  return DetectionResult {
    isNextJs: confidence > 0.5,
    confidence: confidence,
    routerType: structuralResult.routerType,
    fileType: fileType,
    reasons: reasons
  }
}


// =============================================================================
// HELPER FUNCTIONS
// =============================================================================

function determineRouterType(hasAppDir, hasPagesDir, hasSrcApp, hasSrcPages) -> NextRouterType {
  hasApp = hasAppDir or hasSrcApp
  hasPages = hasPagesDir or hasSrcPages

  if hasApp and hasPages {
    return NextRouterType.HYBRID
  } else if hasApp {
    return NextRouterType.APP_ROUTER
  } else if hasPages {
    return NextRouterType.PAGES_ROUTER
  }
  return NextRouterType.UNKNOWN
}


function isValidAppDirectory(pathSegments, appIndex) -> boolean {
  // Valid: app/, src/app/
  // Invalid: node_modules/*/app/, .next/app/

  if appIndex == 0 {
    return true  // app/ at root
  }

  if appIndex == 1 and pathSegments[0] == "src" {
    return true  // src/app/
  }

  // Check for invalid parent directories
  invalidParents = ["node_modules", ".next", "dist", "build", ".turbo"]
  for i in range(0, appIndex) {
    if pathSegments[i] in invalidParents {
      return false
    }
  }

  return true
}


function isValidPagesDirectory(pathSegments, pagesIndex) -> boolean {
  // Similar logic to isValidAppDirectory
  if pagesIndex == 0 {
    return true
  }

  if pagesIndex == 1 and pathSegments[0] == "src" {
    return true
  }

  invalidParents = ["node_modules", ".next", "dist", "build", ".turbo"]
  for i in range(0, pagesIndex) {
    if pathSegments[i] in invalidParents {
      return false
    }
  }

  return true
}


function findProjectRoot(filePath: Path) -> Path? {
  // Walk up looking for package.json or next.config.*
  currentDir = getParentDirectory(filePath)

  while currentDir != "/" {
    if exists(currentDir / "package.json") {
      return currentDir
    }

    for configFile in NEXT_CONFIG_FILES {
      if exists(currentDir / configFile) {
        return currentDir
      }
    }

    currentDir = getParentDirectory(currentDir)
  }

  return null
}


function isAtProjectRoot(filePath: Path) -> boolean {
  projectRoot = findProjectRoot(filePath)
  fileDir = getParentDirectory(filePath)

  return projectRoot == fileDir or
         projectRoot / "src" == fileDir
}


function findAppDirectory(projectRoot: Path) -> Path? {
  candidates = [
    projectRoot / "app",
    projectRoot / "src" / "app"
  ]

  for candidate in candidates {
    if exists(candidate) and isDirectory(candidate) {
      return candidate
    }
  }

  return null
}


function findPagesDirectory(projectRoot: Path) -> Path? {
  candidates = [
    projectRoot / "pages",
    projectRoot / "src" / "pages"
  ]

  for candidate in candidates {
    if exists(candidate) and isDirectory(candidate) {
      return candidate
    }
  }

  return null
}


// =============================================================================
// PUBLIC API
// =============================================================================

function detectNextJs(path: Path, options: DetectionOptions = {}) -> DetectionResult {
  depth = options.depth ?? "structural"  // "quick" | "structural" | "deep"

  if isDirectory(path) {
    switch depth {
      case "quick":
        return quickDetectFolder(path)
      case "structural":
      case "deep":
        return structuralDetectFolder(path)
    }
  } else {
    switch depth {
      case "quick":
        return quickDetectFile(path)
      case "structural":
        return structuralDetectFile(path)
      case "deep":
        return deepDetectFile(path)
    }
  }
}


// =============================================================================
// USAGE EXAMPLES
// =============================================================================

// Quick check for folder
result = detectNextJs("/projects/my-app", { depth: "quick" })
// => { isNextJs: true, confidence: 0.85, routerType: APP_ROUTER, ... }

// Deep analysis for specific file
result = detectNextJs("/projects/my-app/app/dashboard/page.tsx", { depth: "deep" })
// => { isNextJs: true, confidence: 0.98, fileType: PAGE, routerType: APP_ROUTER,
//      reasons: ["App Router special file: page", "Exports metadata object",
//                "Uses next/navigation", "Confirmed: parent is Next.js project"] }

// Detect file type in API route
result = detectNextJs("/projects/my-app/app/api/users/route.ts", { depth: "structural" })
// => { isNextJs: true, confidence: 0.90, fileType: ROUTE_HANDLER, routerType: APP_ROUTER }
```

---

## Key Design Decisions

| Decision                                    | Rationale                                                                                |
| ------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Three-layer approach**                    | Allows caller to trade off speed vs. confidence based on their use case                  |
| **Path validation before assuming Next.js** | Prevents false positives from `node_modules/some-lib/app/` or `.next/` cache directories |
| **Separate Pages vs App Router detection**  | They have different file conventions and exports                                         |
| **Confidence scoring**                      | Allows downstream tools to decide thresholds for action                                  |
| **Reason accumulation**                     | Provides explainability for debugging and UX                                             |
| **Project root walking**                    | A file in `app/` isn't Next.js unless the project itself is                              |
| **Content-based validation**                | Catches edge cases like a `page.tsx` that doesn't actually export a component            |

This would slot nicely into Vipr's technical debt analysis — you could use quick detection for scanning large codebases and deep detection for detailed file analysis.
