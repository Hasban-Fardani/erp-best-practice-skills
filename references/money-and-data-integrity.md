# Money Handling & Data Integrity

## Money: Always Integer (Smallest Unit)

Never store money as float or decimal. Always use integer in the smallest unit.
For IDR: store in **sen** (1 Rupiah = 100 sen). Use `BIGINT` in the database.

```
// Why floats break financial software:
0.1 + 0.2 === 0.30000000000000004  // true in JavaScript
// Multiply across 100 employees × 12 months = unexplainable discrepancies in reports
```

Rules:
- Database column: `BIGINT` / `integer` — never `DECIMAL`, `FLOAT`, or `NUMERIC`
- User inputs `4000000` (Rupiah) → multiply by 100 before saving
- Display to user → divide by 100 → format with locale formatter
- Every multiplication with a rate (tax, BPJS, discount) → `Math.round()` / `round()` immediately

```php
// Laravel — store
$gajiPokok = (int) ($request->gaji_pokok * 100);  // Rp 4.000.000 → 400000000

// Display
number_format($gajiPokok / 100, 0, ',', '.');  // → "4.000.000"

// BPJS calculation — always round immediately
$potonganBpjs = (int) round($gajiPokok * $rates->bpjs_employee_rate);
```

```typescript
// TypeScript
const gajiPokok = 4_000_000 * 100;           // store
const display = gajiPokok / 100;             // display
const potongan = Math.round(gajiPokok * 0.02); // calc
```

## Rates Must Come From the Database, Not Code

Tax rates, BPJS percentages, overtime multipliers, and discount limits change
with government regulations. Hardcoding them creates silent bugs when rates change.

```php
// Wrong — breaks when PPN changes
$ppn = $subtotal * 0.11;

// Right — from config table, no deployment needed
$ppn = (int) round($subtotal * $settings->ppn_rate);

// Best — per document, preserves historical rate
$ppn = (int) round($subtotal * $document->tax_rate);
```

Always store the rate at the time of document creation — not a live reference.
An invoice created when PPN was 11% must always show 11%, even if the rate later changes to 12%.

## Idempotency: Financial Operations Must Not Duplicate

Any financial operation that can be called more than once (due to retry, double-click,
network timeout) must be protected with an **idempotency key**.

Operations that require idempotency: payroll processing, journal commits, invoice creation,
payment webhook handling, status updates that trigger financial side effects.

Pattern:
1. Client generates a UUID before submitting
2. Server checks if that key was already processed → return the same result, don't re-execute
3. If not seen before → execute inside a transaction → save key + result atomically

```php
// Laravel
DB::transaction(function () use ($idempotencyKey, $params) {
    $existing = PayrollIdempotency::where('key', $idempotencyKey)->first();
    if ($existing) return $existing->result;  // already done

    $result = $this->doProcessPayroll($params);

    PayrollIdempotency::create(['key' => $idempotencyKey, 'result' => $result]);
    return $result;
});
```

## Database Transactions: All or Nothing

Every operation touching more than one financial table must be inside a single transaction.
No exceptions.

```php
// Right
DB::transaction(function () {
    Payroll::create($payrollData);
    Journal::create($journalData);
    JournalLine::insert($lines);
    // If any fails → all rollback
});

// Wrong — partial failure leaves inconsistent data
Payroll::create($payrollData);
Journal::create($journalData);  // if this fails, payroll row is already committed
```

## Audit Trail: Append-Only, Never Deleted

Every financial change must produce an audit record:
- Who changed it (user ID + name snapshot — snapshot because users can be deleted)
- When (server timestamp, not client)
- What changed (before and after values, not just the new value)

The audit table is append-only. The application role must have only `INSERT` permission on it —
no `UPDATE`, no `DELETE`.

```php
// Inside the same transaction as the data change
AuditEvent::create([
    'user_id'            => $userId,
    'user_name_snapshot' => $userName,
    'action'             => 'UPDATE',
    'resource_type'      => 'payroll',
    'resource_id'        => $payrollId,
    'before_state'       => $currentData,
    'after_state'        => $newData,
    'occurred_at'        => now(),  // server time
]);
```

The audit trail answers "who changed this and when?" — it does not guarantee the correctness
of the data. That is the user's responsibility. The platform's responsibility is that
the audit record itself is accurate and unmodifiable.

## Double-Entry Balance: Debit Must Equal Credit

For any ERP with accounting/journal functionality:

```
SUM(debit lines) must equal SUM(credit lines)
```

Validate before committing. If not balanced → route to suspense account, not reject the operation.
Blocking the user's operation because of a journal imbalance freezes their workflow.

```php
$totalDebit  = collect($lines)->where('position', 'debit')->sum('amount');
$totalCredit = collect($lines)->where('position', 'credit')->sum('amount');

if ($totalDebit !== $totalCredit) {
    // Route difference to suspense — don't throw an exception that blocks the user
    $this->routeToSuspense($lines, 'JOURNAL_IMBALANCE');
}
```

## Pre-Release Checklist: Financial Code

- [ ] Money stored as integer (sen/smallest unit), not float or string
- [ ] Every rate multiplication uses `round()` immediately
- [ ] Multi-table operations wrapped in a single transaction
- [ ] Idempotency key checked before executing financial operations
- [ ] Audit event created inside the same transaction as the data change
- [ ] Rates (tax, BPJS, discounts) read from config/database, not hardcoded
- [ ] Historical documents store their rate at creation time
- [ ] No money values logged to console or error tracking (PII risk)
