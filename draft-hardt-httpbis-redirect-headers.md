%%%
title = "HTTP Redirect Headers"
abbrev = "Redirect Headers"
ipr = "trust200902"
area = "Applications and Real-Time"
workgroup = "HTTP"
keyword = ["http", "redirect", "oauth", "security", "privacy"]

[seriesInfo]
status = "standard"
name = "Internet-Draft"
value = "draft-hardt-httpbis-redirect-headers-latest"
stream = "IETF"

date = 2025-12-15T00:00:00Z

[[author]]
initials = "D."
surname = "Hardt"
fullname = "Dick Hardt"
organization = "Hellō"
  [author.address]
  email = "dick.hardt@hello.coop"

[[author]]
initials = "S."
surname = "Goto"
fullname = "Sam Goto"
organization = "Google"
  [author.address]
  email = "goto@google.com"

%%%

.# Abstract

This document defines HTTP headers that enable browsers to pass redirect parameters securely during HTTP redirects without exposing them in URLs. The Redirect-Query header carries parameters traditionally sent via URL query strings, the Redirect-Origin header provides browser-verified origin authentication, and the Redirect-Path header enables path-based redirect validation. These headers address security and privacy concerns in authentication and authorization protocols such as OAuth 2.0 and OpenID Connect by preventing parameter leakage through browser history, Referer headers, server logs, and analytics systems.

{mainmatter}

# Introduction

Authentication and authorization protocols (OAuth [@!RFC6749], OpenID Connect [@OIDC], SAML) use browser redirects to navigate users between applications and authorization servers. These redirects must carry protocol parameters, which historically appear in URLs or POSTed forms.

Current redirect mechanisms have significant limitations: URL parameters leak sensitive data through browser history, Referer headers, server logs, analytics, and JavaScript access. POST-based redirects expose parameters in DOM form fields and require SameSite=None cookies for cross-site operation. The Referer header is unreliable for origin verification as it may be stripped, rewritten, or removed by privacy tools and enterprise proxies.

This document defines three HTTP headers that address these limitations by moving redirect parameters into browser-controlled headers that are not exposed in URLs or the DOM:

- **Redirect-Query** - Carries parameters traditionally sent via URL query strings
- **Redirect-Origin** - Provides browser-verified origin authentication
- **Redirect-Path** - Enables path-based redirect validation

**Incremental Deployment:** A key feature of this specification is that deployment does not require coordination between parties. Each party (client application, browser, authorization server) can independently add support for Redirect Headers. Full functionality emerges naturally when all three parties support the mechanism, but partial deployment gracefully degrades to existing URL-based behavior. This allows for organic adoption without requiring synchronized upgrades across the ecosystem.

# Redirect Headers

Three headers work together during top-level 302/303 redirects:

| Header | Set by | Direction | Purpose |
|--------|--------|-----------|---------|
| Redirect-Query | Server (client or AS) | Both directions | Carry parameters without URLs |
| Redirect-Origin | Browser | Both directions | Mutual origin authentication |
| Redirect-Path | Client | Client to AS | Validate redirect_uri path |

**Browser behavior:** Only processes these headers during top-level redirects. Ignores them for normal requests or embedded resources.

## Redirect-Query

Carries redirect parameters using URL query string encoding.

```
Redirect-Query: "code=SplxlOBe&state=123"
```

- Replaces URL query parameters
- Parsed using standard URL query string parsing
- Prevents exposure via browser history, Referer, logs, and analytics
- Set by server (client or AS)

## Redirect-Origin

Browser-supplied origin of the page initiating the redirect.

```
Redirect-Origin: "https://app.example"
```

**Properties:**

- Set ONLY by the browser (cannot be spoofed by scripts)
- Enables AS to verify which client initiated the request
- Enables client to verify response came from correct AS
- Provides cryptographic mutual authentication (browser-mediated)

## Redirect-Path

Optional path prefix for additional redirect_uri validation.

**Client sets:**
```
Redirect-Path: "/app1/"
```

**Browser validates:**

1. Checks current URL path begins with /app1/
2. If valid: includes Redirect-Path in request to AS
3. If invalid: omits header (client cannot lie)

**AS enforces:**

- redirect_uri MUST begin with Redirect-Origin + Redirect-Path

This prevents redirect_uri manipulation attacks within the same origin.

# Use Cases

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

# OAuth Redirect Security Threats

**Scope:** Redirect Headers specifically addresses OAuth and OIDC web-based redirect flows between websites where sensitive parameters are passed via URL query strings. This proposal does NOT address form_post mechanisms where data appears in the DOM, as that attack vector requires different mitigations.

## Authorization Code Theft from URL Query Strings

The primary security weakness is in the **authorization server's response** containing the authorization code in the URL query string. When an AS redirects back to the client with ?code=...&state=..., the authorization code is exposed through multiple vectors:

**Known attack vectors:**

1. **Browser history leakage** - Authorization codes stored in browser history can be retrieved by attackers with device access
   - Reference: OAuth 2.0 Security Best Current Practice [@OAUTH-SECURITY-TOPICS]

