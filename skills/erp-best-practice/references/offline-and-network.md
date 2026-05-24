# Offline & Unstable Network Reality

The default assumption for Indonesian office ERPs should be **unstable network**,
not offline-first. Most operators are online most of the time, but the connection
drops for 5–30 seconds several times per day. The system must survive these drops
without losing work.

True offline mode (warehouse scanner in a remote location, field sales in a basement
without signal) requires additional design — covered at the end.

## Network Conditions to Design For

| Environment | Latency | Drop rate | Bandwidth | Notes |
|---|---|---|---|---|
| Jakarta office, wired | 20–60ms | < 0.1% | 50–100 Mbps | Best case |
| Jakarta office, WiFi | 50–150ms | 1–3% (brief drops) | 5–30 Mbps | Typical office |
| Suburb office, ADSL | 100–250ms | 3–8% | 5–20 Mbps | Common in regional offices |
| Mobile 4G (sales rep) | 100–500ms | 5–15% | 2–20 Mbps | Variable by area |
| Mobile 3G fallback | 300–1500ms | 10–30% | 0.5–3 Mbps | Real in rural areas |
| VPN (cross-region) | 200–600ms | 2–10% | varies | Often the bottleneck |

If your design assumes "request succeeds in < 200ms," you're designing for the
demo. Real operators see the spinner every day.

## Optimistic UI Boundaries

Optimistic UI (apply change locally, sync to server, reconcile) reduces perceived
latency. But misuse causes data corruption: the user thinks the action succeeded
when it silently failed.

**Use optimistic UI for:**
- Status toggles (mark as read, archive)
- Reordering items
- Adding/removing tags or labels
- UI-only preferences (column visibility, view mode)

**Never use optimistic UI for:**
- Financial mutations (payment, invoice creation, journal posting)
- Approvals and rejections
- Stock movements
- Anything with downstream effects (sending email, notifying customer)
- Any operation that requires a server-side ID to be assigned (invoice number)

For non-optimistic operations, show the spinner, accept the latency, return a
confirmed result. Lying about the state is worse than being slow.

## Retry States

Network failure must be visible and recoverable. Default behavior on a save failure:

1. **Detect the failure type:**
   - Timeout (no response in 30s)
   - 5xx server error
   - 4xx client error (validation, auth)
   - Network unreachable (`navigator.onLine === false` or fetch threw)

2. **Behavior by type:**

| Failure | UI response | Retry behavior |
|---|---|---|
| Timeout | "Save is taking longer than expected. [Retry] [Cancel]" | Manual retry, same idempotency key |
| 5xx | "Could not save (server error). We're notified. [Retry]" | Manual retry, same idempotency key |
| 4xx validation | "Could not save: [field] [reason]. [Fix and retry]" | User fixes, normal submit |
| 4xx auth (401/403) | Open re-auth modal, preserve form state, retry after auth | Auto-retry with same idempotency key |
| Network unreachable | "You appear to be offline. Your work is saved as a draft. [Retry when online]" | Auto-retry on `online` event |

3. **Always preserve the form state.** Never reset fields on failure. The
   operator filled them once; do not make her do it again.

## Idempotency Under Retry

Every retry of a financial or destructive operation must use the same idempotency
key as the original attempt. The client generates the key once, before the first
submit. See [[concurrency]] for the full pattern.

If the client cannot tell whether the original request succeeded (e.g., timeout
after submit), the retry must be safe. Idempotency makes this safe.

❌ BAD: timeout on first submit → client generates a new key for retry → server
   creates two invoices.

✅ GOOD: timeout on first submit → client retries with the *original* key →
   server recognizes the key, returns the previously-stored result or the
   in-progress state.

## Reconnect Messaging

When the connection drops, the operator should know:

```
[banner — top of page, sticky]
  ⚠ Connection lost. Trying to reconnect... (attempt 3 of 10)
```

When it returns:

```
[toast]
  ✓ Reconnected. Your work was preserved.
```

Detection:
```javascript
window.addEventListener('offline', () => showOfflineBanner());
window.addEventListener('online', () => {
    hideOfflineBanner();
    showReconnectedToast();
    retryPendingRequests();
});
```

The `online`/`offline` events are imperfect — they detect network adapter state,
not actual connectivity. Augment with periodic lightweight pings or by inferring
from API call success rates.

## Autosave to Draft

For any form with > 5 fields or > 30 seconds of input time, autosave to draft
every 60 seconds.

Rules:
- Save to a `draft_state` field on the record, or to a separate `drafts` table
  keyed by (user_id, resource_type, resource_id_or_temp_uuid)
- Draft state is **not** the same as a saved record — it does not affect
  reports, does not trigger workflows, does not show in lists
- Show "Draft saved 13:24" in a fixed position the operator can see while
  scrolling
- On reload, prompt: "You have a draft from 13:24. [Restore] [Discard]"
- Drafts expire after 7 days of inactivity, or when the record is saved as final

Anti-patterns:
- Autosaving to the final record (live saves while editing). Looks futuristic,
  causes accidental partial-save bugs and confuses reports.
- Autosaving sensitive fields to a server draft (NIK, salary). Use local storage
  with care, or skip autosave for these fields.
- Autosaving an approval state. Catastrophic — the operator thinks they're
  drafting an approval and it accidentally goes through.

## Draft Storage Layers

Three options, in order of robustness:

