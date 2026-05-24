# Money Handling & Data Integrity

The rules in this file are the highest-stakes in the skill. Violations cause
financial loss, legal exposure, or unrecoverable corruption. Treat every rule
here as CRITICAL unless explicitly noted otherwise.

For severity classification and proportional response: [[severity]].
For concurrent-operation safety on top of these rules: [[concurrency]].
For audit and replay infrastructure: [[observability]].

Code examples in this file are stack-agnostic pseudocode. Translate the
**pattern**, not the syntax. Brief notes for major stacks appear at the end of
each section where the idiom matters.

## Money: Always Integer (Smallest Unit)

Severity: **CRITICAL.** Never store money as float or decimal. Always use
integer in the smallest unit. For IDR: store in **sen** (1 Rupiah = 100 sen).
Use a 64-bit integer column.

```
// Why floats break financial software (this is true in every language
// that uses IEEE 754 — JS, Python, Ruby, Java, C#, Go, ...):
0.1 + 0.2 !== 0.3
// Multiply across 100 employees × 12 months = unexplainable discrepancies in reports
```

Rules:
- Database column: 64-bit integer (`BIGINT` in PostgreSQL/MySQL, `INTEGER` in
  SQLite, `NUMERIC(19,0)` if your DB has no native bigint) — never `DECIMAL`,
  `FLOAT`, `NUMERIC` with scale, `MONEY`, or string.
- User inputs `4000000` (Rupiah) → multiply by 100 before saving.
- Display to user → divide by 100 → format with locale formatter.
- Every multiplication with a rate (tax, BPJS, discount) → round to integer
  **immediately** after the multiplication.

```
// Pseudocode — same pattern in every stack
gaji_pokok_sen   = to_int(user_input_rupiah * 100)        // store
display_rupiah   = format_currency(gaji_pokok_sen / 100)  // display
potongan_bpjs    = round_to_int(gaji_pokok_sen * rates.bpjs_employee_rate)
```

**Stack notes:**
- **JavaScript / TypeScript:** values fit in `Number` up to 2^53 (~9.007 quadrillion).
  For larger or for safety, use `BigInt`. Avoid `Number.prototype.toFixed` for
  storage — it returns a string with rounding behavior that differs from `Math.round`.
- **Python:** use `int`. Avoid the `decimal` module for storage (use it only
  if a regulation explicitly requires DECIMAL semantics). Avoid `float`.
- **Java / Kotlin:** use `long`. `BigDecimal` is acceptable but slower and
  serializes inconsistently across drivers.
- **C# / .NET:** use `long`. `decimal` is acceptable for in-memory work but
  store as `long` in DB to avoid driver-level conversion bugs.
- **Go:** use `int64`. Avoid `float64`.
- **Ruby:** use `Integer`. Ruby integers auto-promote to Bignum, but the DB
  column type is what matters — `BIGINT`.
- **PHP:** cast to `int`. On 32-bit systems, prefer string + GMP if any value
  could exceed `PHP_INT_MAX` (rare for IDR ≤ 9.2e18).

## Rates Must Come From the Database, Not Code

Tax rates, BPJS percentages, overtime multipliers, and discount limits change
with government regulations. Hardcoding them creates silent bugs when rates change.

```
// Wrong — breaks when PPN changes (and no one will remember why the report is off)
ppn = subtotal * 0.11

// Right — from config, no deployment needed when rate changes
ppn = round_to_int(subtotal * settings.ppn_rate)

// Best — per document, preserves historical rate
ppn = round_to_int(subtotal * document.tax_rate)
```

Always store the rate at the time of document creation — not a live reference.
An invoice created when PPN was 11% must always show 11%, even if the rate
later changes to 12%. The rate is a snapshot on the document, not a foreign key
to a mutable settings row.

## Idempotency: Financial Operations Must Not Duplicate

Severity: **CRITICAL.** Any financial operation that can be called more than
once (due to retry, double-click, network timeout) must be protected with an
**idempotency key**. For the full pattern under concurrent load, webhook
retries, and number-assignment races: [[concurrency]].

Operations that require idempotency: payroll processing, journal commits,
invoice creation, payment webhook handling, status updates that trigger
financial side effects.

Pattern:
1. Client generates a UUID before submitting.
2. Server checks if that key was already processed → return the same result,
   don't re-execute.
3. If not seen before → execute inside a transaction → save key + result
   atomically.

```
// Pseudocode — stack-agnostic
function processPayroll(idempotencyKey, params):
    begin_transaction:
        existing = idempotency_store.lookup(idempotencyKey, lock=true)
        if existing:
            return existing.response          // already done

        result = do_process_payroll(params)

        idempotency_store.insert({
            key:        idempotencyKey,
            endpoint:   'POST /payroll/process',
            response:   result,
            user_id:    current_user.id,
            created_at: now_utc(),
        })
        return result
    commit
```

**Stack notes for transactions:**
- **Node.js (Knex, Prisma, TypeORM):** all expose `transaction(callback)`.
- **Python (Django):** `transaction.atomic()` decorator/context manager.
- **Python (SQLAlchemy):** `Session.begin()` context manager.
- **Ruby on Rails:** `ActiveRecord::Base.transaction { ... }`.
- **Java (Spring):** `@Transactional` on the service method.
- **Go (database/sql):** `db.BeginTx()` + explicit `Commit`/`Rollback`. Use
  `defer tx.Rollback()` so rollback fires on early return.
- **C# (.NET / EF Core):** `using var tx = db.Database.BeginTransaction()`.
- **PHP (Laravel):** `DB::transaction(function () { ... })`.

## Database Transactions: All or Nothing

