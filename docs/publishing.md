# Publishing Credential Metadata

`registry-cli` handles all credential metadata conversion and publishing as part of the centralized registry build pipeline. There is no need to run a separate tool or GitHub Action on individual repositories.

## How It Works

When `registry-cli build` processes a source repository, it:

1. Clones the repository (main branch)
2. Scans for Markdown files with `vct:` YAML front matter
3. Converts them to credential metadata in all supported formats (`.vctm.json`, `.mdoc.json`, `.vc.json`)
4. Detects `schema-meta.yaml` files for TS11 compliance metadata
5. Aggregates everything into the registry site in a single sweep

## Two Ways to Provide Credential Metadata

### 1. Markdown authoring (recommended)

Write credential definitions as Markdown with YAML front matter. `registry-cli` converts them automatically during the build:

```markdown
---
vct: https://example.com/credentials/my-credential
background_color: "#003366"
text_color: "#FFFFFF"
---

# My Credential

A description of the credential.

## Claims

- `given_name` (string): Given name [mandatory]
- `family_name` (string): Family name [mandatory]
- `email` (string): Email address
- `address` (object): Postal address
    - `street` (string): Street address
    - `city` (string): City
```

Claims can be nested using Markdown sub-lists — use `(object)` for structured data and `(array)` for repeating items. See the [nested claims documentation](https://developers.siros.org/docs/sirosid/registry/registry-cli#nested-claims) for details.

To generate only specific output formats, add `formats:` to the front matter (e.g. `formats: sd-jwt, w3c`). See [per-credential format override](https://developers.siros.org/docs/sirosid/registry/registry-cli#per-credential-formats).

See [Markdown Format](../docs/markdown-format.html) for the full authoring guide.

### 2. Pre-built metadata files

Place `.vctm.json`, `.mdoc.json`, `.vc.json` files directly in the repository. These are used as-is without conversion.

## Normalization

`registry-cli` applies normalization rules during conversion to ensure credential metadata conforms to the latest specification:

| Rule | Description |
|------|-------------|
| `ensure-display-array` | Ensure `display` is an array |
| `rename-lang-to-locale` | Fix legacy `lang` → `locale` |
| `set-display-locale-default` | Default locale to `en-US` |
| `set-display-name-from-root` | Copy `name` to display entries |
| `remove-empty-svg-template-properties` | Clean up empty `properties` |
| `remove-empty-description` | Remove empty descriptions |
