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

This document defines HTTP headers that enable secure parameter passing and mutual authentication during browser redirects. The Redirect-Query header carries parameters in browser-controlled headers instead of URLs, preventing leakage through browser history, Referer headers, server logs, and analytics systems. The Redirect-Origin header provides browser-verified origin authentication that cannot be spoofed or stripped, enabling reliable mutual authentication between parties. The optional Redirect-Path header allows servers to request path-specific origin verification. Together, these headers address critical security and privacy concerns in authentication and authorization protocols such as OAuth 2.0 and OpenID Connect.

{mainmatter}

# Introduction

Authentication and authorization protocols (OAuth [@!RFC6749], OpenID Connect [@OIDC], SAML) use browser redirects to navigate users between applications and authorization servers. These redirects must carry protocol parameters, which historically appear in URLs or POSTed forms.

This document addresses two distinct problems in redirect-based protocols:

1. **Parameter Leakage**: URL parameters leak sensitive data (authorization codes, tokens, session identifiers) through browser history, Referer headers, server logs, analytics systems, and JavaScript access. POST-based redirects expose parameters in DOM form fields. Both mechanisms make sensitive data visible to unintended parties.

2. **Origin Verification**: Current mechanisms for verifying the origin of redirects are unreliable. The Referer header may be stripped, rewritten, or removed by privacy tools and enterprise proxies, preventing reliable mutual authentication between parties.

This document defines two HTTP headers that address these problems:

- **Redirect-Query** - Carries parameters in a browser-controlled header instead of URLs, preventing leakage while always being paired with origin verification
- **Redirect-Origin** - Provides browser-verified origin authentication that cannot be spoofed by scripts or manipulated by intermediaries

A third header, **Redirect-Path**, allows servers to request path-specific origin verification when finer-grained validation is needed.

**Note on independent use**: While origin verification without parameter passing is theoretically possible (server sends Redirect-Path, browser responds with Redirect-Origin), no specific use cases have been identified for this configuration.

**Incremental Deployment:** A key feature of this specification is that deployment does not require coordination between parties. Each party (client application, browser, authorization server) can independently add support for Redirect Headers. Full functionality emerges naturally when all three parties support the mechanism, but partial deployment gracefully degrades to existing URL-based behavior. This allows for organic adoption without requiring synchronized upgrades across the ecosystem.

# Redirect Headers

Two headers work together during top-level 302/303 redirects, with an optional third header for path-specific validation:

| Header | Set by | Direction | Purpose |
|--------|--------|-----------|---------|
| Redirect-Query | Server | Both directions | Carry parameters without URLs |
| Redirect-Origin | Browser | Both directions | Verified origin (+ optional path) |
| Redirect-Path | Server | Server to Browser | Request path-specific origin |

**Browser behavior:** Only processes these headers during top-level redirects. Ignores them for normal requests or embedded resources.

## Redirect-Query

The Redirect-Query header carries redirect parameters using URL query string encoding. It is set by servers (either the client application or authorization server) in redirect responses.

```
Redirect-Query: "code=SplxlOBe&state=123"
```

**Properties:**

- Set by server in HTTP redirect response (302/303)
- Replaces URL query parameters
- Parsed using standard URL query string parsing
- Prevents exposure via browser history, Referer, logs, and analytics
- When present, browser MUST include Redirect-Origin in the subsequent request

## Redirect-Origin

The Redirect-Origin header provides browser-verified origin authentication. It is set ONLY by the browser and contains the origin (and optionally path) of the page from which the redirect originated.

**Format:** Always ends with `/`

```
Redirect-Origin: "https://app.example/"
Redirect-Origin: "https://app.example/app1/"  (when Redirect-Path validated)
```

**Browser behavior:**

The browser sets Redirect-Origin when either Redirect-Query or Redirect-Path is present in the redirect response:

1. **Base case**: Redirect-Origin is set to the origin of the current page plus `/`
   - Current page: `https://app.example/some/page`
   - Redirect-Origin: `https://app.example/`

