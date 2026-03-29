# Input Control & Contextual Help

## Free Text vs Controlled Input

If a field that feeds into reports accepts free text, reports cannot be aggregated reliably.
This is not a developer failure — it is a consequence of the design decision.

Real example: Sri types "Electrical Installation", Budi types "Elec. Install",
Ani types "installation electrical". Pak Hendra requests a report by job type →
the system cannot aggregate them. Developer gets blamed. Root cause: free text on a categorical field.

Any field that will appear in a filter, group-by, or aggregation must be controlled:

| Field | Why |
|---|---|
| Job type / category | Cannot aggregate free text |
| Document status | Only a finite set of values is valid |
| Department, division, branch | Must be consistent across all users |
| Unit of measure (pcs, kg, m) | Calculations break if inconsistent |
| Document type, transaction type | Must be filterable |
| Customer, vendor, employee name | Must relate to master data |

Free text is appropriate for: notes, work descriptions, full addresses, rejection reasons —
fields that are purely for human reading, never for filters or aggregations.

## Combobox: The Middle Ground

Users feel free because they can type. Developers get clean data.

Flow:
1. User types in the field
2. System shows matching options from master data (autocomplete)
3. No match found → show `+ Add "[typed text]" as new option`
4. New value enters master data — immediately, or after admin approval

✓ Rina can add a new customer name without leaving the form
✓ The new value becomes structured data, reportable next time
✗ Pure free text — every new entry creates noise in reports

## Choosing the Right Input Type

| Options | Use |
|---|---|
| 2–4 | Radio buttons (all visible, 1 click) |
| 5–15 | Dropdown select |
| 16+ | Searchable dropdown / autocomplete |
| Growable master data | Combobox with "Add new" |
| Numeric range (1–10) | Stepper or text input, not dropdown |
| Date | Date picker, not three separate dropdowns |

✓ Status field with 4 values → radio buttons
✗ Dropdown for "quantity 1–10" — more clicks than just typing the number

## Communicating Controlled Input to Users

To Sri: *"If you type freely, the system can't calculate the report by category
because 'Electrical Installation' and 'Elec. Install' are read as two different things.
This dropdown keeps your own reports accurate."*

To a manager who insists on free text: show a concrete example of dirty data
from a similar free-text field. If they still want it after seeing the data,
that is a business decision they must explicitly own — not a developer decision.

## Contextual Help: Reducing Support Questions

Users don't switch to a manual when stuck. They ask the nearest person who answers faster.
Dimas asks Sri. Sri WhatsApps the developer. This is natural human behavior, not laziness.

Solution: embed help in the application exactly where it's needed.
Well-implemented contextual help reduces support tickets by up to 30%.

Help hierarchy — use the lightest option that solves the problem:

**1. Permanent helper text** — small gray text below the field, always visible:
```
Contract Date  [__________]
               Format: DD/MM/YYYY. Cannot be earlier than today.
```
Filament: `->helperText('Format: DD/MM/YYYY')`
HTML: `<p class="text-sm text-gray-500 mt-1">Format: DD/MM/YYYY</p>`
React/Vue: `<small>` or helper component below the input

**2. Tooltip `?` icon** — question mark beside the label, shows on hover/click, max 2 lines:
```
Tax Rate [?]  → "Used to calculate the final invoice amount. Default from Settings."
```
Filament: `->hint('...')->hintIcon('heroicon-o-question-mark-circle')`
HTML: `[?]` button with a CSS tooltip or lightweight popover

**3. Inline validation message** — appears on blur (field loses focus), not while typing:
```
[red border] "Invalid format. Example: 021-5551234"
```
See `forms.md` for the multi-field error banner pattern (needed for long scrolled forms).

**4. Walkthrough / guided tour** — a sequence of positioned tooltips step by step.
Use only for first-time login and major new feature rollouts.
Always include a visible Skip button. Never auto-trigger again after dismissal.

## Diagnosing Repeated Support Questions

Every repeated support question signals that the UI failed to communicate.

| Question asked repeatedly | Fix in the UI |
|---|---|
| "What does the Approve button do?" | More descriptive label or tooltip |
| "Where do I start?" | Informative empty state with a clear CTA |
| "What's the date format?" | Permanent helper text below the field |
| "Is this field required?" | Consistent required/optional marking |
| "My data disappeared" | Unsaved changes warning (see `forms.md`) |
| "Why can't I type here?" | Tooltip on the controlled/disabled field explaining why |

If 3+ different users ask the same question → fix the UI, don't resend the explanation.
