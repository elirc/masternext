# Systematic Debugging

The method: **reproduce → narrow (bisect the layers) → hypothesize → test cheaply → fix root cause → add regression coverage.** Never fix what you can't reproduce; never close a bug without the test that would have caught it.

The repo's layers give you natural bisection points: browser ↔ server action/route ↔ lib policy ↔ Prisma ↔ Postgres ↔ Inngest ↔ external service. For each bug, the first question is *which boundary is the value wrong at?*

Tools available: browser devtools + network tab; Node debugger / `console.error` tags (grep for `[UPDATE_ACCOUNT]`, `[ISSUE_INVOICE]`, `[AUDIT_LOG_WRITE_FAILED]`); Prisma query logging (dev logs `error`/`warn` — [lib/prisma.ts:17](../../../lib/prisma.ts#L17); bump to `["query"]` to see SQL); the Inngest dev dashboard for function runs/retries; Playwright trace viewer on failures.

---

## Scenario 1: "My account edit saved but the list still shows the old name"

Reproduction: edit an account, navigate back to the list, see stale data.
First question: is the write wrong (DB) or the read stale (cache)?
Narrowing:
1. Query the DB directly / re-fetch the detail page — new value present? → write is fine, it's cache.
2. Check the action for `revalidatePath` — [update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62). Present? Check the *argument* matches the list route segment.
Likely root cause: a `revalidatePath` with the wrong path string, or a client component holding server data in `useState` and never re-reading ([fake-code #9](../04-code-reading-gym/03-fake-code-contrasts.md)).
Regression test: E2E — edit, navigate, assert new value visible.
Senior lesson: "saved but stale" is almost never a DB bug; cache invalidation is the usual suspect in RSC apps.
Interview version: "Walk me through debugging a stale-data bug in a Next.js app" — narrate the write-vs-cache bisection.

## Scenario 2: "Two invoices got the same number"

Reproduction: hard — needs concurrency. Reproduce with two simultaneous `issueInvoice` calls on the same series (a script firing two promises).
First question: is numbering non-atomic, or did someone bypass the transaction?
Narrowing:
1. Confirm both calls hit the same series and overlapped in time (logs/timestamps).
2. Read the call site: is `{ isolationLevel: "Serializable" }` present at [issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128)? Is there a *second* caller of `consumeNextNumber` (`rg "consumeNextNumber"`) that omits it?
3. Enable query logging; observe whether both transactions read the counter before either wrote.
Likely root cause: a new caller without Serializable, or Serializable aborts being swallowed and retried incorrectly.
Cheap test: integration test firing N parallel issuances, asserting N distinct numbers.
Regression: that test, in CI.
Senior lesson: correctness invariants owned by callers *will* be violated by the next caller; the fix is to move the invariant into `consumeNextNumber` (lock internally). Interview version: this is the whole "how do you debug a race condition" answer — reproduce by forcing concurrency, confirm the read-modify-write window, fix by relocating the lock.

## Scenario 3: "Enrichment overwrote a field a user just edited"

Reproduction: trigger enrichment; while it runs, edit one of the target fields in the UI; the field reverts.
First question: read-time vs write-time emptiness.
Narrowing: [enrich-contact.ts:123-138](../../../inngest/functions/enrich-contact.ts#L123-L138) — the emptiness check uses `contact` loaded at [:65](../../../inngest/functions/enrich-contact.ts#L64-L80), before the agent ran. If the field was empty *then* and the user filled it *during*, enrichment still writes.
Likely root cause: lost update — the classic read-then-write-later race, here with a multi-second gap.
Cheap test: unit-test the merge logic proves the *intended* behavior; the bug is timing, so the real fix is a final re-read inside a scoped `updateMany` that only writes still-empty fields (`where: { id, position: null }`-style).
Regression: hard to E2E; document the invariant and add a conditional-write test.
Senior lesson: "apply only to empty fields" is idempotency-flavored but not concurrency-safe across a long-running job.

## Scenario 4: "Webhook says delivered but our status is still 'sent'... sometimes"

Reproduction: intermittent; correlate a send whose `update-send-record` step and delivered webhook are close in time.
First question: ordering — did the delivered webhook arrive before we wrote `status: "sent"`?
Narrowing: [resend route:35-42](../../../app/api/campaigns/webhooks/resend/route.ts#L34-L42) only promotes to delivered `if (send.status === "sent")`. If delivery beat the send-record write ([send-step.ts:53-68](../../../inngest/functions/campaigns/send-step.ts#L53-L68)), the guard skips and status stays wrong permanently.
Likely root cause: a race between our own status write and the provider's callback, with a guard that assumes our write wins.
Cheap test: unit-test the webhook handler with `status: "queued"` and event `delivered` — observe it's dropped.
Regression: that test + broadening the guard to accept any pre-delivered status.
Senior lesson: never assume your write lands before a third party's callback; make reconciliation order-independent.

## Scenario 5: "PDF is blank / issuance 'failed' but the invoice is ISSUED"

Reproduction: issue an invoice with MinIO misconfigured.
First question: did issuance fail or did the *PDF* fail? They're decoupled by design.
Narrowing: [issue-invoice.ts:131-207](../../../actions/invoices/issue-invoice.ts#L131-L207) — the transaction (status→ISSUED, number assigned) commits first; PDF runs after in a try/catch that logs `[ISSUE_INVOICE] PDF generation failed` and swallows. So "ISSUED with no PDF" is the *expected* degraded state, not corruption.
Likely root cause: MinIO creds/bucket, or the live-account PDF data path ([risk #10](../09-reference/risk-register.md)).
Fix: `regenerate-pdf` once storage is fixed ([regenerate-pdf.ts](../../../actions/invoices/regenerate-pdf.ts)); no data repair needed.
Senior lesson: recognize *intentional* partial-failure states before you "fix" them — the log tag and comment tell you it's a decision. Interview version: "how do you debug when part of an operation succeeds and part fails?" — identify the transaction boundary first; everything after it is best-effort here.

## Meta-drill

Pick any scenario, and before reading the narrowing steps, write your own bisection plan and the single cheapest probe you'd run first. Self-grade: Strong = your first probe would falsify a whole half of the search space (e.g., "re-fetch detail page" instantly separates write-bug from cache-bug in Scenario 1).