2. **With Redirect-Path**: If the server includes Redirect-Path in the response, the browser validates the path claim:
   - Server sends: `Redirect-Path: "/app1/"`
   - Browser checks: Does current page path start with `/app1/`?
   - If YES: `Redirect-Origin: "https://app.example/app1/"`
   - If NO: `Redirect-Origin: "https://app.example/"` (path claim rejected)

**Properties:**

- Set ONLY by the browser (cannot be spoofed by scripts or intermediaries)
- Enables receiving party to verify the origin of the redirect
- Provides mutual authentication between parties
- Always ends with `/` for consistent parsing
- May include validated path when Redirect-Path is used

## Redirect-Path

The Redirect-Path header allows a server to request path-specific origin verification. It is set by the server in the redirect response as a claim about the current page's path. The browser validates this claim and, if valid, includes the path in Redirect-Origin.

**Format:** Must start and end with `/`

```
Redirect-Path: "/app1/"
```

**Server behavior:**

The server includes Redirect-Path in the redirect response when it wants the receiving party to know not just the origin, but also a specific path within that origin.

**Browser validation:**

1. Server sends: `Redirect-Path: "/app1/"`
2. Browser checks: Does the current page path start with `/app1/`?
3. If valid: Include path in `Redirect-Origin: "https://example.com/app1/"`
4. If invalid: Ignore the path claim, use origin only: `Redirect-Origin: "https://example.com/"`

This mechanism prevents path manipulation attacks where an attacker might try to redirect from an unexpected path within the same origin. The server cannot lie about its path because the browser enforces validation.

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

## Header Confidentiality

Redirect-Query carries sensitive parameters (authorization codes, tokens, session identifiers) that MUST be protected with the same care as credentials. Servers MUST:

- Configure logging systems to exclude or redact Redirect-Query header values
- Treat Redirect-Query with the same confidentiality as Authorization headers
- Use TLS for all redirects carrying Redirect-Query to prevent network observation

Network intermediaries (proxies, load balancers, CDNs) can observe header values in transit. Deployment environments with untrusted intermediaries require additional protection beyond this specification.

## Browser Implementation Requirements

Browsers MUST enforce strict isolation of Redirect headers:

- JavaScript MUST NOT be able to read or set Redirect-* headers via any API
- Browser extensions MUST NOT have access to Redirect-* headers
- Only the browser's redirect handling mechanism can create or consume these headers
- Headers MUST only be processed during top-level navigation redirects, never for subresource requests

Failure to enforce these restrictions would allow malicious scripts to forge origin claims or steal sensitive parameters.

## Origin Verification Limits

Redirect-Origin provides browser-mediated mutual authentication but has limitations:

- It verifies the browser's understanding of origin, not the server's identity
- It does not authenticate the user or establish a secure channel
- It supplements but does NOT replace redirect_uri validation and registration
- Servers MUST continue to validate redirect_uri against registered values

Redirect-Path provides additional validation within an origin but cannot prevent attacks where the attacker controls a legitimate path within the same origin.

## Browser Trust Model

This specification assumes an honest browser implementation. It cannot protect against:

- Compromised or malicious browsers
- Browser bugs that fail to enforce header restrictions
- Browser extensions with elevated privileges
- Debugging tools that modify headers

