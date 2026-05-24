---
name: erp-best-practice
description: >
  Production-grade ERP reasoning system covering UX/UI, forms, tables, financial
  integrity, concurrency, observability, exception handling, performance, network
  reality, mobile workflows, legacy migration, and Indonesian compliance (UU PDP,
  BPJS, NIK/NPWP, IDR). Use this skill when: designing or reviewing ERP screens,
  forms, workflows, or data models; auditing financial code for safety; planning
  migrations from spreadsheets or legacy systems; reasoning about operator
  behavior, recoverability, race conditions, or audit trails; making tradeoff
  decisions between competing principles. Triggers on enterprise app UX, admin
  experience, internal tools, accounting workflows, payroll, invoicing,
  approvals, or back-office software questions.
---

# ERP Best Practice

A reasoning system for building, reviewing, and operating ERP systems that
real Indonesian operators actually use. Optimized for operational throughput,
financial correctness, and recoverability — not for visual impressiveness.

## Core Philosophy

ERP systems are judged by:
- Operator speed and confidence
- Low cognitive load
- Recoverability from mistakes
- Financial correctness
- Workflow continuity
- Long-term maintainability
- Operational stability

Not by visual impressiveness or SaaS design trends. Every recommendation in this
skill is filtered through that lens.

When principles conflict, resolve via the priority hierarchy in `references/tradeoffs.md`:
financial correctness > legal/compliance > data integrity > user trust >
recoverability > auditability > throughput > onboarding clarity > visual aesthetics.

## Stack-Agnostic

