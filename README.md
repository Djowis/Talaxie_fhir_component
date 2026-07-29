# talaxie-fhir-components

Custom [Talaxie](https://www.talaxie.org/) (Talend Open Studio fork) components for [FHIR](https://hl7.org/fhir/) integration.

| Component | Purpose | Status |
|---|---|---|
| `tFHIRInput` | Executes a FHIR search (HTTP GET), follows Bundle pagination, and streams each resource as a row. | Stable |

## Requirements

- Talaxie Studio 8+ (legacy `javajet` user component model)
- Job execution JVM: Java 11+ (`java.net.http` client)
- Jackson Databind (declared in the component descriptor, resolved by the Studio via Maven)

## Installation

1. Download the latest release and unzip it.
2. Copy the `tFHIRInput` folder (the folder itself, not its parent, not the zip) into your user components directory.
3. In the Studio: *Window > Preferences > Components*, set **User component folder** to the directory *containing* `tFHIRInput`, then Apply.
4. The component appears in the palette under **Healthcare/FHIR**.

The component folder must sit at depth 1 of the user components directory: the Studio does not scan subfolders and does not accept zip files for legacy components.

## tFHIRInput

`tFHIRInput` is a start-of-flow component (no input connector, one main output).

### Output schema (fixed)

| Column | Type | Content |
|---|---|---|
| `resourceId` | String | Logical id of the resource (`Resource.id`) |
| `resourceType` | String | FHIR resource type |
| `lastUpdated` | String | `meta.lastUpdated`, raw ISO 8601 instant (kept as text on purpose: converting to a date type would impose an implicit timezone decision; do it downstream with the pattern of your choice) |
| `resourceJson` | String | Complete resource as raw JSON, ready for `tExtractJSONFields` or any JSON-aware processing |

### Parameters

| Parameter | Notes |
|---|---|
| FHIR server base URL | e.g. `"https://hapi.fhir.org/baseR4"`. Context variables supported, as in any Talend field. |
| Resource type | e.g. `"Patient"`, `"Organization"` |
| Search parameters | Key/value table. Values are URL-encoded at runtime; names are sent as-is so FHIR modifiers (`family:exact`) work. Use `_count` here to control page size. |
| HTTP headers | Name/value table of custom headers sent with every request to the FHIR server (not to the OAuth2 token endpoint). Typical use: API keys such as `ESANTE-API-KEY` for the French Annuaire Santé. Values are plain text fields: for secrets, reference a context variable rather than hardcoding the key in the job. |
| Authentication | None / Basic / Bearer token / OAuth2 client credentials |
| Follow HTTP redirects (3xx) | Enabled by default |
| Max pages | Pagination safety guard (default 50). Reaching it logs a truncation warning and stops; it is not an error. |
| Timeout (ms) | Applied to connection and each request |
| Die on error | Enabled by default: any HTTP or parsing failure kills the job (fail fast). When disabled, the error is logged to stderr and stored in `ERROR_MESSAGE`. |

### Returned values (globalMap)

`NB_LINE` (resources emitted), `NB_PAGES` (pages fetched), `ERROR_MESSAGE`.

### Pagination

The component follows `Bundle.link[relation=next]` URLs exactly as provided by the server, page after page, streaming rows as it goes (one page in memory at a time). Empty pages that still carry a `next` link — a real-world behavior of some servers — are traversed transparently.

### OAuth2 notes

- Grant type: `client_credentials` only (machine-to-machine). Interactive flows do not apply to batch ETL.
- Client credentials are sent in the token request body (`client_secret_post`). Servers requiring `client_secret_basic` are not supported yet.
- The access token is obtained once, before pagination starts. Extractions running longer than the token lifetime will fail on a 401; token refresh is not implemented.
- Token endpoint errors surface the RFC 6749 `error` / `error_description` fields. Secrets are never logged.

## Column mappings (FHIRPath)

Add your own columns to the output schema (next to the four fixed ones), then map each to a FHIRPath expression in the *Column mappings* table. Column names are entered without quotes (schema identifiers); expressions are quoted string values, as with any Talend field.

- A mapped column must already exist in the schema, otherwise the job fails at build time with a clear message naming the missing column.
- Empty result → null. Single value → the value. Multiple values → comma-joined. Complex elements are rendered as JSON; refine the expression (`.first()`, `.where(...)`) to get a scalar.
- Invalid FHIRPath syntax fails the job unconditionally at startup, even if "Die on error" is off: bad syntax is a design error, not a runtime condition. A valid expression matching nothing yields null (missing data, not an error).
- Example (French Annuaire Santé, pick the RPPS identifier among several): `identifier.where(system='https://rpps.esante.gouv.fr').value.first()`

At startup HAPI logs a few benign `WARN ... Unable to load resource: profiles-*.xml` lines: this component parses but does not validate, so profile resources are intentionally not shipped. Silence them in your project's own logging configuration if desired.

## TLS / certificates

The component uses the JVM's default SSL context. If the target server's certificate chain is not trusted by the Java runtime (`PKIX path building failed`), fix it at the runtime level: import the missing CA into the JRE's `cacerts` (`keytool -importcert -cacerts ...`), or use `tSetKeystore` at the start of the job. By design, this component will never offer a "disable certificate verification" option.

## Known limitations (v1 scope)

- Read/search only: no write (`tFHIROutput`), no `$export` bulk, no profile validation.
- FHIR R4 tested; other versions may work (the component only relies on Bundle structure) but are not supported.
- No typed per-resource column mapping: the resource is delivered as raw JSON by design.

## Versioning note

The `VERSION` attribute inside `tFHIRInput_java.xml` is constrained by the Studio's EMF loader to a plain decimal (`1.0`, `1.1`, ...). Semantic versioning of this project lives in Git tags and releases.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
