# Registry Catalog and Source Ownership

Use this reference when designing or reviewing a private UI/block registry for
Laravel ERP. The registry is a discovery and source-copy system, not a runtime
dependency and not proof that every benchmark component already exists.

## Coverage is not readiness

Keep two facts separate:

1. a component name exists in an official benchmark catalog;
2. an internal item is source-owned, previewed, documented, tested, and safe to
   copy.

Use explicit status values:

- `source-owned`: an internal item can be opened and verified;
- `adapted`: a related internal pattern exists, but it is not API-equivalent;
- `planned`: backlog only;
- `reference-only`: comparison or inspiration, not a delivery promise;
- `rejected`: deliberately excluded with a recorded reason.

Never convert a benchmark count into a maturity claim. A catalog with 68 names
may still have only a small number of source-owned ERP patterns.

## Evidence contract for each source-owned item

Require one canonical item record and page containing:

- purpose, use case, and non-use case;
- preview and interactive examples on the same page;
- source tree and copy/install boundary;
- normal, loading, empty, error, disabled, responsive, keyboard, and compact
  states when relevant;
- Pest contract and browser evidence for local interaction;
- framework, adapter, version, provenance, compatibility, dependency, and
  bundle impact;
- upgrade, migration, and rollback notes.

A screenshot only proves a visible moment. It does not prove authorization,
mutation correctness, accessibility, or a valid source contract.

## Discovery and distribution

Keep one global search for human discovery. Facets are part of that search, not
a second search with the same scope. A coverage page explains benchmark status;
it does not duplicate the search bar.

For a private Laravel registry, a shadcn-shaped JSON response may reuse the
stable fields `$schema`, `name`, `title`, `description`, `type`, `dependencies`,
`registryDependencies`, `categories`, and `files`. Add ERP-specific metadata
for purpose, use case, provenance, compatibility, and tests. Keep the endpoint
behind internal access and treat it as source distribution, never as a public
URL or a runtime import.

## AI accountability

An agent may say “ready” only when the item status and evidence support it. The
agent output must name the source, version/snapshot, test result, and unknowns.
If semantic search is unavailable, say lexical fallback; do not pretend that a
meaning index exists.

Synthetic fixtures are mandatory for internal examples. Redact PII, secrets,
client identifiers, and unpublished business rules before indexing or preview.

## Slice learning log

- The official benchmark list was stored with capture date and URLs instead of
  being hardcoded only in prose.
- A daisyUI name was initially mapped to the wrong slug in the manifest; the
  syntax/test pass caught and corrected it before handoff. This is why the
  manifest must be validated as data, not trusted by visual inspection.
- The current registry has a source-owned baseline, not full parity. Planned
  entries remain visibly planned until their source and tests exist.
- Sass deprecation output from Bootstrap is a dependency maintenance warning,
  not evidence that the coverage implementation failed. Record it separately
  and do not hide it in a green build claim.
