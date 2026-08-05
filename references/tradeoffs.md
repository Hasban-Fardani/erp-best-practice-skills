# Tradeoff Doctrine

Real ERP decisions are conflicts between principles. The goal is not to satisfy
every rule, but to know which one wins and why.

## Priority Hierarchy

When principles conflict, resolve in this order. Higher numbers never override lower numbers.

1. **Financial correctness** — money figures, journal balance, tax calculations match reality.
2. **Legal & compliance** — UU PDP, tax law, labor law, audit trail completeness.
3. **Data integrity** — referential consistency, no orphaned records, transaction atomicity.
4. **User trust** — operator confidence that the system did what it said.
5. **Recoverability** — operator can undo a mistake or reconstruct lost work.
6. **Auditability** — who did what, when, with before/after state.
7. **Operational throughput** — speed of routine tasks, clicks per task.
8. **Onboarding clarity** — discoverability for new users.
9. **Visual aesthetics** — typography hierarchy, spacing, color refinement.

A choice that improves #7 by sacrificing #1 is wrong. A choice that improves #9
by sacrificing #5 is wrong. A choice that improves #5 by adding two clicks (#7)
is correct.

Severity classifications in [[severity]] map roughly to this ladder:
CRITICAL = #1–#3, HIGH = #4–#6, MEDIUM = #7–#8, PREFERENCE = #9.

## Core Tradeoffs

### Consistency vs Speed

Consistent UI patterns reduce cognitive load across the system. But a perfectly
consistent pattern is sometimes too slow for high-frequency workflows.

**Default:** Consistency wins. Use the same modal, drawer, and form patterns everywhere.

**Exception when:**
- A workflow runs 100+ times per day per operator (warehouse scanning, payment posting)
- Measured time difference is > 2 seconds per task
- The deviation is documented and isolated to one screen

**Avoid when:**
- The "speed" comes from removing confirmation on a destructive action — see [[human-factors]]
- The deviation cascades (one custom pattern justifies the next)

### Strict Validation vs Fast Drafting

Strict validation catches errors early. Strict validation also blocks Rina from
saving a half-finished quotation she wants to come back to tomorrow.

**Default:** Two-tier validation. Drafts accept incomplete data and skip business rules.
Final submission enforces everything.

**Exception when:**
- The field is structurally required (foreign key, money field that breaks calculation)
- Saving an invalid draft causes silent data corruption elsewhere
- A regulator requires field-completeness on every persisted record (rare)

**Avoid when:**
- "Strict on draft" is just developer convenience — pushes work onto the operator
- The system can't distinguish draft from final state (fix the model first)

### Auditability vs Edit Flexibility

Operators want to fix their mistakes by editing the record. Audit requires the
history to be preserved.

**Default:** Edits allowed on draft and pending states. Once approved/posted,
edits become **adjustments** (new entries that reference the original) rather
than overwrites. See [[exceptions-and-recovery]] for retroactive correction patterns.

**Exception when:**
- A specific role (finance manager, controller) is granted post-approval edit rights
  AND each edit produces a paired audit + reason field
- Pre-release/pre-distribution stage where no downstream system has consumed the data

**Avoid when:**
- "Edit" silently overwrites a posted journal entry — this destroys the audit trail
- The reason field for the edit is optional or free-text-only with no category

### Density vs Discoverability

Pak Hendra's reports page can show 50 KPIs on one screen (density) or 10 with
generous spacing (discoverability). Sri's data entry screen has the same tension.

**Default:** Density for power users on screens they see daily. Discoverability
for screens used weekly or less, and for any first-week-of-use experience.

**Exception when:**
- The screen serves both populations — provide a "compact view" toggle, default to discoverable
- A specific persona explicitly requests density (finance month-end, controller dashboard)

**Avoid when:**
- Density is achieved by removing labels or shrinking fonts below the 16px floor
- "Power user" is a guess — observe actual use before optimizing for it

### Realtime Processing vs Eventual Consistency

Stock decrement, balance update, invoice posting — should they happen in the
same request that triggered them, or in a background job?

**Default:** Synchronous for anything the operator needs to see confirmed before
the next action. Asynchronous for anything that doesn't block the next step.

