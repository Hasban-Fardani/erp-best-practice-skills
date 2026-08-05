---
name: erp-best-practice
description: >
  Production-grade ERP reasoning system covering UX/UI, forms, tables, production
  safety, AI accountability, and JavaScript delivery alongside financial
  integrity, concurrency, observability, exception handling, performance, network
  reality, mobile workflows, legacy migration, and Indonesian compliance (UU PDP,
  BPJS, NIK/NPWP, IDR). Use this skill when: designing or reviewing ERP screens,
  forms, workflows, or data models; auditing financial code for safety; planning
  migrations from spreadsheets or legacy systems; reasoning about operator
  behavior, recoverability, race conditions, or audit trails; making tradeoff
  decisions between competing principles. Triggers on enterprise app UX, admin
  experience, internal tools, accounting workflows, payroll, invoicing,
  approvals, or back-office software questions. Treat every application as
  production unless the user explicitly labels it a disposable sandbox.
---

# ERP Best Practice

A reasoning system for building, reviewing, and operating ERP systems that
real Indonesian operators actually use. Optimized for operational throughput,
financial correctness, recoverability, and safe change delivery — not for
visual impressiveness.

## Core Philosophy

ERP systems are judged by:
- Operator speed and confidence
- Low cognitive load
- Recoverability from mistakes
- Financial correctness
- Workflow continuity
- Long-term maintainability
- Operational stability
- Safe production changes with an explicit evidence trail
- Honest AI output that separates observed facts, inference, and unknowns

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

## Production Assumption and AI Accountability

Default stance: the application is already in production and supports real
operators for years. A local checkout is not permission to treat data, queues,
cron, or deployment state as disposable. If the environment is unknown, mark
it **UNKNOWN** and ask for the missing evidence before making a risky change.

Every implementation report must separate:

1. **Observed** — directly read from source, command output, test output, or a
   browser trace.
2. **Inferred** — a conclusion derived from observed evidence; state the chain.
3. **Unknown** — not checked or not provable from the available evidence.

Never claim a command was run, a test passed, a browser was checked, a file was
read, or a deployment succeeded without the reproducible command and output.
Screenshots are evidence of what was visible at one moment, not proof of server
authorization, database integrity, network ownership, or production readiness.

If an AI-generated change is not tested or inspected, say **NOT RUN**. If the
requested result depends on unavailable credentials, external state, or a
missing decision, say **BLOCKED** and request only that decision. Do not use
confidence, flattery, or a plausible explanation to fill an evidence gap.

## Production-Safe Command Policy

Before any command that can mutate data, code, process state, or deployment
state, identify the environment and show the exact target. In production,
default to read-only diagnostics and generated artifacts for human review.

Never execute these in production without explicit operator approval, a backup
or rollback plan, and a maintenance/incident record:

- `php artisan migrate:fresh`, `migrate:refresh`, `db:wipe`, `db:seed`, or an
  unscoped destructive data command.
- `php artisan tinker` that writes data, deletes records, changes permissions,
  dispatches jobs, or invokes domain mutations.
- `php artisan queue:flush`, `queue:restart`, `horizon:terminate`, or worker
  restarts without confirming queue impact and deployment ownership.
- `php artisan config:clear`, `cache:clear`, `route:clear`, `view:clear`, or
  `optimize:clear` during traffic without a release plan and cache warm-up.
- `composer update`, `composer remove`, `npm update`, `npm install` with
  dependency range changes, or lockfile rewrites on a live host.
- `git reset --hard`, `git clean -fd`, force pushes, direct production edits,
  or commands that replace uncommitted work.
- Applying Supervisor, crontab, systemd, Docker, Nginx, PHP-FPM, or firewall
  changes directly when only a configuration artifact was requested.

Safe default sequence: inspect → generate a dry-run/receipt → review target and
blast radius → test in a production-like environment → deploy through the
team's approved mechanism → verify health and business behavior → record the
result. When a dangerous command appears necessary, provide it as **INFO ONLY**
with its risk and ask the user to run or approve it; do not run it yourself.

## Struktur Backend dan Quality Gate

