# erp-best-practice

> A Claude AI skill for building ERP systems that are actually used — not just impressive on demos.

**Author:** Hasban Fardani
**Language:** English  
**Scope:** UX/UI, form design, data integrity, Indonesian compliance

---

## Overview

`erp-best-practice` is a structured knowledge base packaged as a Claude skill. It gives Claude a grounded, opinionated framework for answering questions about ERP system design — covering everything from pixel-level typography decisions to architecture-level principles like idempotency and audit trails.

The skill is built around a core insight: most ERP systems fail not because of missing features, but because the UX was designed for what management *wanted to see*, not what the actual daily user *needs to do*. Every recommendation in this skill is filtered through that lens.

---

## What It Covers

**Foundational**

| Domain | Reference File |
|---|---|
| Severity classification (CRITICAL / HIGH / MEDIUM / PREFERENCE) | `references/severity.md` |
| Tradeoff doctrine and priority hierarchy | `references/tradeoffs.md` |
| Mode selection (generation / critique / audit / review / analysis) | `references/review-modes.md` |

**UX / UI**

| Domain | Reference File |
|---|---|
| Typography, color, contrast, spacing | `references/visual-design.md` |
| Table alignment, row style, empty states, pagination | `references/table-design.md` |
| Forms: create, edit, multi-step, validation, unsaved changes | `references/forms.md` |
| Modal, drawer, full page — decision rules and anatomy | `references/navigation.md` |
| Free text vs dropdown, combobox, contextual help | `references/input-control.md` |
| ERP principles: discovery, permissions, filter persistence, do/don't | `references/erp-principles.md` |

**Operational engineering**

| Domain | Reference File |
|---|---|
| Money as integer, idempotency, transactions, audit trail, double-entry | `references/money-and-data-integrity.md` |
| Race conditions, idempotency, locking, webhook retries, numbering | `references/concurrency.md` |
| Audit lineage, correlation IDs, replayability, reconciliation | `references/observability.md` |
| Emergency edits, retroactive correction, approval reversal, session expiry | `references/exceptions-and-recovery.md` |
| Pagination, exports, slow queries, low-end device reality | `references/performance.md` |
| Autosave, retry, unstable network, true offline patterns | `references/offline-and-network.md` |
| Operator psychology, panic clicks, confirmation fatigue, trust | `references/human-factors.md` |

**Context**

| Domain | Reference File |
|---|---|
| Warehouse scanning, field sales, mobile approvals | `references/mobile-erp.md` |
| Spreadsheet/WhatsApp replacement, gradual adoption | `references/legacy-and-migration.md` |
| IDR format, timezone, NIK/NPWP, BPJS, UU PDP compliance | `references/indonesia-compliance.md` |

---

## User Personas

The skill grounds all recommendations in four realistic Indonesian ERP user archetypes:

**Sri, 44 — Operations Admin.** Inputs quotations and contracts daily. 1366×768 monitor, non-technical, asks the developer when confused. Frustrated by placeholder-as-label, filter resets, and errors only visible at the top of long forms.

**Pak Hendra, 58 — Finance Manager.** Views reports and approves documents. Wears reading glasses. Gray or small text is unreadable. Will request IT to "simplify" anything requiring more than 3 clicks.

**Rina, 31 — Marketing Staff.** Comfortable with technology, moves fast. Frustrated by too many mandatory fields when she just wants a quick draft. Expects autocomplete for customer names.

**Dimas, 27 — HR Staff.** Tech-savvy but new to this specific ERP. Reads forms top to bottom before filling. Needs clear labels and informative empty states.

---

## Key Principles

### For UX/UI Questions
- Single-column forms always — two columns cause Z-pattern scanning and missed fields
- Label above field always — placeholders disappear when typing
- Error summary banner + inline errors together for forms with 10+ fields
- Horizontal lines only in tables — no zebra striping
- Numbers right-aligned, text left-aligned, headers match their column
- Status badges always use color + icon + text, never color alone
- Font weight system: `400` body / `500` labels / `600` section titles / `700` page titles

### For Data & Architecture Questions
- Money always stored as integer in smallest unit (sen for IDR) — never float
- All rates (tax, BPJS, discounts) must come from the database, never hardcoded
- Store the rate at document creation time — historical records must be immutable
- All financial operations: idempotency key + database transaction + audit trail
- Audit events are append-only — no UPDATE, no DELETE on audit tables
- Filter state must persist across back navigation

