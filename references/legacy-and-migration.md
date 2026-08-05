# Legacy & Migration Reality

Most Indonesian SMB ERPs do not replace another ERP. They replace:
- A constellation of Excel spreadsheets (the actual source of truth)
- A WhatsApp group where "approval" happens
- A legacy system from 2014 that nobody fully understands
- Manual paper processes the office has used for 20 years

If you treat your project as a greenfield SaaS deployment, you will lose. The
existing system works — imperfectly, painfully, but it works. The replacement
must earn its place.

## What You're Actually Replacing

Map the **real** current state before designing the new state:

| What management says | What actually happens |
|---|---|
| "We use SAP for accounting" | SAP for posting, Excel for analysis, WhatsApp for approval |
| "Sales reps use the CRM" | Reps use WhatsApp Business + a personal Excel, the CRM is updated weekly by an intern |
| "We have an inventory system" | The system exists; warehouse staff write on paper, key-in once a week |
| "Approval is in the system" | Approval is in WhatsApp; system is updated after |
| "Invoices go out from the ERP" | Invoices are generated in Excel, the ERP is updated retroactively |

The gap between "documented" and "actual" is the design space. Build for actual.

## Pre-Migration Discovery

Before writing code, capture:

1. **The current workflow, end-to-end.** Who initiates? Who does what next?
   What's the longest delay? What's the most common error?
2. **The Excel templates in use.** Get copies. Read every column. Note formulas.
   Note the manual cleanup people do (the formula breaks, they paste-value over it).
3. **The WhatsApp groups.** What's discussed? Who approves? How are approvals
   recorded? How are exceptions handled?
4. **The historical data.** How many records? How dirty? What's the agreed-upon
   "starting point" for migration (March 1 of next fiscal year? Last completed
   month? Last completed quarter)?
5. **The seasonal cycle.** Don't migrate the week before payroll. Don't migrate
   the day month-end close starts.

What you learn here decides whether the project succeeds.

## Gradual Adoption Patterns

Big-bang migrations fail. Operators reject systems that show up overnight
demanding their full workflow.

**Pattern A — Parallel run:**

```
Month 1: New system available. Old process continues.
         Operators try the new system for 1–2 workflows. Old system is canonical.

Month 2: Operators expand to 5–6 workflows. Old system still canonical.
         Daily reconciliation: do both systems show the same number?

Month 3: Switch canonical to new system. Old system continues read-only.
         Weekly reconciliation.

Month 4+: Old system archived, read-only access for historical lookup.
```

This is slower, more expensive, and the only way that consistently works.

**Pattern B — Department-by-department:**

Roll out to one department first (sales operations, then warehouse, then finance).
Each department adopts fully before the next starts. Cross-department workflows
have a defined handoff between systems during the transition.

**Pattern C — Workflow-by-workflow:**

Roll out by business workflow rather than by department. Quotation flow first,
then invoice flow, then collection flow. Each workflow is end-to-end on the new
system before the next starts.

Choose by stakeholder appetite. Department-by-department reduces conflict.
Workflow-by-workflow gets to value faster. Big-bang gets you fired.

## Import Validation

Spreadsheets are dirty. Always.

For every imported field:

1. **Schema validation.** Type, format, length, regex.
2. **Business validation.** Foreign keys exist, dates are reasonable, amounts are non-negative
   where required, totals match line items.
3. **Duplicate detection.** Same customer name spelled three ways → cluster and
   present for review, do not blindly create three customers.
4. **Reconciliation totals.** Sum of imported amounts must equal sum of source
   spreadsheet. If they don't match, abort and report which rows differ.
5. **Sample review.** First 10 imported records visible side-by-side with source
   spreadsheet — operator approves before bulk runs.

Anti-patterns:

❌ Import 10,000 records, 837 fail silently with "validation error." Operator
   discovers two months later when reports are short.
✅ Import surfaces failures inline with row reference, error reason, and
   suggested fix. Operator can edit and retry, row by row or in bulk.

❌ Import strips data the operator didn't realize was important (notes column,
   the "category" they made up in 2019).
✅ Import preserves all source columns in a `migration_metadata` JSON column
   on each record, available for inspection if anything looks wrong post-launch.

❌ Operator imports the same file twice → duplicate records.
✅ Import detects (file_hash) and (per-row idempotency key from the file) →
   reuses results from the prior import.

## Reconciliation Workflow

Reconciliation is the most-skipped phase and the most damaging to skip.

Required reconciliation outputs after every migration phase:

| Comparison | What it proves |
|---|---|
| Count of source records vs target records | No records lost |
| Sum of source amounts vs target amounts | No financial drift |
| Count by category / status | No records misclassified |
| Sample of 10 random records, full field comparison | No silent corruption |
| Date range coverage (earliest to latest) | No partial periods |
| Sum of debit / credit in journal | Double-entry balance preserved |

Run these reconciliations daily during parallel run. Operator-facing dashboard,
not a developer SQL query. When reconciliation breaks, the system must say so
loudly — silent drift is the killer.

## Spreadsheet Replacement

