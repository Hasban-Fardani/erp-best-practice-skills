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
once (retry, double-click, network timeout) must be protected with an
**idempotency key**: the client generates a UUID before submitting, the server
records the key with the result, and a repeat request returns the stored
result without re-executing.

Operations that require it: payroll processing, journal commits, invoice
creation, payment webhook handling, and any status update that triggers
financial side effects.

Full implementation (pseudocode, key lifetime by caller, key-source selection,
row-level lock vs unique-constraint variants, stack notes for `SELECT FOR
UPDATE`): [[concurrency]] § Idempotency. Apply that pattern; this file owns
the *which financial operations require it*, [[concurrency]] owns the *how*.

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

**Stack notes — transaction syntax:**
- **Node.js (Knex, Prisma, TypeORM):** `transaction(callback)`.
- **Python (Django):** `transaction.atomic()` decorator / context manager.
- **Python (SQLAlchemy):** `Session.begin()` context manager.
- **Ruby on Rails:** `ActiveRecord::Base.transaction { ... }`.
- **Java (Spring):** `@Transactional` on the service method.
- **Go (`database/sql`):** `db.BeginTx()` + explicit `Commit`/`Rollback`. Use
  `defer tx.Rollback()` so rollback fires on early return.
- **C# / .NET (EF Core):** `using var tx = db.Database.BeginTransaction()`.
- **PHP (Laravel):** `DB::transaction(function () { ... })`.

## Audit Trail: Financial Mutation Rules

Severity: **CRITICAL.** Every financial change must produce an audit record
inside the **same database transaction** as the data change. If the
transaction rolls back, the audit record must roll back too — silently
losing audit on rollback is one of the most common failures in audit
libraries that default to async writes.

Financial-specific requirements (in addition to the general audit pattern
in [[observability]] § Mutation History):

- **Inside the same transaction as the data change.** Verify your audit
  library writes synchronously — many default to async/out-of-transaction.
- **Append-only at the database level.** Application role has `INSERT` only —
  revoke `UPDATE` and `DELETE` via GRANT. Application code alone is not enough.
- **User identity by snapshot, not foreign key.** Users get deactivated,
  renamed, or reassigned. The audit must remain interpretable years later.
- **Before AND after state.** "Status changed to Approved" without the
  previous value is useless when reconstructing.
- **Money values: never log to console or error tracker.** Audit table only.

For the full audit event structure, ORM diff helpers, library options by
stack, and operator-facing history panel pattern: [[observability]] §
Mutation History.

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
