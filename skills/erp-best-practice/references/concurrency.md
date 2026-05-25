# Concurrency Doctrine

> **Scope.** Owns: idempotency pattern, optimistic and pessimistic locking,
> race-condition primitives, document-number assignment, webhook deduplication,
> distributed/named locks, retry-safe operation design.
> **See also:** [[money-and-data-integrity]] for which financial operations
> require these primitives · [[observability]] for correlation IDs and audit
> of concurrent operations · [[offline-and-network]] for client-side retry
> semantics · [[exceptions-and-recovery]] for accidental-submission UX.

ERP systems handle simultaneous operations from real humans (Sri and Rina hit
Save on the same record at the same moment) and from systems (a payment gateway
retries a webhook three times in 200ms because the first response timed out).

If concurrency is not handled deliberately, the failure modes are: duplicate
invoices, wrong stock counts, double-billed customers, corrupted journal entries,
and no good way to figure out which run was the real one.

## Core Principles

1. **Identify which operations are concurrency-sensitive.** Not all are.
2. **Pick the right primitive for each one** (idempotency, optimistic locking,
   pessimistic locking, sequence reservation, queue serialization).
3. **Make conflicts visible.** A silent overwrite is worse than a noisy reject.
4. **Make retries safe.** Anything that can be called twice should produce the
   same result the second time, or refuse with a clear reason.

## Operations That Require Concurrency Defense

| Operation | Why | Primitive |
|---|---|---|
| Invoice / quotation number assignment | Two operators hit Save → same number issued | DB sequence or row-level lock on a counter |
| Stock decrement on shipment | Two pickers ship from the same item, stock goes negative | Row-level lock (`SELECT FOR UPDATE`) + check |
| Payment webhook ingestion | Provider retries on timeout | Idempotency key from provider |
| Payroll batch run | Operator clicks Run twice during slow load | Idempotency key + job lock |
| Approval action | Two approvers click Approve in parallel | Optimistic version check |
| Edit on shared master record | Sri and Rina edit the same customer | Optimistic version check |
| Bank reconciliation matching | Two operators match the same transaction | Pessimistic lock during match session |
| Journal posting from invoice approval | Approve clicked twice; one approval, one duplicate journal | Idempotency on the approval, transaction on the journal |

## Idempotency

**Definition:** an operation is idempotent if running it N times produces the same
result as running it once.

**Mechanism:** client generates a unique key (UUID) before the request. Server
records the key with the result. Subsequent requests with the same key return
the stored result without re-executing.

```
// Server pseudocode — financial operation
function processPayroll(idempotencyKey, params):
    begin_transaction:
        // Lock the idempotency row so two concurrent requests with the
        // same key cannot both pass the "not yet processed" check
        existing = idempotency_store.lookup(idempotencyKey, lock=true)

        if existing:
            return existing.response          // already done

        result = do_process(params)

        idempotency_store.insert({
            key:         idempotencyKey,
            endpoint:    'POST /payroll/process',
            response:    result,
            user_id:     current_user.id,
            created_at:  now_utc(),
        })
        return result
    commit
```

```
// Client pseudocode — generate key once, retry safely
idempotencyKey = uuid_v4()

function submit():
    response = http_post('/api/payroll/process',
        headers = { 'Idempotency-Key': idempotencyKey },
        body    = params)
    return response.body

// If the network fails halfway, retry with the SAME key — safe.
// Never regenerate the key on retry.
```

**Stack notes — row-level locks for the idempotency check:**
- **PostgreSQL / MySQL (raw):** `SELECT ... FOR UPDATE`.
- **Node + Prisma:** `prisma.$transaction([ ... ], { isolationLevel: 'Serializable' })`
  or raw `SELECT ... FOR UPDATE` via `$queryRaw`.
- **Node + Knex / TypeORM:** `.forUpdate()` on the query builder.
- **Python + Django:** `Model.objects.select_for_update().get(...)` inside
  `transaction.atomic()`.
