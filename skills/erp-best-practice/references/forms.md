# Forms: Create, Edit, Multi-Step, Validation

> **Scope.** Owns: form layout (single vs two-column), label placement,
> required/optional marking, field grouping, validation flow, error-summary
> pattern, submit-button rules, multi-step wizards, unsaved-changes warning,
> edit-vs-create differences, tab order. **See also:** [[input-control]] for
> which input type to use per field · [[navigation]] for modal/drawer/page
> decision · [[exceptions-and-recovery]] § Browser Refresh for draft autosave
> · [[money-and-data-integrity]] for rate-snapshot storage · [[concurrency]]
> § Idempotency for double-submit protection.

## Layout

**Default: single column.** Severity: MEDIUM. Two-column layouts cause Z-pattern
scanning — Sri skips fields, especially on long forms.

**Use two-column (paired fields side-by-side) when:**
- Fields are short, logically related, and visually scanned as a unit
  (City + Province + ZIP; First Name + Last Name; Date + Time)
- The form is short overall (< 8 fields total) and density helps discoverability
- A line-item table is being entered (qty + unit price + total in one row is correct)

**Avoid two-column when:**
- Either column has fields > 20 characters wide on average
- The two columns are unrelated (Name + Address Line 1 + Phone + City — wrong)
- The form is > 15 fields (Z-pattern fatigue compounds)
- Targeting mobile parity — collapses badly and creates inconsistent layout

Form container max-width: **680–760px** for single-column. Wider only with
deliberate density justification. Avoid full-width on > 1366px monitors.
Autofocus first field when the page or modal opens.

## Labels

**Rule: label above the field.** Severity: HIGH. Not inside (placeholder) or
beside it. This is one of the most common ERP failure patterns and is rarely
worth the visual savings of placeholder-as-label.

✓ Permanent label above → always visible while Sri is typing
✗ Placeholder as label → disappears on focus, Pak Hendra forgets what to fill in

**Exception (rare):** single-field search inputs ("Search invoices...") where
the field's purpose is unambiguous from context and there is no other field to
confuse it with. Even here, an above-label is fine; the savings are aesthetic.

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

Form UX rules for fields that depend on configurable rates (PPN, discount,
BPJS, DP ratio):

- Show the rate that *will be applied* as a read-only field next to the
  calculated amount: `PPN (11%)  Rp 440.000`. Operators must be able to
  verify "yes, the right rate was used" without checking settings.
- On edit of an existing document, show the rate **snapshotted at document
  creation**, not the current live rate. Label it: `PPN at creation: 11%`.
- Never let the form recalculate using a different rate than what was
  stored on the document — this silently rewrites history.
- If a rate field is editable (admin override for a specific document),
  require a reason field and audit the override.

Storage, calculation, and rate-snapshot enforcement belong to the backend —
see [[money-and-data-integrity]] § Rates Must Come From the Database, Not
Code for the pattern. For Indonesian-specific rate sources (PPN, PPh, BPJS
caps): [[indonesia-compliance]].

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
Severity: MEDIUM.

**Reset/Clear buttons: default to omitting them.** They almost never help and
they consistently hurt — Sri's most common destructive misclick is hitting
"Reset" thinking it's "Submit." Severity of inclusion: HIGH risk.

**Exception:** filter panels where "Clear filters" is the explicit, expected
action — and labeled "Clear filters", not "Reset." See [[erp-principles]] § Filter Persistence.

Disable the button after click to prevent double-submit. Show a spinner during
processing. Always pair with server-side idempotency — visual disable is not
enough. See [[concurrency]] § Idempotency.

Don't disable Save when the form has errors. Let the user click, then reveal
all errors at once. A disabled button with no explanation forces Sri to scan
the entire form to guess what's wrong.

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

Save step data to component / form state only — commit to database on the
final confirmation. Combine with draft autosave (see below) so the operator
does not lose work if a tab closes mid-wizard.

**Common form-state patterns by stack:**
- **React:** `useReducer` for multi-step state; or a form library (React Hook
  Form, Formik); or a store (Zustand, Redux Toolkit).
- **Vue:** Pinia store for cross-step state; `vee-validate` for validation.
- **Svelte:** writable store per wizard; `svelte-forms-lib`.
- **Angular:** Reactive Forms with a parent `FormGroup` per wizard.
- **Server-rendered (Rails, Django, Laravel, ASP.NET):** session-backed wizard
  state, or a `draft_state` JSON column on the target record; framework
  conventions (Rails Wicked, Django FormWizard, Filament MultiStepForm,
  ASP.NET wizard control).
- **HTMX / Hotwire:** server-side state in session or draft row; client only
  swaps partials per step.

## Unsaved Changes Warning

Trigger when the user navigates away, closes a drawer, or closes the browser tab
while the form has unsaved changes.

```
"You have unsaved changes. Are you sure you want to leave?"
[Leave]    [Stay on this page]  ← primary
```

Two layers are needed:

1. **In-app navigation blocking** — when the operator clicks a link, closes a
   drawer/modal, or uses router back. Shown as a custom confirmation dialog
   (consistent styling, "Stay" as primary action).
2. **Browser-level `beforeunload`** — when the operator closes the tab or
   refreshes. Browsers only show a generic message; you cannot style it.

```js
// Browser-level — works in every web framework
window.addEventListener('beforeunload', (e) => {
    if (formIsDirty) { e.preventDefault(); e.returnValue = ''; }
});
```

**Stack-specific helpers for in-app navigation blocking:**
- **React Router v6+:** `useBlocker(isDirty)` — render your own confirmation modal
  when `blocker.state === 'blocked'`.
- **Vue Router:** `onBeforeRouteLeave((to, from, next) => { ... })`.
- **Angular Router:** `CanDeactivate` route guard.
- **Svelte (SvelteKit):** `beforeNavigate` from `$app/navigation`.
- **Next.js (App Router):** `useRouter` + custom `Link` interception, or
  `useBlocker` shim.
- **Rails / Django / Laravel server-rendered:** combine `beforeunload` (above)
  with a confirmation modal on internal navigation actions (clicking a link
  inside the form area).
- **HTMX:** `hx-confirm` or a `htmx:beforeRequest` listener checking dirty state.
- **Filament:** `->unsavedChangesAlerts()` on the form builder.

For modals/drawers: trigger the unsaved-changes confirmation on close attempt
(X button, ESC, backdrop click) — same dialog as navigation blocking.

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
Full draft storage strategy and offline considerations: [[offline-and-network]] § Autosave to Draft.

## Tab Order & Conditional Fields

Tab order must follow visual field order, not DOM rendering order.
Sri uses Tab to move between fields — jumps to hidden fields confuse her.

Default: DOM order = visual order = tab order. Only override with explicit
`tabindex` when CSS reorders fields away from DOM order (rare in well-designed
forms — usually a smell). Avoid `tabindex` values > 0 globally; they create
unmaintainable jump-orders.

In server-rendered forms (Rails, Django, Laravel, Filament, ASP.NET): field
order in the form definition typically becomes DOM order, so visual = tab as
long as CSS doesn't rearrange.

When field B is only relevant after field A is filled:
- Show B as disabled (dimmed) — signals "waiting for input above"
- Never hide B entirely — sudden appearance confuses Dimas who reads forms top to bottom first