Baca `references/code-structure-and-quality.md` ketika mengubah backend Laravel.
Profil default guard adalah `balanced` dengan batas 100 baris fisik di `app/`.
Gunakan `strict` untuk boundary kecil/berisiko tinggi. Batas 101–200 hanya
boleh melalui profil `exception` dan manifest ber-owner, expiry, alasan, serta
receipt. Pecah berdasarkan tanggung jawab, bukan membuat abstraction berlapis
untuk memindahkan satu baris. Urutan method tetap `public` → `protected` →
`private`, dan inheritance aplikasi maksimal dua tingkat. Utamakan komposisi
dengan boundary yang mudah dicari.

Guard wajib dijalankan sebelum handoff. PHP Insights harus minimal 50 untuk
quality, complexity, architecture, dan style; skor 80 atau lebih adalah target
hijau, tetapi jangan mengorbankan keterbacaan untuk mengejarnya. `composer test`
dan `composer quality` harus menyertakan output nyata, bukan klaim.

## UI and Delivery Rules for Laravel ERP

For Blade + Bootstrap ERP screens:

- Indonesian is the default interface language; use translation keys for new
  user-facing copy and keep English as an explicit alternate locale.
- A screen must support a readable `complete` mode and a denser `compact` mode;
  density must not remove labels, status text, keyboard access, or primary row
  actions.
- Table primary actions stay visible in the row. If a secondary dropdown is
  used, it must be collision-safe on the last visible row and must not be
  clipped by a table footer or overflow container.
- Live table filters use a 300–400 ms debounce, preserve every active filter in
  the URL, reset only the page number, abort stale requests, and show a visible
  active-filter summary. With JavaScript, prefer fetch + partial replacement +
  `history.pushState`; retain a normal GET fallback without JavaScript.
- Every filter request has `aria-busy`, disabled controls, a table loading
  overlay that keeps old data as context, and a clear loading detail. Use an
  inline summary for successful filtering; use toast/SweetAlert for request
  feedback or notifications, not for every keystroke.
- Native select is preferred for 2–3 fixed options. More than 3 or dynamic
  options must be searchable. Tom Select is the default optional lazy capability
  for this Bootstrap stack; Select2 is an explicit jQuery tradeoff. Livewire
  adapters must account for `wire:ignore` and destroy/re-init lifecycle.
- Compact mode reduces prose and spacing only. It never hides tags, status text,
  labels, loading/error/recovery state, keyboard paths, or primary actions. A
  controlled set of status/actions uses shared width and height tokens so rows
  remain vertically aligned.
- Default sorting is a business decision, never an incidental database order.
  Document date precedence, stable tie-breakers, multi-column sorting, and
  A–Z/Z–A text sorting; cover each with a Pest test.
- Keep browser-local modal, toast, tab, collapse, and filter state local. Load
  JavaScript libraries per capability/page where the bundler supports it.

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
| Production safety, AI accountability, command guardrails | `references/production-safety-and-ai.md` |

### UX / UI

| Topic | File |
|---|---|
| Typography, color, contrast, spacing | `references/visual-design.md` |
| Table alignment, row design, actions | `references/table-design.md` |
| Forms: create, edit, multi-step, errors, validation | `references/forms.md` |
| Modal, drawer, full page — when to use which | `references/navigation.md` |
| Input type decisions: free text, dropdown, combobox, contextual help | `references/input-control.md` |
| Live filter, searchable select, loading, alert/toast, compact mode | `references/filtering-and-feedback.md` |
| Taste Skill yang diadaptasi untuk shell, table, form, registry preview | `references/erp-taste-profile.md` |
| Registry coverage, source ownership, private distribution, AI evidence | `references/registry-catalog.md` |
| Core ERP principles, do/don't, data integrity | `references/erp-principles.md` |

### Operational Engineering

| Topic | File |
|---|---|
| Struktur backend, profil 50/100/exception, inheritance, PHP Insights | `references/code-structure-and-quality.md` |
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

## Ownership Map (Single Source of Truth)

Cross-cutting concepts appear in multiple files for context. When in doubt,
read the **canonical owner** first; other files reference it but should not be
treated as authoritative for the same concept.