- **Python + SQLAlchemy:** `session.query(Model).with_for_update().filter(...)`.
- **Ruby on Rails:** `Model.lock.find_by(...)` inside a `transaction` block.
- **Java + Spring Data JPA:** `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the
  repository method.
- **Go + database/sql:** `tx.QueryRow("SELECT ... FOR UPDATE", ...)`.
- **C# + EF Core:** raw SQL `FromSqlRaw("SELECT ... FOR UPDATE", ...)`, or
  use `LOCK TABLE` explicitly.
- **PHP + Laravel/Eloquent:** `Model::where(...)->lockForUpdate()->first()`.

**Alternative: a unique constraint on the idempotency key** plus an `INSERT
... ON CONFLICT DO NOTHING` (PG) / `INSERT IGNORE` (MySQL) followed by reading
back the stored response. Slightly simpler, equivalent safety.

### Idempotency Key Lifetime

Keys are not garbage. Keep them for at least the retry window of every system
that calls you:

| Caller | Minimum retention |
|---|---|
| Browser (user double-click) | 24 hours |
| Internal background jobs | 7 days |
| External payment webhook (Midtrans, Xendit) | 30 days |
| Reconciliation / refund flows | 90 days |

Store with an expiry column. Prune in batches at night, not inline on insert.

### What to Use as the Key

| Source | Good for |
|---|---|
| Client-generated UUID | Browser submissions, mobile app |
| Provider-supplied transaction ID | Payment webhooks (Midtrans `order_id`, Xendit `external_id`) |
| Natural business key | Already-unique identifiers (invoice number once issued) |
| Hash of request body | Last resort — fragile if any field changes |

Do not use timestamps. Do not use auto-increment IDs. The key must be generated
*before* the operation is attempted.

## Race Conditions in Numbering

Document numbers (INV-2026-001, QUOT-2026-042) must be unique and ideally
sequential without gaps for accounting reasons.

**Anti-pattern (broken in every language):**
```
// BROKEN under concurrency — classic read-then-write race
last = invoices.order_by_desc('id').first()
next = (last?.number_seq ?? 0) + 1
invoices.insert({ number: "INV-2026-" + next, ... })
// Two requests run this simultaneously → both compute next = 43
// Both insert INV-2026-043 → unique constraint violation,
// or worse: no constraint and silent duplicates
```

**Pattern A — DB sequence (default, recommended):**
```sql
-- PostgreSQL
CREATE SEQUENCE invoice_number_seq START 1;
SELECT nextval('invoice_number_seq');  -- atomic, never returns the same value twice

-- MySQL 8+
-- Sequences are not native. Use AUTO_INCREMENT on a dedicated counter table,
-- or use the locked-counter pattern below.

-- SQL Server
CREATE SEQUENCE invoice_number_seq START WITH 1;
SELECT NEXT VALUE FOR invoice_number_seq;

-- Oracle
CREATE SEQUENCE invoice_number_seq START WITH 1;
SELECT invoice_number_seq.NEXTVAL FROM DUAL;
```
Gaps occur if the surrounding transaction rolls back — the sequence has already
incremented. Acceptable for most use cases. Inform finance: "sequence reflects
attempts, not completions; gaps are normal and auditable."

**Pattern B — Row-level locked counter (works on any DB):**
```
// Pseudocode
begin_transaction:
    counter = number_counter
        .where('key', 'invoice_2026')
        .for_update()
        .first()

    next = counter.value + 1
    counter.update(value=next)
    return next
commit
```
Slower under high concurrency (serializes). Safe and gap-free if the surrounding
transaction commits.

**Pattern C — Reservation queue:**
A background worker reserves blocks of 100 numbers. Clients draw from the
reserved block. Fast under load, requires reconciliation if blocks are
partially used.

Choose A by default. B if you cannot use sequences (legacy MySQL, restricted
DDL). C only at high write volume (> 100 invoices/min) or where DB sequences
are unavailable.

## Stock Mutation Conflicts

Two warehouse operators ship from the same SKU at the same moment. Both see
"stock: 5". Both ship 3. Naive code writes 2, then 2, leaving the system at 2
when reality is -1.

**Pattern:**
```
// Pseudocode
function decrementStock(skuId, qty, shipmentId):
    begin_transaction:
        stock = stock_table
            .where('sku_id', skuId)
            .for_update()           // pessimistic lock — second request waits
            .first()

        if stock.quantity < qty:
            raise InsufficientStock(
                available = stock.quantity,
                requested = qty)

        stock.update(quantity = stock.quantity - qty)

        stock_movement.insert({
            sku_id:      skuId,
            change:      -qty,
            reason:      'shipment',
            reference:   shipmentId,
            occurred_at: now_utc(),
        })
    commit
