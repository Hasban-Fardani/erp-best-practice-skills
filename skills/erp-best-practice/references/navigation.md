# Navigation Patterns: Modal, Drawer, Full Page

## Decision Tree

```
Destructive action or needs confirmation?         → Modal
User needs background data as reference?          → Drawer
Complex, multi-step, or has sub-navigation?       → Full Page
Task repeated many times daily, friction minimal? → Inline edit
Short form (≤ 5 fields), no reference needed?     → Modal
Medium form (4–10 fields), list context helps?    → Drawer
Everything else?                                  → Full Page
```

## Modal

Use for:
- Confirmation dialogs: delete, cancel, approve, override
- Short input: 1–3 fields that need no external reference
- File / image preview
- Single-question prompts (rejection reason, comment before approval)

Do not use for:
- Forms with more than 5 fields
- Tables or data grids
- Multi-step wizards
- A modal opened from inside another modal
- Error messages (use inline or toast instead)
- Feature announcements or onboarding
- Auto-triggered modals without a user action

Anatomy of a correct modal:
```
┌──────────────────────────────────────────────┐
│  Delete Quotation #PEN-2024-001?          [X] │
│                                               │
│  This action cannot be undone. The record     │
│  will be permanently removed.                 │
│                                               │
│              [Cancel]  [Yes, Delete ←primary] │
└──────────────────────────────────────────────┘
```

- Specific title: `"Delete Quotation #PEN-2024-001?"` not `"Confirm"`
- Max 2–3 lines of body text
- Specific primary action: `"Yes, Delete"` not `"OK"`
- Always a secondary escape: Cancel button + X button
- ESC key closes the modal
- Clicking the backdrop closes only if there are no unsaved changes

Modal sizing:

| Content | Width |
|---|---|
| Simple confirmation | 400–480px |
| Form 3–5 fields | 520–600px |
| Form with a small table | Use drawer instead |
| Complex content | Use full page |

## Drawer / Side Panel

Best for sub-tasks too complex for a modal but not needing full page navigation.
Pak Hendra can read a detail drawer on the right while keeping the list visible on the left.

Use for:
- Viewing record detail while keeping the list visible
- Edit form (4–10 fields) where surrounding data may be referenced
- Filter & sort panel
- Quick document preview before approving

Sizing:

| Use Case | Width |
|---|---|
| Simple detail view | 360–480px |
| Edit form 4–8 fields | 480–560px |
| Form with sub-sections | 600–680px |
| More than that | → Full Page |

Rules:
- Always show a close button (X) at the top right
- Clicking the overlay closes only if there are no unsaved changes
- If unsaved changes exist, trigger the unsaved changes confirmation first
- Title must be specific: `"Quotation #PEN-001 Detail"` not just `"Detail"`
- On mobile: drawer becomes a full-screen bottom sheet

## Full Page

Use for:
- Long forms: quotations, contracts, RAB, invoices
- Onboarding master data (new employee, new customer)
- Pages with their own tab navigation
- Approval flows with history, timeline, and comments
- Multi-step wizards (10+ fields)

After submitting a new record: navigate to the **detail page** of that record.
Include a "Back to list" link.

✗ Silent redirect to list — Sri doesn't know if her save actually worked
✓ Land on `Quotation #PEN-2024-001` with a success toast

## Primary-Detail (Master-Detail) Layout

For wide screens (≥ 1024px) where users frequently navigate a list and inspect items:
show list on the left (~60%) and detail panel on the right (~40%).

Works well for: approval queues, document lists, employee records.
On smaller screens, collapse the detail panel to full-screen.

## Toast Notifications

| Type | Dismiss behavior | Severity if violated |
|---|---|---|
| Success | Auto-dismiss after 3–4 seconds | MEDIUM |
| Info | Auto-dismiss after 4–5 seconds | PREFERENCE |
| Warning | Manual dismiss only | HIGH |
| Error | **Manual dismiss only** | HIGH |

✗ Error toast auto-dismisses — Sri misses it and doesn't know her action failed
✓ Error toast stays until she reads and dismisses it

For trust-building microcopy and the full operator-psychology model:
[[human-factors]] § Trust-Building Microcopy.

## Cross-References

- Confirmation modal anatomy for destructive actions: [[exceptions-and-recovery]] § Accidental Submission
- Confirmation fatigue (don't confirm everything): [[human-factors]]
- Modal/drawer behavior under network failure: [[offline-and-network]]
- Mobile alternatives (bottom sheet, full-screen overlay): [[mobile-erp]]
