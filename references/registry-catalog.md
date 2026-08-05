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

## Catalog taxonomy is part of the user model

Expose separate first-class scopes for `component`, `layout`, `block`, and
`backend`/`test` items. Do not flatten them into one list and expect a badge or
description to explain the difference. The scope answers a different question:

- **Component** — one reusable UI primitive or field/control;
- **Layout** — page chrome or structural shell such as header, drawer, footer,
  hero, or stack;
- **Block** — a composed ERP workflow slice such as a filtered table or a long
  form section;
- **Backend** — a server-side recipe, test recipe, or operational case without
  pretending it is a visual component.

The homepage should expose these scopes as visible catalog navigation. A
component card may link to a detail page, but a layout or block must not be
silently reclassified as a component merely because it has a source file.
Source ownership is a second axis, not a replacement for taxonomy.

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

The canonical item page starts with the real rendered fixture. Do not add a
large “Preview” CTA to a page that is already the preview, and do not create
tabs that only scroll to sections while looking like separate application
states. Keep the documentation linear and let the source/test sections follow
the fixture.

If an item has more than one source layer, show each real file in a file list
with its layer (`Frontend` or `Backend`), path, syntax-highlighted contents,
and a copy action. “Source available” means files exist and still require
review; “source complete” requires the declared test evidence to be verified.
Never display a source count as if it were a source-complete count.

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

## Dynamic authoring and MCP boundary

The database/schema alone does not make a registry dynamic. A PHP seeder or
benchmark config that must be edited for every item is a baseline importer, not
an authoring system. The registry needs a draft/revision/evidence workflow that
can be called by the web UI, an API, a command, and an internal MCP facade
through the same application service.

Minimum safe authoring flow:

```text
create draft → validate → render safe fixture → attach Pest/browser evidence
→ request review → approve → publish revision → generate read-only install plan
```

MCP should be read-only by default. It may create or update a draft with an
optimistic revision check, but it must not publish or overwrite a consuming
project without explicit human approval, an auditable diff, and a receipt.
Suggested tools are search, get item, get source, create draft, update draft,
validate, attach evidence, request review, publish-after-approval, and install
plan. If a meaning index is unavailable, return lexical fallback and say so.

Do not execute arbitrary Blade supplied by AI or an author as a web preview.
Use a whitelisted declarative fixture schema, a registered renderer, or escaped
source display. This keeps manual authoring flexible without turning the
private registry into an XSS/code-execution surface.

## Interactive preview contract

Classify every item as `static`, `browser-interactive`, or `server-boundary`.
Visible controls must either work in the preview or be clearly non-actionable;
a decorative button that does nothing is a broken contract. Browser-interactive
fixtures must prove trigger/state transition, dismiss or reset, keyboard and
ARIA behavior, loading/error/success copy, no unexpected network request, and
no console error. Server-boundary fixtures may use synthetic data, but must
clearly separate local simulation from authorization, query, transaction,
mutation, and test behavior in the consuming project.

For blocks, a single input is not enough evidence. A table block should show
the workflow states relevant to its job: filters, pagination, loading,
empty/error, status/action geometry, and the server boundary. A form block
should show section navigation, error recovery, draft/conflict indication, and
the dependency rule when those are part of its purpose.

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
- A source item was first absorbed into the component scope because its source
  slug was reused as a mapping key. Explicitly exclude block-owned source from
  component mapping and test component/layout/block/backend counts separately.
- A detail page called a validation error summary “preview” even though it did
  not render the input field. The fixture now renders the actual field, label,
  invalid state, and helper/error copy; a prose error summary alone is not a
  field preview.
- A mobile browser check found that a long example path in a flex row expanded
  the page to 424px at a 390px viewport. The source panel was not the cause;
  the fix was `min-width: 0` on the example content and `overflow-wrap: anywhere`
  for paths. Treat long paths, code, and labels as mobile layout fixtures.
- A feature test reached the backend preview and exposed a missing namespace
  separator (`IlluminateSupportStr`). The browser path had not covered that
  catalog result. Keep direct feature coverage for each preview rendering type;
  a visual smoke test alone is insufficient.
- A preview can contain a real-looking `<button>`, `<input>`, or close control
  while still being a static illustration. Treat every visible control as a
  contract: either wire and test the state transition, or render it as clearly
  non-actionable content. This prevents a “preview” page from teaching a false
  interaction model.
- An empty icon button exposed an incomplete Lucide import map. Maintain an
  icon manifest and fail validation when a template requests an unregistered
  icon; a visually blank action is a usability and accessibility defect, not a
  cosmetic detail.
- A large PHP seeder plus preview mapping was initially mistaken for dynamic
  registry authoring. A database-backed read model is not enough: authors need
  draft/revision/evidence services that can be used by the UI, command, API, and
  MCP without editing application code for every new item.
- A first browser probe used a fixed short wait for a lazy-loaded modal and
  produced a false negative. Use a runtime readiness marker such as
  `window.erpUi.modal` after `networkidle` before testing the interaction; keep
  the original probe and corrected probe in the receipt so the evidence remains
  auditable.
- A browser-interactive fixture initially relied on an empty `aria-current`
  attribute. Use an explicit `aria-current="true"` only for the active item and
  remove the attribute otherwise; presence alone is not a reliable state value.
- A feature assertion initially searched for a short class substring and
  matched a legitimate data attribute. Assert the exact semantic marker when a
  test is meant to prove absence; broad substring checks create false confidence.
- A button-option CSS rule initially styled only `div` elements while the
  accessible fixture correctly used `button` elements. Style by role/fixture
  contract, not by the first element type used in the prototype.
- The first catalog service grew to 325 lines because it mixed source loading,
  item mapping, evidence status, and query presentation. Split only along those
  named responsibilities: source assembly, item mapping, and source evidence;
  keep benchmark labels and preview types in configuration so future authoring
  can replace the data layer without rewriting the presenter.
- A refactor test assumed a seeded source was complete because its preview was
  visible; the actual contract correctly classified it as partial. Readiness is
  evidence-driven, not inferred from visual availability, and tests should
  allow only the explicitly permitted evidence states.
