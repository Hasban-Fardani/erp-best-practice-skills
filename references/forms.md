# Forms: Create, Edit, Multi-Step, Validation

## Layout

Single column always. Two-column layouts cause Z-pattern scanning — Sri skips fields.
Exception: short logically-related fields side-by-side (City + Province + ZIP).

Form container max-width: **680–760px**. Never full-width.
Autofocus first field when the page or modal opens.

## Labels

Label above the field, always. Not inside (placeholder) or beside it.

✓ Permanent label above → always visible while Sri is typing
✗ Placeholder as label → disappears on focus, Pak Hendra forgets what to fill in

Placeholder is allowed only as a format example, never as a label replacement:
```
Label:       "Contract Date"
Placeholder: "DD/MM/YYYY"
```

## Required vs Optional

- Most fields required → mark optional ones with **(Optional)**
- Most fields optional → mark required ones with **\***
- Never mix both conventions in the same form.

## Field Size Matches Expected Input

✓ "Full Name" field wider than "ZIP Code (5 digits)"
✗ All fields the same width → Sri doesn't know if she should write long or short

## Field Grouping & Order

1. Primary identity (document number, date, title)
2. Relations/references (customer, project, department)
3. Content details (items, descriptions, quantities)
4. Calculations (subtotal, tax, discount, total)
5. Notes / attachments

## Tax & Dynamic Calculation Fields

Never hardcode rates. Tax rates (PPN), discounts, and DP ratios must be configurable.

```php
// Wrong — breaks when regulations change
$ppn = $subtotal * 0.11;

// Better — from config, no deployment needed to update
$ppn = $subtotal * config('erp.ppn_rate');

// Best — from database, user-configurable per document or per period
$ppn = $subtotal * ($document->tax_rate ?? TaxRate::current()->rate);
```

Always store the rate at creation time — not a live reference.
Historical invoices must preserve their original calculation even after rate changes.
For BPJS rates and Indonesian tax specifics: see `indonesia-compliance.md`.
For money-as-integer storage pattern and rate calculation rules: see `money-and-data-integrity.md`.

## Validation

Validate on blur (when field loses focus). On submit, re-validate all fields.

### Multi-Field Error Problem

For forms with 10+ fields: red borders alone are not enough.
Sri is scrolled to the bottom — she won't scroll back up to hunt for a red field she can't see.

Solution: **error summary banner + inline errors, both at the same time**.

```
┌──────────────────────────────────────────────────┐
│ ⚠  3 fields need attention before saving:        │
│    • Contract Date — required                    │  ← clickable, scrolls to field
│    • Customer — must be selected from the list   │
│    • Item quantity — must be greater than 0      │
└──────────────────────────────────────────────────┘
```

The banner:
- Fixed/sticky at the top of the form or drawer
- Each item is a clickable link that scrolls to and focuses that field
- Inline red border + message still appears below each affected field

Auto-scroll to the first error field after submit.

✓ Sticky banner with links + inline messages below each field
✗ Only red borders — invisible when the user has scrolled past them
✗ Only a banner — user can't see which field is affected without scrolling

## Submit Button

✓ Specific label: "Save Quotation", "Create Invoice"
✗ Generic label: "Submit", "Send", "OK"

Remove Reset/Clear buttons entirely — they almost never help, they consistently hurt.
Disable the button after click to prevent double-submit. Show a spinner during processing.

Don't disable Save when the form has errors. Let the user click, then reveal all errors at once.
A disabled button with no explanation forces Sri to scan the entire form to guess what's wrong.

## Forms with 10–20+ Fields: Multi-Step

Break into steps when a form has more than ~10 fields or spans multiple logical domains.

Step structure example for a quotation:
```
Step 1: Document Info   → Number, date, customer, project
Step 2: Line Items      → Repeatable item rows
Step 3: Financial       → Tax, discount, totals, payment terms
Step 4: Review          → Read-only summary before final save
```

Progress indicator — mandatory, always visible even while scrolled:
```
● Document Info  ─  ○ Line Items  ─  ○ Financial  ─  ○ Review
```

- Completed steps: solid dot
- Current step: highlighted dot
- Future steps: gray dot (reachable, not locked)
- Allow going back to completed steps without losing data

Save step data to component state only — commit to database on the final confirmation.

Framework patterns:
- Livewire: `$this->stepData['step1']`
- React/Vue: component state or form store (Zustand, Pinia)
- Blade + Alpine: `x-data` with nested objects per step

## Unsaved Changes Warning

Trigger when the user navigates away, closes a drawer, or closes the browser tab
while the form has unsaved changes.

```
"You have unsaved changes. Are you sure you want to leave?"
[Leave]    [Stay on this page]  ← primary
```

Framework implementations:

```php
// Filament v3+
->unsavedChangesAlerts()
```

```php
// Livewire (non-Filament) — Blade + Alpine
// In Livewire component: public bool $isDirty = false;
// In Blade:
<div x-data @beforeunload.window="if ($wire.isDirty) $event.preventDefault()">
```

```js
// React — React Router v6
import { useBlocker } from 'react-router-dom';
const blocker = useBlocker(isDirty);
// Show confirm modal when blocker.state === 'blocked'
```

```js
// Vue 3 + Vue Router
onBeforeRouteLeave((to, from, next) => {
  if (isDirty.value) {
    confirm('You have unsaved changes. Leave?') ? next() : next(false);
  } else next();
});
```

```js
// Vanilla JS / plain Blade (browser-native dialog)
window.addEventListener('beforeunload', (e) => {
  if (formIsDirty) { e.preventDefault(); e.returnValue = ''; }
});
```

## Edit Form Specifics

Pre-filled data must look editable. Data in edit forms often looks static —
Sri doesn't realize she can click on it.

✓ Input fields keep a visible border and white background even when filled
✓ Cursor changes to text cursor on hover
✗ Data rendered as `<span>` or `<p>` outside an `<input>` — looks read-only

Create vs Edit differences:

| Aspect | Create | Edit |
|---|---|---|
| Initial state | Empty | Pre-filled |
| Main risk | Forgetting to fill a field | Accidentally changing a field |
| Submit label | "Save [Name]" | "Update [Name]" |
| After submit | Navigate to detail page | Stay on page or return to detail |

For single-field edits in a table (status toggle, quantity):
use inline edit — click to activate, Enter or blur to save, show "Saved ✓" for 1.5s.

For long forms (RAB, quotation with many items): auto-save to **draft** every 60 seconds.
Show: `"Draft saved: 13:24"`. Never auto-save to published/final state.

## Tab Order & Conditional Fields

Tab order must follow visual field order, not DOM rendering order.
Sri uses Tab to move between fields — jumps to hidden fields confuse her.

In React: use `tabIndex` explicitly if DOM order differs from visual order.
In Filament: field order in `->schema([])` determines tab order.

When field B is only relevant after field A is filled:
- Show B as disabled (dimmed) — signals "waiting for input above"
- Never hide B entirely — sudden appearance confuses Dimas who reads forms top to bottom first
