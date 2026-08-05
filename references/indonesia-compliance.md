# Indonesia Compliance: IDR, NIK/NPWP, BPJS, Timezone, UU PDP

Rules based on regulations current as of March 2026.
Always verify against the latest regulation before implementing.

PII handling rules in this file are **CRITICAL** by default — violations expose
the company to UU PDP penalties (up to 2% of annual revenue per Pasal 57).
Severity per rule is noted inline; absence of a tag means CRITICAL.

## IDR Display Format

Indonesian Rupiah uses period as thousands separator, never comma.

```
Rp 4.000.000   ✓
Rp 4,000,000   ✗
```

Always format on output using a locale-aware formatter — never with manual
string manipulation. Manual formatting breaks on edge cases (negatives, zero,
very large numbers).

**Stack examples** (input: `amount_sen` = 400_000_000, i.e. Rp 4.000.000):

```javascript
// JavaScript / TypeScript / Node — built-in
new Intl.NumberFormat('id-ID', {
    style: 'currency', currency: 'IDR',
    minimumFractionDigits: 0,
}).format(amount_sen / 100)
// → "Rp 4.000.000"

// Compact form for tables/badges
new Intl.NumberFormat('id-ID', { notation: 'compact', style: 'currency', currency: 'IDR' })
    .format(amount_sen / 100)
// → "Rp 4 jt"
```

```python
# Python — using babel (pip install babel)
from babel.numbers import format_currency
format_currency(amount_sen / 100, 'IDR', locale='id_ID')
# → "Rp 4.000.000,00"  (override fraction_digits=0 if needed)
```

```ruby
# Ruby on Rails — built-in
ActionController::Base.helpers.number_to_currency(
    amount_sen / 100.0,
    unit: 'Rp ', separator: ',', delimiter: '.', precision: 0)
# → "Rp 4.000.000"
```

```java
// Java — built-in
NumberFormat fmt = NumberFormat.getCurrencyInstance(new Locale("id", "ID"));
fmt.setMaximumFractionDigits(0);
fmt.format(amount_sen / 100.0);
// → "Rp 4.000.000"
```

```csharp
// C# / .NET — built-in
(amount_sen / 100m).ToString("C0", new CultureInfo("id-ID"));
// → "Rp 4.000.000"
```

```go
// Go — golang.org/x/text/message + currency
p := message.NewPrinter(language.Make("id-ID"))
p.Sprintf("Rp %d", amount_sen / 100)
// → "Rp 4.000.000"
```

```php
// PHP — using intl extension
$fmt = new NumberFormatter('id-ID', NumberFormatter::CURRENCY);
$fmt->setAttribute(NumberFormatter::MAX_FRACTION_DIGITS, 0);
$fmt->formatCurrency($amount_sen / 100, 'IDR');
// → "Rp 4.000.000"
```

## Timezone: UTC in Backend, Local in Frontend

Store all timestamps as UTC. Display converted to the user's timezone.
Never store a timestamp without timezone information.

```
// Database column types — pick the timezone-aware variant
PostgreSQL: TIMESTAMPTZ           (recommended)
MySQL:      TIMESTAMP             (stored UTC, displayed in session TZ — convert in app)
            or DATETIME with explicit "always store UTC" application convention
SQL Server: DATETIMEOFFSET
Oracle:     TIMESTAMP WITH TIME ZONE
SQLite:     TEXT in ISO-8601 with 'Z' suffix
```

```
// Pseudocode
// Store
document.contract_date = current_time_utc()

// Display — convert to the user's timezone
display = format_in_timezone(document.contract_date, user.timezone)
```

Indonesia uses three timezones — none of them observe DST:

```
WIB  = 'Asia/Jakarta'    UTC+7 — Java, Sumatra, West & Central Kalimantan
WITA = 'Asia/Makassar'   UTC+8 — Bali, NTB, NTT, East & South Kalimantan
WIT  = 'Asia/Jayapura'   UTC+9 — Maluku, Papua
```

"Payroll period March" means a specific range in the operator's local timezone:
```
2026-03-01 00:00:00 WIB → stored as 2026-02-28 17:00:00 UTC
2026-03-31 23:59:59 WIB → stored as 2026-03-31 16:59:59 UTC
```

Period boundaries must be computed in the user's timezone, then converted to
UTC for the query. Computing in UTC first and then offsetting produces
off-by-one-day bugs on the period edges.

## NIK (Nomor Induk Kependudukan)

16-digit number. Not random — encodes province, city, and birth date.
Female: digits 7–8 (date portion) are increased by 40.

Validation:
```
// Pseudocode — single rule: 16 numeric digits
function validateNik(nik) -> { valid, error }:
    if not matches(nik, /^\d{16}$/):
        return { valid: false,
                 error: 'NIK harus 16 digit angka. Contoh: 3175012345678901' }
    return { valid: true }
```

UU PDP No. 27/2022 — NIK is sensitive personal data:
- Must be encrypted in the database (deterministic encryption so it can be queried)
- Must never be sent to external AI providers
- Must never appear in logs, console output, or error tracking
- Access must be role-restricted and logged in the audit trail