### For Indonesian Compliance
- IDR display: `Rp 4.000.000` (period as thousands separator, never comma)
- All timestamps stored UTC, displayed in user's timezone (WIB/WITA/WIT)
- NIK and NPWP: validated, encrypted deterministically, never logged or sent to external AI
- BPJS rates from config table with salary caps — not hardcoded
- UU PDP No. 27/2022: sensitive data classification enforced throughout

---

## File Structure

```
erp-best-practice/
├── SKILL.md                              # Entry point — personas, routing, usage pattern
├── README.md                             # This file
└── references/
    │  # Foundational
    ├── severity.md                       # CRITICAL / HIGH / MEDIUM / PREFERENCE classification
    ├── tradeoffs.md                      # Priority hierarchy, conflict resolution
    ├── review-modes.md                   # Generation / critique / audit / review / analysis
    │
    │  # UX / UI
    ├── visual-design.md                  # Typography & color
    ├── table-design.md                   # Table patterns
    ├── forms.md                          # Form design, multi-step, validation
    ├── navigation.md                     # Modal, drawer, full page
    ├── input-control.md                  # Input types & contextual help
    ├── erp-principles.md                 # Core do/don't principles
    │
    │  # Operational engineering
    ├── money-and-data-integrity.md       # Financial integrity rules
    ├── concurrency.md                    # Race conditions, idempotency, locking
    ├── observability.md                  # Audit, correlation, replay, reconciliation
    ├── exceptions-and-recovery.md        # Emergency edits, reversals, session loss
    ├── performance.md                    # Pagination, exports, slow queries
    ├── offline-and-network.md            # Autosave, retry, unstable network
    ├── human-factors.md                  # Operator psychology and trust
    │
    │  # Context
    ├── mobile-erp.md                     # Mobile workflows that matter
    ├── legacy-and-migration.md           # Spreadsheet/WhatsApp replacement
    └── indonesia-compliance.md           # Indonesia-specific compliance
```

Each reference file is self-contained and focused on a single domain. Claude reads only the files relevant to the question — not all at once. The mode and escalation rules in `review-modes.md` decide which additional files to load.

---

## Installation

### Via Claude Code CLI (Recommended)

```bash
claude skills install https://github.com/Hasban-Fardani/erp-best-practice-skills.git
```

### Via Git (Manual Clone)

```bash
# Clone the repository into your Claude skills directory
git clone https://github.com/Hasban-Fardani/erp-best-practice-skills.git ~/.claude/skills/erp-best-practice
```

### Manual Installation

1. Clone or download this repository
2. Place the `erp-best-practice/` folder in your Claude skills directory (`~/.claude/skills/`)
3. Ensure `SKILL.md` is at the root with valid YAML frontmatter
4. Restart Claude Code or reload skills

---

## How Claude Uses This Skill

When a question touches ERP design, Claude:

1. Identifies the relevant domain from the question
2. Reads the corresponding reference file (1–2 files max)
3. Answers with concrete numbers, ✓/✗ examples, and persona context
4. For new feature decisions: checks `erp-principles.md` first
5. For modal vs page decisions: reads `navigation.md`

Claude does not read all files at once. The skill is designed for progressive disclosure — lightweight routing in `SKILL.md`, full detail in the reference files.

---

## What This Skill Does Not Cover

- Frontend framework-specific implementation details beyond code snippets
- Backend API design and REST/GraphQL conventions
- Database schema design (ERD, normalization)
- DevOps, CI/CD, and deployment
- Multi-tenancy architecture
- Specific library or package comparisons
- Generic SaaS UX trends or visual design fashion

The skill is opinionated about *what* to build and *how to reason* about ERP
tradeoffs. It is deliberately silent on framework wars and architectural
fashion that don't change operational outcomes.

---

## Regulatory Basis (Indonesia Compliance)

Rules in `indonesia-compliance.md` are based on regulations current as of **March 2026**:

- UU PDP No. 27/2022 — Personal Data Protection
- PP No. 44–46/2015 — BPJS TK rates
- Perpres No. 82/2018 — BPJS Kesehatan rates
- Permenaker No. 5/2023 — Overtime calculation
- Permenkominfo No. 5/2020 — PSE registration

**Always verify against the latest regulation before implementing.** Rates and rules change.

---

## License