2. **Server log exposure** - Authorization codes visible in web server access logs, proxy logs, and load balancer logs
   - Codes can be extracted in real-time or from archived logs

3. **Referer header leakage** - When the callback page loads third-party resources (images, scripts, analytics), the authorization code leaks via the Referer header
   - Reference: OAuth 2.0 authentication vulnerabilities [@PORTSWIGGER-OAUTH]

4. **Browser-swapping attacks** - Attackers exploit scenarios where authorization codes leak through shared URLs or when users switch browsers during the flow
   - Discussion: OAuth-WG Browser-Swapping thread

5. **URL sharing** - Users may inadvertently share URLs containing authorization codes after errors or confusion

6. **Analytics and crash reporting** - Authorization codes captured by analytics systems, error tracking, and monitoring tools

**What Redirect Headers mitigates:**

By moving the authorization code from the URL query string to the Redirect-Query header, **all of these attack vectors are eliminated**. The authorization code never appears in:

- URLs (no browser history exposure)
- Referer headers (no leakage to third parties)
- Server logs (when servers are configured to not log sensitive headers)
- User-visible locations (no accidental sharing)

**Important clarification:**

The security concern is specifically the **authorization server's response** with the authorization code. The **client's authorization request** to the AS (containing client_id, redirect_uri, state) does not have known security concerns from being in the URL, as these parameters are not sensitive credentials. However, moving them to headers provides consistency and reduces URL clutter.

# OAuth Incremental Deployment

Redirect Headers is designed for **incremental adoption** - each party (client, browser, authorization server) can independently add support, with functionality emerging when all parties support it.

## How It Works

**Clients** can start sending parameters in both locations:

- URL query string (for backward compatibility)
- Redirect-Query header (signaling support)

**Browsers** forward Redirect- headers when present:

- No special detection needed
- Simply forward the headers during redirects

**Authorization Servers** detect support and respond accordingly:

- If request includes Redirect-Query → AS knows both client and browser support it
- AS can then send response using **only** Redirect-Query header (no URL parameters)

## Adoption Path

Each party adds support independently, in any order:

```
Clients add support
  ├─ Send params in URL + Redirect-Query
  └─ Look for response in header, fall back to URL

Browsers add support
  ├─ Forward Redirect-* headers when present
  └─ No configuration needed

Authorization Servers add support
  ├─ Detect support when Redirect-Query received
  ├─ Confirm: client + browser both support it
  └─ Send response ONLY in Redirect-Query (omit URL params)

Result: Once all three support it → authorization code sent in header, not URL
```

**No coordination required** - each party adds support independently, and the system naturally converges to the secure behavior once all three support it. The client can immediately start sending both, browsers simply forward headers, and authorization servers detect support from incoming requests.

# Feature Discovery

Some protocols may wish to explicitly discover browser support for Redirect Headers before relying on them. While not required for security (backward compatibility ensures graceful fallback), feature discovery can optimize behavior.

## Using Client Hints

Servers can advertise support and request browser capabilities using Client Hints [@!RFC8942]:

**Server advertises support:**
```
Accept-CH: Redirect-Supported
```

**Browser responds with capability:**
```
Redirect-Supported: ?1
```

## Optimization for OAuth

Once both browser and AS support is confirmed through feature discovery, OAuth clients MAY send parameters **only** in Redirect-Query, omitting URL parameters entirely. This:

- Reduces URL length
- Prevents any possibility of parameter leakage via URLs
- Maintains backward compatibility (unsupported browsers fall back to URL parameters)

**Note:** Feature discovery is optional. The incremental deployment model works without explicit discovery - authorization servers detect support by receiving Redirect-Query headers in requests.

# Security Considerations

**Security:**

- Redirect-Query contains sensitive data - servers MUST treat as confidential
- Browsers MUST prevent JavaScript from reading or setting redirect headers
- Redirect-Path supplements but does NOT replace redirect_uri validation
- Network middleboxes (proxies, load balancers) can still observe headers
- Defense assumes honest browser (cannot protect against browser compromise)
- Transition period: URLs still leak until clients stop sending URL parameters

# Privacy Considerations

**Privacy:**

- Authorization codes and tokens removed from browser history
- No Referer leakage to third-party resources
- No exposure to JavaScript, browser extensions, or DOM inspection
- Reduced logging in analytics, crash reports, and server logs
- User-visible URLs are clean and don't reveal sensitive parameters

# IANA Considerations

This document registers three new HTTP header fields in the "Hypertext Transfer Protocol (HTTP) Field Name Registry" defined in [@!RFC9110].

## Redirect-Query Header Field

Header field name: Redirect-Query

Applicable protocol: http

Status: standard

Author/Change controller: IETF

Specification document(s): [this document]

## Redirect-Origin Header Field

Header field name: Redirect-Origin

Applicable protocol: http

Status: standard

Author/Change controller: IETF

Specification document(s): [this document]

## Redirect-Path Header Field

Header field name: Redirect-Path

Applicable protocol: http

Status: standard

