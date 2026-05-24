# Review-Oriented Modes

This skill supports five interaction modes. Each calls for a different stance,
different output shape, and different cited evidence. Decide which mode applies
before responding, and stay in it.

## Mode Selection

Infer the mode from the user's request. When ambiguous, prefer the most
defensive mode (audit > review > critique > analysis > generation).

| Mode | Trigger phrases | Stance |
|---|---|---|
| **Generation** | "Build...", "Design...", "Create the form for...", "How should I structure..." | Constructive, opinionated, single recommended path |
| **Critique** | "What's wrong with...", "Roast this...", "Where would you push back..." | Direct, exhaustive on issues, severity-tagged |
| **Audit** | "Audit this...", "Find the bugs...", "Is this safe to release?", "Compliance check..." | Pre-release adversarial; assume something is broken |
| **Review** | "Review this PR...", "Take a look at...", "What do you think of..." | Balanced — strengths and issues both, prioritized |
| **Operational Analysis** | "Why might Sri struggle with...", "How will this scale...", "What breaks at year 2..." | Predictive; reason about operational failure modes |

## Mode: Generation

**Goal:** ship the right artifact the first time.

**Output shape:**
- One recommended approach, stated up front
- Concrete numbers, not ranges (pick 480px, not "around 400–600px")
- Cite the source rule from the relevant reference file
- Persona impact ("Sri will use this 80×/day, so single column matters more than density")
- If there are valid alternatives, list 1–2 and explain when to switch
- For implementation tasks: include code in the project's stack — read the
  existing codebase to determine framework, ORM, and conventions, and match them.
  If the stack is unknown, write the pattern in pseudocode plus a stack-agnostic
  description and ask which stack to translate to.

**Avoid:**
- "It depends" without then deciding what it depends on
- Listing every option without committing
- Trendy frontend patterns when an established ERP pattern already exists

## Mode: Critique

**Goal:** find what's broken before users do.

**Output shape:**
- Default to finding 3–5 issues minimum (no "looks good to me" unless you've
  genuinely verified each section)
- Each issue tagged by severity ([[severity]]): CRITICAL, HIGH, MEDIUM, PREFERENCE
- Each issue cites the rule it violates and the source reference file
- Each issue includes the specific fix
- Order: CRITICAL first, then HIGH, then MEDIUM, then PREFERENCE
- Show, don't just tell — quote the problematic code/text/screenshot detail

**Stance rules:**
- A CRITICAL issue is non-negotiable; do not soften it because the user seems
  attached to the design
- A PREFERENCE issue is not a critique — note it once and move on
- If there are no CRITICAL or HIGH issues, say so explicitly — operators trust
  this skill to be loud when something is broken

## Mode: Audit

**Goal:** decide whether something is safe to release.

**Output shape:**
- Pre-release checklist walked top to bottom, item by item
- Pass / fail / unknown for each
- For "unknown": state what evidence is missing
- Block-release issues called out separately at the top
- Final verdict: SAFE / SAFE WITH CAVEATS / NOT SAFE

**Walked checklists (load from the relevant reference file):**
- Money operations → [[money-and-data-integrity]] Pre-Release Checklist
- Concurrency → [[concurrency]] Failure Scenarios to Test
- Indonesia compliance → [[indonesia-compliance]] Pre-Release Checklist
- Observability → [[observability]] Pre-Release Checklist
- Network reliability → [[offline-and-network]] Pre-Release Checklist
- Exception handling → [[exceptions-and-recovery]] Pre-Release Checklist
- Performance → [[performance]] Pre-Release Checklist
- Mobile → [[mobile-erp]] Pre-Release Checklist
- Migration → [[legacy-and-migration]] Pre-Release Checklist

**Stance rules:**
- Default to "NOT SAFE" until evidence proves otherwise
- Asking "show me where idempotency is implemented" is a valid audit step;
  do not infer from absence
- If access to the code is read-only and a check requires running something,
  state the check needed and that it must be verified

## Mode: Review

**Goal:** balanced assessment for a colleague.

**Output shape:**
- Brief summary (2–3 sentences) of what was reviewed
- "Working well" section (2–4 items) — what to keep
- "Issues" section — severity-tagged, like critique but limited to the top 5–8
  most actionable
- "Worth considering" section — items that are not issues per se but worth
  thinking about
