# Production Safety & AI Accountability

> **Scope.** Owns the operating stance for AI-assisted ERP work: production
> assumptions, evidence contracts, command guardrails, change boundaries,
> release verification, and honest reporting. This file is the canonical owner
> for whether an action may be executed or must be returned as information only.

## Default Environment Assumption

Assume the application is already in production unless the user explicitly
labels it a disposable sandbox. The code checkout may be local, but the
database, queues, scheduled jobs, uploads, credentials, and operators are
real. A request to make a change is not permission to alter production data or
process state.

If the environment is not proven, mark it **UNKNOWN**. Do not silently infer
that `APP_ENV=local`, a localhost URL, or a test database represents the
deployment target.

## Evidence Contract

Every report and handoff separates these categories:

| Category | Meaning | Acceptable evidence |
|---|---|---|
| **OBSERVED** | Directly established fact | File contents, exact command output, test result, browser trace, deploy receipt |
| **INFERRED** | Conclusion derived from observed facts | State the facts and the reasoning chain |
| **UNKNOWN** | Not checked or not provable | Name the missing check or decision |
| **NOT RUN** | A required check was intentionally not executed | Explain why and what remains |
| **BLOCKED** | Progress cannot continue without a decision or external state | Ask for the smallest unblocker |

Never write “verified”, “tested”, “fixed”, “deployed”, or “done” without the
command, scope, and output that support the word. A screenshot proves only the
visible state captured in that moment. It does not prove authorization,
database integrity, network ownership, queue behavior, accessibility, or
production readiness.

For AI-generated work, require at least:

1. A source diff review.
2. A deterministic test for the changed server behavior.
3. A build check for changed frontend assets.
4. A browser check for changed interaction or layout.
5. A second-pass review that looks for scope drift, hidden network calls,
   unsafe commands, missing authorization, and unsupported claims.

If one is missing, report the exact status instead of compensating with a
confident explanation.

## Command Guardrails

### Read-only by default in production

The safe default sequence is:

```text
inspect target → generate dry-run/receipt → review blast radius
→ test in production-like environment → approved deployment
→ health check → business smoke check → receipt
```

Commands that inspect configuration, routes, schema, logs, processes, release
manifests, or generated artifacts are generally read-only, but still confirm
the target before running them.

### Do not execute these in production without explicit approval

- `php artisan migrate:fresh`, `migrate:refresh`, `db:wipe`, `db:seed`, or an
  unscoped destructive data command.
- Write-capable `php artisan tinker` or database consoles.
- `php artisan queue:flush`, `queue:restart`, `horizon:terminate`, or worker
  restarts without queue-impact review.
- `php artisan config:clear`, `cache:clear`, `route:clear`, `view:clear`, or
  `optimize:clear` during traffic without a release/cache-warm plan.
- `composer update`, `composer remove`, `npm update`, or lockfile rewrites on a
  live host.
- `git reset --hard`, `git clean -fd`, force push, or direct edits that can
  discard work.
- Applying Supervisor, crontab, systemd, Docker, Nginx, PHP-FPM, firewall, or
  permission changes when the request only asks for a plan or artifact.

When one of these appears necessary, return it as **INFO ONLY** with the
reason, target, blast radius, rollback, and approval needed. Do not run it.

## Laravel ERP Change Boundary

- Browser-local modal, toast, dropdown, tab, collapse, and filter state must
  remain local; do not add Livewire/fetch merely to change cheap UI state.
- Every business mutation needs an explicit HTTP/server boundary, policy,
  validation, transaction/concurrency decision, audit decision, and Pest test.
- Default UI language is Indonesian. New copy uses translation keys; English is
  an explicit alternate locale, not an accidental default.
- `complete` and `compact` modes may change density and explanatory copy, but
  never remove primary actions, status text, labels, keyboard access, or error
  recovery.
- Table primary actions remain visible in the row. Any secondary dropdown must
  avoid clipping at the last row and must be tested against overflow containers.
- Default sorting is an explicit business rule. Document date precedence,
  stable tie-breakers, multi-column sorting, and text A–Z/Z–A behavior; test
  each rule.
- Use lazy capability imports for frontend libraries when Vite can split the
  chunk without making the interaction unreliable. Measure the resulting
  entry and async chunk sizes.

## Review Questions Before Handoff

- What changed, and why is each file in scope?
- Which facts were observed versus inferred?
- What production data, process, queue, or deployment state could this touch?
- Which command was run, against which target, and what was its output?
- What remains unknown or not run?
- Can a junior developer locate the source, server boundary, and test without
  following more than two indirections?

## Verification Failure and Improvisation Log

A failed verification attempt is evidence about the tool boundary, not proof
that the implementation is wrong or right. Record the exact failed command,
the observed error, the minimal correction, and the successful rerun.

- Check `--help` or the project script before adding flags from memory; command
  names that look alike do not guarantee the same options.
- Do not call Laravel helpers such as `base_path()` while a test file is being
  loaded before the application container boots; use a deterministic path for
  tool fixtures or boot the proper test case.
- Use the repository's or workspace's bundled runtime when available. If a
  validator lacks a package, install it into an isolated temporary location or
  report `NOT RUN`; never weaken validation silently.
- When a quality command reports pre-existing failures, run a targeted check on
  changed files, preserve the full result as `FAIL`, and do not relabel it as
  green without a baseline receipt.
