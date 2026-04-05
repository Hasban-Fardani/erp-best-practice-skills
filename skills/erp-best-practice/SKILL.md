---
name: erp-best-practice
description: >
  Complete UX/UI and architecture guide for building ERP systems that are
  actually used — not just impressive on demos. Use this skill when: designing
  forms (create/edit/multi-step), data tables, modals/drawers, visual design
  (typography/color), input controls (free text vs dropdown), contextual help,
  navigation patterns, or discussing core ERP principles. Always trigger for
  questions about enterprise app UX, form design, admin experience, ERP
  architecture decisions, or frontend best practices for internal tools.
---

# ERP Best Practice

## User Personas

Internalize these personas before answering any ERP UX question.

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

Read only the file relevant to the question — not all at once.

| Topic | File |
|---|---|
| Typography, color, contrast, spacing | `references/visual-design.md` |
| Table alignment, row design, actions | `references/table-design.md` |
| Forms: create, edit, multi-step, errors, validation | `references/forms.md` |
| Modal, drawer, full page — when to use which | `references/navigation.md` |
| Input type decisions: free text, dropdown, combobox | `references/input-control.md` |
| Core ERP principles, do/don't, data integrity | `references/erp-principles.md` |
| Money as integer, idempotency, transactions, audit trail, double-entry | `references/money-and-data-integrity.md` |
| IDR format, timezone, NIK/NPWP, BPJS rates, UU PDP data classification | `references/indonesia-compliance.md` |

---

## Usage Pattern

1. Identify which domain the question falls into
2. Read only the relevant reference file (1–2 max)
3. Answer with concrete numbers, ✓/✗ examples, and persona context
4. For new feature decisions: always check `erp-principles.md` first
5. When in doubt whether to use a modal or page: read `navigation.md`


## Author

Hasban Fardani - Indonesian Software Engineer