| Concept | Canonical Owner | Other files (context only) |
|---|---|---|
| Idempotency pattern (pseudocode, key lifetime, key source) | `concurrency.md` § Idempotency | `money-and-data-integrity.md`, `exceptions-and-recovery.md`, `forms.md` |
| Audit event structure (before/after, ORM diffs, libraries) | `observability.md` § Mutation History | `money-and-data-integrity.md` (financial-specific rules only), `erp-principles.md` (principle only) |
| Money-as-integer storage and rate snapshots | `money-and-data-integrity.md` | `forms.md` (UI display only), `indonesia-compliance.md` (IDR formatting) |
| Database transactions (syntax, anti-patterns, stack notes) | `money-and-data-integrity.md` § Database Transactions | `concurrency.md` (defers) |
| Optimistic / pessimistic locking | `concurrency.md` | `exceptions-and-recovery.md`, `offline-and-network.md` (cite only) |
| Pagination — server strategy (cursor/offset, indexing) | `performance.md` § Pagination Doctrine | `table-design.md` (defers) |
| Pagination — UI (page numbers, totals, infinite-scroll exceptions) | `table-design.md` § Pagination | `performance.md` (defers) |
| Loading state — principle (every round-trip needs feedback) | `erp-principles.md` § Loading & Feedback States | (cited from many) |
| Loading state — duration thresholds (which indicator when) | `performance.md` § Loading State Thresholds | `erp-principles.md` (defers) |
| Operator psychology (panic, confirmation fatigue, fear) — the WHY | `human-factors.md` | (cited from many) |
| Recovery/defense patterns (undo, soft delete, accidental submit) — the HOW | `exceptions-and-recovery.md` | `human-factors.md` (defers for implementation) |
| Modal anatomy (title, body, buttons, ESC/Enter, sizing) | `navigation.md` § Modal | `exceptions-and-recovery.md`, `human-factors.md` (defer) |
| Confirmation philosophy (when to require, friction by severity) | `human-factors.md` § Confirmation Fatigue | `exceptions-and-recovery.md` (defers) |
| Filter persistence (URL/session, checklist) | `erp-principles.md` § Don't reset filters | `table-design.md` (cites) |
| Live filter/request feedback contract | `filtering-and-feedback.md` | `erp-principles.md`, `performance.md`, `table-design.md` |
| Empty state — UI presentation | `table-design.md` § Empty State | `human-factors.md` (microcopy only) |
| Empty state — microcopy | `human-factors.md` § Trust-Building Microcopy | `table-design.md` (cites) |
| Error message microcopy | `human-factors.md` § Error Messages That Help | `erp-principles.md` (principle only) |
| Unsaved-changes warning | `forms.md` § Unsaved Changes Warning | `exceptions-and-recovery.md` (cites), `navigation.md` (cites) |
| Autosave to draft | `offline-and-network.md` § Autosave Draft | `forms.md` (cites), `exceptions-and-recovery.md` (cites) |
| Retry logic — server (webhook idempotency) | `concurrency.md` § Payment Callback Retries | `observability.md` (cites) |
| Retry logic — client (exponential backoff) | `offline-and-network.md` | `concurrency.md` (cites) |
| Session expiry | `exceptions-and-recovery.md` § Session Expiration | `offline-and-network.md` (cites) |
| PII handling (NIK, NPWP, BPJS, redaction) | `indonesia-compliance.md` | `observability.md` (never-log list cites) |
| Backend file size, method order, inheritance, PHP Insights thresholds | `code-structure-and-quality.md` | `production-safety-and-ai.md` (evidence only) |
| Taste rules yang benar-benar cocok untuk ERP | `erp-taste-profile.md` | Taste Skill source (adapted, not copied) |
| Severity classification (CRITICAL/HIGH/MEDIUM/PREFERENCE) | `severity.md` | (cited from all) |
| Priority hierarchy (financial > legal > integrity > ...) | `tradeoffs.md` | (cited from all) |

If two files appear to disagree, the canonical owner wins. Open an issue on
the non-canonical file — it has drifted.

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