```

Anti-patterns:
- `UPDATE stock SET quantity = quantity - ? WHERE sku_id = ?` **without** the
  `SELECT FOR UPDATE` first — atomic at the row level for the decrement, but
  loses the chance to validate `quantity >= qty` before decrementing → negative
  stock.
- Decrementing without an immutable movement log — no audit when reconciling.
- Allowing negative stock silently — finance discovers it three months later.

Alternative (safer in some DBs): `UPDATE stock SET quantity = quantity - ?
WHERE sku_id = ? AND quantity >= ?` — atomic check-and-decrement. Returns
`rows_affected = 0` if insufficient stock, which the application interprets
as failure. Still write the movement log on success.

For high-volume goods (FMCG warehouses): consider reserving stock at the start
of the picking workflow, not at shipment.

## Payment Callback Retries

Payment gateways retry webhooks aggressively. Midtrans retries up to 5 times
over 24 hours. Xendit retries with exponential backoff. A naive handler that
records each retry as a separate payment doubles or triples the customer's
ledger.

**Required pattern (stack-agnostic):**

```
// Webhook endpoint pseudocode
function handleWebhook(request):
    // 1. Extract provider key — varies by provider
    providerKey = request.body.order_id          // Midtrans
    // OR     request.body.external_id           // Xendit
    // OR     request.body.id                    // Stripe-style
    // OR     request.headers['X-Provider-Event-Id']

    // 2. Verify signature BEFORE anything else
    if not verify_signature(request.raw_body, request.headers['X-Signature']):
        return http_response(401, { error: 'invalid signature' })

    // 3. Dedup + process in one transaction
    begin_transaction:
        existing = webhook_event
            .where('provider_key', providerKey)
            .for_update()
            .first()

        if existing:
            // Already processed — return 200 so provider stops retrying
            return http_response(200, { status: 'already_processed' })

        webhook_event.insert({
            provider:     'midtrans',
            provider_key: providerKey,
            payload:      request.body,           // store the raw payload
            raw_body:     request.raw_body,       // store the raw bytes too
            received_at:  now_utc(),
        })

        process_payment(request.body)
    commit

    return http_response(200, { status: 'processed' })
```

**Stack notes for signature verification:**
- Use the provider's official SDK if available — they handle constant-time
  comparison and the specific HMAC algorithm correctly.
- If hand-rolling: use a constant-time string compare (`crypto.timingSafeEqual`
  in Node, `hmac.compare_digest` in Python, `subtle.ConstantTimeCompare` in Go,
  `MessageDigest.isEqual` in Java) — `==` is vulnerable to timing attacks.
- Compute HMAC over the **raw request body bytes**, not the parsed JSON.
  Re-serializing changes byte order and breaks the signature.

Rules:
- Always verify signature first. Always.
- Always return 200 on duplicate (so provider stops retrying), not 4xx or 5xx.
- Always wrap dedup + processing in one transaction.
- Always store the raw payload before processing, even if you parse it elsewhere.
  When reconciliation fails six months later, you need the original bytes.

## Optimistic Locking for Shared Records

Sri opens the customer record at 10:00. Rina opens it at 10:01. Rina saves at
10:05. Sri saves at 10:07 — overwriting Rina's changes without knowing.

**Pattern:** every editable record carries a `version` (integer, increments on
each update). The form sends the version it loaded. The server rejects if the
DB version no longer matches.

```sql
-- Migration (any DB)
ALTER TABLE customer ADD COLUMN version INTEGER NOT NULL DEFAULT 1;
```

```
// Update pseudocode — conditional update on version
rows_affected = execute_sql("""
    UPDATE customer
    SET name = ?, email = ?, ..., version = version + 1
    WHERE id = ? AND version = ?
""", new_name, new_email, ..., id, expected_version)

if rows_affected == 0:
    // Either the record was deleted, or its version changed → 409 Conflict
    return http_response(409, {
        error:   'conflict',
        message: 'This record was modified by another user. Reload to see the latest version.',
    })