```
// Pseudocode — deterministic encryption: same input → same ciphertext (queryable)
encrypted_nik = encrypt_deterministic(nik, key = config.nik_encryption_key)
employee = employees.where('nik_encrypted', encrypted_nik).first()
```

**Stack notes — deterministic encryption:**
- **Don't roll your own AES-ECB.** ECB is deterministic but leaks block patterns.
- **Use AES-SIV** (RFC 5297) — built for deterministic-encryption-with-queryability.
  Libraries: `cryptography` (Python, `Aes.SIV`), `miscreant` (Node, Go, Java, Ruby),
  Google Tink (`DeterministicAead`).
- **Alternative for query-only use cases:** store a keyed HMAC of the NIK as a
  separate `nik_hash` column for lookups, plus the AES-GCM-encrypted real NIK
  for retrieval. Query by hash, decrypt on read. This avoids needing
  deterministic encryption entirely.
- **Rotate keys** — design for key rotation from day 1. Store `key_version`
  alongside each ciphertext so you can decrypt old values during a rotation.

## NPWP (Nomor Pokok Wajib Pajak)

Two valid formats:
- Old (15 digits): `XX.XXX.XXX.X-XXX.XXX`
- New (16 digits, NIK-based, effective 2024): same as NIK for individual taxpayers

```
// Pseudocode
function validateNpwp(npwp) -> { valid, error }:
    cleaned = remove_chars(npwp, '.-')      // strip formatting
    if length(cleaned) not in [15, 16]:
        return { valid: false,
                 error: 'NPWP harus 15 atau 16 digit. Format lama: XX.XXX.XXX.X-XXX.XXX' }
    return { valid: true }
```

Same protection rules as NIK — encrypt, restrict access, no external AI, always audit.

## BPJS Rates

Rates change with government regulation. Always store in a config table, never hardcode.

Current rates (per PP No. 44/2015 & Perpres No. 82/2018, as of March 2026):

| Component | Employee | Employer |
|---|---|---|
| BPJS TK — JHT | 2% | 3.7% |
| BPJS TK — JP | 1% | 2% |
| BPJS Kesehatan | 1% | 4% |

BPJS Kesehatan has a salary cap (Rp 12.000.000/month as of March 2026 — store in config):

```
// Pseudocode — all rates and caps from a config/regulation table
rates        = regulation_config.current()
cap_kes_sen  = rates.bpjs_kes_salary_cap        // e.g. 1_200_000_00 (in sen)
base_kes_sen = min(gaji_pokok_sen, cap_kes_sen)
potongan_sen = round_to_int(base_kes_sen * rates.bpjs_kes_employee_rate)
```

BPJS TK (JHT + JP) has no salary cap — calculated from full base salary.

## PPh 21

Phase 0 (current common practice): manual input field only.
Label: `"PPh 21 (calculated manually — refer to DJP calculator at pajak.go.id)"`
This is a legal obligation — the field must exist in payroll, even if calculation is manual.

Automated calculation requires consultation with a licensed tax consultant before implementation.

## UU PDP Data Classification

Per UU No. 27/2022 Pasal 4:

| Data | Classification | Must Encrypt | External AI |
|---|---|---|---|
| NIK | Sensitive | Yes (deterministic) | Never |
| NPWP | Sensitive | Yes (deterministic) | Never |
| Bank account number | Sensitive | Yes (AES-256) | Never |
| Salary amount | Confidential | Access control | Never |
| Employee name | Personal | Access control | Never |
| Company name | Business | Not required | Allowed (anonymized) |
| Schema structure | Technical | Not required | Allowed |

Safe to send to AI: field names and data types, validation rules, error messages,
rate values from config, anonymized examples ("a salary field stored as integer in sen").

Never send to AI: actual NIK values, salary figures, employee names, bank account numbers,
or any transaction data with real amounts.

## Pre-Release Checklist: Indonesia-Specific

- [ ] IDR amounts display with period as thousands separator (Rp 4.000.000) [HIGH]
- [ ] All timestamps stored as UTC, displayed in user's timezone (WIB/WITA/WIT) [HIGH]
- [ ] NIK validated as 16 digits before saving [HIGH]
- [ ] NPWP validated as 15 or 16 digits before saving [HIGH]
- [ ] NIK and NPWP encrypted (deterministic) in database [CRITICAL]
- [ ] No NIK/NPWP values in logs, console, or error tracking [CRITICAL]
- [ ] BPJS rates and caps read from config table, not hardcoded [HIGH]
- [ ] PPh 21 manual field present in payroll form [HIGH — legal obligation]
- [ ] Data classification verified before sending anything to external services [CRITICAL]
- [ ] Access to sensitive fields is role-restricted [CRITICAL]
- [ ] Access to sensitive fields is logged in audit trail [CRITICAL — see [[observability]]]
- [ ] Local storage / browser cache contains no sensitive PII [CRITICAL — see [[offline-and-network]]]

## Cross-References

- General audit and access-log infrastructure: [[observability]]
- PII handling in offline draft storage: [[offline-and-network]]
- Migration of historical PII data: [[legacy-and-migration]]