| Operation | Default | Reason |
|---|---|---|
| Save quotation → assign number | Sync | Operator needs the number to communicate with the customer |
| Send PDF to customer email | Async | Email delivery is slow; success is independent of save |
| Decrement stock on shipment | Sync | Subsequent shipments depend on current stock |
| Recalculate dashboard aggregates | Async | Stale-by-seconds is acceptable; recompute on read or schedule |
| Post journal entry on invoice approval | Sync (in same DB transaction) | Journal must reflect the invoice atomically |
| Send WhatsApp notification | Async | Network-dependent, retry-safe |

**Exception when:**
- The "sync" operation depends on a slow external service (don't block the operator on a third-party API)
- A queue is genuinely unavailable in the deployment environment

**Avoid when:**
- Async is chosen to "fix" performance — investigate the slow query first
- The operator has to refresh the page to see whether the async job succeeded (see [[observability]])

### Strict Permissions vs Practical Coverage

Locked-down RBAC is safer in theory. In practice, every gap is a WhatsApp message
to the admin asking for a one-time override, which creates audit chaos.

**Default:** Permissions match real-world delegation patterns, not org-chart fantasy.
Build a "request access" workflow for genuine edge cases. Audit every grant.

**Exception when:**
- Regulatory requirement (segregation of duties for financial approvals)
- The data is so sensitive that one accidental view is a compliance incident (HR salary, NIK)

**Avoid when:**
- Permissions are so tight that operators share accounts to get work done — this is worse than open access
- Override workflow is undocumented (becomes WhatsApp + manual SQL)

## Decision Framework

When you face a tradeoff, work through these in order:

1. **Identify which principles are in conflict.** Name them, don't hand-wave.
2. **Locate them on the priority hierarchy.** The higher one wins by default.
3. **Check if the lower-priority principle is degraded badly enough to override.**
   "Slightly less aesthetic" doesn't override "operator slower by 2 clicks."
   "Operator slower by 30 seconds × 200 times/day" might override "slightly less consistent."
4. **State the tradeoff explicitly in the design note.** Future readers should know
   the choice was deliberate, not accidental.
5. **Set the severity of the loser.** If you sacrificed CRITICAL for HIGH, you made
   a mistake. If you sacrificed PREFERENCE for HIGH, you made the right call.

## Example Walkthroughs

### "Operators want the Approve button to skip confirmation."

Conflict: throughput (#7) vs recoverability (#5) vs auditability (#6).

Approve is irreversible (triggers downstream journal, notifications, locks the document).
Skipping confirmation gains ~1 second per approval. Pak Hendra approves ~30/day.
Saved time per day: 30 seconds. Cost of one accidental approval: 30 minutes
of reversal paperwork plus a confused customer.

Resolution: keep confirmation. Reduce friction by making the confirmation specific
and one-key-dismissable (Enter to confirm), not by removing it. See [[human-factors]].

### "Finance wants to edit the posted journal entry."

Conflict: edit flexibility (#7) vs auditability (#6) vs financial correctness (#1).

Direct edit on a posted journal destroys the historical record and breaks any
downstream reconciliation. But the underlying need (correcting a wrong amount)
is real.

Resolution: provide a "Reverse and Reissue" workflow. Original entry stays.
A reversing entry is posted with reason. A new correct entry is posted.
All three are linked. See [[exceptions-and-recovery]] for the pattern.

### "Marketing wants free text on the 'Industry' field."

Conflict: input speed (#7) vs report aggregability (#3 — data integrity).

Free text wins on speed for Rina. It destroys quarterly reports for Pak Hendra.

Resolution: combobox. Rina types freely, system autocompletes existing values,
"Add new" creates a structured entry. Speed preserved for her, structure
preserved for reporting. See [[input-control]].

## What This Doctrine Does Not Solve

It does not tell you when a stakeholder is wrong. It tells you which principle
to defend. The conversation with the stakeholder is a separate skill.

When you find yourself unable to resolve a tradeoff by this framework, the most
likely cause is that an unstated principle is in play (compliance constraint,
vendor lock-in, political ownership of a screen). Surface it before deciding.