Author/Change controller: IETF

Specification document(s): [this document]

# Implementation Status

**Note to RFC Editor:** Please remove this section before publication.

**Specification status:** Exploratory draft

**Browser support:** Not yet implemented (proposed specification)

**Server support:** Reference implementations needed

This specification requires:

- Browser vendors to implement header handling
- Authorization servers to support Redirect-Query
- Client applications to adopt the pattern

Deployment strategy: Backward compatible - clients send both URL and headers during transition.

{backmatter}

# OAuth Example: Before and After

This appendix provides a detailed comparison of OAuth flows with and without Redirect Headers to illustrate the differences in security and functionality.

## Without Redirect Headers (current OAuth)

**Client Website returns to Browser:**
```
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123&redirect_uri=...
```

**Browser navigates, sends to AS:**
```
GET /authorize?client_id=abc&state=123&redirect_uri=...
Host: as.example
Referer: https://app.example/login
```
← Unreliable, may be stripped

**AS returns code to Browser:**
```
HTTP/1.1 302 Found
Location: https://app.example/cb?code=SplxlOBe&state=123
```
← Leaked in URL

**Browser sends code to Client Website:**
```
GET /cb?code=SplxlOBe&state=123
Host: app.example
Referer: https://as.example/consent
```
← In browser history, logs, analytics
← Third-party resources see code via Referer

**Problems:**

- Authorization code appears in URL (history, logs, Referer, extensions)
- No cryptographic origin verification (Referer is optional and unreliable)

## With Redirect Headers

**Client Website returns to Browser:**
```
HTTP/1.1 302 Found
Location: https://as.example/authorize?client_id=abc&state=123
Redirect-Query: "client_id=abc&state=123"
Redirect-Path: "/app1/"
```

**Browser navigates, adds origin and forwards to AS:**
```
GET /authorize?client_id=abc&state=123
Host: as.example
Redirect-Origin: "https://app.example"
Redirect-Path: "/app1/"
Redirect-Query: "client_id=abc&state=123"
```
← Browser-supplied, cannot be spoofed

**AS validates and returns to Browser:**
```
HTTP/1.1 302 Found
Location: https://app.example/cb
Redirect-Query: "code=SplxlOBe&state=123"
```
← No parameters in URL!

**Browser forwards back to Client Website:**
```
GET /cb
Host: app.example
Redirect-Origin: "https://as.example"
Redirect-Query: "code=SplxlOBe&state=123"
```
← Clean URL
← Client verifies this
← Not in URL, history, or Referer

**Benefits:**

- Authorization code never appears in URLs
- Mutual origin authentication (browser-verified)
- Backward compatible (browsers/servers without support fall back to URL parameters)

**Requirements:**

- If Redirect-Query received in request: AS MUST use Redirect-Query for response
- Client MUST verify Redirect-Origin matches expected AS
- AS MUST verify Redirect-Origin matches expected client
- When Redirect-Query is present, client MUST ignore URL parameters and use only header parameters

# Acknowledgments

The authors would like to thank early reviewers for their valuable feedback and insights that helped shape this proposal: Jonas Primbs, Warren Parad.

<reference anchor='RFC6749' target='https://www.rfc-editor.org/info/rfc6749'>
  <front>
    <title>The OAuth 2.0 Authorization Framework</title>
    <author initials='D.' surname='Hardt' fullname='D. Hardt' role='editor'><organization /></author>
    <date year='2012' month='October' />
  </front>
</reference>

<reference anchor='OIDC' target='https://openid.net/specs/openid-connect-core-1_0.html'>
  <front>
    <title>OpenID Connect Core 1.0</title>
    <author initials='N.' surname='Sakimura' fullname='N. Sakimura'><organization /></author>
    <author initials='J.' surname='Bradley' fullname='J. Bradley'><organization /></author>
    <author initials='M.' surname='Jones' fullname='M. Jones'><organization /></author>
    <author initials='B.' surname='de Medeiros' fullname='B. de Medeiros'><organization /></author>
    <author initials='C.' surname='Mortimore' fullname='C. Mortimore'><organization /></author>
    <date year='2014' month='November' />
  </front>
</reference>

<reference anchor='OAUTH-SECURITY-TOPICS' target='https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics'>
  <front>
    <title>OAuth 2.0 Security Best Current Practice</title>
    <author initials='T.' surname='Lodderstedt' fullname='T. Lodderstedt' role='editor'><organization /></author>
    <author initials='J.' surname='Bradley' fullname='J. Bradley'><organization /></author>
    <author initials='A.' surname='Labunets' fullname='A. Labunets'><organization /></author>
    <author initials='D.' surname='Fett' fullname='D. Fett'><organization /></author>
    <date year='2024' month='October' />
  </front>
</reference>

<reference anchor='PORTSWIGGER-OAUTH' target='https://portswigger.net/web-security/oauth'>
  <front>
    <title>OAuth 2.0 authentication vulnerabilities</title>
    <author><organization>PortSwigger</organization></author>
    <date year='2024' />
  </front>
</reference>
