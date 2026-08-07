# Design: TS11-Compliant Catalogue of Attestations for registry.siros.org

**Status:** Strawman / Draft  
**Date:** 2026-04-23  
**Author:** leifj  
**References:**
- [TS11 — Specification of interfaces and formats for the catalogue of attributes and the catalogue of attestations](https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/main/docs/technical-specifications/ts11-interfaces-and-formats-for-catalogue-of-attributes-and-catalogue-of-schemes.md)
- [TS11 OpenAPI spec (Annex A.3)](https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/main/docs/technical-specifications/api/ts11-cat-of-attestations-jwt-openapi31.yml)
- [TS11 JSON Schema (Annex A.2)](https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/main/docs/technical-specifications/api/ts11-json-cat-attestations-data-model.json)
- [Attestation Rulebooks Catalog](https://github.com/eu-digital-identity-wallet/eudi-doc-attestation-rulebooks-catalog)

## 1. Goal

Evolve registry.siros.org from a VCTM-only registry into a TS11-compliant **Catalogue of Attestations** that:

1. Exposes the TS11 `GET /schemas` and `GET /schemas/{schemaId}` API with JWS-signed responses
2. Preserves the existing VCTM, mDOC, and W3C VC credential files as the format-specific schemas referenced by `SchemaMeta.schemaURIs`
3. Retains **git-based workflow** as the sole authoring and authorization mechanism — no write API
4. Continues to be deployable as a static site with a thin API layer

## 2. Principles

| # | Principle | Rationale |
|---|-----------|-----------|
| P1 | **Git is the write API** | Authorization, versioning, review, and audit trail are handled by GitHub branch protection and PR review. No `PUT`/`DELETE` endpoints are implemented. |
| P2 | **Build-time signing via PKCS#11** | JWS signatures are produced at build time using the same 3-tier PKCS#11 signing model as trust-lists (dev/softhsm/yubihsm). Production uses YubiHSM2. No runtime key access required. |
| P3 | **Static-first** | All API responses are pre-built as static files. A CDN or GitHub Pages serves them directly. |
| P4 | **VCTM as schema content** | Existing VCTM/mDOC/W3C VC files are the format-specific data schemas. `SchemaMeta` is a governance envelope that references them. |
| P5 | **Incremental adoption** | The existing VCTM registry URLs and site continue to work. TS11 endpoints are additive. |
| P6 | **Go toolchain** | The build pipeline and signing tool are implemented in Go, aligning with the broader SIROS toolchain (g119612, registry-cli, trust-lists, go-cryptoutil). |

## 3. Data Model Mapping

### 3.1 SchemaMeta — the governance envelope

Each credential currently produces files like:
```
/<org>/<slug>.vctm.json   (SD-JWT VC type metadata)
/<org>/<slug>.mdoc.json   (mDOC document type)
/<org>/<slug>.vc.json     (W3C VC)
```

TS11 requires a `SchemaMeta` object per attestation type. We introduce a **new source file** in each contributing repository's `vctm` branch — authored in YAML for readability:

```
<slug>.schema-meta.yaml    (preferred)
<slug>.schema-meta.json    (also supported)
```

Only governance-decision fields require manual authoring. All other fields are **inferred at build time**:

| Field | Source | Manual? |
|-------|--------|---------|
| `attestationLoS` | `schema-meta.yaml` | Yes — governance decision |
| `bindingType` | `schema-meta.yaml` | Yes — governance decision |
| `trustedAuthorities` | `schema-meta.yaml` | Yes — deferred, optional |
| `version` | `schema-meta.yaml`, git tag, or `"0.1.0"` default | Optional |
| `supportedFormats` | Auto-detected from co-located `*.vctm.json`, `*.mdoc.json`, `*.vc.json` | No |
| `schemaURIs` | Built from detected format files + registry URL | No |
| `rulebookURI` | Auto-detected from co-located `rulebook.md` | No |
| `id` | Deterministic UUID v5 from `org/slug` | No |

Minimal example for `vctm_pid_arf_1_5.schema-meta.yaml`:

```yaml
attestation_los: iso_18045_high
binding_type: key
# version: 1.0.0  (optional — defaults to git tag or 0.1.0)
# trusted_authorities: deferred
```

At build time, the builder:
1. Reads `<slug>.schema-meta.yaml` (or `.json`) from the source repo
2. Assigns a deterministic UUID `id` (derived from `org + slug`, stable across rebuilds)
3. Infers `supportedFormats` from which format files exist for this credential
4. Populates `schemaURIs` from the discovered format files:
   ```json
   "schemaURIs": [
     { "formatIdentifier": "dc+sd-jwt", "uri": "https://registry.siros.org/sirosfoundation/vctm_pid_arf_1_5.vctm.json" },
     { "formatIdentifier": "mso_mdoc", "uri": "https://registry.siros.org/sirosfoundation/vctm_pid_arf_1_5.mdoc.json" }
   ]
   ```
5. Sets `rulebookURI` from co-located `rulebook.md` → `https://registry.siros.org/<org>/<slug>/rulebook.html`
6. Writes the complete `SchemaMeta` to `dist/api/v1/schemas/<uuid>.json`

### 3.2 Credentials without schema-meta.json

Credentials that do not yet have a `schema-meta.json` file continue to appear on the human-readable site but are **excluded** from the TS11 API responses. This allows incremental migration.

### 3.3 Format mapping

| VCTM format file | TS11 `formatIdentifier` |
|------------------|------------------------|
| `*.vctm.json`    | `dc+sd-jwt`            |
| `*.mdoc.json`    | `mso_mdoc`             |
| `*.vc.json`      | `jwt_vc_json`          |

## 4. API Design

### 4.1 Endpoint layout (static files)

```
dist/
  api/
    v1/
      openapi.yaml                         # TS11 OpenAPI spec (compatibility signaling)
      schemas.jwt                          # GET /schemas (full list, JWS-signed)
      schemas/
        <uuid>.jwt                         # GET /schemas/{schemaId} (JWS-signed)
        <uuid>.json                        # unsigned JSON (convenience, non-normative)
      orgs.json                            # GET /orgs (SIROS extension, §4.3b)
      orgs/
        <org>/
          schemas.jwt                      # GET /orgs/{org}/schemas (SIROS extension, §4.3b)
      .well-known/
        jwks.json                          # JWKS for signature verification
  <org>/
    <slug>/
      rulebook.html                        # rendered Attestation Rulebook
```

### 4.2 GET /schemas

**Static file:** `api/v1/schemas.jwt`

A JWS compact serialization containing a JWT payload conforming to `SignedSchemaListPayload`:

```json
{
  "iss": "https://registry.siros.org",
  "iat": 1745366400,
  "data": {
    "total": 4,
    "limit": 100,
    "offset": 0,
    "data": [
      { /* SchemaMeta */ },
      { /* SchemaMeta */ }
    ]
  }
}
```

Since this is a static file, **filtering and pagination are not server-side** for the global list. Two approaches:

- **Option A (recommended for now):** Serve the full list. The catalogue is expected to remain small enough (hundreds, not millions) that a single response is practical. Clients filter locally.
- **Option B (implemented for organization filtering, §4.3b):** Generate pre-computed filtered views at build time for a common query pattern. Other dimensions (e.g. by format) can follow the same approach if a real need arises.

### 4.3 GET /schemas/{schemaId}

**Static file:** `api/v1/schemas/<uuid>.jwt`

A JWS compact serialization containing the single `SchemaMeta` object.

### 4.3b GET /orgs and GET /orgs/{org}/schemas — organization filtering (SIROS extension)

**Static files:** `api/v1/orgs.json`, `api/v1/orgs/<org>/schemas.json` (+ `.jwt` when signing is enabled)

**Not part of TS11** — the spec has no "organization" concept, and `SchemaMeta`'s JSON Schema (Annex A.2) sets `additionalProperties: false` at the top level, so adding an org/repo field directly to the object would break TS11 conformance validation for every credential. Filtering is instead expressed as a **separate resource collection**: `/orgs/{org}/schemas` returns the exact same `PaginatedSchemaList` shape as the global list, containing only that organization's TS11-compliant schemas — each item byte-for-byte identical to (and as TS11-conformant as) the one served from `/schemas`. This works identically on static GitHub Pages hosting and on `registry-cli serve` (the org-scoped files are written by `build` and served as static files either way).

`/orgs.json` is an index of every organization known to this registry instance (including ones with zero TS11-compliant schemas, for discoverability), each with `name`, `schemaCount`, and `schemasURL`.

There is intentionally no static resource for an arbitrary *set* of organizations — clients wanting several orgs fetch each one's list and merge client-side. A curated, named "collection" (a fixed set of orgs declared in `sources.yaml`) could follow the same static-file pattern if a real recurring need shows up; not built speculatively.

Repo-level filtering (as opposed to organization) was considered and deferred: every current `sources.yaml` entry maps one organization to one source repo, so it would add a second dimension with no actual distinction in the data yet. Add it later, following the same pattern, if an org ever spans multiple repos.

A live query-parameter filter (`?organization=`) on `registry-cli serve` was also considered — it's a smaller change (an in-memory index in `pkg/apihandler`, never touching the `SchemaMeta` shape either) but was deferred since it only benefits self-hosted deployments; `registry.siros.org` itself is static and can't serve it live without an additional edge-function layer.

### 4.4 Write operations

Not implemented. The design document explicitly states:

> *Administration of the catalogue is performed through the git-based workflow. Contributors submit schema-meta.json files via pull requests to their credential repositories. Review and merge of PRs constitutes the authorization decision. The TS11 `PUT` and `DELETE` methods are intentionally omitted from this deployment.*

TS11 §4.5.3 requires that "operations SHALL be authorised only for the entity that has registered the attestation schema." Git branch protection rules and CODEOWNERS files fulfill this requirement — the entity (GitHub org) controls merge rights to their own repo.

### 4.5 JWS Signing

**Algorithm:** ES256 (ECDSA with P-256 and SHA-256)  
**Key management:** Reuses the SIROS 3-tier PKCS#11 signing model established in `trust-lists` and `g119612`:

| Mode | Backend | Runner | Use case |
|------|---------|--------|----------|
| `dev` | Ephemeral SoftHSM2 | GitHub-hosted | CI/PR validation |
| `softhsm` | Persistent SoftHSM2 | Self-hosted | Staging |
| `yubihsm` | YubiHSM2 hardware | Self-hosted | Production |

**Signing flow:**
1. `registry-cli build` produces unsigned JSON payloads and the static site
2. `registry-cli sign` signs each payload as JWS compact serialization via PKCS#11
3. Signed `.jwt` files are written to `dist/`
4. The corresponding public key is published as `jwks.json`

This reuses the existing `ThalesGroup/crypto11` → `miekg/pkcs11` stack and the `go-cryptoutil` algorithm registry, avoiding a separate key management approach. The same `PKCS11_URI` environment variable pattern is used:

```bash
# Example for dev mode
PKCS11_URI="pkcs11:module=/usr/lib/softhsm/libsofthsm2.so;pin=1234;token=registry"
```

The `x-jku-url` header specified in the TS11 OpenAPI is embedded in the JWS header as the `jku` parameter, pointing to `https://registry.siros.org/api/v1/.well-known/jwks.json`.

### 4.6 CORS and CDN

GitHub Pages provides:
- HTTPS with valid certificates
- Global CDN (Fastly)
- Basic DDoS protection

For additional TS11 §5.4/§5.5 compliance, a Cloudflare proxy layer can be added in front of the custom domain to provide:
- Rate limiting
- WAF rules
- Cache headers tuned for the API paths

## 5. Build Pipeline Changes

### 5.1 Current flow

```
GitHub topic search → discover repos → fetch VCTM files → render HTML + JSON → deploy to Pages
```

### 5.2 Proposed flow

```
read sources.yaml
  → resolve meta-sources (e.g. github:topic/vctm) into concrete repos
  → union with explicitly listed repos
  → fetch VCTM files (as today)
  → fetch schema-meta.yaml files (new)
  → merge SchemaMeta envelopes with discovered formats (new)
  → assign deterministic UUIDs (new)
  → sign API responses with JWS (new)
  → render HTML + JSON + JWT files
  → deploy to Pages
```

Source discovery is driven by a `sources.yaml` manifest (see §12.6 for the full format). Each entry is either an explicit git repo URL or a **meta-source** that resolves to a set of repos at build time via a platform-specific discovery mechanism:

| Meta-source pattern | Resolves via | Example |
|---------------------|-------------|---------|
| `github:topic/<topic>` | GitHub API search for repos with topic | `github:topic/vctm` |
| `gitlab:topic/<topic>` | GitLab API search | `gitlab:topic/vctm` |
| `git:<url>` | Direct git clone | `git:https://git.example.org/repo.git` |

This keeps auto-discovery available while making the source set explicit, auditable, and forge-agnostic.

### 5.3 New build dependencies

| Dependency | Purpose |
|------------|---------|
| `js-yaml` (npm) | Parse `*.schema-meta.yaml` source files |
| `marked` (npm) | Render `rulebook.md` → HTML (already declared) |
| Go signing tool | JWS signing via PKCS#11 — reuses `g119612/pkg/jws` |

### 5.4 UUID generation

Deterministic UUIDs (v5, namespace `https://registry.siros.org`) derived from `<org>/<slug>`:

```javascript
const { v5: uuidv5 } = require('uuid');
const NAMESPACE = '6ba7b811-9dad-11d1-80b4-00c04fd430c8'; // URL namespace
const id = uuidv5(`https://registry.siros.org/${org}/${slug}`, NAMESPACE);
```

This ensures stable IDs across rebuilds without requiring a database.

## 6. Source Repository Changes

### 6.1 Schema-meta authoring

Contributors add a `<slug>.schema-meta.yaml` (or `.json`) alongside their existing VCTM files in the `vctm` branch. YAML is the preferred format for authoring — it reduces the per-credential overhead to 2-3 lines of governance metadata:

```yaml
attestation_los: iso_18045_high
binding_type: key
```

All other `SchemaMeta` fields are inferred at build time (see §3.1). The registry-cli tool can be extended to:

1. **Validate** `schema-meta.yaml` against the TS11 JSON schema at publish time
2. **Scaffold** a minimal `schema-meta.yaml` from an existing VCTM file
3. **Warn** when governance fields are missing

### 6.2 vctm-registry.json extension

The per-repo `.well-known/vctm-registry.json` manifest gains an optional field:

```json
{
  "credentials": [
    {
      "slug": "vctm_pid_arf_1_5",
      "vctm": "vctm_pid_arf_1_5.vctm.json",
      "mdoc": "vctm_pid_arf_1_5.mdoc.json",
      "schemaMeta": "vctm_pid_arf_1_5.schema-meta.json"
    }
  ]
}
```

### 6.3 Attestation Rulebooks

TS11 §4.2 requires a human-readable Attestation Rulebook for each attestation type, referenced by `rulebookURI`.

**Primary approach — self-hosted markdown:**

Contributors author a `rulebook.md` file alongside their VCTM files in the credential repo. The EC's [rulebook template](https://github.com/eu-digital-identity-wallet/eudi-doc-attestation-rulebooks-catalog/tree/main/template) provides the starting scaffold. At build time:

1. The builder fetches `rulebook.md` from the source repo's `vctm` branch
2. Renders it to HTML using `marked`
3. Publishes at `https://registry.siros.org/<org>/<slug>/rulebook.html`
4. Auto-populates `rulebookURI` in the `SchemaMeta` envelope

This gives rulebooks the same git-based authoring, review, and versioning as VCTMs themselves. The rendered page is styled consistently with the rest of the registry site.

**Fallback — external reference:**

For standardised attestation types (PID, EHIC, diplomas), `rulebookURI` can point to the EC's [Attestation Rulebooks Catalog](https://github.com/eu-digital-identity-wallet/eudi-doc-attestation-rulebooks-catalog) by setting an explicit `rulebook_uri` in `schema-meta.yaml`:

```yaml
attestation_los: iso_18045_high
binding_type: key
rulebook_uri: https://github.com/eu-digital-identity-wallet/eudi-doc-attestation-rulebooks-catalog/tree/main/rulebooks/pid
```

An explicit `rulebook_uri` takes precedence over auto-detection of `rulebook.md`.

## 7. Site Changes

### 7.1 Human-readable additions

The existing credential detail pages (`/<org>/<slug>.html`) gain a new section/tab showing the TS11 governance metadata:

- Attestation Level of Security
- Binding Type
- Supported Formats (already shown, now aligned with TS11 enum)
- Trust Framework references
- Link to Attestation Rulebook
- Schema UUID and API endpoint link

### 7.2 New documentation pages

- `docs/ts11.html` — Explains TS11 compliance, API usage, JWS verification
- `docs/api.html` — API reference (generated from the subset of the TS11 OpenAPI we implement)

## 8. What This Design Does NOT Cover

| Aspect | Reason |
|--------|--------|
| `PUT /schemas/{schemaId}` | Git PRs serve as the write mechanism |
| `DELETE /schemas/{schemaId}` | Removing a credential = removing files from the repo |
| Server-side query filtering (`?attestationLoS=`, etc.) on the *static* site | `registry.siros.org` is static; clients filter the full list. (`registry-cli serve` — a live, self-hosted mode added after this document was written — does support these as real query-parameter filters via `pkg/apihandler`, for anyone who runs it.) |
| Pagination with `limit`/`offset` | Full list served; catalogue is small |
| Catalogue of Attributes (TS11 §2) | Deferred to Phase 5. Read-only approach viable; see §11 for design. |
| Runtime JWS signing | Build-time signing is sufficient for a read-only catalogue |
| OAuth 2.0 protected write access (TS11 §5.2.2) | No write endpoints |

## 9. Migration Plan

### Phase 1: Go toolchain scaffolding and schema-meta authoring
- Create `sirosfoundation/registry-cli` repo with `build` and `sign` subcommands
- Port `build.js` discovery and fetching logic to Go, driven by `sources.yaml`
- Ship default Go html/template files with the tool
- Add `schema-meta.yaml` and `rulebook.md` files to `sirosfoundation/demo-credentials` and `SUNET/vc` repos
- Extend registry-cli with `schema-meta.yaml` validation and scaffolding

### Phase 2: Build pipeline and signing
- Implement SchemaMeta inference, UUID generation, and TS11 JSON schema validation in `registry-cli build`
- Implement `registry-cli sign` with 3-tier PKCS#11 model
- Generate `api/v1/schemas.jwt`, `api/v1/schemas/<uuid>.jwt`, `jwks.json`, and `openapi.yaml`
- Set up PKCS#11 signing in GitHub Actions (dev mode with SoftHSM2 initially)

### Phase 3: Site instance conversion
- Convert `registry.siros.org` repo to a site instance: `sources.yaml`, template overrides, static assets, Makefile
- Add TS11 metadata section to credential detail template
- Add `docs/ts11.html` and `docs/api.html` pages
- Archive `scripts/build.js` and remove Node.js dependencies

### Phase 4: Validation (1 day)
- Validate generated `SchemaMeta` against TS11 JSON schema
- Verify JWS signatures using published JWKS
- Test with a TS11-aware client (or write a simple one)

### Phase 5: Catalogue of Attributes (future)
- Define `*.attr.json` and `*.attr-schema.json` file conventions in source repos
- Extend build pipeline to collect attribute definitions and JSON Schemas
- Generate `api/v1/attributes.jwt` and `api/v1/attributes/<id>.jwt` endpoints
- Publish JSON Schema files at `api/v1/attributes/schemas/<name>.json`
- Add human-readable attribute listing pages to the site
- See §11 for detailed design

## 10. Open Questions

1. **Rulebook hosting:** Resolved — registry.siros.org will host rendered rulebooks from `rulebook.md` files authored in git alongside VCTMs. External `rulebookURI` references supported as fallback via explicit `rulebook_uri` in `schema-meta.yaml`.

2. **Trust authority bootstrapping:** Deferred. The `trustedAuthorities` field is deeply tied to the ETSI Trusted Lists infrastructure. SIROS already has tooling for this (g119612, trust-lists, go-trust with LoTE support), but the integration design needs dedicated attention. For now, `trustedAuthorities` is optional and can be empty (`[]`). A future design iteration will address linking `trustedAuthorities` entries to published trust lists from the `trust-lists` repo.

3. **Catalogue of Attributes:** Resolved — the read-only, git-based approach extends naturally to attributes. Deferred to Phase 5; see §11.

4. **Key rotation:** When the signing key is rotated, old `.jwt` files become unverifiable unless the old public key remains in the JWKS. Define a key rotation policy (e.g. keep previous key in JWKS for one build cycle).

5. **Compatibility signaling:** Resolved — publish the TS11 OpenAPI specification as a static file at `api/v1/openapi.yaml`. This is the most standard and EU-aligned approach for automated API discovery. A Cloudflare proxy can add HTTP `Link` headers pointing to it later if needed. No TS11-specific well-known endpoint is defined in the spec, so OpenAPI + optional DCAT-AP metadata is the pragmatic choice.

6. **Signing tool packaging:** Resolved — `registry-cli sign` is a subcommand of the unified `registry-cli` binary. The signing logic lives in `pkg/jwssign/` and is generic (signs arbitrary JSON files via PKCS#11). It imports `g119612/pkg/jws` and `go-cryptoutil` as Go module dependencies.

## 11. Catalogue of Attributes (Phase 5)

TS11 §2 defines a **Catalogue of Attributes** alongside the Catalogue of Attestations. The two are operationally independent but complementary. The read-only, git-based approach applies cleanly to attributes — and in fact fits *better*, since TS11 defines no write API for attributes at all.

### 11.1 Comparison with Catalogue of Attestations

| Aspect | Catalogue of Attestations (§4) | Catalogue of Attributes (§2) |
|--------|-------------------------------|-----------------------------|
| Core object | `SchemaMeta` envelope | `Attribute` object |
| Schema content at URI | VCTM/mDOC/W3C VC files (exist) | JSON Schema files (need authoring) |
| Trust/governance | `trustedAuthorities` array | `authenticSources` array (per-country endpoints) |
| Write API in spec | `PUT`/`DELETE` defined (we skip) | None defined — inherently read-only |
| Discovery API | `GET /schemas` | ETSI TS 119 478 §5.1 (similar GET-based) |
| Source of truth | Credential issuers | Member states / authentic source operators |

### 11.2 Data model

The TS11 `Attribute` class (§2.1) contains:

| Field | Card. | Description |
|-------|-------|-------------|
| `identifier` | [1..1] | Unique URI identifier (namespace + local ID + version) |
| `name` | [1..*] | Language-tagged friendly names |
| `description` | [1..*] | Language-tagged descriptions |
| `nameSpace` | [0..1] | URI of the attribute's namespace |
| `distributions` | [1..*] | Array of `SchemaDistribution` (accessURL + mediaType) |
| `contactInfo` | [1..*] | Contact URIs |
| `legalBasis` | [0..*] | ELI URIs to legal basis |
| `semanticDataSpecification` | [0..1] | URI to OOTS Semantic Repository definition |
| `authenticSources` | [1..*] | Array of `DataService` (country + endpoint) |

Sub-classes:
- **`SchemaDistribution`**: `{ accessURL, mediaType }` — points to the actual JSON Schema (media type `application/json-schema`)
- **`DataService`**: `{ country, nationalSubID?, endpointURL, endpointDescription }` — per-member-state verification endpoint

### 11.3 Source file convention

Contributors add attribute definitions alongside VCTM files in the `vctm` branch:

```
attributes/
  birthdate.attr.json            # Attribute metadata object
  birthdate.attr-schema.json     # JSON Schema for the attribute
  address.attr.json
  address.attr-schema.json
```

Example `birthdate.attr.json`:

```json
{
  "identifier": "urn:eudi:attribute:birthdate:1.0",
  "name": [
    { "value": "Date of Birth", "lang": "en" },
    { "value": "Geburtsdatum", "lang": "de" }
  ],
  "description": [
    { "value": "The date of birth of the natural person", "lang": "en" }
  ],
  "nameSpace": "urn:eudi:pid:1",
  "contactInfo": ["https://siros.org/contact"],
  "legalBasis": ["http://data.europa.eu/eli/reg/2014/910/oj"],
  "authenticSources": []
}
```

The `distributions` and `authenticSources` arrays are populated at build time:
- `distributions` is generated from the co-located `*.attr-schema.json` file
- `authenticSources` is taken from the source file (may be empty for pilot deployments)

### 11.4 Build output

```
dist/
  api/
    v1/
      attributes.jwt                          # Full attribute list, JWS-signed
      attributes/
        <id-slug>.jwt                         # Individual attribute, JWS-signed
        <id-slug>.json                        # Unsigned JSON (convenience)
        schemas/
          birthdate.json                      # Actual JSON Schema file
          address.json
```

### 11.5 Authentic sources

The `authenticSources` field contains per-member-state verification endpoints operated by national authorities (e.g. Swedish Tax Agency for birthdate). This data:

1. Must come from actual authentic source operators — not something credential issuers can author unilaterally
2. Changes when countries onboard or update infrastructure
3. In the EC's vision, is managed through the OOTS/SDG infrastructure

For a SIROS deployment, `authenticSources` will typically be empty (`[]`) initially and populated as countries/sources join the pilot. The `[1..*]` cardinality in the spec is a SHOULD-level requirement for production catalogues; a pilot catalogue can relax this.

### 11.6 Relationship to attestation schemas

Attestation schemas (`SchemaMeta`) reference attributes indirectly — the format-specific schema files (VCTMs) contain `claims` arrays that define individual attributes. TS11 §4.1 notes that attribute definitions can be provided either:

- By reference to the catalogue of attributes (a URI), or
- Defined inline within the attestation schema

The VCTM `claims` array currently defines attributes inline. In Phase 5, claims could optionally carry a `catalogueAttributeRef` field pointing to the corresponding entry in the catalogue of attributes, linking the two catalogues.

## 12. Toolchain Architecture

### 12.1 Rationale for Go migration

The current Node.js builder (`scripts/build.js`, ~750 lines) handles discovery, fetching, grouping, and HTML rendering. With TS11, the pipeline must also parse YAML, validate against JSON Schema, render markdown rulebooks, infer governance metadata, generate UUIDs, and produce JWS-signed outputs via PKCS#11.

Maintaining this as a hybrid Node.js + Go pipeline (Node.js for site generation, Go for signing) introduces unnecessary integration complexity. Migrating to a pure Go toolchain:

- **Aligns with the SIROS ecosystem** — g119612, registry-cli, go-trust, go-cryptoutil, go-wallet-backend are all Go
- **Single build artifact** — one statically-linked binary per tool, no `node_modules`
- **Unified crypto stack** — PKCS#11 signing reuses `ThalesGroup/crypto11` and `go-cryptoutil` directly
- **Simpler CI/CD** — `go install` in GitHub Actions, no Node.js setup step

### 12.1.1 Separation of tool and site

The `registry-cli` tool and the `registry.siros.org` site are separate concerns:

- **`registry-cli`** is a reusable, site-agnostic Go binary. It knows how to discover credential repos, infer SchemaMeta, render templates, and sign outputs — but carries no opinion about branding, templates, or which credentials to include. Lives in its own repository (`sirosfoundation/registry-cli`).
- **`registry.siros.org`** is a *site instance* — the SIROS-specific configuration, templates, static assets, and `sources.yaml` that drive a particular deployment. It consumes `registry-cli` as a build dependency.

This separation lets other organisations deploy a private TS11-compliant catalogue by creating their own site repo with custom `sources.yaml`, templates, and branding, while reusing the same `registry-cli` binary.

### 12.2 Repository layout

The tool and the site live in separate repositories:

**`sirosfoundation/registry-cli`** — the reusable tool:

```
registry-cli/
  cmd/
    registry-cli/           # single binary with subcommands
      main.go
      build.go              # `registry-cli build` subcommand
      sign.go               # `registry-cli sign` subcommand
  pkg/
    discovery/              # sources.yaml parsing + meta-source resolution
    schemameta/             # schema-meta.yaml parsing, inference, validation
    render/                 # HTML template rendering + markdown (goldmark)
    registry/               # vctm-registry.json generation
    jwssign/                # PKCS#11 JWS signing logic
  templates/                # default Go html/template files (shipped with tool)
  go.mod
  go.sum
  Makefile
  README.md
```

**`sirosfoundation/registry.siros.org`** — the SIROS site instance:

```
registry.siros.org/
  sources.yaml              # which credential repos to include
  templates/                # site-specific template overrides (optional)
  static/                   # CSS, images, branding
  docs/                     # design docs (this file)
  Makefile                  # wraps registry-cli commands
  .github/workflows/        # CI/CD: build → sign → deploy
```

The `registry-cli` tool ships **default templates** that produce a functional TS11-compliant site out of the box. A site instance can override individual templates by placing files in its own `templates/` directory — the tool merges site-specific templates over the defaults.

### 12.3 `registry-cli build`

Replaces `scripts/build.js`. Responsibilities:

1. **Discovery** — Parse `sources.yaml`, resolve meta-sources (e.g. `github:topic/vctm`) into concrete repo URLs, union with explicitly listed repos
2. **Fetching** — Clone or fetch VCTM/mDOC/W3C VC files, `schema-meta.yaml`, `rulebook.md` from each repo's `vctm` branch
3. **Inference** — Auto-populate `SchemaMeta` fields:
   - `supportedFormats` from discovered format files
   - `schemaURIs` from registry URL + format file paths
   - `rulebookURI` from co-located `rulebook.md` or explicit `rulebook_uri`
   - `id` from deterministic UUID v5
   - `version` from YAML, git tag, or default
4. **Validation** — Validate assembled `SchemaMeta` against TS11 JSON schema (Annex A.2)
5. **Rendering** — Generate HTML site via Go templates, render `rulebook.md` via goldmark
6. **Output** — Write `dist/` with HTML site, JSON files, unsigned API payloads, static assets, `openapi.yaml`

**Key Go dependencies:**

| Package | Purpose |
|---------|---------|
| `html/template` | HTML site generation |
| `gopkg.in/yaml.v3` | Parse `schema-meta.yaml` |
| `github.com/yuin/goldmark` | Render `rulebook.md` → HTML |
| `github.com/google/uuid` | Deterministic UUID v5 |
| `github.com/santhosh-tekuri/jsonschema/v5` | TS11 JSON schema validation |

**CLI interface:**

```
registry-cli build \
  --output dist/ \
  --base-url https://registry.siros.org \
  --sources sources.yaml \
  --templates templates/
```

### 12.4 `registry-cli sign`

Generic JWS signing subcommand. Not registry-specific — signs any JSON files in a directory via PKCS#11.

**Responsibilities:**

1. Read unsigned JSON files from an input directory (or glob pattern)
2. Wrap each in a JWT payload with `iss` and `iat` claims
3. Sign as JWS compact serialization (ES256) via PKCS#11
4. Write `.jwt` files alongside the `.json` originals
5. Generate `jwks.json` with the public key

**Key Go dependencies:**

| Package | Purpose |
|---------|---------|
| `github.com/ThalesGroup/crypto11` | PKCS#11 interface |
| `github.com/go-jose/go-jose/v4` | JWS compact serialization |
| `github.com/sirosfoundation/go-cryptoutil` | Algorithm registry, ECDSA format handling |

**CLI interface:**

```
registry-cli sign \
  --input dist/api/v1/ \
  --pattern "*.json" \
  --pkcs11-uri "pkcs11:module=...;pin=...;token=..." \
  --key-label registry-signing \
  --issuer https://registry.siros.org \
  --jku https://registry.siros.org/api/v1/.well-known/jwks.json \
  --jwks-output dist/api/v1/.well-known/jwks.json
```

The subcommand also supports a **list mode** for signing a single aggregate payload (the `schemas.jwt` response containing all `SchemaMeta` objects):

```
registry-cli sign \
  --input dist/api/v1/schemas/ \
  --aggregate dist/api/v1/schemas.jwt \
  --pkcs11-uri "..." \
  ...
```

### 12.5 Build and CI/CD

**Makefile targets (site repo):**

```makefile
REGISTRY_CLI_VERSION ?= latest

install:        ## Install registry-cli
	go install github.com/sirosfoundation/registry-cli/cmd/registry-cli@$(REGISTRY_CLI_VERSION)

generate:       ## Run registry-cli build to generate dist/
	registry-cli build --output dist/ --base-url https://registry.siros.org \
	  --sources sources.yaml --templates templates/ --static static/

sign:           ## Sign API payloads
	registry-cli sign --input dist/api/v1/ --pkcs11-uri "$$PKCS11_URI" \
	  --key-label registry-signing --issuer https://registry.siros.org

site: install generate sign   ## Full pipeline
```

**GitHub Actions workflow (site repo):**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest          # or self-hosted for yubihsm mode
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - run: make install
      - run: make generate
      - name: Sign (dev mode)
        run: |
          # Install SoftHSM2, generate ephemeral key
          make sign PKCS11_URI="pkcs11:module=/usr/lib/softhsm/libsofthsm2.so;..."
      - uses: actions/upload-pages-artifact@v3
        with: { path: dist/ }
  deploy:
    needs: build
    uses: actions/deploy-pages@v4
```

### 12.6 Source manifest (`sources.yaml`)

The `sources.yaml` file declares where credential data comes from. Each entry is either an **explicit repo** or a **meta-source** that resolves to repos at build time. This decouples `registry-cli` from any specific forge while preserving auto-discovery for platforms that support it.

```yaml
# sources.yaml — source manifest for registry-cli build

sources:
  # Meta-source: auto-discover all GitHub repos with the "vctm" topic
  # in the sirosfoundation org
  - github:topic/vctm?org=sirosfoundation

  # Explicit repos (any git host)
  - git:https://github.com/example-org/example-credential.git
  - git:https://gitlab.example.eu/national-id/pid-schema.git

  # GitLab meta-source (hypothetical)
  # - gitlab:topic/vctm?group=eu-wallet

defaults:
  branch: vctm            # default branch to fetch from (overridable per-repo)
```

**Meta-source resolution rules:**

| Scheme | Resolver | Auth | Notes |
|--------|----------|------|-------|
| `github:topic/<t>` | GitHub Search API (`topic:<t>`) | `GITHUB_TOKEN` env var | Optional `?org=` filter |
| `gitlab:topic/<t>` | GitLab Projects API (`topic=<t>`) | `GITLAB_TOKEN` env var | Optional `?group=` filter |
| `git:<url>` | Direct `git clone --depth 1` | SSH key or credential helper | Any git-compatible host |

Meta-sources are resolved once at the start of a build. The resolved repo set is logged for auditability. If a repo appears in both a meta-source result and an explicit entry, the explicit entry's settings (branch, path overrides) take precedence.

### 12.7 Migration from Node.js

The migration is incremental:

1. Create `sirosfoundation/registry-cli` repo with Go module structure
2. Implement `registry-cli build` in Go, initially producing identical output to `build.js`
3. Validate output parity with a diff test (`build.js` output vs `registry-cli build` output)
4. Add TS11 features (schema-meta, rulebooks, API payloads) in `registry-cli build`
5. Implement `registry-cli sign`
6. Convert `registry.siros.org` from a monolith to a site instance: move `sources.yaml`, templates, and static assets to the root; replace `scripts/build.js` with `Makefile` calling `registry-cli`
7. Switch GitHub Actions to the Go pipeline
8. Archive `scripts/build.js` and remove Node.js dependencies

The Handlebars templates are converted to Go templates at step 2. Default templates ship with `registry-cli`; SIROS-specific overrides stay in `registry.siros.org/templates/`.