The patterns in this skill are framework- and language-agnostic. Code examples
in reference files are written as pseudocode showing the *pattern*, with brief
notes for major stacks (Node/TypeScript, Python/Django/SQLAlchemy, Ruby on
Rails, Java/Spring, Go, C#/.NET, PHP/Laravel) where the idiom matters. Translate
the pattern to whatever stack the project uses — when you have access to the
codebase, read it first to match existing framework, ORM, and conventions.

## User Personas

Internalize these personas before answering any ERP question.

---

**Sri, 44 — Operations Admin**
Inputs quotations, contracts, and RAB every day. 1366×768 monitor, browser
at 100% zoom, frequently multitasks with WhatsApp open. Non-technical.
When an error is unclear, she asks the developer directly via WhatsApp.

Most common frustrations:
- Field labels disappear when she starts typing (placeholder-as-label)
- Errors show only at the top of the page after she's scrolled all the way down
- Filter state resets every time she navigates back to the list
- Free-text fields she fills "correctly" don't show up cleanly in reports

---

**Pak Hendra, 58 — Finance Manager**
Views reports, approves documents, occasionally inputs data. Wears reading
glasses. Gray or small text is unreadable. More than 3 clicks for an approval
and he'll request IT to "simplify it." Never reads manuals. When confused,
calls Sri.

---

**Rina, 31 — Marketing Staff**
Creates proposals, tracks prospects, monitors deal pipelines. Comfortable
with technology (uses CRM, Google Workspace). Moves fast — frustrated by
too many mandatory fields when she just wants to save a draft quickly. Expects
autocomplete for customer names, not manual typing from scratch.

---

**Dimas, 27 — HR Staff**
Manages employee records, attendance, leave requests. Tech-savvy but new to
this specific ERP. Asks "why does this field exist?" a lot. Needs clear field
labels and good empty states to understand what data is expected.

---

## Reference Files

Read only the files relevant to the question. Cross-doctrine escalation rules
in `references/review-modes.md` determine when to load additional files.

### Foundational (load when reasoning across domains)

| Topic | File |
|---|---|
| Severity classification: CRITICAL / HIGH / MEDIUM / PREFERENCE | `references/severity.md` |
| Tradeoff doctrine and priority hierarchy | `references/tradeoffs.md` |
| Mode selection (generation, critique, audit, review, analysis) | `references/review-modes.md` |

### UX / UI

| Topic | File |
|---|---|
| Typography, color, contrast, spacing | `references/visual-design.md` |
| Table alignment, row design, actions | `references/table-design.md` |
| Forms: create, edit, multi-step, errors, validation | `references/forms.md` |
| Modal, drawer, full page — when to use which | `references/navigation.md` |
| Input type decisions: free text, dropdown, combobox, contextual help | `references/input-control.md` |
| Core ERP principles, do/don't, data integrity | `references/erp-principles.md` |

### Operational Engineering

| Topic | File |
|---|---|
| Money as integer, transactions, audit trail, double-entry, rate snapshots | `references/money-and-data-integrity.md` |
| Race conditions, idempotency, optimistic locking, webhook retries | `references/concurrency.md` |
| Audit lineage, correlation IDs, replayability, reconciliation visibility | `references/observability.md` |
| Emergency edits, retroactive correction, approval reversal, session expiry | `references/exceptions-and-recovery.md` |
| Pagination, exports, slow queries, low-end device reality | `references/performance.md` |
| Autosave, retry, unstable network, true offline patterns | `references/offline-and-network.md` |
| Operator psychology, panic clicks, confirmation fatigue, trust-building | `references/human-factors.md` |

### Context

| Topic | File |
|---|---|
| Warehouse scanning, field sales, mobile approvals, when mobile is NOT needed | `references/mobile-erp.md` |
| Spreadsheet/WhatsApp replacement, gradual adoption, reconciliation | `references/legacy-and-migration.md` |
| IDR format, timezone, NIK/NPWP, BPJS rates, UU PDP data classification | `references/indonesia-compliance.md` |

---

## Usage Pattern

1. **Identify the mode.** Generation, critique, audit, review, or operational
   analysis? See `references/review-modes.md`. When unclear, prefer the more
   defensive mode.

2. **Identify the domain.** UX, financial, concurrency, compliance, etc.
   Load only the reference file(s) needed — 1–3 typically, occasionally more
   for cross-domain questions.

3. **Apply cross-doctrine escalation.** If the workflow touches money, always
   load `money-and-data-integrity.md`. If it touches PII, always load
   `indonesia-compliance.md`. If concurrent operators are involved, load
   `concurrency.md`. Full escalation table in `review-modes.md`.

4. **Tag severity.** When citing a rule or flagging an issue, mark its level
   (CRITICAL / HIGH / MEDIUM / PREFERENCE) per `severity.md`. Match the
   strength of recommendation to severity.

5. **Resolve conflicts via the priority hierarchy.** When principles compete,
   `tradeoffs.md` decides which wins.

6. **Answer with concrete numbers, ✓/✗ examples, and persona impact.** Generic
   advice is the failure mode this skill exists to prevent.

## Quick Severity Reference

| Class | Examples |
|---|---|
| **CRITICAL** | Money as float; no idempotency on financial ops; NIK to external AI; missing audit on financial mutation; missing transaction boundary |
| **HIGH** | Placeholder-as-label; free text on reporting field; WCAG AA contrast fail; filter reset on back; controlled input missing; unsaved-changes warning missing |
| **MEDIUM** | Inconsistent required marking; weak empty state copy; generic submit button label; missing helper text |
| **PREFERENCE** | Specific font weight; zebra vs lines; exact drawer width; color palette choice |

Full index: `references/severity.md`.

## Anti-Pattern Reminders

- Never optimize visual aesthetics over operator throughput
- Never make destructive actions easier than recoverable ones
- Never trust the "happy path" — design for refresh-mid-form and network drops
- Never assume year-1 data volume (design for year 2: 10–100× more rows)
- Never confuse "mobile parity" with "every workflow on mobile"
- Never replace a working spreadsheet with a worse system overnight
- Never silently overwrite, silently discard, or silently fail

## Author

Hasban Fardani — Indonesian Software Engineer
