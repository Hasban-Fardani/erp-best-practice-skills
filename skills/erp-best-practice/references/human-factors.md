# Human Failure Psychology

ERP operators are not the user from a usability lab. They are stressed,
multitasking, repetitive-task workers with personal stakes in the outcome
(Sri gets blamed if a customer's invoice is wrong; Pak Hendra approves things
he doesn't fully read because he has 40 to get through before lunch).

Design for the human under pressure, not for the rational maximizer.

## Operator State Model

At any moment an operator is in one of these states. The UI should support
the worst case, not assume the best.

| State | Signals | Design response |
|---|---|---|
| Focused | First hour of shift, doing routine work | Standard density, normal confirmations |
| Distracted | Phone ringing, WhatsApp open, child asks a question | Make critical actions hard to fat-finger |
| Rushed | End of day, deadline pressure | Save state aggressively; prevent destructive shortcuts |
| Frustrated | After 3+ failed attempts at the same task | Show recovery options visibly; lower the cognitive cost of "ask for help" |
| Panicked | Just clicked the wrong button | Provide undo; show that work was preserved |
| Resigned | Has stopped reading messages | Visual + textual + spatial cues for the same information |

## Panic Clicking

Symptom: button click feels unresponsive → user clicks again, harder, three times.

Causes:
- Latency between click and visible feedback > 200ms
- Spinner appears in a location the user is not looking at
- Modal opens behind another window
- Action succeeded but no confirmation message

Defenses:

1. **Instant visual feedback on click.** Button disables and shows a spinner the
   instant it's pressed — not after the server responds. See [[forms]] § Submit Button.
2. **Idempotency on submit handlers.** Even if Sri clicks 5 times, only one
   record gets created. See [[concurrency]].
3. **Optimistic UI for known-safe actions.** Status toggle, mark-as-read,
   reorder — apply the change locally first, reconcile with server.
4. **Confirmation toast for completed actions.** Even if the change was visible
   in the UI, a "Saved" toast makes it explicit. Pak Hendra checks for the toast.

❌ BAD: button has no `disabled` state, fires the same POST 5 times in 800ms,
   server creates 5 invoices with consecutive numbers.

✅ GOOD: button disables on the `mousedown` event. Server receives 5 requests
   with the same idempotency key, returns the first result 5 times. UI shows
   one invoice, one success toast.

## Confirmation Fatigue

If every action is confirmed, the operator clicks "Yes" without reading. The
confirmation has become wallpaper.

Rules:

- **Confirm only the irreversible.** Save, edit, move, sort, filter — never.
  Delete, approve, post, reverse, send — always.
- **Make the confirmation specific.** "Delete?" is wallpaper. "Delete invoice
  INV-2026-042?" is not.
- **Vary the friction by severity.** Low-stakes destructive: simple modal.
  High-stakes: typed confirmation. See [[exceptions-and-recovery]].
- **Never confirm twice for the same action.** If the user clicked "Approve" and
  confirmed, do not pop a "Are you sure you want to approve?" again.

❌ BAD: "Save changes?" modal on every form save. By week two, Sri clicks Yes
   without looking — including the day she's saving the wrong record.

✅ GOOD: Save is silent and shows a "Saved 13:24" toast. Delete shows
   "Delete quotation QUOT-001?" modal. Reverse-posted-invoice requires typing
   the invoice number.

## Distrust of Destructive Actions

Operators have been burned. They know "Delete" sometimes means "permanently gone."
They develop defensive habits: never deleting anything, archiving everything,
keeping screenshots of forms before saving.

Lean into this distrust productively:

1. **Soft delete by default.** "Delete" moves to trash. Trash has a 30-day
   retention. Restore is one click. See [[exceptions-and-recovery]].
2. **Distinguish "Archive" from "Delete."** Archive removes from common views
   but is intentionally permanent-feeling. Delete is the trash. Both are
   reversible; the difference is operator intent.
3. **Make Undo visible.** 10-second toast after any soft delete: "Quotation
   moved to trash. [Undo]" Sri uses this constantly.
4. **Heavy lift for hard delete.** Hard delete is a separate workflow — admin
   role, typed confirmation, audit reason, paired approval for high-stakes records.

## Fear of Losing Data

The single most common operator anxiety: "did my work save?"

Triggers:
- Network blip in the middle of a long form
- Browser refresh after entering 30 fields
- Boss calls while the form is open
- Closed the wrong tab

Defenses build trust by *visibly* preserving work:

1. **Autosave to draft** every 60 seconds on long forms. Show "Draft saved: 13:24"
   in a fixed position the user can see without scrolling.
2. **Unsaved changes warning** on navigation and tab close. See [[forms]].
3. **Recovery prompt on reload.** "You have an unsaved draft from 13:24.
   [Restore] [Discard]"
4. **Explicit success on save.** Specific toast: "Quotation QUOT-001 saved."
   Not "Done" or "Success."
5. **Display the saved record after creation.** Land on the detail page of the
   new record, not the list. See [[navigation]].

The trust compounds. Operators who have been rescued by autosave once will
trust the system with longer forms. Operators who have ever lost 30 fields of
work will avoid the system or screenshot every form first.

## Uncertainty After Save

Even with a success toast, operators want to *see* the saved state. They will:

- Navigate to the list, search for their record, open it, scroll through every field
- Take a screenshot "in case"
- Re-enter the form to verify

The UI should make this verification trivial:

- After create: land on the detail page, fields read like the form, with edit affordances
- After edit: stay on the detail page or return to detail, never silently redirect
- After bulk action: show a result summary ("23 invoices updated, 0 failed")
- Use the same record number/title in the toast as in the URL — the operator
  can verify by glancing at the address bar

## Repetitive Workflow Exhaustion

Sri creates 80 quotations a day. By quotation 60, her attention is in 8% mode.
Errors made in this state include:
- Selecting the wrong customer from a dropdown that auto-completed too early
- Tab-pasting a field value from her clipboard into a wrong field
- Saving without filling a required field she didn't see (because she's scrolled past it)

