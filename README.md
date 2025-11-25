# Redirect Headers: Explainer Draft

_Last updated: 2025-11-15_

**Author:** Dick Hardt
**Email:** dick.hardt@hello.coop

## TL;DR

Redirect Headers move sensitive parameters from URLs to HTTP headers during browser redirects:
- **Redirect-Query** - Redirect parameters (replaces URL query string)
- **Redirect-Origin** - Browser-supplied origin (enables mutual authentication)
- **Redirect-Path** - Optional path validation (prevents redirect_uri manipulation)

**Why:** Authorization codes, tokens, and other sensitive data leak through URLs (browser history, Referer header, logs, analytics). Redirect Headers keep this data in headers that browsers control and don't expose to JavaScript or third parties.

**Primary use case:** Web server based OAuth/OIDC flows, but works for any protocol using browser redirects (SAML, proprietary auth flows). 

SPA, inapp mobile browsers are out of scope of this proposal.

---

## Table of Contents
1. Introduction
2. Problems with Existing Redirect Mechanisms
3. Redirect Headers
   - 3.1 Redirect-Query
   - 3.2 Redirect-Origin
   - 3.3 Redirect-Path
4. Use Cases
5. OAuth Example: Before and After
6. Security and Privacy Considerations
7. Optimization: Omitting URL Parameters
8. Implementation Status
9. Related Documents

## 1. Introduction

Authentication and authorization protocols (OAuth, OpenID Connect, SAML) use browser redirects to navigate users between applications and authorization servers. These redirects must carry protocol parameters, which historically appear in URLs or POSTed forms.

**Problem:** URLs leak sensitive data through browser history, Referer headers, server logs, analytics, and JavaScript access.

**Solution:** Redirect Headers move parameters into browser-controlled HTTP headers that aren't exposed in URLs or the DOM.

## 2. Problems with Existing Redirect Mechanisms
### URL-based redirects
- Parameters appear in the URL  
- URLs enter browser history  
- URLs are visible to scripts and extensions  
- URLs leak via the `Referer` header  
- URLs are logged in servers, proxies, analytics, crash reporting  

### POST redirects
- Parameters appear in DOM form fields  
- Visible to JavaScript  
- POST bodies may be logged or inspected  
- Cross-site POST requires `SameSite=None` cookies  

### `Referer` is unreliable
- May be stripped, rewritten, or truncated  
- May be removed by privacy tools and enterprise proxies  

Redirect Headers solve these limitations.

## 3. Redirect Headers

Three headers work together during top-level 302/303 redirects:

| Header | Set by | Direction | Purpose |
|--------|--------|-----------|---------|
| Redirect-Query | Server (client or AS) | Both directions | Carry parameters without URLs |
| Redirect-Origin | Browser | Both directions | Mutual origin authentication |
| Redirect-Path | Client | Client → AS | Validate redirect_uri path |

**Browser behavior:** Only processes these headers during top-level redirects. Ignores them for normal requests or embedded resources.  

### 3.1 Redirect-Query

Carries redirect parameters using URL query string encoding.

```http
Redirect-Query: "code=SplxlOBe&state=123"
```

- Replaces URL query parameters
- Parsed using standard URL query string parsing
- Prevents exposure via browser history, Referer, logs, and analytics
- Set by server (client or AS)

### 3.2 Redirect-Origin

Browser-supplied origin of the page initiating the redirect.

```http
Redirect-Origin: "https://app.example"
```

**Properties:**
- Set ONLY by the browser (cannot be spoofed by scripts)
- Enables AS to verify which client initiated the request
- Enables client to verify response came from correct AS
- Provides cryptographic mutual authentication (browser-mediated)

### 3.3 Redirect-Path

Optional path prefix for additional redirect_uri validation.

**Client sets:**
```http
Redirect-Path: "/app1/"
```

**Browser validates:**
1. Checks current URL path begins with `/app1/`
2. If valid: includes Redirect-Path in request to AS
3. If invalid: omits header (client cannot lie)

**AS enforces:**
- redirect_uri MUST begin with `Redirect-Origin` + `Redirect-Path`

This prevents redirect_uri manipulation attacks within the same origin.

---

## 4. Use Cases

**Primary: OAuth and OpenID Connect**
- Authorization code flow
- Implicit flow (though deprecated)
- Hybrid flows

**Other authentication protocols:**
- SAML assertions
- Proprietary SSO flows
- Any protocol requiring browser-mediated parameter passing

