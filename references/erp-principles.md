# ERP Principles: Do & Don't

## The Core Test

ERP that gets used: Sri completes her daily tasks without contacting the developer.
ERP that's for show: beautiful dashboard, but Sri struggles with data entry every day
because UX was designed for what the manager wanted to see, not what she needs to do.

## Must Do

**Discovery before writing code.**
Observe the actual daily workflow of admins — not just interview managers.
Map who actually inputs data vs who only views reports.
Document the manual process (Excel, paper) that already works before replacing it.

Rina uses a Google Sheet to track prospects. Before replacing it with an ERP module,
understand what columns she uses, what she ignores, and what she adds manually.
Skip this and you'll build a module she won't use.

**Design for the least technical user.**
Every frequent task must be completable in 3 clicks or fewer.
Error messages must explain what to do, not just what went wrong.

✓ "Contract Date cannot be in the past. Please select today or a future date."
✗ "SQLSTATE[23000]: Integrity constraint violation"

**Roles and permissions based on actual access needs, not job titles.**
Ask: "Who needs access to this data, to do what specific action?"
Not: "What does this job title probably need to see?"

Dimas (HR) needs to read employee contracts but must not edit salary data.
Rina (marketing) needs to create quotations but not approve them.
Don't lump these together by department.

**Audit trail from day one.**
Log every meaningful action: create, update, approve, reject, delete.
Store: who, when, what changed (before and after values — not just the new value).
Retrofitting audit trails later is expensive and unreliable.
In Laravel: `spatie/laravel-activitylog` — install before building any CRUD.
For implementation pattern with before/after state and idempotency: see `money-and-data-integrity.md`.

**Validate at the database level.**
Foreign keys, unique constraints, and check constraints must be in migrations.
Don't rely only on Laravel validation rules — someone will bypass the form via API or Tinker.

**Soft release: one department first.**
Deploy to Sri's team. Watch where she gets stuck. Fix it. Then expand.
Never do a full rollout on day one.

## Must Not Do

**Don't build features nobody has asked for yet.**
Unused features still need maintenance and clutter the UI.
For SaaS: validate 1–2 paying clients before building a third module.

**Don't design UX based on what management wants to see.**
Pak Hendra wants animated KPI cards. Sri needs the Create form to load in under 2 seconds.
The person who uses the system most should be heard most in UX decisions.

**Don't reset filters on back navigation.**
Sri sets filters (date range, status, department) → opens a record → goes back → filters are gone.
This is the most common unspoken frustration in enterprise apps.

In Filament: `->persistFiltersInSession()`
In Laravel + Livewire: persist filter state to URL query params or session
In React/Vue SPA: store filters in route state or global store, restore on mount

**Don't hardcode business logic that changes.**
Tax rates, discounts, payment terms, approval thresholds — all configurable.
See `forms.md` for the full tax/dynamic calculation pattern.

**Don't do big-bang migration.**
Shutting down all manual processes on day one leaves no fallback.
Run the new system in parallel with the existing process until the team is confident.

**Don't skip technical documentation.**
ERD + main business flow descriptions + notes on non-obvious architecture decisions.
Not for the client — for yourself 6 months later and for any new developer.

## Loading & Feedback States

Every server round-trip must have visible feedback.

| State | UI behavior |
|---|---|
| Saving / submitting | Button disabled + spinner immediately |
| Long-running task (PDF, export) | Progress indicator or "Processing…" state |
| Success / error feedback | See `navigation.md` → Toast Notifications |

✗ Button does nothing for 3 seconds — Sri clicks it again (double submit)
✓ Button immediately goes disabled and shows a spinner

## Filter Persistence Checklist

Before releasing any list or table page:
- [ ] Active filters are visible as chips or filled filter fields
- [ ] Filters survive navigating to a record and pressing Back
- [ ] Filter state survives a page refresh (URL params or session)
- [ ] "Clear all filters" button is visible when filters are active
- [ ] Empty state message reflects the currently active filter context
