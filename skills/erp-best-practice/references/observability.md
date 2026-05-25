# Observability & Traceability

> **Scope.** Owns: audit-event structure with before/after state, ORM diff
> helpers, off-the-shelf audit libraries by stack, correlation IDs, document
> lineage, operator-facing history panels, replayability, financial
> reconciliation visibility, log vs audit distinction, never-log PII list.
> **See also:** [[money-and-data-integrity]] for financial-specific audit
> rules (append-only at DB level, transaction co-commit) · [[concurrency]]
> for idempotency-key generation · [[indonesia-compliance]] for PII redaction
> requirements.

When finance asks "why is the March payroll Rp 4.500.000 short?", you need to
reconstruct the state of every input, every rate, every rounding decision, and
every operator action across multiple subsystems. If you cannot do this in under
an hour, your observability is insufficient.

ERP observability is not application metrics (latency, error rate). It is
**business observability**: who did what, to which record, with what inputs,
producing what result, and how to replay it.

## The Four Questions

Every observable system should answer these for any business event:

1. **Who** triggered it (user, system job, external webhook — with identity captured)?
2. **When** did it happen (server timestamp, not client)?
3. **What** changed (before state, after state, inputs used)?
4. **Why** did it happen (the business reason — approval, retry, scheduled run, manual override)?

If your audit log answers 1, 2, 3 but not 4, half your reconciliation questions
will still require asking the operator. Capture the *reason* as a structured
field, not buried in a comment.

## Audit Lineage

Every financial document has a lineage — the chain of upstream events that
produced its current state.

```
Quotation QUOT-001 (created by Rina, 2026-03-01)
  └── Approved by Pak Hendra, 2026-03-02
       └── Converted to Sales Order SO-001, 2026-03-03
            └── Approved by Pak Hendra, 2026-03-04
                 └── Shipment SHIP-001 created, 2026-03-05
                      └── Stock decremented (50 units, ref: SHIP-001)
                 └── Invoice INV-001 issued, 2026-03-05
                      └── Journal JE-001 posted
                      └── Email sent to customer@example.com (msg-id: abc123)
            └── Payment PAY-001 received, 2026-03-10
                 └── Journal JE-002 posted (matched to INV-001)
                 └── Invoice INV-001 status → Paid
```

Lineage requirements:
- Every document references its parent(s) by ID (foreign key, not free-text)
- Every state transition writes an audit event with parent + child IDs
- Reports can render the lineage backward (from invoice → quotation) and forward
  (from quotation → all downstream)
- Reversal documents preserve lineage (see [[exceptions-and-recovery]])

## Mutation History

Audit events capture not just "this changed" but the full before/after state of
the changed fields.

```
// Pseudocode — emit one audit event per mutation, inside the same transaction
audit_event.insert({
    user_id:            user.id,
    user_name_snapshot: user.name,   // snapshot — user could be renamed/deleted later
    user_role_snapshot: user.role,
    action:             'UPDATE',
    resource_type:      'invoice',
    resource_id:        invoice.id,
    resource_key:       invoice.number,        // human-readable for reports
    before_state:       previous_values,       // JSON of what the row was
    after_state:        new_values,            // JSON of what the row became
    changed_fields:     diff_keys,             // ["amount", "tax_rate"]
    reason:             request.body.change_reason,
    reason_category:    request.body.change_category,
    correlation_id:     request.headers['X-Correlation-ID'],
    idempotency_key:    request.headers['Idempotency-Key'],
    occurred_at:        now_utc(),             // server time, UTC
    ip_address:         request.client_ip,
    user_agent:         request.headers['User-Agent'],
})
```

**Getting the before/after diff in your ORM:**
- **SQLAlchemy:** `inspect(instance).attrs.<field>.history` per attribute, or
  hook into `before_update` event.
- **Django:** override `save()` and read `self._loaded_values` (set in `from_db`),
  or use `django-simple-history` / `django-auditlog`.
- **Rails (ActiveRecord):** `record.changes` and `record.previous_changes` after save.
- **Hibernate / JPA:** Envers (`@Audited`) writes shadow tables automatically.
- **Sequelize / TypeORM:** `instance._previousDataValues` (Sequelize) or
  `subscribers` with `beforeUpdate` hooks (TypeORM).
- **Prisma:** no built-in diff — read the current row before update, compare
  manually. Or use `prisma-extension-audit`.
- **EF Core:** `ChangeTracker.Entries()` exposes `OriginalValues` and `CurrentValues`.
- **Eloquent:** `$model->getOriginal()`, `$model->getDirty()`, `$model->getChanges()`.

