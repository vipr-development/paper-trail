# CSP Testing Guide

## Quick Reference for Testing Content Security Policy

### Current CSP Configuration

Location: `clients/desktop/src/main/index.ts` (lines 89-107)

```typescript
'Content-Security-Policy': [
  "default-src 'self';",           // Default: only from same origin
  "script-src 'self';",             // Scripts: only from same origin
  "style-src 'self' https://fonts.googleapis.com;",  // Styles: self + Google Fonts CSS
  "font-src 'self' https://fonts.gstatic.com;",      // Fonts: self + Google Fonts CDN
  "img-src 'self' data:;",          // Images: self + data URIs (base64)
  "connect-src 'self';",            // XHR/fetch: only to same origin
  "object-src 'none';",             // Plugins: none allowed
  "base-uri 'self';",               // <base> tag: only same origin
  "form-action 'self';",            // Forms: only submit to same origin
  "frame-ancestors 'none';",        // Embedding: cannot be embedded
].join(' ')
```

### Testing Procedure

#### 1. Start Development Server

```bash
cd /Users/jamesleebaker/Codespace/vipr
pnpm --filter @vipr/desktop dev
```

The DevTools console will open automatically.

#### 2. Monitor Console for CSP Violations

CSP violations appear as **red errors** in the console:

```
Refused to load the stylesheet 'https://example.com/style.css' because it violates
the following Content Security Policy directive: "style-src 'self' https://fonts.googleapis.com"
```

#### 3. Test All Pages

Navigate through each page and watch for violations:

| Page          | Route            | What to Check                                  |
| ------------- | ---------------- | ---------------------------------------------- |
| Overview      | `/`              | Dashboard charts, stat cards, dark mode toggle |
| Files         | `/files`         | Table rendering, file icons, sorting           |
| Issues        | `/issues`        | Issue cards, severity badges, filtering        |
| Anti-Patterns | `/anti-patterns` | Pattern cards, expandable details              |
| Hotspots      | `/hotspots`      | Heatmap visualization, file metrics            |
| Dependencies  | `/dependencies`  | Dependency graph (if using visualization)      |
| Settings      | `/settings`      | All settings panels, toggles, inputs           |
| About         | `/about`         | Version info, links                            |

#### 4. Test Interactive Elements

For each page, verify:

- [ ] Dropdowns open/close without errors
- [ ] Modals open/close without errors
- [ ] Tooltips appear on hover
- [ ] Charts/visualizations render
- [ ] Dark mode toggle works
- [ ] All fonts load correctly
- [ ] All icons/images display

#### 5. Check Network Tab

Open DevTools Network tab:

1. Filter by "All"
2. Look for **red/failed requests** (status code 0 or failed)
3. Check if failures are due to CSP (will show in console too)

#### 6. Production Build Testing

After dev testing passes, test production build:

```bash
pnpm --filter @vipr/desktop build
pnpm --filter @vipr/desktop start
```

Production may have different CSP behavior:

- Assets served via `file://` protocol instead of `http://localhost:5173`
- No Vite HMR (Hot Module Replacement) injection
- Minified/bundled resources

### Common CSP Violations and Fixes

#### Violation: Inline Styles

**Error**:

```
Refused to apply inline style because it violates CSP directive "style-src 'self'"
```

**Cause**: Component using `style={{...}}` attribute or styled-components

**Fix**:

- Move styles to CSS file
- Use Tailwind utility classes instead
- If absolutely necessary: Add `'unsafe-inline'` to `style-src` (NOT RECOMMENDED)

#### Violation: Inline Scripts

**Error**:

```
Refused to execute inline script because it violates CSP directive "script-src 'self'"
```

**Cause**: `<script>` tag in HTML or `dangerouslySetInnerHTML` with scripts

**Fix**:

- Move script to external file
- Use React event handlers instead of inline `onclick`
- Remove any `eval()` or `Function()` calls

#### Violation: External Resources

**Error**:

```
Refused to load font from 'https://unauthorized-cdn.com/font.woff2'
```

**Fix**:

- If legitimate: Add domain to appropriate CSP directive with comment explaining why
- If not needed: Remove the resource

**Example Exception**:

```typescript
// Allow Mapbox for future dependency graph visualization
"font-src 'self' https://fonts.gstatic.com https://api.mapbox.com;",
```

### Expected CSP Compliance

The application **should pass CSP testing** because:

✅ **Tailwind v4** - All styles compiled at build time
✅ **React** - No inline event handlers
✅ **Vite** - All scripts bundled and served from self
✅ **Google Fonts** - Explicitly allowed in CSP
✅ **SVG Icons** - Embedded as data URIs (allowed)
✅ **No eval()** - TypeScript/React don't use eval

### Reporting Violations

If you find a CSP violation during testing:

1. **Copy the full console error message**
2. **Note the page/component that triggered it**
3. **Determine if the resource is necessary**
4. **Document the violation**:

```markdown
**CSP Violation Report**

- **Page**: Settings
- **Component**: ThemeToggle
- **Error**: Refused to load style from 'https://cdn.example.com/theme.css'
- **Necessary**: No - Remove third-party theme library
- **Proposed Fix**: Use local CSS file or Tailwind classes
```

### Testing Checklist

```markdown
## CSP Testing Checklist

### Development Mode

- [ ] No console errors on app start
- [ ] Overview page renders without CSP violations
- [ ] Files page renders without CSP violations
- [ ] Issues page renders without CSP violations
- [ ] Anti-Patterns page renders without CSP violations
- [ ] Hotspots page renders without CSP violations
- [ ] Dependencies page renders without CSP violations
- [ ] Settings page renders without CSP violations
- [ ] About page renders without CSP violations
- [ ] Dark mode toggle works
- [ ] All modals open/close correctly
- [ ] All dropdowns function correctly
- [ ] Charts/visualizations render
- [ ] Fonts load from Google Fonts
- [ ] SVG icons display correctly

### Production Build

- [ ] App starts without CSP errors
- [ ] All pages render correctly
- [ ] All interactive elements work
- [ ] No failed network requests
- [ ] Fonts and icons load correctly
```

### CSP Relaxation (If Needed)

If testing reveals legitimate needs for CSP relaxation:

**Before**:

```typescript
"style-src 'self' https://fonts.googleapis.com;",
```

**After** (with justification):

```typescript
// JUSTIFICATION: Third-party charting library requires inline styles
// ALTERNATIVES CONSIDERED: Self-hosted charts, but library has critical security updates
// RISK: Inline styles could enable CSS injection attacks
// MITIGATION: Library is well-maintained, styles are sanitized by React
"style-src 'self' https://fonts.googleapis.com 'unsafe-inline';",
```

### CSP Reporting Endpoint (Future)

Consider adding CSP reporting to catch violations in production:

```typescript
'Content-Security-Policy': [
  // ... existing directives ...
  "report-uri /api/csp-violations;",
].join(' ')
```

Then implement handler:

```typescript
ipcMain.handle('csp:report-violation', (event, report) => {
  logger.warn('CSP violation detected', report);
  // Could send to analytics/error tracking
});
```

## Summary

CSP testing ensures the application:

1. Doesn't execute untrusted scripts
2. Doesn't load resources from unauthorized origins
3. Doesn't use dangerous inline code patterns

Follow this guide after any major UI changes or dependency updates.
