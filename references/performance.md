# Performance Reality

> **Scope.** Owns: data-volume budget, server-side pagination strategy
> (cursor vs offset), indexing discipline, query patterns to avoid,
> loading-state duration thresholds, debounce timing, background exports,
> reporting pre-aggregation, low-end-device constraints, performance metrics.
> **See also:** [[table-design]] § Pagination for the visual UI of pagination
> · [[erp-principles]] § Loading & Feedback States for the principle that
> every round-trip needs feedback · [[concurrency]] for lock-contention
> patterns · [[offline-and-network]] for network-induced latency.

Most ERP performance problems are not solved by faster servers. They are caused
by treating data sets as small in the design phase and discovering at year two
that "All Invoices" has 240,000 rows, the export button times out, and the
search box freezes the browser.

Performance work in ERPs is a data-volume budget, not a benchmark.

## Volume Reality

By year two of any active Indonesian SMB ERP:

| Entity | Typical volume |
|---|---|
| Customers / vendors | 500 – 5,000 |
| Products / SKUs | 1,000 – 50,000 |
| Invoices / quotations | 30,000 – 300,000 |
| Stock movements | 200,000 – 2,000,000 |
| Journal entries | 500,000 – 5,000,000 |
| Audit events | 1,000,000 – 50,000,000 |
| Payroll lines | 10,000 – 100,000 per year |

Designs that work on the demo database (50 rows) collapse here. Assume the
upper bound. The cost of designing for 100k rows from day one is small; the
cost of retrofitting pagination, indexing, and async exports under production
load is enormous.

## Pagination Doctrine (Server Strategy)

