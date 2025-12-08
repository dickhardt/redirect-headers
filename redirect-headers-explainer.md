# Redirect Headers: Explainer Draft

_Last updated: 2025-11-15_

**Author:** Dick Hardt  
**Email:** dick.hardt@hello.coop  

## Table of Contents
1. Introduction  
2. Problems with Existing Redirect Mechanisms  
3. Redirect Headers  
   - 3.1 Redirect-Query  
   - 3.2 Redirect-Origin  
   - 3.3 Redirect-Path 
4. How OAuth Works Today  
5. OAuth Flow Using Redirect Headers  
6. Client Hint Optimization  
7. Security Considerations  
8. Privacy Considerations  
9. Related Documents

## 1. Introduction
Many authentication and authorization protocols combine direct server-to-server communication with browser-mediated navigation. OAuth, OpenID Connect, SAML, and many proprietary sign-in flows require sending the user from the application to an Authorization Server, then returning them to the application with the result.

Because these navigation steps happen through the browser, the protocol must carry parameters during redirects. Historically, browsers only support two mechanisms for carrying redirect parameters: URL query and POSTed form fields.

Redirect Headers introduce a new, browser-controlled mechanism for carrying redirect parameters without exposing them in URLs or the DOM.

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
Redirect Headers introduce three browser-related headers that are only added during a top-level 302 redirect.  
If a response is not a redirect, browsers ignore these headers and do not forward them.

The headers are:
- `Redirect-Query`, redirect parameters encoded like a query string  
- `Redirect-Origin`, the origin of the initiating page  
- `Redirect-Path`, an optional path prefix validated by the browser  

### 3.1 Redirect-Query
Example (AS → Client response):

```
Redirect-Query: code=SplxlOBeZQQYbYS6WxSbIA&state=123
```

The client parses `Redirect-Query` the same way it parses a URL query component.  
This moves authorization codes out of URLs, reducing exposure via browser history, `Referer`, analytics, crash reporting, and extension access.

### 3.2 Redirect-Origin
Browser-supplied origin of the initiating page.

Example:
```
Redirect-Origin: https://app.example
```

Properties:
- Set only by the browser  
- Not modifiable by scripts  
- Identifies the origin initiating the redirect  
- Allows the AS to know which application initiated the request  
- Allows the client to know the response came from the AS  

This enables mutual origin authentication through the browser.

### 3.3 Redirect-Path
Clients may provide:
```
Redirect-Path: /app1/
```
The browser then:

1. Verifies that the current top-level URL path **begins with** `/app1/`.  
2. If the check passes, the browser includes the same `Redirect-Path` value on the redirect request to the Authorization Server.  
3. If the check fails, the browser omits `Redirect-Path` entirely (the client cannot lie about it).

The AS can then enforce:

- `redirect_uri` MUST begin with `Redirect-Origin` plus `Redirect-Path`.

## 4. How OAuth Works Today

### Step 1 — Client sends OAuth request via 302

```
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&...state=123&
```

### Step 2 — Browser navigates to AS with full URL and Referer

```
GET /authorize?client_id=abc&...&state=123
Host: as.example
Referer: https://app.example/login
```

### Step 3 — AS authenticates then returns the authorization code in the URL

```
HTTP/1.1 302 Found
Location: https://app.example/cb?code=SplxlOBe&state=123
```

### Step 4 — Browser returns to the client and leaks the code via Referer

```
GET /cb?code=SplxlOBe...&state=123
Host: app.example
Referer: https://as.example/consent
```

### Why this is a problem

- Third-party resources on the callback page receive the full URL containing `code=...` via `Referer`.  
- URLs appear in browser history, server logs, CDN logs, analytics, and telemetry.  
- URLs are visible to JavaScript and browser extensions.  
- OAuth cannot authenticate the origin of the initiating webpage using only URL parameters and optional `Referer`.

Redirect Headers eliminate these problems.



## 5. OAuth Flow Using Redirect Headers
### Step 1 — Client sends OAuth request with URL parameters and Redirect-Query
```
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&...state=123
Redirect-Query: client_id=abc&...state=123
Redirect-Path: /app1/
```


### Step 2 — Browser adds Redirect-Origin and forwards Headers
```
GET /authorize?client_id=abc&...state=123
Host: as.example
Redirect-Origin: https://app.example
Redirect-Path: /app1/
Redirect-Query: client_id=abc&...state=123
```
> If the browser does not support Redirect Headers, they are not sent.

### Step 3 — AS returns using Redirect Headers
```
HTTP/1.1 302 Found
Location: https://app.example/cb
Redirect-Query: code=SplxlOBeZQQYbYS6WxSbIA&state=123
```

If Redirect Headers are received by a Redirect Header capable AS, it MUST NOT include response parameters in the URL.

> If the AS does not support Redirect Headers, none are sent.

### Step 4 — Browser forwards back to client
```
GET /cb
Host: app.example
Redirect-Origin: https://as.example
Redirect-Query: code=SplxlOBeZQQYbYS6WxSbIA&state=123
```

If Redirect-Query is present, the client MUST only use those parameters.
The client MUST verify the Redirect-Origin is for the AS.  

## 6. Client Hint Optimization
Clients normally send parameters in both the URL and Redirect-Query.

Browsers without Redirect Header support ignore them.  
AS instances without support ignore them.

To optimize:
```
Accept-CH: Redirect-Supported
```

Browser replies:
```
Redirect-Supported: ?1
```

When both:
1. Browser supports Redirect Headers  
2. AS responds using Redirect Headers  

Then the client MAY omit URL parameters.

## 7. Security Considerations
- Servers must treat Redirect-Query as sensitive  
- Middleboxes may still observe headers  
- Redirect-Path does not replace redirect_uri validation  
- Browsers must prevent scripts from setting redirect headers  
- Cannot defend against a fully compromised browser  
- URLs still leak until clients stop sending parameters in the URL  

## 8. Privacy Considerations
- Few parameters appear in URLs  
- Reduced Referer leakage  
- Reduced analytics and telemetry exposure  
- No DOM exposure of redirect parameters  

## 9. Related Documents
- [OAuth 2.0 Security Best Current Practice (RFC 9126)](https://tools.ietf.org/rfc/rfc9126.html)  
- [OAuth 2.0 Threat Model and Security Considerations (RFC 6819)](https://tools.ietf.org/rfc/rfc6819.html)
- [Fetch Metadata Request Headers](https://w3c.github.io/webappsec-fetch-metadata/)
- [W3C Privacy Principles](https://w3ctag.github.io/privacy-principles/)
- [Client Hints Infrastructure](https://tools.ietf.org/html/rfc8942)