Severity: **CRITICAL.** Every operation touching more than one financial table
must be inside a single transaction. No exceptions.

```
// Right — all or nothing
begin_transaction:
    payroll.create(payrollData)
    journal.create(journalData)
    journal_line.insert_many(lines)
    // If any fails → all rollback
commit

// Wrong — partial failure leaves inconsistent data
payroll.create(payrollData)
journal.create(journalData)   // if this fails, payroll row is already committed
```

Guard against silent failure modes:
- **Auto-commit mode:** some drivers default to auto-commit. Confirm your ORM /
  driver requires explicit `begin`.
- **Long-running transactions:** keep transactions short. Don't make external
  API calls inside them — the lock window blocks other writers.
- **Nested transactions:** most DBs implement nested transactions as
  savepoints. Read your ORM's docs — silent partial commits are possible.

## Audit Trail: Append-Only, Never Deleted

Severity: **CRITICAL.** Every financial change must produce an audit record:
- Who changed it (user ID + name snapshot — snapshot because users can be
  renamed, reassigned, or deleted)
- When (server timestamp, not client)
- What changed (before and after values, not just the new value)

The audit table is append-only. The application database role must have only
`INSERT` permission on it — no `UPDATE`, no `DELETE`. Enforce at the database
level (separate role / GRANT), not just in application code.

```
// Pseudocode — inside the same transaction as the data change
audit_event.insert({
    user_id:            user.id,
    user_name_snapshot: user.name,
    user_role_snapshot: user.role,
    action:             'UPDATE',
    resource_type:      'payroll',
    resource_id:        payroll.id,
    resource_key:       payroll.number,   // human-readable for reports
    before_state:       previous_values,
    after_state:        new_values,
    changed_fields:     diff_keys,
    occurred_at:        now_utc(),        // server time, not client
    correlation_id:     request.correlation_id,
    idempotency_key:    request.idempotency_key,
})
```

The audit trail answers "who changed this and when?" — it does not guarantee
the correctness of the data. That is the user's responsibility. The platform's
responsibility is that the audit record itself is accurate and unmodifiable.

For correlation IDs, lineage, and operator-facing history panels:
[[observability]].

**Common audit-trail libraries by stack** (use one before writing your own):
- **Node / TypeScript:** Prisma + a manual `audit_log` table; or `objection-db-errors`.
- **Python (Django):** `django-simple-history`, `django-auditlog`.
- **Python (SQLAlchemy):** `sqlalchemy-continuum`.
- **Ruby on Rails:** `paper_trail`, `audited`.
- **Java (Spring):** `Spring Data Envers` (Hibernate Envers).
- **C# (.NET):** `Audit.NET`, EF Core change tracker.
- **Go:** typically hand-rolled — keep the audit insert inside the same `Tx`.
- **PHP (Laravel):** `spatie/laravel-activitylog`, `owen-it/laravel-auditing`.

In all cases: verify the library writes inside your transaction. Some default
to async / out-of-transaction writes, which silently lose audit records on
transaction rollback.

## Double-Entry Balance: Debit Must Equal Credit

For any ERP with accounting/journal functionality:

```
SUM(debit lines) must equal SUM(credit lines)
```

Validate before committing. If not balanced → route the difference to a
suspense account, do not reject the operation. Blocking the user's operation
because of a journal imbalance freezes their workflow and forces them to fix
something they may not have the context to fix.

```
// Pseudocode
total_debit  = sum(line.amount for line in lines if line.position == 'debit')
total_credit = sum(line.amount for line in lines if line.position == 'credit')

if total_debit != total_credit:
    // Route difference to suspense — don't throw an exception that blocks the user
    route_to_suspense(lines, reason='JOURNAL_IMBALANCE')
    notify_finance_async(journal_id, difference=total_debit - total_credit)
```

The suspense entry is itself an audit-trailed adjustment. Finance reviews
suspense balances during reconciliation and re-classifies entries to their
correct accounts.

## Editing Posted Financial Documents

Severity: **CRITICAL.** Posted invoices, processed payroll, and committed
journal entries must not be edited in place. The audit trail will lose
historical truth and any downstream reconciliation will silently disagree
with the system.

Use **reverse and reissue** instead — see [[exceptions-and-recovery]]
§ Retroactive Correction Pattern for the full workflow.

❌ BAD: `UPDATE invoices SET amount = ? WHERE id = ?` on a posted invoice.

✅ GOOD: Create a reversal entry for `-original_amount`, then a new invoice
   for the corrected `amount`, both referencing the original. Original status
   → `Reversed`. Reason field captured (controlled vocabulary).

## Pre-Release Checklist: Financial Code

- [ ] Money stored as integer (sen/smallest unit), not float or string [CRITICAL]
- [ ] Every rate multiplication uses `round()` to integer immediately [CRITICAL]
- [ ] Multi-table operations wrapped in a single transaction [CRITICAL]
- [ ] Idempotency key checked before executing financial operations [CRITICAL]
- [ ] Audit event created inside the same transaction as the data change [CRITICAL]
- [ ] Rates (tax, BPJS, discounts) read from config/database, not hardcoded [HIGH]
- [ ] Historical documents store their rate at creation time [CRITICAL]
- [ ] No money values logged to console or error tracking [CRITICAL — PII]
- [ ] Posted documents are not editable in place; reversal workflow exists [CRITICAL]
- [ ] Webhook handlers verify signature before processing [CRITICAL — see [[concurrency]]]
- [ ] Number assignment is race-safe (sequence or locked counter) [HIGH — see [[concurrency]]]
- [ ] Audit table has INSERT-only permission at the DB level [CRITICAL]
- [ ] Transactions do not contain external API calls (no slow IO inside locks) [HIGH]
