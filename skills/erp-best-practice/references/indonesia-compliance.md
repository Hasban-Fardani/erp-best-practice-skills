# Indonesia Compliance: IDR, NIK/NPWP, BPJS, Timezone, UU PDP

Rules based on regulations current as of March 2026.
Always verify against the latest regulation before implementing.

## IDR Display Format

Indonesian Rupiah uses period as thousands separator, never comma.

```
Rp 4.000.000   ✓
Rp 4,000,000   ✗
```

```php
// Laravel
number_format($amount / 100, 0, ',', '.');
// → "4.000.000" then prepend "Rp "

// Full: "Rp 4.000.000"
```

```typescript
new Intl.NumberFormat('id-ID', {
    style: 'currency', currency: 'IDR',
    minimumFractionDigits: 0,
}).format(amount / 100)
// → "Rp 4.000.000"
```

Compact display for tables/badges:
```typescript
new Intl.NumberFormat('id-ID', { notation: 'compact', style: 'currency', currency: 'IDR' })
    .format(amount / 100)
// → "Rp 4 jt"
```

## Timezone: UTC in Backend, Local in Frontend

Store all timestamps as UTC. Display converted to the user's timezone.
Never store a timestamp without timezone information.

```php
// Database: always timestamptz (PostgreSQL) or datetime with UTC offset
// Store
$document->contract_date = now()->utc();

// Display — convert to user's local timezone
$document->contract_date->setTimezone($user->timezone);
```

```typescript
const TIMEZONES = {
    WIB:  'Asia/Jakarta',   // UTC+7 — Java, Sumatra, West & Central Kalimantan
    WITA: 'Asia/Makassar',  // UTC+8 — Bali, NTB, NTT, East & South Kalimantan
    WIT:  'Asia/Jayapura',  // UTC+9 — Maluku, Papua
};

// "Payroll period March" means:
// 2026-03-01 00:00:00 WIB → stored as 2026-02-28 17:00:00 UTC
// 2026-03-31 23:59:59 WIB → stored as 2026-03-31 16:59:59 UTC
```

## NIK (Nomor Induk Kependudukan)

16-digit number. Not random — encodes province, city, and birth date.
Female: digits 7–8 (date portion) are increased by 40.

Validation:
```php
function validateNik(string $nik): array {
    if (!preg_match('/^\d{16}$/', $nik)) {
        return ['valid' => false,
                'error' => 'NIK harus 16 digit angka. Contoh: 3175012345678901'];
    }
    return ['valid' => true];
}
```

UU PDP No. 27/2022 — NIK is sensitive personal data:
- Must be encrypted in the database (deterministic encryption so it can be queried)
- Must never be sent to external AI providers
- Must never appear in logs, console output, or error tracking
- Access must be role-restricted and logged in the audit trail

```php
// Deterministic encryption: same input → same ciphertext (queryable)
$encryptedNik = encrypt_deterministic($nik, config('app.nik_key'));
Employee::where('nik_encrypted', $encryptedNik)->first();
```

## NPWP (Nomor Pokok Wajib Pajak)

Two valid formats:
- Old (15 digits): `XX.XXX.XXX.X-XXX.XXX`
- New (16 digits, NIK-based, effective 2024): same as NIK for individual taxpayers

```php
function validateNpwp(string $npwp): array {
    $cleaned = preg_replace('/[.\-]/', '', $npwp);
    if (!in_array(strlen($cleaned), [15, 16])) {
        return ['valid' => false,
                'error' => 'NPWP harus 15 atau 16 digit. Format lama: XX.XXX.XXX.X-XXX.XXX'];
    }
    return ['valid' => true];
}
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

```php
// All from config — rates and caps
$rates    = RegulationConfig::current();
$capKes   = $rates->bpjs_kes_salary_cap;      // e.g. 1_200_000_00 (in sen)
$baseKes  = min($gajiPokok, $capKes);
$potongan = (int) round($baseKes * $rates->bpjs_kes_employee_rate);
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

- [ ] IDR amounts display with period as thousands separator (Rp 4.000.000)
- [ ] All timestamps stored as UTC, displayed in user's timezone (WIB/WITA/WIT)
- [ ] NIK validated as 16 digits before saving
- [ ] NPWP validated as 15 or 16 digits before saving
- [ ] NIK and NPWP encrypted (deterministic) in database
- [ ] No NIK/NPWP values in logs, console, or error tracking
- [ ] BPJS rates and caps read from config table, not hardcoded
- [ ] PPh 21 manual field present in payroll form
- [ ] Data classification verified before sending anything to external services
