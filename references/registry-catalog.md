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

## Visual catalog contract

The registry homepage is a discovery surface, not a prose index. A preview card
must render a synthetic but recognizable fixture of the component or block:
buttons should look like buttons, a table should show rows, a form should show
fields and state, and a loading item should show loading. A sentence such as
“component for handling forms” is context, not a preview.

Keep the fixture visible before the explanatory copy. A detail page may add
purpose, use case, source, and test contract below the fixture, but it must not
fall back to title-plus-description when the item is a visual component. For
backend or test recipes, show a short real source/example excerpt instead of
pretending a UI fixture exists.

Benchmark inventory and internal discovery have different audiences. Store the
full inventory and provenance for maintainers, but do not expose a benchmark
matrix or competitor names in the normal office-facing catalog unless the user
is explicitly auditing coverage. The visible catalog should show the internal
component count and an honest readiness status; a count is never a claim that
every item is source-owned.

Use a visual grid for a large component inventory, one global search for title,
purpose, tags, and backend context, and category facets rather than a second
search bar. On mobile, collapse the grid to one column and keep the same fixture
above the status and source link.

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

## Safe source installation and upgrades

Every published item must expose a read-only lifecycle plan before an installer
is considered:

- `strategy`: normally `copy-with-review`;
- `review_required`: explicit boolean, defaulting to `true`;
- `overwrite`: use `never` or another explicitly approved policy, never an
  implicit overwrite;
- file paths, sizes, and SHA-256 digests;
- one source digest for the file set;
- breaking changes, migration, dependency, and rollback notes.

The plan is evidence for a human or agent to prepare and inspect a diff. It is
not permission to write into a consuming project. A future installer must add
dry-run, path/conflict checks, explicit approval, backup or reversible diff,
fixture-project tests, and a receipt before it can mutate files.

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
- An empty `breaking_changes` list is valid metadata. Validate it with
  `present|array`, not only `required|array`; a contract test caught this exact
  seed regression before the source plan could be trusted.
- A registry preview was initially implemented as descriptive copy for some
  items. Browser review found that the word “preview” was misleading; visual
  items now require a rendered fixture, while backend/test items show a real
  source excerpt. This distinction is part of the acceptance contract.
- A homepage count originally described only source records even though the
  product goal was a complete discoverable catalog. Keep inventory count and
  source readiness as separate fields so “71 catalog items” cannot be read as
  “71 production-ready items”.
- A full benchmark matrix made the internal coverage page look like a research
  report and confused office readers. Keep benchmark evidence in maintainer
  documentation and make the everyday catalog visual and concise.
