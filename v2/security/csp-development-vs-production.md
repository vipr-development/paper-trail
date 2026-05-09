# Content Security Policy: Development vs Production

## Overview

Vipr desktop application uses different CSP (Content Security Policy) configurations for development and production environments to balance security with developer ergonomics.

## Development Mode CSP

### Configuration

```typescript
csp: {
  defaultSrc: ["'self'"],
  scriptSrc: ["'self'", "'unsafe-inline'"],     // ← Allow inline for Vite HMR
  styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],  // ← Allow inline for HMR
  fontSrc: ["'self'", 'https://fonts.gstatic.com'],
  imgSrc: ["'self'", 'data:'],
  connectSrc: ["'self'", 'ws:', 'wss:'],        // ← Allow WebSocket for HMR
  objectSrc: ["'none'"],
  baseUri: ["'self'"],
  formAction: ["'self'"],
  frameAncestors: ["'none'"],
}
```

### Why Permissive?

Vite's development server requires `'unsafe-inline'` for:

1. **Hot Module Replacement (HMR)** - Injects inline `<style>` tags dynamically to apply CSS changes without full page reload
2. **Dev Client Script** - May inject inline scripts to establish WebSocket connection for HMR
3. **WebSocket Connection** - Requires `ws:` and `wss:` in `connectSrc` for live reload communication

Without these directives, the development experience breaks:

- CSS changes don't apply (CSP blocks inline styles)
- HMR WebSocket fails (CSP blocks connection)
- Dev tools injection fails (CSP blocks inline scripts)

### Security Trade-offs

**Risk**: Inline script/style execution is allowed, which could be exploited if:

- A malicious npm package injects code during development
- A compromised Vite plugin serves malicious content
- Local network attacker intercepts `ws://` HMR traffic

**Mitigation**:

- Development mode only runs on `localhost` (not exposed to external networks)
- Vite dev server is trusted first-party code
- Production builds use strict CSP (see below)

## Production Mode CSP

### Configuration

```typescript
csp: {
  defaultSrc: ["'self'"],
  scriptSrc: ["'self'"],                        // ← No unsafe-inline
  styleSrc: ["'self'", 'https://fonts.googleapis.com'],  // ← No unsafe-inline
  fontSrc: ["'self'", 'https://fonts.gstatic.com'],
  imgSrc: ["'self'", 'data:'],
  connectSrc: ["'self'"],                       // ← No WebSocket origins
  objectSrc: ["'none'"],
  baseUri: ["'self'"],
  formAction: ["'self'"],
  frameAncestors: ["'none'"],
}
```

### Why Strict?

Production builds:

- **All styles are static CSS files** - Vite bundles styles into `.css` files at build time
- **No inline scripts** - All JavaScript is bundled into `.js` files
- **No HMR required** - Production app doesn't need hot reload infrastructure

This strict CSP prevents:

- XSS attacks via inline script injection
- Style injection attacks
- Data exfiltration via unauthorized external connections
- Clickjacking via frame embedding

## Implementation Details

### Config Selection

`src/shared/config.ts` exports environment-specific configurations:

```typescript
export const config: AppConfig = ConfigSchema.parse(
  process.env['NODE_ENV'] === 'production' ? productionConfig : developmentConfig
);
```

Electron's main process reads `NODE_ENV`:

- Development: `pnpm dev` → `NODE_ENV=development` → permissive CSP
- Production: `pnpm package` → `NODE_ENV=production` → strict CSP

### CSP Application

`src/main/index.ts` applies CSP via `webRequest.onHeadersReceived`:

```typescript
mainWindow.webContents.session.webRequest.onHeadersReceived((details, callback) => {
  callback({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': [buildCSPHeader()],
    },
  });
});
```

The `buildCSPHeader()` function in `config.ts` constructs the CSP header string from the active configuration.

## Testing CSP

### Development Testing

1. Start dev server: `pnpm --filter @vipr/desktop dev`
2. Open DevTools Console
3. Verify NO CSP violations for:
   - Vite HMR style injection
   - WebSocket connection (`ws://localhost:5173`)
   - Dev client scripts

4. Test HMR:
   - Change a Tailwind class in a component
   - Save file
   - Verify styles update instantly without page reload

### Production Testing

1. Build production app: `pnpm --filter @vipr/desktop package`
2. Run packaged app
3. Open DevTools Console
4. Verify CSP violations ARE blocked:
   - Inline `<script>` tags
   - Inline `style` attributes
   - External script/style URLs (except allowed Google Fonts)

5. Verify app functionality:
   - Styles load correctly from bundled CSS
   - JavaScript executes from bundled files
   - No HMR infrastructure present

## Future Considerations

### Nonce-based CSP

Instead of `'unsafe-inline'` in development, we could use nonces:

```typescript
// Generate nonce per request
const nonce = crypto.randomBytes(16).toString('base64');

// Apply to CSP
styleSrc: ["'self'", `'nonce-${nonce}'`, 'https://fonts.googleapis.com']

// Inject nonce into Vite HTML template
<style nonce="${nonce}">...</style>
```

**Benefits**: More secure even in development
**Drawbacks**: Requires Vite plugin to inject nonces, adds complexity

### Subresource Integrity (SRI)

For external resources (Google Fonts), we could add SRI hashes:

```typescript
fontSrc: ["'self'", 'https://fonts.gstatic.com'],
// Add SRI to link tags in index.html
<link href="..." integrity="sha384-..." crossorigin="anonymous">
```

**Benefits**: Prevents MITM attacks on external resources
**Drawbacks**: Hashes must be updated when fonts change

## Security Best Practices

1. **Never weaken production CSP** - Always maintain `'self'` only for scripts/styles
2. **Minimize allowed origins** - Only add external origins when absolutely necessary
3. **Use environment detection** - Never ship development CSP to production
4. **Monitor violations** - Log CSP violations in production to detect attacks
5. **Test both modes** - Verify CSP doesn't break legitimate functionality

## Related Documentation

- Phase 4: Security Hardening (`documentation/docs/feature-development/electron-app/round-one/phase-04-security-hardening.md`)
- Electron Security Guide: https://www.electronjs.org/docs/latest/tutorial/security
- MDN CSP Reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP

## Troubleshooting

### Problem: Styles not loading in development

**Symptom**: Console shows `Refused to apply inline style because it violates CSP`

**Solution**: Verify `developmentConfig.security.csp.styleSrc` includes `'unsafe-inline'`

### Problem: HMR WebSocket fails

**Symptom**: Console shows `WebSocket connection failed` or CSP violation for `ws://localhost:5173`

**Solution**: Verify `developmentConfig.security.csp.connectSrc` includes `'ws:'` and `'wss:'`

### Problem: Production app blocked by CSP

**Symptom**: Legitimate scripts/styles blocked in packaged app

**Solution**:

1. Check if resources are from allowed origins (`'self'`, Google Fonts)
2. Verify resources are bundled correctly (not loaded dynamically)
3. Review `productionConfig.security.csp` for missing allowed origins

### Problem: CSP not updating after config change

**Symptom**: Changed CSP config but violations persist

**Solution**:

1. Rebuild: `pnpm --filter @vipr/desktop build`
2. Restart dev server (not just HMR refresh)
3. Clear Electron cache: Delete `~/Library/Application Support/vipr-desktop` (macOS)
