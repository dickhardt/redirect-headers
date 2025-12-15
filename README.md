# HTTP Redirect Headers

This is the working area for the individual Internet-Draft, "HTTP Redirect Headers".

* [Editor's Copy](https://dickhardt.github.io/redirect-headers/draft-hardt-httpbis-redirect-headers.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-hardt-httpbis-redirect-headers)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-hardt-httpbis-redirect-headers)
* [Compare Editor's Copy to Individual Draft](https://dickhardt.github.io/redirect-headers/#go.draft-hardt-httpbis-redirect-headers.diff)

## Abstract

This document defines HTTP headers that enable browsers to pass redirect parameters securely during HTTP redirects without exposing them in URLs. The Redirect-Query header carries parameters traditionally sent via URL query strings, the Redirect-Origin header provides browser-verified origin authentication, and the Redirect-Path header enables path-based redirect validation. These headers address security and privacy concerns in authentication and authorization protocols such as OAuth 2.0 and OpenID Connect.

## Additional Resources

* [Explainer Document](explainer.md) - Detailed explanation, use cases, and examples

## Contributing

See the [guidelines for contributions](https://github.com/dickhardt/redirect-headers/blob/main/CONTRIBUTING.md).

Contributions can be made by creating pull requests.
The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## Authors

- Dick Hardt (Hellō)
- Sam Goto (Google)