- Specific suggestions, not vague ones ("split the customer field into autocomplete
  + add new", not "improve the customer input")

**Stance rules:**
- Acknowledge strengths genuinely — operators are demoralized by reviews that
  only criticize
- But never soften a CRITICAL or HIGH issue to maintain niceness
- For PR reviews: comment on diff scope only, not surrounding code unless invited

## Mode: Operational Analysis

**Goal:** predict failure modes before they happen in production.

**Output shape:**
- Walk the workflow as one of the personas — Sri at 4pm tired, Pak Hendra
  approving 40 documents before lunch, Rina in a hurry to leave for a customer
  meeting
- Identify the point where the workflow fails or degrades
- Quantify where possible: "at 10k records this query becomes a 4s page load"
- Identify which reference doctrine applies: concurrency, observability,
  human factors, performance
- Recommend the specific change

**Stance rules:**
- "It'll be fine" is not analysis. Find at least one failure mode.
- Common blind spots: year-2 data volume, multi-operator concurrency, network
  flake at the wrong moment, repetitive-task fatigue, operator off-system workaround

## Mode-Specific Output Examples

### Generation — "How should I build the quotation create form?"

> Use a multi-step form (4 steps: Document Info → Line Items → Financial →
> Review). For Quotations specifically, ~20 fields total justifies this.
>
> **Step structure:** [from [[forms]] § Multi-Step]
>
> **Critical patterns to include:**
> - Autosave to draft every 60s (per [[offline-and-network]]) — quotations are
>   long, network drops will lose work otherwise
> - Tax rate stored at document creation, not live reference (per
>   [[money-and-data-integrity]])
> - Customer field as combobox (autocomplete + add new), per [[input-control]]
>
> **Stack-specific notes:** match the project's existing form library and ORM.
> Most admin frameworks (Filament, Django admin, ActiveAdmin, ASP.NET Core
> scaffolding) and form libraries (React Hook Form, vee-validate, Reactive
> Forms) have built-in multi-step + draft helpers — use them rather than
> rolling your own.
>
> **Persona check:** Rina creates ~30 quotations/week — she'll appreciate the
> draft autosave when she gets pulled into a meeting mid-form.

### Critique — "What's wrong with this design?"

> Found 4 issues (1 CRITICAL, 2 HIGH, 1 MEDIUM):
>
> **🔴 CRITICAL — Total stored as DECIMAL(12,2)**
> Per [[money-and-data-integrity]]: money must be BIGINT in sen, not DECIMAL.
> Float math will produce sub-cent discrepancies that compound across reports.
> Fix: migrate column to BIGINT, store as `(int) round($total * 100)`.
>
> **🟠 HIGH — Placeholder used as label on customer field**
> Per [[forms]]: label above field, always. Placeholder disappears on focus.
> Sri will not remember what the field is for after typing 3 characters.
> Fix: add `<label>` above, keep placeholder as format example only.
>
> **🟠 HIGH — No idempotency on submit handler**
> Per [[concurrency]]: financial operations must use idempotency keys. Double-
> click on Submit will create two quotations with consecutive numbers. Fix:
> add idempotency middleware, client generates UUID before submit.
>
> **🟡 MEDIUM — Submit button labeled "Submit"**
> Per [[forms]]: use specific labels. "Save Quotation" reduces ambiguity.

### Audit — "Is this payroll module safe to release?"

> Verdict: **NOT SAFE** — 2 release-blocking issues.
>
> **🔴 Block-release issues:**
> 1. Salary stored as `decimal(10,2)` — not integer in sen. Violates
>    [[money-and-data-integrity]] CRITICAL rule.
> 2. No idempotency key on `POST /payroll/process` — duplicate runs will create
>    duplicate payroll entries. Violates [[concurrency]] CRITICAL rule.
>
> **Pre-release checklist** (from [[money-and-data-integrity]] +
> [[indonesia-compliance]]):
>
> | Check | Status |
> |---|---|
> | Money as integer in sen | ❌ FAIL — uses decimal |
> | Rate multiplication uses round() | ⚠ UNKNOWN — not visible in code reviewed |
> | Multi-table ops in transaction | ✓ PASS — explicit transaction wrapper present |
> | Idempotency on financial endpoint | ❌ FAIL — no key check |
> | Audit event in same transaction | ✓ PASS |
> | Rates from config table | ✓ PASS — uses RegulationConfig |
> | Historical rate snapshot | ⚠ UNKNOWN — need to verify |
> | NIK never logged | ⚠ UNKNOWN — review log lines |
> | BPJS cap applied | ✓ PASS |
> | PPh 21 field present | ✓ PASS |
>
> **To verify:** show me where rate is snapshot at calculation time, and grep
> the codebase for log lines that include `nik`, `salary`, or `gaji`.

### Operational Analysis — "How will this scale?"

> Walking the workflow as Sri at year 2 with ~80,000 invoices:
>
> **Where it breaks:**
>
> 1. **Invoice list page** — current query `SELECT * FROM invoices ORDER BY
>    created_at DESC LIMIT 25 OFFSET ?` becomes slow past page 200 (offset
>    scans 5,000 rows). Sri opens a customer's invoice history from 18 months
>    ago and waits 6 seconds.
>    Fix: cursor pagination per [[performance]].
>
> 2. **Customer combobox** — at 5,000 customers, the autocomplete fetches all
>    of them on focus. Sri types and waits 800ms for filtering.
>    Fix: server-side search with debounce 300ms, return top 20 matches.
>
> 3. **Concurrent number assignment** — two ops staff hit Save at the same
>    moment, both get INV-2026-12847, one save fails with constraint violation,
>    operator confused.
>    Fix: DB sequence per [[concurrency]] § Race Conditions in Numbering.
>
> 4. **Export to Excel** — Sri exports "all invoices this year" → server tries
>    to hold 80k rows in memory → OOM or 5-minute hang.
>    Fix: background export per [[performance]] § Exports.

## When to Refuse to Pick a Mode

If the user's request mixes modes confusingly ("Build this AND tell me what's
wrong with the existing version AND audit the migration plan"), break it apart:

> "Three different tasks here. Let me handle them in order:
> 1. **Audit** the existing version (defensive, finds gaps)
> 2. **Critique** the migration plan (issues + severities)
> 3. **Generate** the new design (informed by 1 + 2)"

Sequencing matters. Generating without auditing means you may rebuild a bug.

## Reviewing Screenshots

When the user shares an interface screenshot for review:

1. **Identify the screen type** (list, form, modal, dashboard).
2. **Walk it as the persona who uses it most.** Sri for ops screens, Pak Hendra
   for approval/reporting, Rina for sales/CRM, Dimas for HR.
3. **Check the relevant reference files for that screen type.**
4. **Tag issues by severity.**

For screenshots, common HIGH-severity findings:
- Placeholder-as-label
- Status communicated by color alone (no icon, no text)
- Tiny gray text below 16px or below 4.5:1 contrast
- Save button generically labeled
- No empty state copy
- Mixed alignment (number column left-aligned, money column right)
- No filter persistence indicator
- Action buttons too close to each other (Approve next to Reject)
- Modal with too many fields

Common CRITICAL findings (rarer but high-impact):
- Free-text field on a categorical reporting field
- Sensitive data (NIK, salary) visible without role check
- Destructive action without confirmation
- Money displayed in wrong format (Rp 4,000,000 instead of Rp 4.000.000)

## Reviewing Workflow Proposals

When the user describes a proposed workflow (rather than a built screen):

1. **Find the failure modes** — what happens when the network drops mid-step,
   when two operators run it concurrently, when the operator clicks the wrong
   button, when the underlying record changed between load and submit?
2. **Check the regulatory and financial requirements** — audit trail, transaction
   atomicity, rate snapshots, PII handling.
3. **Walk it as the lowest-skill persona who will use it.** If Sri can complete
   it without contacting the developer, it passes.
4. **Identify cross-doctrine escalations:**
   - Touches money → load [[money-and-data-integrity]]
   - Touches PII → load [[indonesia-compliance]]
   - Concurrent operators → load [[concurrency]]
   - Reversible? → load [[exceptions-and-recovery]]
   - On mobile? → load [[mobile-erp]]

## Cross-Doctrine Escalation Rules

When a question touches multiple domains, always load the higher-stakes doctrine.

| If the workflow involves... | Always also load |
|---|---|
| Money or financial mutation | [[money-and-data-integrity]], [[concurrency]], [[observability]] |
| PII (employee data, NIK, NPWP, bank account, salary) | [[indonesia-compliance]] |
| Approval or destructive action | [[exceptions-and-recovery]], [[human-factors]] |
| List or table with > 1k expected rows | [[performance]] |
| External callback (webhook, payment) | [[concurrency]], [[observability]] |
| Long form (15+ fields) or multi-step | [[forms]], [[offline-and-network]], [[exceptions-and-recovery]] |
| Mobile context | [[mobile-erp]], [[offline-and-network]] |
| Replacing an existing process | [[legacy-and-migration]] |
| Any conflict between two principles | [[tradeoffs]] |

Never answer a financial workflow question without [[money-and-data-integrity]].
Never answer a PII question without [[indonesia-compliance]].

## Pre-Response Self-Check

Before producing the output:

- [ ] Have I identified the correct mode?
- [ ] Have I loaded the right reference files (not too many, not too few)?
- [ ] Are my severity tags accurate per [[severity]]?
- [ ] Have I cited the source rule for non-obvious claims?
- [ ] Have I named the persona impact where relevant?
- [ ] Have I checked for cross-doctrine escalations?
- [ ] Am I committing to a recommendation or hedging?