```

**Stack notes:**
- Many ORMs support this natively: SQLAlchemy (`version_id_col`), Hibernate /
  JPA (`@Version`), Doctrine (`@Version`), EF Core (`[ConcurrencyCheck]` or
  `IsConcurrencyToken()`), Eloquent (manual or `optimistic-locking` packages).
- For ORMs without native support, the raw SQL above works everywhere.
- An alternative to a numeric `version` is a per-row `updated_at` timestamp with
  microsecond precision — works, but loses clarity on "what changed" semantics
  and is sensitive to clock skew across replicas.

UI response to 409:
- Modal: "This customer was edited by Rina at 10:05. Your changes were not saved."
- Diff view: side-by-side of "your edits" vs "current saved state"
- Options: [Discard mine] [Keep mine, overwrite hers] [Merge manually]

Never silently overwrite. Never silently discard. The operator must choose.

## Pessimistic Locking — When Optimistic Is Wrong

Use pessimistic locks (`SELECT FOR UPDATE`) when:
- The operation is short (< 1 second of work)
- Conflicts are common (high-traffic shared counter)
- Resolution after conflict is impossible or very expensive (stock, sequence)

Avoid pessimistic locks when:
- The user has the record open in a form for minutes — you'd lock everyone else out
- The work after the lock involves external API calls (lock + slow IO = disaster)
- Read-heavy workloads where most accesses are not edits

## Retry-Safe Operations

Make these idempotent or naturally retry-safe:

| Operation | Make safe by |
|---|---|
| Sending notifications | Idempotency key on the notification record |
| Generating PDFs | Idempotency key + cache by content hash |
| Posting journal entries | Idempotency key on the source transaction |
| Decrementing stock | Movement log with idempotency reference |
| Email/WhatsApp delivery | Outbox pattern: write intent to DB, worker delivers + marks done |
| Bulk import | Per-row idempotency on (file_hash, row_number) |

## Queue Serialization

Some operations cannot run in parallel for the same key:
- Two payroll runs for the same period
- Two month-end close jobs
- Two reconciliation jobs on the same bank account

Pattern: use a named lock or a queue with concurrency=1 per key.

```
// Pseudocode — distributed named lock
acquired = lock_store.acquire("payroll-run:" + period,
                              ttl_seconds   = 600,
                              wait_seconds  = 5)
if not acquired:
    raise AlreadyRunning("Another worker is processing payroll for " + period)

try:
    run_payroll(period)
finally:
    lock_store.release("payroll-run:" + period)
```

**Stack notes — distributed locks:**
- **Redis (most common):** `SET key value NX PX 600000` for acquire, Lua script
  for safe release. Or use a library: `node-redlock` (Node), `redis-lock` (Python),
  `Redlock-rb` (Ruby), `Redisson` (Java), `redsync` (Go).
- **PostgreSQL advisory locks:** `pg_try_advisory_lock(key)` / `pg_advisory_unlock(key)`.
  Free if you already use PG. Tied to the session — release on disconnect.
- **ZooKeeper / etcd:** for stricter consistency requirements in distributed
  setups. Overkill for most ERPs.
- **In-memory (single-instance) lock:** acceptable only if you have one app
  server. Breaks the moment you scale horizontally.

If another worker already holds the lock, block for 5 seconds then fail loudly.
Do not silently skip — the operator must know whether the run happened.

## Cross-Tab Conflicts

Sri opens the same quotation in two browser tabs by accident. She edits in tab A,
edits in tab B, saves both. Tab B's save wins, tab A's edits are lost.

Defenses:
- Optimistic version check (above) — tab A's save returns 409
- BroadcastChannel API to notify other tabs of save events (advanced)
- "Open in same tab" links from the list page (reduce accidental duplicates)

For high-stakes records (in-flight approval, posting in progress): consider
single-tab enforcement via a session lock.

## Failure Scenarios to Test

Before releasing any concurrency-sensitive feature, simulate:

- [ ] Two concurrent submits with the same idempotency key → one success, one returns prior result
- [ ] Two concurrent submits with different keys → both succeed, no data corruption
- [ ] Webhook delivered twice with same provider key → processed once
- [ ] Webhook delivered with invalid signature → rejected, not processed
- [ ] Optimistic update with stale version → 409, UI shows conflict resolution
- [ ] Stock decrement under contention → no negative stock, no lost decrements
- [ ] Document number assignment under contention → no duplicates
- [ ] Queue lock held → second worker blocks then fails loudly, does not silently skip
- [ ] Network timeout during save → retry produces same record, not duplicate
- [ ] Session expires mid-save → re-auth and resubmit produces one record, not two

## Cross-References

- Money operations and transaction boundaries: [[money-and-data-integrity]]
- Recovery from accidental submission and duplicate user actions: [[exceptions-and-recovery]]
- Audit trail for concurrent operations: [[observability]]
- Network-induced retries and offline behavior: [[offline-and-network]]
