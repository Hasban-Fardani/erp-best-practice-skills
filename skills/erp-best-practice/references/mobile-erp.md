# Mobile ERP Doctrine

ERP on mobile is not "the desktop UI but smaller." It is a specific set of
operational workflows that happen away from a desk. Designing the wrong things
for mobile (a 30-field quotation form on a 5" screen) is worse than not
supporting mobile at all.

Start by identifying *which* workflows actually need mobile, not by mandating
mobile parity for everything.

## Workflows That Need Mobile

| Workflow | Why mobile | Persona |
|---|---|---|
| Warehouse picking / receiving | Operator walks the warehouse, scans SKUs | Warehouse staff |
| Stock count / opname | Physical inventory counts in aisles | Warehouse staff + auditors |
| Delivery confirmation | Driver hands over goods, captures signature/photo | Driver |
| Field sales (visit, order, collection) | Sales rep at customer site | Field sales |
| Service technician dispatch | Technician at customer site, parts used, work done | Technician |
| Approval (lightweight) | Manager approves quotation while away from desk | Pak Hendra, controller |
| Attendance check-in / check-out | Employee clocks in (often with geofence + photo) | All staff |
| Expense submission with receipt photo | Employee submits expense on the spot | All staff |

## Workflows That Do NOT Need Mobile

| Workflow | Why not |
|---|---|
| Creating a full quotation (30+ fields) | Form is too long; data entry is faster on desktop |
| Payroll processing | High-stakes, irreversible — should be done deliberately on desktop |
| Financial reports / dashboards | Better on larger screens; not time-critical |
| Master data setup (customers, products, accounts) | Bulk operation, screen real estate matters |
| Month-end close | Heavy concentration, multi-tab work |
| Bank reconciliation | Comparison-heavy, needs wide tables |
| Multi-step wizards with > 10 fields | Will be abandoned on mobile |
| Audit log review | Power-user work, desktop |

**Stating "mobile not required" is not a failure.** It is a deliberate choice
that prevents resources being spent on workflows that real operators will not
use on mobile anyway.

## Mobile-First Design for the Right Workflows

For workflows that need mobile (above), design mobile-first — meaning the mobile
flow is primary and the desktop view is the secondary projection. Otherwise the
mobile experience is always a degraded version.

### Warehouse Scanning

Critical constraints:
- One hand often holds the scanner or product
- Bright fluorescent or sunlight environments — contrast must be aggressive
- Gloves common — touch targets ≥ 48px
- Offline-frequent (warehouse RF coverage gaps)
- Errors are physical — wrong scan = wrong shipment leaves the building

Design:
- Single primary action per screen ("Scan next item")
- Numeric input large and centered, with stepper buttons (`+ -`) for adjustments
- Scan confirmation: audible beep + visual flash + haptic (if available)
- Mismatch alert: distinct sound + red banner — easy to spot in a busy aisle
- Offline queue with sync status visible (see [[offline-and-network]])
- Session persistence across app crashes — pick lists don't disappear

### Field Sales

Critical constraints:
- Variable network (3G fallback common)
- Customer is watching — perceived professionalism matters
- Long sessions (full visit ≈ 30–60 min)
- Battery: should run a full day

Design:
- Pre-cached customer + product data for offline-capable visit flow
- Quick order entry: scan product or pick from a short list, not search-heavy
- Capture customer signature with stylus or finger
- Generate visit summary as PDF on device, send via email/WhatsApp before leaving
- Submit visit when reconnected — operator sees clear sync status
- No surprise sync delays — never block "next visit" on previous sync

### Mobile Approvals

Critical constraints:
- Approver is in a meeting / lunch / car
- Sees only the document snippet (number, amount, requester)
- Cannot easily reference other documents
- One bad approval = real money lost

Design:
- Summary view: requester, amount, type, top 3 line items, attached files
- "View full detail" expands inline — does not navigate away
- Approve / Reject buttons separated visually (color, position, ≥ 24px apart)
- Reject requires reason (controlled vocabulary, not free text)
- Approve requires confirmation step for amounts above threshold
- All approvals on mobile have the same audit fields as desktop
- For very high-stakes approvals: require desktop, mobile shows "Open on desktop to approve"

### Attendance Check-In

- Geofence verification (within X meters of office)
- Optional selfie capture
- Offline-capable (record locally, sync when online — common at coverage edge)
- Late check-in handling: capture timestamp, route to manager for approval if outside window
- No silent failure: if GPS fails, present manual workflow with reason

## When Mobile Parity Is NOT Required

Be explicit about this in the design phase:

> "Quotation creation: desktop only. Mobile users see 'Open on desktop to create.'"

> "Payroll run: desktop only. Mobile users see 'Open on desktop to process.'"

> "Financial reports: desktop primary. Mobile shows summary cards only,
> not detailed tables."

This is not laziness. It is a deliberate decision to focus mobile development
on the workflows that genuinely need it, and to prevent operators from being
forced into a broken UX out of policy.

## Mobile-Specific UX Rules

| Aspect | Mobile rule |
|---|---|
| Touch target minimum | 44–48px (WCAG, plus reality of gloves and fat fingers) |
| Primary action position | Bottom of screen — reachable by thumb |
| Body font size | 16px minimum — prevents iOS auto-zoom on focus, comfortable read |
| Form layout | Single column always (no exceptions on narrow screens) |
| Modal alternative | Bottom sheet — slides up, dismissable by swipe down |
| Drawer alternative | Full-screen overlay on mobile |
| Table alternative | Card list — each row becomes a card with primary action |
| Multi-select | Long-press to enter selection mode, not always-visible checkboxes |
| Keyboard avoidance | Input fields scroll into view above keyboard, never hidden |
| Numeric input | Use `inputmode="numeric"` to trigger numeric keypad |
| Date input | Native date picker, not custom calendar (smaller, faster, accessible) |

## Mobile Performance Constraints

Mobile devices in Indonesian field operations:
- Mid-range Android (4GB RAM, 64GB storage)
- Battery often at 30–60% by midday
- Network: 4G primary, 3G fallback common
- Background apps: WhatsApp, browser, navigation

Implications:
- App / page weight matters more (slower CPU, smaller bundle budget)
- Animations: keep minimal — drain battery and feel laggy on mid-range
- Background sync: respect device sleep — don't drain battery polling
- Storage: prune offline cache aggressively (don't accumulate forever)

## Native App vs Responsive Web

| Factor | Native | Responsive web |
|---|---|---|
| True offline support | Excellent | Limited (PWA helps but with quirks) |
| Device features (scanner, camera, GPS) | Direct access | Via web APIs, with permission friction |
| Distribution | App stores (review process, updates lag) | Instant deploy |
| Maintenance cost | High (iOS + Android + web) | Low (one codebase) |
| User installation friction | Install once, then easy | Zero install |
| Login persistence | Long-lived token | Frequent re-auth |
| Use when | Warehouse, field sales, technicians | Approvals, attendance, expense |

For Indonesian SMB ERPs, responsive web (PWA) covers most needs at a fraction
of the cost. Reserve native for operations that genuinely require it
(scanning at high frequency, offline-mandatory, device-feature-heavy).

## Anti-Patterns

❌ Tiny "Approve" / "Reject" buttons rendered next to each other in a mobile list.
   Pak Hendra approves a Rp 50.000.000 invoice by mistake while in a meeting.
✅ Tap a row → opens summary view → action buttons full-width, separated,
   require explicit confirm step.

❌ Showing the desktop table on mobile with horizontal scroll.
   Operator misses key columns, makes wrong decisions.
✅ Card list: each invoice becomes a card with the 3 most-important fields.
   Tap for full detail.

❌ Quotation creation on mobile with 30 form fields stacked vertically.
   Operator abandons after field 8.
✅ "Quotation creation is desktop-only. Tap to open in your desktop browser
   on next sign-in." OR: mobile supports a 3-field "quick quote draft" that
   completes on desktop.

❌ Optimistic offline writes that fail silently when sync conflicts arise.
   Warehouse shows stock = 50, but customer's order was already shipped by
   another picker — system desync.
✅ Offline writes queue with explicit sync status. Conflicts surface to operator:
   "Stock for SKU-001 changed since you went offline. Review: [your action]
   vs [current state]."

## Pre-Release Checklist: Mobile

- [ ] Identified which workflows actually need mobile (not all of them)
- [ ] Stated which workflows are desktop-only (and how mobile users are routed)
- [ ] Touch targets ≥ 44px on critical actions
- [ ] Single-column forms on mobile (no exceptions)
- [ ] Approve/Reject buttons visually separated and confirmed
- [ ] High-stakes approvals require confirmation (or desktop)
- [ ] Offline support for workflows that genuinely need it
- [ ] Sync status visible to operator (queue depth, last sync, conflicts)
- [ ] Tested on mid-range Android with throttled 4G
- [ ] PWA install prompts for power users (warehouse, field sales)
- [ ] Battery and bundle budget honored

## Cross-References

- Offline storage and sync: [[offline-and-network]]
- Human factors on small screens: [[human-factors]]
- Performance budgets: [[performance]]
- Audit trail for mobile actions: [[observability]]
- Approval reversal patterns: [[exceptions-and-recovery]]
