# ERP Principles: Do & Don't

> **Scope.** Owns: top-level do/don't statements — discovery before code,
> design-for-least-technical-user, role-based access intent, audit-trail
> principle, DB-level validation, soft release, anti-feature-creep, filter
> persistence, loading-feedback principle. This file is the *principle*
> layer; implementation belongs to the linked specialist files. **See also:**
> linked files inline (each principle ends with the file that owns its
> implementation).

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

**Audit trail from day one.** Severity: CRITICAL for financial mutations, HIGH for the rest.
Log every meaningful action: create, update, approve, reject, delete.
Store: who, when, what changed (before and after values — not just the new value).
Retrofitting audit trails later is expensive and unreliable — the gap period
can never be reconstructed.

Pick a library that writes inside your DB transaction (so a rollback discards
the audit too). Full audit-event structure, ORM diff helpers, library options,
and operator-facing history panel: [[observability]] § Mutation History.
Financial-specific audit rules (append-only at DB level, money never logged):
[[money-and-data-integrity]] § Audit Trail.

**Validate at the database level.**
Foreign keys, unique constraints, and check constraints must be in migrations.
Don't rely only on application-layer validation — someone will bypass the form
via the API, the admin shell, a background job, or a hand-written SQL query.
Application validation is for friendly error messages; DB constraints are for
data integrity.

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
This is the most common unspoken frustration in enterprise apps. Severity: HIGH.

Implementation options (any framework):
- **URL query params** (preferred): filter state lives in the URL, survives
  reloads, bookmarkable, back/forward "just works." E.g. `/invoices?status=draft&date_from=2026-03-01`.
- **Session/server-side state**: per-user filter memory across pages, survives
  reloads. Use when filters are too complex for URLs or are sensitive.
- **Local/session storage**: client-only filter memory. Survives reloads, not
  shareable, can leak across users on shared machines.

Most table libraries and admin frameworks expose a one-line config for this
(Filament `->persistFiltersInSession()`, TanStack Table state in URL,
Django admin `list_filter` with `?` URL, Rails Ransack with permanent params).
Use the built-in option when available — hand-rolled often misses edge cases
(pagination interaction, multi-tab consistency).

Release checklist for any list or table page:
- [ ] Active filters are visible as chips or filled filter fields
- [ ] Filters survive navigating to a record and pressing Back
- [ ] Filter state survives a page refresh (URL params or session)
- [ ] "Clear all filters" button is visible when filters are active
- [ ] Empty state message reflects the currently active filter context

**Don't hardcode business logic that changes.**
Tax rates, discounts, payment terms, approval thresholds — all configurable.
See [[forms]] § Tax & Dynamic Calculation Fields for the UI pattern and
[[money-and-data-integrity]] § Rates Must Come From the Database for storage.

**Don't do big-bang migration.**
Shutting down all manual processes on day one leaves no fallback.
Run the new system in parallel with the existing process until the team is confident.

**Don't skip technical documentation.**
ERD + main business flow descriptions + notes on non-obvious architecture decisions.
Not for the client — for yourself 6 months later and for any new developer.

## Loading & Feedback States

Every server round-trip must have visible feedback. Severity: HIGH (silent UI
causes panic-clicking and duplicate submissions).

| State | UI behavior |
|---|---|
| Saving / submitting | Button disabled + spinner immediately |
| Long-running task (PDF, export) | Progress indicator or "Processing…" state |
| Success / error feedback | See [[navigation]] → Toast Notifications |

✗ Button does nothing for 3 seconds — Sri clicks it again (double submit)
✓ Button immediately goes disabled and shows a spinner

Visual disable is not enough. Pair with server-side idempotency on financial
operations — see [[concurrency]]. For which indicator to use at which
duration (skeleton vs spinner vs progress vs background-job handoff):
[[performance]] § Loading State Thresholds. For the operator-psychology
view (why waiting feels unsafe): [[human-factors]] § Panic Clicking.