**Off-the-shelf audit libraries by stack** (use one before writing your own):
- **Node / TypeScript:** Prisma + a manual `audit_log` table; `objection-db-errors`.
- **Python (Django):** `django-simple-history`, `django-auditlog`.
- **Python (SQLAlchemy):** `sqlalchemy-continuum`.
- **Ruby on Rails:** `paper_trail`, `audited`.
- **Java (Spring):** `Spring Data Envers` (Hibernate Envers).
- **C# (.NET):** `Audit.NET`, EF Core change tracker.
- **Go:** typically hand-rolled — keep the audit insert inside the same `Tx`.
- **PHP (Laravel):** `spatie/laravel-activitylog`, `owen-it/laravel-auditing`.

**Critical for financial code:** verify the library writes inside your
transaction. Many default to async / out-of-transaction writes, which
silently lose audit records on transaction rollback.

Rules:
- The audit table is **append-only**. Application role has `INSERT` only —
  no `UPDATE`, no `DELETE`. Enforce at the database level, not just the application.
- Capture user identity by snapshot, not just FK. Users get deactivated, renamed,
  reassigned. The audit record must remain interpretable years later.
- Store the **before AND after** state, not just the new value. "Status changed
  to Approved" without the previous status is useless when reconstructing.
- For redacted or sensitive fields (NIK, salary), audit captures *that* the field
  changed and by whom, but the values are encrypted or masked per [[indonesia-compliance]].

## Correlation IDs

A single business action often spans many systems: HTTP request → controller →
service → DB → queue → background job → external API → webhook callback → DB write.

Without a correlation ID threaded through, debugging "why did this fail" means
guessing which log lines belong together.

Pattern:
- Generate a UUID at the entry point (HTTP middleware, queue listener entry)
- Pass it to every downstream call (HTTP header `X-Correlation-ID`, queue job
  payload, log context)
- Every log line includes it
- Every audit event includes it
- Every external API call passes it on (if the target system supports correlation
  IDs) or stores it in the local outbound record

```
// Pseudocode — request middleware
function correlationMiddleware(request, next):
    correlationId = request.headers['X-Correlation-ID'] ?? uuid_v4()

    // Bind to the logging context for this request
    logger.bind_context({ correlation_id: correlationId })

    // Forward to downstream calls and to the response
    request.headers['X-Correlation-ID'] = correlationId
    response = next(request)
    response.headers['X-Correlation-ID'] = correlationId

    return response
```

**Stack notes — binding correlation IDs to logs and async work:**
- **Node.js:** `async_hooks` / `AsyncLocalStorage` (built in) gives per-request
  context across async/await boundaries. Libraries: `pino-http`, `cls-hooked`.
- **Python:** `contextvars` (3.7+) for async-safe context. Libraries: `structlog`,
  `loguru`, Django's `request_id` middleware.
- **Ruby:** `Thread.current[:correlation_id]` for sync, or
  `ActiveSupport::CurrentAttributes`. Rails has built-in `X-Request-Id`.
- **Java:** SLF4J `MDC` (Mapped Diagnostic Context). For reactive code,
  `Project Reactor` `Context`.
- **Go:** `context.Context` carrying the correlation ID. Pass it to every call
  including DB and HTTP clients.
- **C# / .NET:** `Activity.Current` (built-in W3C trace context) or
  `AsyncLocal<T>`.
- **PHP:** request-scoped container binding (Laravel: `app()->instance()`),
  pass explicitly to queued jobs (job payloads must be serializable).

For background jobs: the correlation ID must travel in the job payload, not
just the request context. The worker re-binds it on dequeue.

Six months later when an operator says "the system charged the customer twice
on March 5th, here's the screenshot," you have the correlation ID from the
screenshot's request, and one query returns every related log line, audit event,
and downstream API call.

## Event Tracing for Background Work

Every background job records:

| Field | Why |
|---|---|
| Job class + parameters | Reproduce locally if needed |
| Started at, completed at | Calculate duration, detect runaway jobs |
| Status (queued, running, succeeded, failed) | Operator-facing job dashboard |
| Attempt number | Distinguish first run from retry |
| Last error (if failed) | Diagnose without grepping logs |
| Triggered by (user ID, scheduled, webhook source) | Who/what scheduled this |
| Correlation ID | Tie back to the originating action |
| Idempotency key | Verify the job ran exactly once |

Operator-facing surface: a "Background Jobs" page filtered to *their* records.
"My export is still processing" should be answerable from the UI, not by
asking the developer.

## Replayability

