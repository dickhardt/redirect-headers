# HTTP Redirect Headers

This repository contains the IETF Internet-Draft for HTTP Redirect Headers.

This repository uses the [IETF I-D Template](https://github.com/martinthomson/i-d-template) for building drafts.

## Draft Document

- **Latest draft**: [draft-hardt-httpbis-redirect-headers](draft-hardt-httpbis-redirect-headers.md)
- **Explainer**: [explainer.md](explainer.md) - Detailed explanation and use cases

## Abstract

This document defines HTTP headers that enable browsers to pass redirect parameters securely during HTTP redirects without exposing them in URLs. The Redirect-Query header carries parameters traditionally sent via URL query strings, the Redirect-Origin header provides browser-verified origin authentication, and the Redirect-Path header enables path-based redirect validation.

## Building the Draft

### First Time Setup

The first time you run `make`, it will automatically clone the i-d-template and install required tools (mmark, xml2rfc, etc.) in a local virtual environment.

### Local Build

Build the draft:
```bash
make
```

View the generated HTML:
```bash
open draft-hardt-httpbis-redirect-headers.html
```

Clean generated files:
```bash
make clean
```

## Automated Builds

GitHub Actions automatically builds the draft on every push to main and publishes to GitHub Pages. The latest rendered version is available at:

https://dickhardt.github.io/redirect-headers/

## Submitting to IETF

When ready to submit a new draft version:

1. Create and push a tag:
   ```bash
   git tag -a draft-hardt-httpbis-redirect-headers-00 -m "Version 00"
   git push origin draft-hardt-httpbis-redirect-headers-00
   ```

2. GitHub Actions will automatically:
   - Build the versioned draft
   - Upload it to the IETF datatracker
   - Archive it in the repository

## Authors

- Dick Hardt (Hellō) - dick.hardt@hello.coop
- Sam Goto (Google) - goto@google.com

## Contributing

This draft is in early development. Feedback and suggestions are welcome via GitHub issues.

## License

See [LICENSE](LICENSE) for details.
