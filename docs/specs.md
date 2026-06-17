# Site Builder for registry.siros.org

## TL;DR

The registry site is built by [registry-cli](https://github.com/sirosfoundation/registry-cli), a Go tool that discovers credential repositories from `sources.yaml`, clones them, detects credential metadata, validates against the TS11 JSON Schema, and generates the static site deployed to GitHub Pages.

## Architecture

### Build Pipeline

1. **Source Resolution** — `sources.yaml` declares repositories via GitHub topic autodiscovery (`github:topic/vctm?org=...`) and explicit git URLs (`git:https://...`). Sources can optionally specify a `path` to restrict scanning to a subfolder (see [path targeting](https://developers.siros.org/docs/sirosid/registry/registry-cli#path-targeting)).
2. **Repository Cloning** — Each source repository is cloned (shallow, default branch from `sources.yaml`)
3. **Credential Detection** — The tool scans for `schema-meta.yaml`, `.vctm.json`, `.mdoc.json`, `.vc.json` files and Markdown files with `vct:` front matter
4. **Markdown Conversion** — Markdown credential files are converted to metadata JSON by registry-cli's built-in credential conversion library (no external binary required)
5. **TS11 Validation** — Each credential is validated against the TS11 JSON Schema; only compliant schemas appear in the API
6. **Site Generation** — HTML pages, API payloads (`/api/v1/schemas.json`), OpenAPI spec, and DCAT-AP catalogue are generated
7. **Deployment** — GitHub Actions deploys the built site to GitHub Pages every 6 hours

### URL Structure

```
https://registry.siros.org/<org>/<slug>.vctm.json     # SD-JWT VC metadata
https://registry.siros.org/<org>/<slug>.mdoc.json      # mso_mdoc configuration
https://registry.siros.org/<org>/<slug>.vc.json        # W3C VC schema
https://registry.siros.org/api/v1/schemas.json         # TS11 catalogue API
https://registry.siros.org/api/v1/schemas/<id>.json    # Individual schema
https://registry.siros.org/catalog.jsonld              # DCAT-AP catalogue
```

### Schema Identifiers

Each credential is assigned a deterministic UUID v5 identifier derived from `org/slug`, using the TS11 namespace UUID. This ensures stable, reproducible IDs across builds.

### Build Dependencies

- **Go 1.22+** — registry-cli is a Go binary
- **registry-cli** — the build tool (`go install github.com/sirosfoundation/registry-cli/cmd/registry-cli@latest`)
- **GITHUB_TOKEN** — for GitHub API access during topic-based autodiscovery

## VCTM Publication

Repositories can provide credential metadata in two ways:

1. **Pre-built metadata files** — Place `.vctm.json`, `.mdoc.json`, `.vc.json` files directly in the repository alongside a `schema-meta.yaml` for TS11 compliance
2. **Markdown authoring** — Write credential definitions as Markdown with `vct:` YAML front matter; registry-cli converts them automatically

See [Markdown Format](../docs/markdown-format.html) for the credential authoring format.

