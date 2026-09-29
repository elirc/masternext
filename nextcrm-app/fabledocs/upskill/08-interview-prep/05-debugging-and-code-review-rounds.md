# Debugging and Code Review Rounds

Timed simulations. Do them against a clock; the interview skill is narrating your reasoning aloud, not just reaching the answer.

---

## Debugging Round 1 (15 min): "Some invoices have duplicate numbers"

Setup: production reports two invoices with number `INV-2026-0042`.
Your narration should hit, in order:
1. **Reproduce**: can't from one request — hypothesize concurrency; write a script firing two `issueInvoice` promises on one series.
2. **Locate**: `consumeNextNumber` is read-modify-write ([numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30)).
3. **Explain the safety net**: the Serializable wrapper ([issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128)) normally prevents it.
4. **Form hypotheses**: a second caller without Serializable (`rg "consumeNextNumber"`), or serialization aborts being swallowed/retried wrong.
5. **Fix root cause**: relocate the lock into the helper (`SELECT FOR UPDATE`) so it's caller-independent.
6. **Regression**: N-parallel-issuance integration test asserting distinct numbers.
Interviewer follow-ups: "Why not just retry?" (works, but the invariant should be owned by the function). "What does Serializable cost?" (abort-retry under contention). Grading: **Strong** = you reproduced by *forcing* concurrency and fixed by relocating the invariant, not by adding a band-aid retry.

## Debugging Round 2 (15 min): "Enrichment overwrote a user's manual edit"

Setup: a rep edited a contact's phone; minutes later it reverted to an old value.
Narration:
1. **Reproduce**: trigger enrichment, edit a field mid-run, observe revert.
2. **Locate**: merge applies to fields empty at the *read* ([enrich-contact.ts:64-80, 123-138](../../../inngest/functions/enrich-contact.ts#L120-L138)) — a multi-second gap between read and write.
3. **Classify**: lost update (read-then-write-later race).
4. **Cheapest probe**: confirm the field was empty at enrichment start (logs/timestamps).
5. **Fix**: re-check emptiness at write time via a scoped `updateMany` that only writes still-empty columns, or version/timestamp guard.
6. **Regression**: conditional-write unit test.
Follow-ups: "Is Inngest retrying making it worse?" (retries re-run the whole fn incl. the write). Grading: **Strong** = named it a lost update and proposed a write-time re-check, not just "add a lock."

## Debugging Round 3 (12 min): "Saved but the list is stale"

Setup: account edit persists but the list shows old data.
Narration: bisect write-vs-cache first (re-fetch detail page → new value? then it's cache). Locate `revalidatePath` ([update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62)); check the path arg matches the list segment, or a client component mirrors server data in `useState`. Fix root cause (correct revalidate / stop mirroring). Regression: E2E edit→navigate→assert. Grading: **Strong** = your first probe splits the search space in half instantly.

## Debugging Round 4 (12 min): "Webhook shows delivered but our status stuck on 'sent'"

Setup: intermittent; some sends never move to delivered.
Narration: hypothesize ordering — the delivered webhook arrived before our own `status: "sent"` write; the handler guards `if (send.status === "sent")` ([resend route:35-42](../../../app/api/campaigns/webhooks/resend/route.ts#L34-L42)) so an early callback is dropped forever. Probe: unit-test the handler with `status: "queued"` + delivered event. Fix: broaden the guard to any pre-delivered status (order-independent reconciliation). Grading: **Strong** = you distrusted your own write landing before a third-party callback.

---

## Review Round 1 (15 min): the "remove Serializable for speed" PR

Use [review kata 2](../04-code-reading-gym/04-review-katas.md). Deliver a spoken review:
- **Block** with the concrete failure: duplicate legal numbers under concurrency; the "errors" being removed are the isolation working.
- **Educate kindly**: point at numbering.ts:13 and explain the read-modify-write.
- **Counter-propose**: FOR UPDATE in the helper (kata A) or retry-on-abort; ask for a load test before any isolation change.
Follow-up: "The author says it's slow — are you just blocking them?" → No: I'm offering a faster-safe path and a way to measure the actual contention. Grading: **Strong** = turned a bad PR into the right PR.

## Review Round 2 (15 min): the "new opportunities PATCH endpoint" PR

Use [review kata 1](../04-code-reading-gym/04-review-katas.md). Spoken review must:
- **Block**: reintroduces the advisory's IDOR pattern (no object-level scope) + mass assignment (raw body spread).
- **Cite precedent**: the contacts route + `tryScopedUpdateContact` as the template to copy.
- **Important**: missing `deletedAt` guard, missing test.
- Offer to pair on the scope OR-clause.
Follow-up: "How would you make sure this class of bug can't come back?" → tests per write path + a lint rule / enforcement wrapper (Project 1). Grading: **Strong** = you named the systemic fix, not just the local one.

---

## Self-scoring for all rounds

- **Basic**: reached a plausible cause/finding.
- **Solid**: correct method (reproduce→narrow→hypothesize→fix→test) or correct B/I/O triage, spoken aloud.
- **Strong**: your first probe/comment collapsed the problem space, you fixed the *root* (relocated the invariant, order-independent reconciliation, systemic enforcement), and you named the regression test. That's the mid→senior line interviewers are scoring.
