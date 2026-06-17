# registry.siros.org

The SIROS Credential Type Registry — a public catalogue of credential type metadata implementing [ETSI TS11](https://www.etsi.org/).

## Overview

This project builds and deploys the static site at `https://registry.siros.org`. The site is built by [registry-cli](https://github.com/sirosfoundation/registry-cli), a Go tool that:

- **Discovers** credential repositories via `sources.yaml` (GitHub topic search and explicit git URLs)
- **Clones** each repository and detects credential definitions (`.vctm.json`, `.mdoc.json`, `.vc.json`, `schema-meta.yaml`)
- **Converts** Markdown credential definitions to metadata using the built-in credential conversion library
- **Validates** credentials against the TS11 JSON Schema
- **Generates** a static HTML site, TS11-compliant API (`/api/v1/schemas.json`), and DCAT-AP catalogue
- **Signs** API responses with JWS (PKCS#11) when configured

## How It Works

1. `sources.yaml` declares which repositories to include — either by GitHub topic autodiscovery or explicit git URLs
2. `registry-cli build` clones the repos, discovers credentials, and builds the site
3. GitHub Actions runs the build every 6 hours and deploys to GitHub Pages

### Autodiscovery

Repositories tagged with the `vctm` topic on GitHub are automatically discovered. The `sources.yaml` file can also list explicit git URLs for repos that don't use topics or aren't on GitHub.

### Credential Detection

For each repository, registry-cli looks for:

- **`schema-meta.yaml`** files — TS11 SchemaMeta envelopes declaring attestation level of security, binding type, and rulebook
- **`.vctm.json`** / **`.mdoc.json`** / **`.vc.json`** files — credential metadata in SD-JWT VC, mso_mdoc, and W3C VC formats
- **Markdown credential files** with `vct:` front matter — automatically converted to metadata by registry-cli (no external tool needed). Supports [nested object/array claims](https://developers.siros.org/docs/sirosid/registry/registry-cli#nested-claims) via sub-lists and [per-credential format override](https://developers.siros.org/docs/sirosid/registry/registry-cli#per-credential-formats) via `formats:` front matter.

## URL Structure

Credential metadata is accessible at:

```
https://registry.siros.org/<org>/<slug>.vctm.json
https://registry.siros.org/<org>/<slug>.mdoc.json
https://registry.siros.org/<org>/<slug>.vc.json
```

The TS11 API is at:

```
https://registry.siros.org/api/v1/schemas.json       # All TS11-compliant schemas
https://registry.siros.org/api/v1/schemas/<id>.json   # Individual schema
```

## Configuration

Source repositories are declared in `sources.yaml`:

```yaml
defaults:
  branch: vctm

sources:
  # Autodiscover repos tagged "vctm" in an org
  - "github:topic/vctm?org=sirosfoundation"

  # Explicit git repository
  - "git:https://github.com/example/credentials.git"

  # With organization label
  - url: "git:https://github.com/example/creds.git"
    organization: "Example Org"

  # Restrict to a subfolder (monorepo support)
  - url: "git:https://github.com/example/monorepo.git"
    path: "credentials/production"
```

See the [sources configuration reference](https://developers.siros.org/docs/sirosid/registry/registry-cli#path-targeting) for details on subfolder path targeting.

## Development

```bash
# Install registry-cli (requires Go 1.22+)
make install

# Build locally
make build

# Preview at http://localhost:8000
make serve
```

## Repository Structure

```
sources.yaml        # Source repository declarations
templates-go/       # HTML templates (Go html/template)
static/             # CSS, images, favicon
Makefile            # Build targets
.github/workflows/  # CI: build + deploy to GitHub Pages
```

## License

Apache-2.0
