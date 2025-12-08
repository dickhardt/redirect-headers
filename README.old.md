# Redirect Headers
## Explainer Draft

_Last updated: 2025-11-13_

**Author:** Dick Hardt  
**Email:** dick.hardt@hello.coop  

## Introduction

Many web authentication and authorization systems such as OAuth 2.0, OpenID Connect, and SAML use **browser‑mediated communication** where one party sends the user to another party for interaction, then receives the user back with a result.

### Example: OAuth Login Flow
1. User clicks "Login with Google" on `app.example`
2. Browser redirects to `accounts.google.com` with OAuth parameters in the URL
3. User authenticates with Google
4. Google redirects back to `app.example/callback?code=abc123&state=xyz`

This creates a fundamental architectural pattern: **protocol parameters are embedded in browser navigations**.

### Current Browser Mechanisms Have Security Issues

**URL Redirects** expose sensitive data:
- OAuth codes appear in browser history, server logs, and Referer headers
- Readable by JavaScript, extensions, and browser subsystems
- Leaked through analytics, CDNs, and proxy logs

**Form POSTs** avoid URL exposure but create new problems:
- Parameters visible in DOM and to JavaScript
- Require `SameSite=None` cookies, weakening session protection
- Cross-site POST bodies logged by WAFs and proxies

**Neither provides reliable origin validation** for the initiating site.

Because browsers lack a secure primitive for redirect parameters, all browser‑mediated identity protocols inherit these vulnerabilities.


## Goals

- **Transport redirect parameters securely** without URL exposure
- **Preserve `SameSite=Lax` cookie behavior** for session protection  
- **Provide strong origin and path binding** to prevent cross-app attacks
- **Keep parameters hidden from JavaScript** and browser extensions
- **Maintain backward compatibility** with existing OAuth/OIDC flows
- **Enable graceful fallback** when browser support is unavailable

---

# Proposed Solution: Browser transmits parameters with HTTP headers.

The browser transmits OAuth redirect parameters using two new `Redirect-` prefixed headers. These headers are browser-controlled and invisible to JavaScript.

## How It Works

### Client → Authorization Server
```
Redirect-Origin: https://app.example/app1/
Redirect-Query: client_id=abc&scope=openid&redirect_uri=https://app.example/app1/cb&state=123
```

### Authorization Server → Client  
```
Redirect-Origin: https://as.example/
Redirect-Query: code=SplxlOBeZQQYbYS6WxSbIA&state=123
```

- **`Redirect-Origin`** — Originating site's origin plus path prefix
- **`Redirect-Query`** — OAuth parameters in query-string format  
- **Browser-enforced** — JavaScript cannot read, modify, or spoof these headers

> Note: `Redirect-Query` could be a structured header. Eg: 
>
>`Redirect-Query: oauth; qs="client_id=abc&scope=openid&..."`

---

# Why Path Prefix Validation Matters

Origin-only validation is insufficient for sites hosting multiple applications:

**Multi-app scenario:**
- `https://example.com/app/` (shopping app)
- `https://example.com/admin/` (admin panel)

**Attack without path binding:**
1. Attacker controls shopping app OAuth flow
2. Sets `redirect_uri=https://example.com/admin/callback`  
3. User authenticates, believing they're logging into shopping
4. Authorization code delivered to admin panel under attacker's control

**Prevention with path binding:**
Browser sends: `Redirect-Origin: https://example.com/app/`  
Authorization Server enforces: `redirect_uri` MUST begin with `https://example.com/app/`  
Attack blocked: `https://example.com/admin/callback` doesn't match path prefix.

---

# OAuth Flow Example

### 1. Feature Detection
Client requests capability:
```
Accept-CH: Redirect-Supported  
```

Browser responds if supported:
```
Redirect-Supported: ?1
```

### 2. Client → Authorization Server
```
Redirect-Origin: https://app.example/oauth/
Redirect-Query: client_id=myapp&scope=openid&...
```

### 3. Authorization Server Validation
- Verify `redirect_uri` starts with `Redirect-Origin` path  
- Parse and validate OAuth parameters normally
- Authenticate user

### 4. Authorization Server → Client
```
Redirect-Origin: https://as.example/
Redirect-Query: code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```

### 5. Client Validation  
- Verify expected authorization server origin
- Validate state parameter  
- Exchange code for tokens

### 6. Fallback Behavior
If `Redirect-Supported` is absent, client falls back to standard URL parameters.

---

# Security and Privacy Benefits

## Security Properties
- **No sensitive parameters in URLs** — eliminates most common leak vectors
- **Browser-enforced origin and path binding** — prevents cross-app impersonation attacks  
- **Preserves `SameSite=Lax` cookies** — maintains existing session security
- **Headers invisible to JavaScript** — prevents malicious script access

## Privacy Properties  
Redirect parameters no longer appear in:
- Browser history and bookmarks
- Server, CDN, and proxy logs (by default)
- Referer headers sent to third parties
- Analytics and telemetry systems
- OS/browser URL processing subsystems

## Remaining Exposure Risks
Headers may still appear in debug logs, enterprise proxies, or packet captures. However, headers are far less likely to be logged than URLs, which appear in access logs by default.

---

# Implementation Considerations

## Browser Support Detection
Authorization servers check for `Redirect-Origin` header presence — no separate discovery needed.

## Backward Compatibility  
Clients gracefully fall back to URL parameters when browser support is unavailable.

## Structured Headers Option
`Redirect-Query` could use structured header format:
```
Redirect-Query: oauth; qs="client_id=abc&scope=openid%20profile"
```

---

# References

- [OAuth 2.0 Security Best Current Practice (RFC 9126)](https://tools.ietf.org/rfc/rfc9126.html)  
- [OAuth 2.0 Threat Model and Security Considerations (RFC 6819)](https://tools.ietf.org/rfc/rfc6819.html)
- [Fetch Metadata Request Headers](https://w3c.github.io/webappsec-fetch-metadata/)
- [W3C Privacy Principles](https://w3ctag.github.io/privacy-principles/)
- [Client Hints Infrastructure](https://tools.ietf.org/html/rfc8942)

---

# Next Steps

- **Browser vendor engagement** — Chrome, Firefox, Safari security teams
- **Standards coordination** — W3C WebAppSec WG and OAuth WG alignment  
- **W3C TAG review** — architectural feedback and web platform integration
- **Prototype development** — browser implementation or polyfill extension
- **Security analysis** — formal review of threat model and mitigations