Designs that help repetitive operators:

1. **Sticky context.** Customer name visible at the top of every step of a long form.
   The operator can confirm "right customer" without scrolling up.
2. **Keyboard-driven.** Tab order matches visual order, Enter advances logically,
   Esc cancels. Mouse usage is a tax on speed.
3. **Defaults from the last record.** "Same customer as previous" / "Same project
   as previous" — opt-in, not silent, but available.
4. **Verification step on high-stakes fields.** Customer and amount get an extra
   moment of visual attention (highlighted on review step, summary banner).
5. **No surprise pop-ups.** Never insert a modal during the form-fill flow that
   wasn't there yesterday. Repetitive work is muscle memory; new UI breaks the
   memory.

## Confirmation Hierarchy (by Reversibility)

| Action type | Confirmation pattern |
|---|---|
| Reversible by Undo (10s window) | Toast with [Undo] link, no modal |
| Reversible by same operator within session | Lightweight inline confirm |
| Reversible by admin (soft delete, retract) | Modal with specific title, [Cancel] [Confirm] |
| Reversible only by reverse-and-reissue | Modal + typed confirmation OR modal + reason field |
| Irreversible, affects external party (sent invoice, paid out salary) | Modal + typed confirmation + reason + secondary approver if high-stakes |
| Irreversible, mass action (delete 200 records) | Multi-step flow: confirm, type confirmation, optional admin approval |

Match the friction to the consequence. Over-friction trains operators to ignore
confirmations. Under-friction guarantees a costly mistake.

## "Heavy" Actions Should Feel Heavy

The visual treatment of a destructive button should match its consequence.

❌ BAD: "Reverse Invoice" button rendered the same as "Save Draft" — same color,
   same size, same position. Operator misclicks once a month, and finance has to
   reconstruct the posting.

✅ GOOD: "Reverse Invoice" is in a separate section labeled "Danger Zone," is
   red-outlined (not red-filled — red-filled is visually loud and trains the eye
   to ignore it), requires opening an expander to access, and shows a typed
   confirmation modal.

The visual weight functions as a forcing function: the operator must consciously
"go to" the destructive action, not graze it.

## Successful Saves Should Feel Explicit and Trustworthy

✗ Silent redirect, no toast. Operator stays anxious.
✗ Generic "Done" toast. Operator can't tell what was done.
✓ "Quotation QUOT-001 saved. View [link] · Create another [link]"

For long-running operations:
✗ Click → wait 8 seconds with no feedback → page appears.
✓ Click → button disables + spinner → progress message ("Generating PDF...
   2 of 5 pages") → completion toast with link to result.

## Error Messages That Help

Errors must explain:
1. What went wrong (in business terms, not technical)
2. What to do about it
3. Where to find help if stuck

❌ BAD: "Validation failed: contract_date must be after today"
✅ GOOD: "Contract Date cannot be in the past. Please select today or a future date."

❌ BAD: "Insufficient stock"
✅ GOOD: "Only 3 units of SKU-001 are available in your warehouse. Reduce the
   quantity, or contact warehouse@company.com to check incoming stock."

❌ BAD: "Network error"
✅ GOOD: "Could not save. Check your internet connection and try again. Your
   work is preserved as a draft."

## Trust-Building Microcopy

| Context | Bad | Good |
|---|---|---|
| After save | "Done" | "Quotation QUOT-001 saved" |
| After autosave | (no message) | "Draft saved 13:24" |
| During processing | "Loading..." | "Generating PDF (3 of 5 pages)..." |
| After delete | "Deleted" | "Invoice moved to trash. [Undo]" |
| After approval | "Approved" | "Approved by Pak Hendra at 14:02. Customer will be notified." |
| On error | "Error" | "Could not save: contract date is required. [Go to field]" |
| Session warning | "Session expiring" | "Your session will expire in 2 minutes. [Stay signed in]" |
| Empty state | "No data" | "No quotations yet this month. [Create the first one]" |

The pattern: specific, action-oriented, names the artifact and the next step.

## Anti-Patterns from Real Indonesian ERPs

❌ Approve button placed next to Reject button, 8px apart, same size, same gray
   styling. Pak Hendra rejects 1 in 50 by accident.
✅ Approve is primary (filled), Reject is secondary (outlined), 24px gap, with
   Reject requiring a reason in a follow-up modal.

❌ "Save" button on a long form disables until all fields are valid. Sri can't
   tell which field is broken without scrolling.
✅ Save is always enabled. Click reveals all errors at once in a sticky banner.
   See [[forms]].

❌ Bulk delete confirmed with "Are you sure?" Sri clicks Yes without reading.
   Deleted 200 records by accident.
✅ Bulk delete shows count and sample: "Delete 200 quotations? First three:
   QUOT-001, QUOT-002, QUOT-003." Type "DELETE 200" to confirm.

❌ Toast "Saved!" then immediately disappears. Pak Hendra missed it because he
   was looking at the next field.
✅ Success toast stays 4 seconds, has a quiet color that doesn't snap attention
   away from work. Error toast stays until dismissed.

## Cross-References

- Recovery patterns and undo: [[exceptions-and-recovery]]
- Confirmation modal anatomy: [[navigation]]
- Form validation and unsaved changes: [[forms]]
- Visual weight and color: [[visual-design]]
- Severity-proportional response: [[severity]]