Operators trust spreadsheets because they can see everything. They distrust
systems that "calculate behind the scenes."

When replacing a spreadsheet:

1. **Show the calculation.** If a field is computed from others, show the inputs.
   On hover: "Total = subtotal (4.500.000) - discount (450.000) + PPN (445.500)
   = 4.495.500"
2. **Match the spreadsheet's column order initially.** Don't reorganize on day 1.
   Operators search visually by position. Match, then evolve.
3. **Export to the same shape they had.** Reports export to a spreadsheet that
   matches their existing template — easy comparison during validation.
4. **Preserve their categories.** If they used "Konsumsi" as an expense category,
   don't rename it to "Food & Beverage." Map their vocabulary.
5. **Allow free-form notes for the things the system doesn't model yet.** Don't
   force every nuance into a structured field on day 1.

## WhatsApp Workflow Replacement

WhatsApp is the universal Indonesian SMB workflow tool. Approval happens there,
clarifications happen there, exceptions happen there. The ERP cannot win against
WhatsApp on speed or familiarity — it must win on traceability and completeness.

Strategy:

1. **Don't try to remove WhatsApp entirely.** You will lose. Integrate.
2. **System-to-WhatsApp notifications.** The system sends a WhatsApp message
   when an approval is needed, with a link back to the system.
3. **Capture the conversation outcome in the system.** If approval happens on
   WhatsApp, the system has a "Record approval from WhatsApp" workflow that
   captures: approver, screenshot of conversation, timestamp, reason. The
   resulting audit trail equals the in-system approval.
4. **Make in-system faster than WhatsApp for routine cases.** Approve from a
   notification with one tap, including from mobile. See [[mobile-erp]].
5. **Trend over time:** show "approvals via system" vs "approvals via WhatsApp
   recorded later." Use this as a leading indicator of adoption.

## Operator Adaptation Curve

Real adoption timeline for an SMB ERP module:

| Phase | Duration | Indicator |
|---|---|---|
| Honeymoon | Week 1–2 | Operators try it, enthusiasm, lots of questions |
| Friction | Week 3–8 | Real frustrations surface; usage drops if not addressed |
| Adaptation | Month 3–6 | Workflows become habit; muscle memory forms |
| Ownership | Month 6+ | Operators request features; treat the system as theirs |

Most projects die in Week 3–8. Stay close to operators in this window. Daily
standups with Sri's team, fast turnaround on small fixes, public acknowledgment
of pain points.

## Migration-Specific UX

For the parallel-run period:

- **Mode indicator visible.** "You are working in the NEW system. The old system
  is still available at [link]."
- **Source attribution on every record.** "Migrated from spreadsheet 2026-03-01"
  or "Created in new system." Operators need to know which records to trust.
- **Easy lookback to old system.** Bookmark, link, "open in old system" button.
  Operators will compare for the first few months.
- **Reconciliation status surfaced.** "Today's invoices: 47 in new system, 47
  in old system. ✓ In sync." Operator confidence comes from visible matching.

## Anti-Patterns

❌ "We'll just import the spreadsheet and shut down the old workflow next week."
   Result: panic, operator revolt, week-long emergency to restore spreadsheets.
✅ Parallel run for 2–3 months. Reconciliation daily. Switch when both systems
   agree for 30 consecutive days.

❌ "The old data is messy, we'll start fresh with the new system."
   Result: operators have to use two systems forever — old for history, new for
   current. Reporting across periods becomes manual.
✅ Migrate at least one full fiscal year of history. Validate. Reconcile.
   Then run the new system as the only system.

❌ "Operators will read the manual and learn the new system."
   Result: nobody reads the manual. Adoption stalls. The project is labeled
   a failure.
✅ Workflow-level training, in person, with each operator on their actual data.
   Documented as short videos for refresh, not a manual.

❌ Migrating into a structure that doesn't match how the business actually thinks.
   Result: every report requires data transformation. Operators don't trust
   the numbers.
✅ Match the business's vocabulary and structure on day 1. Refactor toward
   "ideal" only after operators trust the system.

## Pre-Release Checklist: Migration

- [ ] Discovered the actual current workflow (not the documented one)
- [ ] Collected the real spreadsheet templates in use
- [ ] Identified the WhatsApp workflows being replaced
- [ ] Selected a gradual adoption pattern (parallel / department / workflow)
- [ ] Migration is scheduled outside of seasonal peaks (payroll, month-end, fiscal close)
- [ ] Import script validates schema, business rules, and reconciliation totals
- [ ] Reconciliation dashboard exists and is operator-facing
- [ ] Source attribution captured on every migrated record
- [ ] Old system available read-only during transition
- [ ] Training planned per-operator, per-workflow, on real data
- [ ] Friction-window support plan exists (Week 3–8)
- [ ] Adoption metrics tracked: usage by workflow, by operator, by department

## Cross-References

- Reconciliation totals and financial integrity: [[money-and-data-integrity]]
- Observability for tracking adoption and conflicts: [[observability]]
- Recovery patterns for migration-era data corrections: [[exceptions-and-recovery]]
- Human factors during adaptation friction: [[human-factors]]