This section owns the **server-side strategy**: which query pattern, what page
size, when to switch from offset to cursor. Visual presentation (page-number
control, total-count display, when infinite scroll is and isn't acceptable)
belongs to [[table-design]] § Pagination.

**Default: server-side pagination on every list.**

| Volume in the list | Pattern |
|---|---|
| < 200 rows (rare and bounded — e.g. departments) | Load all, client-side filter/sort |
| 200 – 10,000 rows | Server-side pagination, default 25–50 per page |
| 10,000 – 1,000,000 rows | Server-side pagination + cursor-based (not offset) for deep pages |
| > 1,000,000 rows | Server-side cursor pagination + mandatory filters (date range required) |

❌ BAD: `SELECT * FROM invoices ORDER BY created_at DESC LIMIT 25 OFFSET 200000`
   — offset pagination scans 200,025 rows to return 25.

✅ GOOD: `SELECT * FROM invoices WHERE created_at < ? ORDER BY created_at DESC LIMIT 25`
   — cursor scans from the cursor position, returns 25.

## Indexing Discipline

Every list and search needs an index. Most ERP performance incidents trace to
a missing index on a foreign key or a filter column.

Minimum index coverage:
- Every foreign key column
- Every column used in a WHERE clause on a list query
- Every column used for sorting (`created_at DESC` needs a descending index in some DBs)
- Composite indexes on common filter combinations (e.g., `(customer_id, created_at)` for "this customer's recent invoices")
- Unique constraints on natural business keys (invoice number, NIK, NPWP)

After launch, monitor slow query logs weekly. Add indexes based on real queries,
not anticipated ones. Premature indexing wastes write performance and disk.

## Query Patterns to Avoid

❌ **N+1 in list rendering.** The classic. Each row triggers a query for related data.
   On a 50-row page that's 51 queries instead of 1.
   Fix: eager load (`with()`, `JOIN`, `selectRelated`).

❌ **SELECT *** on wide tables. Pulls back BLOBs, encrypted fields, audit JSON
   when the list only needs name and status.
   Fix: explicit `SELECT id, name, status, created_at`.

❌ **Count queries on huge tables.** `SELECT COUNT(*) FROM journal_entries` on
   a 5M row table can take seconds. Especially bad when run on every page load
   for pagination display.
   Fix: cache the count (5-minute TTL), or use estimated count from `pg_class`
   on PostgreSQL, or show "showing 1–25 of many" instead of exact count for
   deep tables.

❌ **Sorting without an index.** `ORDER BY` on a non-indexed column with > 10k
   rows = full table scan + filesort.
   Fix: index the sort column, or restrict sort options to indexed columns only.

❌ **Searching with leading wildcard.** `WHERE name LIKE '%search%'` cannot use
   a normal index.
   Fix: trigram index (PostgreSQL `pg_trgm`), full-text search (`tsvector`),
   or restrict search to prefix matching (`'search%'`).

❌ **JSON queries without indexes.** `WHERE metadata->>'invoice_type' = 'X'`
   on a million-row table is slow without a generated column or JSON index.
   Fix: extract frequently-queried JSON fields into real columns, or add a
   GIN/expression index.

## Loading State Thresholds (by Duration)

The *requirement* that every server round-trip needs visible feedback lives in
[[erp-principles]] § Loading & Feedback States. This section owns the
**duration-to-feedback-type** mapping — what kind of indicator to use depending
on how long the operation takes.

| Duration | Feedback |
|---|---|
| < 200ms | None (perceived as instant) |
| 200ms – 1s | Subtle spinner or skeleton |
| 1s – 5s | Spinner with explanatory text ("Loading invoices...") |
| 5s – 30s | Progress indicator if measurable; otherwise spinner + "This may take a moment" |
| > 30s | Move to background job with persistent status (see Exports below) |

Anti-pattern: blank screen while data loads. Skeleton rows or shimmer placeholders
give Sri something to look at and signal "the system is working."

## Debounce Strategy

| Input type | Debounce |
|---|---|
| Search box (free typing) | 250–400ms after last keystroke |
| Filter dropdown change | 0ms (immediate) |
| Resize / window events | 100–200ms |
| Autocomplete fetch | 250ms |
| Save form (autosave) | 60s interval, plus on blur of significant fields |
| Real-time validation (free) | 0ms (on blur) — not while typing |

Anti-pattern: debouncing user clicks on buttons. A click is intentional;
debouncing it adds perceived lag.

## Exports

Exports are where ERPs go to die. Sri clicks "Export to Excel" on a list of
80,000 invoices, the browser tab hangs, the server runs out of memory, and the
file never arrives.

**Pattern: every export > 1,000 rows is a background job.**

```
1. Click "Export"
2. Modal: select columns, filters, format (CSV, XLSX, PDF)
3. Click "Generate" → job enqueued, modal closes
4. Toast: "Export job started. We'll notify you when it's ready."
5. Notification (in-app + optional email) when complete with download link
6. Download link valid for 7 days, then auto-deleted
```

Background export benefits:
- Server can process row-by-row, never holding the whole result in memory
- File can be uploaded to object storage (S3, GCS), not held on the app server
- Failures are recoverable (retry the job, not the operator's entire session)
- Operator can continue working — no blocked tab

For small exports (< 1,000 rows, < 5 seconds): synchronous download is fine.
Show a spinner during generation.

## Reporting Queries

Reports are often the slowest queries in an ERP. They aggregate across multiple
years, multiple entities, multiple statuses.

Strategy:

1. **Pre-aggregate when possible.** Daily / monthly summary tables, updated by
   scheduled job or trigger. Reports query the summary, not the raw transactions.
2. **Read replica for reports.** Long-running report queries should not contend
   with transactional writes. Send reports to a read replica.
3. **Materialized views.** For complex aggregations that change less often,
   refresh on a schedule.
4. **Async report generation.** Same pattern as exports: enqueue, notify on
   completion.

Anti-pattern: real-time dashboards that query raw transactional tables on every
page load. Cache the dashboard (5–15 minute TTL) or pre-aggregate.

## Low-End Office Devices

Indonesian SMB offices commonly run:
- 4–8 GB RAM
- Integrated graphics
- Chrome with 15+ tabs open
- Antivirus with browser extension scanning
- 1366×768 monitors

UI implications:
- Avoid heavy client-side data processing (10k-row client-side sort = freeze)
- Limit DOM size (paginate, virtualize long lists)
- Lazy-load heavy widgets (charts below the fold)
- Avoid memory-leaky patterns (event listeners not removed on unmount,
  large arrays held in closure scope)
- Test on a 4GB-RAM laptop with the dev tools throttled, not on your M3 Max

Frontend bundle size:
- Aim < 500 KB gzipped for the initial bundle
- Code-split heavy modules (payroll, reporting) — they don't need to load on the dashboard
- Avoid moment.js (use date-fns or native Intl), avoid lodash full import

## Network Realism

See [[offline-and-network]] for full doctrine. Performance overlaps:

- Office WiFi commonly delivers 5–20 Mbps with 100–300ms latency to JKT data centers
- Mobile data (warehouse staff): 3G fallback is real, 100–800ms latency
- Every API call costs perceptible time; batch where possible
- Compress responses (gzip/brotli) — saves 60–80% on JSON

## Frontend Rendering Patterns

For tables with > 100 rows visible:
- Use virtual scrolling (only render rows in viewport)
- Defer rendering of off-screen complex cells (status badges with icons)

For forms with > 30 fields:
- Multi-step (see [[forms]])
- Or sectioned with collapsible panels — initial render only the first section

For dashboards with multiple charts:
- Lazy-load below-the-fold charts
- Defer heavy chart libraries until needed (dynamic import)
- Avoid auto-refresh polling unless the data genuinely changes by the second

## Measuring Performance

Track and alert on:

| Metric | Target | Alert threshold |
|---|---|---|
| API p50 response time | < 200ms | > 500ms |
| API p95 response time | < 800ms | > 2s |
| List page first paint | < 1s | > 2.5s |
| Form submit success | < 500ms | > 2s |
| Background job duration p95 | (depends on job) | 2x typical |
| Database connection pool usage | < 60% | > 80% |
| Slow query log entries | < 10/day | > 100/day |

Production-grade ERPs have a "Performance" dashboard that surfaces these.
Without it, slow pages get attributed to "the server" and the actual cause
(a missing index, an N+1 query) is invisible.

## Pre-Release Checklist: Performance

- [ ] Every list query uses an index for filtering and sorting
- [ ] Every list page has server-side pagination
- [ ] Foreign keys are indexed
- [ ] No N+1 queries on list pages (eager load relationships)
- [ ] Counts on huge tables are cached or estimated
- [ ] Exports > 1,000 rows are background jobs with notification
- [ ] Reports use pre-aggregation or a read replica
- [ ] Initial JS bundle < 500 KB gzipped
- [ ] Heavy modules code-split
- [ ] Tested on a 4 GB RAM laptop with throttled network
- [ ] Slow query log monitoring is active and reviewed weekly
- [ ] Loading states present for any operation > 200ms

## Cross-References

- Concurrency under load (locks, retries): [[concurrency]]
- Background job observability: [[observability]]
- Offline / network handling: [[offline-and-network]]
- Table pagination and rendering: [[table-design]]