Servers should monitor for anomalous behavior (e.g., Redirect-Origin values that don't match expected patterns) as potential indicators of browser compromise or implementation bugs.

## Transition Period Risks

During incremental deployment, clients may send parameters in both URL and headers for backward compatibility. This dual-sending pattern preserves URL leakage risks until:

- All parties (client, browser, server) support Redirect Headers
- Clients stop including parameters in URLs

Servers SHOULD detect Redirect-Query presence and warn or reject requests that also include sensitive parameters in URLs, to encourage migration away from URL-based parameters.

# Privacy Considerations

## URL History and Referer Leakage

When parameters remain in URLs (as during transition or with non-supporting implementations), sensitive data persists in browser history and may leak via Referer headers. Implementers should consider:

- Browser history is persistent storage that may be accessed by malware, forensic tools, or unauthorized users with device access
- Referer headers are sent automatically to third-party resources, potentially leaking parameters to analytics providers, CDNs, or advertisers
- The transition to headers does not eliminate these risks until all parties stop sending parameters in URLs

This specification does not eliminate all tracking vectors - cookies, browser fingerprinting, and other mechanisms remain unaffected.

## Network Observer Privacy

Network intermediaries can still observe Redirect-* headers in transit, just as they can observe URL parameters today. This specification does not enhance privacy against network-level observers (ISPs, proxies, corporate firewalls). TLS remains essential for protecting parameters from network observation.

In some cases, moving parameters to headers may slightly improve privacy: unlike URL paths (which may be visible in TLS SNI or DNS queries), HTTP headers are encrypted by TLS and not visible to passive network observers.

## Redirect-Origin Privacy Implications

Redirect-Origin explicitly reveals the origin (and optionally path) of the redirecting page. Implementers should consider:

- This is similar to the Referer header but more reliable and always present when Redirect-Query is used
- It enables the receiving party to know definitively where the redirect originated
- Unlike Referer (which users/tools can strip for privacy), Redirect-Origin cannot be disabled by the user when Redirect-Query is present
- This trade-off prioritizes security (mutual authentication) over origin hiding

Protocols using Redirect Headers should only be deployed where mutual knowledge of party identities is acceptable and expected (as in OAuth, where the AS and client already know each other).

## User Control and Transparency

Users have limited control over Redirect Headers:

- Users cannot inspect or modify Redirect-* headers (unlike URL parameters which are visible)
- Users cannot selectively disable Redirect-Origin without breaking functionality
- Browser developer tools may or may not expose these headers depending on implementation

Browser implementations SHOULD provide visibility into Redirect Headers in developer tools for transparency, while maintaining the security restriction that JavaScript cannot access them.

## Server Logging Practices

While Redirect Headers remove parameters from URLs (reducing accidental logging via URL-based logs), servers must implement appropriate logging controls:

- Configure web servers and load balancers to exclude Redirect-Query from access logs
- Ensure application logging redacts sensitive header values
- Be aware that default logging configurations may capture all headers

The shift to headers does not automatically prevent logging - it requires conscious configuration changes.

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
Location: https://as.example/authorize
Redirect-Query: "client_id=abc&state=123&redirect_uri=https://app.example/app1/cb"
Redirect-Path: "/app1/"
```

**Browser validates and adds origin:**
- Current page: `https://app.example/app1/page`
- Redirect-Path claim: `/app1/` ✓ (page path starts with `/app1/`)
- Sets Redirect-Origin: `https://app.example/app1/`

**Browser navigates, sends to AS:**
```
GET /authorize
Host: as.example
Redirect-Origin: "https://app.example/app1/"
Redirect-Query: "client_id=abc&state=123&redirect_uri=https://app.example/app1/cb"
```
← Browser-supplied origin+path, cannot be spoofed
← Parameters not in URL!

**AS validates and returns to Browser:**
- Verifies Redirect-Origin: `https://app.example/app1/`
- Verifies redirect_uri starts with: `https://app.example/app1/`
```
HTTP/1.1 302 Found
Location: https://app.example/app1/cb
Redirect-Query: "code=SplxlOBe&state=123"
```
← No parameters in URL!

**Browser forwards back to Client Website:**
- Current page: `https://as.example/consent`
- Redirect-Query present → Browser sets Redirect-Origin
```
GET /app1/cb
Host: app.example
Redirect-Origin: "https://as.example/"
Redirect-Query: "code=SplxlOBe&state=123"
```
← Clean URL
← Client verifies Redirect-Origin is `https://as.example/`
← Authorization code not in URL, history, or Referer

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

