# Exceptions & Recovery

> **Scope.** Owns the **HOW**: recovery hierarchy (soft delete → undo →
> admin restore → audit → typed confirm), retroactive-correction pattern
> (reverse-and-reissue), approval reversal, accidental-submission defenses,
> duplicate-request handling, browser-refresh draft recovery, session
> expiration handling, partial workflow checkpointing, emergency-edit
> doctrine. The **WHY operators behave this way** belongs to [[human-factors]].
> **See also:** [[concurrency]] § Idempotency for the server-side de-dup
> mechanism · [[navigation]] for modal anatomy · [[forms]] for unsaved-changes
> warning · [[offline-and-network]] for autosave-to-draft mechanics ·
> [[observability]] for the audit fields these recovery paths must write.

ERP systems do not run on happy paths. They run on operators correcting yesterday's
mistakes, undoing accidental clicks, recovering from network drops, and reconstructing
what was lost when a browser tab died mid-form.

Design for the chaos, not the demo.

## Recovery Hierarchy

When choosing how to handle a destructive or irreversible operation, prefer
in this order:

1. **Make it non-destructive.** Soft delete (`deleted_at` timestamp), state change
   (`status = cancelled`), draft → discarded.
2. **Make it reversible by the same operator.** Undo toast for 10 seconds.
   "Cancel" on an in-flight job.
3. **Make it reversible by a privileged operator.** Admin can restore deleted records
   within 30 days.
4. **Make it auditable and reconstructable.** Even if the action stands, the audit
   trail captures enough state to reconstruct what was there.
5. **Make it heavy and confirmed.** Typed confirmation, role check, mandatory reason.
   This is the floor, not the ceiling.

If you find yourself at level 5, ask whether levels 1–4 are genuinely impossible.
Most "must be permanent" requirements turn out to be habit, not constraint.

## Retroactive Correction Pattern

Operators will need to correct posted, approved, or distributed documents.
"Just edit it" destroys the audit trail. "Refuse to edit" pushes the work to
WhatsApp + manual journal entry.

The correct pattern is **reverse and reissue**, linked.

```
Original document (locked)         Reversal document          New document
─────────────────────              ──────────────────         ──────────────
Invoice INV-2026-001                Reversal of INV-001       INV-2026-002
Amount: Rp 5.000.000                Amount: -Rp 5.000.000     Amount: Rp 4.500.000
Status: Reversed                    Reason: Wrong qty         Status: Active
                                    By: finance@co            Refs: INV-001
```

Required fields on the reversal:
- Reason (controlled vocabulary, not free text — see [[input-control]])
- Reference to the original document
- Reference to the new document (if reissue) or null (if pure cancellation)
- Same posting period as original if within the period; current period otherwise
- Same operator authority level as the original posting

The audit trail captures all three documents. Reports filter to "active" by default
so reversed pairs net to zero without manual exclusion.

## Approval Reversal

An approval that has triggered downstream effects (locked the document, notified
the customer, posted to the journal, decremented stock) cannot be "unapproved"
silently.

Pattern:

| State of approval | Reversal mechanism |
|---|---|
| Approved but no downstream effects yet | "Withdraw approval" — state moves back to Pending, audit logged |
| Downstream effects triggered, within same session | "Cancel approval" — runs compensating actions, audit logged |
| Downstream effects triggered, downstream party notified | "Reverse and reissue" — see above |
| Approved by a different person | Original approver can revoke within 1 hour; after that, escalation only |

Never offer a single "Delete approval" button. The label must reflect the consequence.

## Accidental Submission

Sri clicks Submit twice. Rina hits Enter in a wrong field that triggers form submit.
Pak Hendra's mouse jumps and he clicks Approve instead of View.

Defenses, in order:

1. **Idempotency on the submit handler.** See [[concurrency]]. A second click cannot
   create a second record.
2. **Immediate button disable + spinner.** Visual confirmation that the first click
   was received. Prevents the retry click that would otherwise fire.
3. **Specific button labels.** "Save Quotation" is harder to confuse with "Submit
   for Approval" than two buttons both labeled "Submit." See [[forms]].
4. **Spatial separation of destructive and constructive actions.** Approve and
   Reject are 8px apart → wrong button gets clicked. Make them 24px apart minimum,
   and put Reject in a less-primary visual treatment.
5. **Undo window for non-financial actions.** 10-second toast: "Quotation deleted.
   [Undo]". Works for soft delete, archive, status change.
6. **Confirmation modal for irreversible actions.** Specific title, specific button
   label, ESC closes, no auto-action on Enter. See [[navigation]].

For high-stakes destructive actions (delete master data, reverse posted journal,
bulk operations affecting > 50 records): require typed confirmation.

```
Type the document number to confirm reversal:

  [______________]   Required: INV-2026-001

  [Cancel]  [Reverse Invoice]  ← disabled until typed value matches
```

This is friction by design. It distinguishes intent from reflex.

## Duplicate Request Handling

Network blips and retries produce duplicate submissions. The server must distinguish
"same request retried" from "two genuinely separate requests."

| Source of duplicate | Detection | Response |
|---|---|---|
| Double-click on Submit | Idempotency key in request | Return same response, do not re-execute |
| Browser auto-retry on 5xx | Idempotency key | Same as above |
| Payment webhook retried by provider | Provider's idempotency key (Xendit, Midtrans) | Look up by key, return prior result |
| User opens form, walks away, returns, submits | Optimistic version on the underlying record | Reject with "This record was changed by another user. Reload." |
| Same operator on two browser tabs | Cross-tab lock or version check | Warn before save, do not silently overwrite |