| Layer | Survives | Use for |
|---|---|---|
| In-memory (component state) | Page changes within SPA, until reload | Active session only — always combined with one of below |
| Local storage | Page reload, tab close, browser restart | Non-sensitive form drafts |
| Server-side draft table | Tab close, browser switch, device switch, multi-day | Long-form drafts (RAB, quotation with 50 line items) |

For mission-critical long forms: layer all three. In-memory updates immediately,
local storage every 5 seconds, server every 60 seconds. The fallback chain
covers every failure mode.

What NOT to put in local storage:
- NIK, NPWP, bank account numbers
- Salary amounts
- Anything classified Sensitive or Confidential per [[indonesia-compliance]]

Local storage is shared across tabs of the same origin, persists across crashes,
and is readable by any JS that runs on that origin (including XSS payloads).
Treat it as "barely-trusted client state."

## True Offline Mode

Required for:
- Warehouse scanners in areas without WiFi coverage
- Field sales reps in remote regencies
- Auditors at customer sites with no guest network

True offline requires:

1. **Local data cache.** Offline-capable storage (IndexedDB) holds the entity
   data the operator needs: customer list, product catalog, current orders.
2. **Local mutation queue.** Operator actions while offline are queued as
   "intent records" (e.g., "scan SKU-X, qty 1, location A-3-2, by user@X,
   at 14:23 local time").
3. **Sync engine.** When connection returns, the queue is drained to the server
   in order. Each mutation carries its idempotency key.
4. **Conflict resolution.** Server may reject a mutation (stock already shipped,
   record was deleted, version conflict). The sync engine surfaces conflicts
   to the operator, never silently discards.
5. **Status visibility.** UI shows "Offline — 12 actions queued for sync."
   On reconnect: "Syncing... 3 of 12." On completion: "Synced." On conflict:
   "2 actions need your review."

True offline is **expensive to build well.** Reserve it for scenarios where
genuine connectivity is unavailable. Do not add it for "robustness" on top of
an online-with-drops architecture.

## What "Unstable Network" Looks Like in Code

```typescript
async function submitWithRetry(payload: any, idempotencyKey: string) {
    const maxAttempts = 3;
    let lastError;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            const response = await fetch('/api/save', {
                method: 'POST',
                headers: {
                    'Idempotency-Key': idempotencyKey,
                    'X-Correlation-ID': crypto.randomUUID(),
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(payload),
                signal: AbortSignal.timeout(30_000),
            });

            if (response.ok) return await response.json();
            if (response.status === 409) {
                // Optimistic lock conflict — do not retry; surface to user
                throw new ConflictError(await response.json());
            }
            if (response.status >= 400 && response.status < 500) {
                // 4xx errors are user-facing; do not retry
                throw new ValidationError(await response.json());
            }
            // 5xx — retry with backoff
            lastError = new ServerError(response.status);
        } catch (err) {
            if (err instanceof ConflictError || err instanceof ValidationError) throw err;
            lastError = err;
        }

        // Exponential backoff: 1s, 2s, 4s
        if (attempt < maxAttempts) {
            await new Promise(r => setTimeout(r, 1000 * 2 ** (attempt - 1)));
        }
    }

    throw lastError;
}
```

Key properties:
- Same idempotency key across attempts
- New correlation ID per attempt (so logs distinguish them)
- Do not retry 4xx (it'll just fail the same way)
- Exponential backoff on 5xx and network errors
- Hard timeout (30s) on each attempt
- Surface conflict immediately, do not retry

## Common Failure Modes

❌ User clicks Save during a network drop. Spinner appears. Spinner never goes
   away. Sri waits 5 minutes, force-refreshes, loses all her work.
✅ Spinner appears with a timeout. After 10s, message: "This is taking longer
   than expected. We're saving your work locally. [Cancel] [Keep trying]"
   After 30s: "Could not connect. Your work is saved as a draft. [Retry when
   online]"

❌ Save succeeded on the server, but the response was lost in transit. User
   clicks Save again. Two invoices created.
✅ Idempotency key on the request. Server detects the duplicate, returns the
   first result. One invoice.

❌ "Sync" button manually pressed by operator. Sometimes fails silently.
✅ Sync runs automatically on `online` event. Status is visible. Failures
   surface as inline errors with retry, not silent.

❌ Form blocked from saving because validation API call (to check customer
   exists) timed out. Sri can't proceed.
✅ Validation failures from network errors are non-blocking. "Could not verify
   customer — saving as unverified. Verify before approval."

## Pre-Release Checklist: Network Reliability

- [ ] Every form > 5 fields has unsaved-changes warning
- [ ] Every long form (15+ fields) autosaves to draft every 60 seconds
- [ ] Draft state visible to user ("Draft saved 13:24")
- [ ] Form state preserved across failed submit (no field reset on error)
- [ ] Failed saves can be retried with same idempotency key
- [ ] Offline event handled: banner shows, work saved locally
- [ ] Online event handled: pending retries fire automatically
- [ ] Session expiry (401) opens re-auth modal, preserves form state
- [ ] No PII (NIK, salary, bank account) written to local storage
- [ ] Optimistic UI is restricted to safe, non-financial operations
- [ ] Network failure messages are specific and actionable

## Cross-References

- Recovery from session loss and partial submission: [[exceptions-and-recovery]]
- Idempotency and retry semantics: [[concurrency]]
- Background job status visibility: [[observability]]
- Mobile-specific offline scenarios: [[mobile-erp]]
- Performance budgets that interact with network: [[performance]]
