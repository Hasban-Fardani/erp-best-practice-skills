# Severity Classification

Not all ERP rules carry equal weight. Reason proportionally:
match the effort of pushback to the severity of the violation.

When you cite a rule from another reference file, also state its severity.
When you find a violation in review mode, classify it before recommending a fix.

## Levels

| Level | Meaning | Example | Response in review |
|---|---|---|---|
| **CRITICAL** | Causes financial loss, data corruption, legal exposure, or unrecoverable state | Money stored as float; no idempotency on payment webhook; NIK sent to external AI | **Block release.** Must fix before merge. |
| **HIGH** | Breaks daily operator workflow; silent data quality decay; major rework cost later | Placeholder-as-label; free text on a categorical reporting field; no audit trail on financial mutations | **Fix before next release.** Document if deferred. |
| **MEDIUM** | Slows operators; causes confusion; recoverable with workaround | Inconsistent required-field marking; weak empty-state copy; missing helper text | **Schedule for cleanup sprint.** Track. |
| **PREFERENCE** | Style or convention that has alternatives of similar quality | Zebra stripe vs horizontal line; drawer width 480 vs 520; specific accent color | **Suggest only if asked.** Do not block. |

## Classification Rules

**CRITICAL** — meets at least one:
- Money: precision loss, double-charge, missing audit, missing idempotency
- Data integrity: race condition that can corrupt state, missing transaction boundary
- Legal: PII leaked to logs / external AI / unencrypted storage, missing required field for tax/labor law
- Security: privilege escalation, missing authorization check on financial mutation
- Unrecoverable: no way to undo or reconstruct a destructive action

**HIGH** — meets at least one:
- Daily blocker for a primary persona (Sri can't complete a routine task)
- Silent decay (free-text categorical → reports degrade over months)
- Workflow regression (replaces a working spreadsheet with something worse)
- Forces operator into out-of-system workaround (WhatsApp the dev, export to Excel)
- Compounds with scale (fine at 10 records, broken at 10k)

**MEDIUM** — meets at least one:
- Friction during normal operation, but task still completable
- Cosmetic inconsistency that erodes trust over time
- Missing affordance for a less-common workflow
- Onboarding pain that lifts after a week of use

**PREFERENCE** — none of the above. The rule has documented alternatives that
work for different teams. Bring it up only if directly asked or if the codebase
already follows the convention.

## Proportional Response

The severity determines how hard you push back, not whether you push back.

| Severity | Wording when recommending a fix |
|---|---|
| CRITICAL | "This must change before release. Reason: ..." |
| HIGH | "Strongly recommend fixing this. Reason: ... If deferred, track explicitly." |
| MEDIUM | "Worth fixing. Reason: ... Reasonable to bundle into next cleanup." |
| PREFERENCE | "One option is ... Another team might do ... Either is fine." |

Do not escalate PREFERENCE to CRITICAL by tone. Do not soften CRITICAL into PREFERENCE
to avoid friction with a stakeholder. Severity is a property of the rule, not the conversation.

## Common Mistakes in Classification

❌ Treating typography (font weight choice) as CRITICAL.
✓ Typography is mostly PREFERENCE. Contrast ratio failing AA is HIGH (older users can't read it).
   Storing salary as float is CRITICAL.

❌ Treating idempotency as "nice to have."
✓ Idempotency on financial operations is CRITICAL. The first duplicate payroll run
   teaches everyone why.

❌ Treating "we'll add the audit trail later" as MEDIUM.
✓ Missing audit trail on financial mutations is CRITICAL. Retrofitting it is unreliable
   and the gap period can never be reconstructed.

❌ Treating mobile parity as CRITICAL for back-office work.
✓ Mobile parity for warehouse scanning = HIGH. For finance month-end close = PREFERENCE.
   Context decides. See [[mobile-erp]].

## Severity Index for Existing Rules

Quick lookup for the most-cited rules across the skill:

| Rule | File | Severity |
|---|---|---|
| Money stored as integer (sen), not float | [[money-and-data-integrity]] | CRITICAL |
| Idempotency key on financial operations | [[money-and-data-integrity]] | CRITICAL |
| Multi-table financial ops in one transaction | [[money-and-data-integrity]] | CRITICAL |
| Audit trail on financial mutations | [[money-and-data-integrity]] | CRITICAL |
| Rates stored at document creation time | [[money-and-data-integrity]] | CRITICAL |
| Rates from config, not hardcoded | [[money-and-data-integrity]] | HIGH |
| NIK/NPWP encryption + no external AI | [[indonesia-compliance]] | CRITICAL |
| Database-level constraints (FK, unique, check) | [[erp-principles]] | HIGH |
| UTC storage + timezone display | [[indonesia-compliance]] | HIGH |
| WCAG AA contrast (≥ 4.5:1) | [[visual-design]] | HIGH |
| Label above field (not placeholder-as-label) | [[forms]] | HIGH |
| Error summary banner on 10+ field forms | [[forms]] | HIGH |
| Filter persistence across back navigation | [[erp-principles]] | HIGH |
| Controlled input on categorical reporting fields | [[input-control]] | HIGH |
| Unsaved changes warning | [[forms]] | HIGH |
| Status badge = color + icon + text | [[visual-design]] | HIGH |
| Explicit submit button label ("Save Quotation") | [[forms]] | MEDIUM |
| Tabular numerals in number columns | [[table-design]] | MEDIUM |
| Single-column form layout | [[forms]] | MEDIUM (see exceptions there) |
| Drawer vs modal vs page sizing thresholds | [[navigation]] | MEDIUM |
| Specific font weight values (400/500/600/700) | [[visual-design]] | PREFERENCE |
| Horizontal-lines-only vs zebra stripe | [[table-design]] | PREFERENCE |
| Inter vs Roboto vs Nunito font family | [[visual-design]] | PREFERENCE |

See [[tradeoffs]] for how severity interacts with conflicting priorities.