For the full pattern: [[concurrency]] § Idempotency.

## Browser Refresh / Tab Close During Form

The most common operator-side data loss event. Defenses, in order of strength:

1. **Autosave to draft** every 60 seconds for long forms. Show "Draft saved: 13:24".
   Reload restores the draft. See [[offline-and-network]].
2. **`beforeunload` warning** for any form with unsaved changes. See [[forms]].
3. **Local storage shadow** of in-progress form state (use carefully — never for
   sensitive fields like NIK, salary, bank account).
4. **Recovery prompt on next load.** "You have an unsaved draft from 13:24. [Restore] [Discard]"

The combination of autosave + reload prompt covers ~95% of real cases. Local storage
alone is insufficient (cleared by privacy modes, by other tabs, by user cleanup).

What NOT to autosave:
- Money-affecting calculations that compound on next field
- Approval state (autosaving "approved" mid-form is a CRITICAL bug)
- Fields that trigger downstream notifications when written
- PII that should not persist beyond the form's lifecycle

## Session Expiration

Sri is filling a 25-field quotation. Her session expired 8 minutes ago. She clicks
Save. The server rejects with 401. The default behavior is to redirect to login —
and she loses everything.

Defense pattern:

1. **Soft expiry warning at -2 minutes.** Toast: "Your session will expire in 2 minutes.
   [Stay signed in]" — click extends the session without leaving the form.
2. **Inline re-auth on save.** If save fails with 401, open a modal with username +
   password fields. Authenticate without page navigation. Re-submit the original
   request with the new session.
3. **Draft preservation.** Even if the operator clicks away from the re-auth modal,
   the form state is already in the draft store (above). She can return and continue.
4. **Never silently drop a save** because the session expired between form submit
   and server response. This is the worst-case data loss event and the most preventable.

For long-running operators (Sri starts at 8am, never logs out): consider longer
session windows for trusted IPs, with re-auth required for sensitive actions
(approvals, salary edits, master data changes) regardless of session age.

## Partial Workflow Completion

Multi-step workflows die in the middle:
- Step 3 of 4 in a quotation wizard, network drops
- Bulk import: 847 of 1200 rows processed, server restarts
- Payroll run: 60 of 200 employees calculated, queue worker crashes
- PDF generation for an export: 12 of 50 pages rendered, timeout

Pattern: every multi-step operation must have a **resumable checkpoint**.

| Scenario | Checkpoint mechanism |
|---|---|
| Multi-step form (wizard) | Step data autosaved per step. Resume to last completed step. |
| Bulk import | Row-level state: pending / processed / failed. Re-run resumes from pending only. |
| Payroll batch | Per-employee state. Failures isolated. Re-run skips successes. |
| Background job (PDF, report) | Job state in DB with progress %. UI polls or websocket. Restart resumes from last completed page. |

Anti-pattern: "If it fails, just rerun the whole thing." This is acceptable only
when the operation is fully idempotent AND the cost of redoing the work is
trivial (sub-second, no external calls).

## Emergency Edit Doctrine

Sometimes operators need to do things the system says they can't:
- Backdating an invoice because the customer's PO came late
- Editing an employee's salary mid-period for a retroactive raise
- Marking a shipment as delivered before the courier's webhook arrives

These are real business needs. The wrong answer is "the system doesn't allow that."
The right answer is a documented emergency path:

1. **Role gate.** Only specific roles (controller, ops manager) can take the action.
2. **Mandatory reason.** Controlled vocabulary, not free text.
3. **Heavier audit.** Captures normal audit fields plus the override category and
   the policy bypassed.
4. **Optional approval.** For the highest-stakes overrides, require a second approver.
5. **Discoverable.** Don't hide the function — operators will find a workaround
   that's worse. Surface it with a clear "Override" label.
6. **Reported.** Monthly: list of overrides by user, by category. Trend → policy gap.

Anti-pattern: building the emergency edit as a separate "admin script" that the
developer runs via SSH. This means there is no audit, no policy, and the developer
is now in the financial workflow.

## Recovery State Visibility

After any save, the operator must know:
- Was it saved? (specific success message, not generic "OK")
- Where is it now? (status badge, location in workflow)
- What changed? (highlighted fields if relevant)
- Can it be undone? (Undo link if applicable)

Anti-pattern: silent redirect to list after save. Sri does not know if the save
succeeded. She refreshes, scrolls, hunts for her record by date. See [[navigation]].

## Pre-Release Checklist: Exception Handling

- [ ] Every irreversible action has at least confirmation + audit + reason
- [ ] Every form > 5 fields has `beforeunload` warning
- [ ] Every long form (15+ fields) has autosave to draft
- [ ] Every approval action has a defined reversal path (withdraw/cancel/reverse)
- [ ] Every duplicate-prone endpoint has idempotency
- [ ] Every multi-step operation can resume from its last checkpoint
- [ ] Session expiry triggers re-auth without form data loss
- [ ] Emergency-edit paths exist for documented exception scenarios, gated by role + reason
- [ ] Soft delete used as default; hard delete is a separate, heavier action