For incident response, you must be able to:

1. Reconstruct the state of a record at any past timestamp
2. Replay a calculation with the exact inputs and rates that were in effect at the time
3. Re-process a failed webhook from the stored payload

This requires:
- **Snapshotted rates** at calculation time (see [[money-and-data-integrity]])
- **Raw payload storage** for every external event (webhooks, file uploads, imports)
- **Append-only audit** with before/after state (above)
- **No destructive deletes** on transactional data (soft delete with timestamp)

When you cannot replay, every reconciliation question becomes a deposition: a
chain of "I think it was..." answers. Replayability is what turns deposition
into proof.

## Financial Reconciliation Visibility

Finance does reconciliation monthly. They need answers to:

1. What did we receive from each payment provider this month? (Per-provider settlement report)
2. Does the sum of journal entries match the sum of source documents? (Invoice vs journal)
3. Are there orphaned payments? (Received but unmatched to an invoice)
4. Are there orphaned invoices? (Issued but no payment expected — voided, written off)
5. Did every webhook callback produce exactly one journal entry? (No duplicates, no missing)

Build dashboards for these questions before finance asks. The first month-end
close after launch will surface gaps you did not anticipate.

Minimum dashboards:
- Daily journal balance check (total debits = total credits per period)
- Outstanding invoices aged (current, 30, 60, 90+ days)
- Unmatched payments queue (received but not yet allocated)
- Webhook event log (received, processed, failed, with retry status)
- Background job log (succeeded, running, failed — last 7 days)
- Audit event search by user, by resource, by date range

## What to Log vs What to Audit

These are different:

| Property | Log | Audit |
|---|---|---|
| Purpose | Debugging, monitoring | Business accountability, legal evidence |
| Retention | Days to weeks | Years (regulatory minimums apply) |
| Access | Engineers | Finance, compliance, auditors (read), engineers (read for support) |
| Mutability | Can be rotated/dropped | Append-only, never deleted |
| PII handling | Scrub before write | Encrypt sensitive fields, retain |
| Storage | Log aggregator (ELK, Loki, CloudWatch) | DB or compliance-grade storage |

A common mistake: putting financial audit data in the application log. When the
log rotation policy deletes 30 days of data, you have lost legally required
records. Audit is a database table, not a log file.

## Never Log This

Per [[indonesia-compliance]] and general security hygiene:

- NIK, NPWP, KK numbers
- Full bank account numbers (last 4 digits only, if any)
- Passwords, tokens, API keys (even hashed — many hashers are reversible at this size)
- Money amounts tied to a person's name (PII linkage)
- Full request bodies of HR or payroll endpoints
- Full webhook payloads in plaintext logs (store the raw payload in DB instead,
  log only the correlation ID and event type)

If you must include a sensitive value in a log for short-term debugging, scrub
before write — never rely on "we'll redact during ingest."

## Operator-Facing Audit Surface

The audit data exists for users too, not only for developers. Build:

- **Per-record history panel.** On any document, "History" tab shows the audit
  trail with who, when, what changed, and reason. Pak Hendra checks this when
  he sees an unexpected change.
- **Per-user activity log.** For HR / compliance: "Show me everything Rina did
  between March 1 and March 15."
- **Diff view.** Side-by-side before/after for edits. Far more useful than a
  JSON dump of changed fields.

Operators trust systems they can see into. An invisible audit trail is a
half-built audit trail.

## Pre-Release Checklist: Observability

- [ ] Every financial mutation produces an audit event in the same DB transaction
- [ ] Audit table is INSERT-only at the database level (revoked UPDATE, DELETE)
- [ ] User identity captured by snapshot (name, role, ID at the time)
- [ ] Before and after state stored on UPDATE actions
- [ ] Reason field captured for non-routine actions (overrides, reversals, edits)
- [ ] Correlation ID generated at entry, propagated through logs + audit + downstream calls
- [ ] Raw payload stored for every external event (webhook, import, file upload)
- [ ] Document lineage navigable in both directions (parents and descendants)
- [ ] Operator-facing history panel exists on every transactional document
- [ ] Background job state visible to the operator who triggered it
- [ ] Financial reconciliation dashboards exist before finance asks
- [ ] No PII or financial values in application logs

## Cross-References

- Concurrency, idempotency keys, retry semantics: [[concurrency]]
- Money handling, transactions, rate snapshots: [[money-and-data-integrity]]
- PII classification, encryption, what not to log: [[indonesia-compliance]]
- Exception handling and emergency edits that need extra audit: [[exceptions-and-recovery]]