**When NOT to use:**
- Public, non-sensitive parameters that can appear in URLs
- Server-to-server communication (no browser involved)
- Protocols that don't use browser redirects

---

## 5. OAuth Example: Before and After

### Without Redirect Headers (current OAuth)

**Client Website returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123&redirect_uri=...
```

**Browser navigates, sends to AS:**
```http
GET /authorize?client_id=abc&state=123&redirect_uri=...
Host: as.example
Referer: https://app.example/login  ← Unreliable, may be stripped
```

**AS returns code to Browser:**
```http
HTTP/1.1 302 Found
Location: https://app.example/cb?code=SplxlOBe&state=123  ← Leaked in URL
```

**Browser sends code to Client Website:**
```http
GET /cb?code=SplxlOBe&state=123  ← In browser history, logs, analytics
Host: app.example
Referer: https://as.example/consent  ← Third-party resources see code via Referer
```

**Problems:**
- Authorization code appears in URL (history, logs, Referer, extensions)
- No cryptographic origin verification (Referer is optional and unreliable)

---

### With Redirect Headers

**Client Website returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123
Redirect-Query: "client_id=abc&state=123"
Redirect-Path: "/app1/"
```

**Browser navigates, adds origin and forwards to AS:**
```http
GET /authorize?client_id=abc&state=123
Host: as.example
Redirect-Origin: "https://app.example"  ← Browser-supplied, cannot be spoofed
Redirect-Path: "/app1/"
Redirect-Query: "client_id=abc&state=123"
```

**AS validates and returns to Browser:**
```http
HTTP/1.1 302 Found
Location: https://app.example/cb  ← No parameters in URL!
Redirect-Query: "code=SplxlOBe&state=123"
```

**Browser forwards back to Client Website:**
```http
GET /cb  ← Clean URL
Host: app.example
Redirect-Origin: "https://as.example"  ← Client verifies this
Redirect-Query: "code=SplxlOBe&state=123"  ← Not in URL, history, or Referer
```

**Benefits:**
- Authorization code never appears in URLs
- Mutual origin authentication (browser-verified)
- Backward compatible (browsers/servers without support fall back to URL parameters)

**Requirements:**
- If Redirect-Query received in request: AS MUST use Redirect-Query for response
- Client MUST verify Redirect-Origin matches expected AS
- AS MUST verify Redirect-Origin matches expected client
- When Redirect-Query is present, client MUST ignore URL parameters and use only header parameters

---

## 6. Security and Privacy Considerations

**Security:**
- Redirect-Query contains sensitive data - servers MUST treat as confidential
- Browsers MUST prevent JavaScript from reading or setting redirect headers
- Redirect-Path supplements but does NOT replace redirect_uri validation
- Network middleboxes (proxies, load balancers) can still observe headers
- Defense assumes honest browser (cannot protect against browser compromise)
- Transition period: URLs still leak until clients stop sending URL parameters

**Privacy:**
- Authorization codes and tokens removed from browser history
- No Referer leakage to third-party resources
- No exposure to JavaScript, browser extensions, or DOM inspection
- Reduced logging in analytics, crash reports, and server logs
- User-visible URLs are clean and don't reveal sensitive parameters

---

## 7. Optimization: Omitting URL Parameters

For backward compatibility, clients initially send parameters in both URL and Redirect-Query.

Once both browser and AS support is confirmed, clients MAY send parameters only in Redirect-Query.

**Detection using Client Hints:**
```http
Accept-CH: Redirect-Supported
```

Browser responds:
```http
Redirect-Supported: ?1
```

This optimization reduces URL length but isn't required for security benefits.

---

## 8. Implementation Status

**Specification status:** Exploratory draft
**Browser support:** Not yet implemented (proposed specification)
**Server support:** Reference implementations needed

This specification requires:
- Browser vendors to implement header handling
- Authorization servers to support Redirect-Query
- Client applications to adopt the pattern

Deployment strategy: Backward compatible - clients send both URL and headers during transition.

---

## 9. Related Documents
- [OAuth 2.0 Security Best Current Practice (RFC 9126)](https://tools.ietf.org/rfc/rfc9126.html)  
- [OAuth 2.0 Threat Model and Security Considerations (RFC 6819)](https://tools.ietf.org/rfc/rfc6819.html)
- [Fetch Metadata Request Headers](https://w3c.github.io/webappsec-fetch-metadata/)
- [W3C Privacy Principles](https://w3ctag.github.io/privacy-principles/)
- [Client Hints Infrastructure](https://tools.ietf.org/html/rfc8942)
