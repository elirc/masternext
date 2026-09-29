# Side Effects, Async, and Reliability

## Inventory of side effects

| Side effect | Trigger | Where | Reversible? |
| --- | --- | --- | --- |
| Campaign emails to humans | Inngest `campaigns/send-step` | [send-step.ts:40-51](../../../inngest/functions/campaigns/send-step.ts#L40-L51) | **No** — least reversible thing in the repo |
| OTP / invite / notification emails | auth + actions | [lib/auth.ts:70-89](../../../lib/auth.ts#L70-L89), [emails/](../../../emails/) | no |
| Invoice PDF → MinIO | after issue tx | [issue-invoice.ts:197-203](../../../actions/invoices/issue-invoice.ts#L197-L203) | yes (regenerate) |
| AI enrichment writes + API spend | Inngest `enrich/*` | [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts) | field writes yes; **spend no** |
| Embedding refresh | `crm/account.saved` etc. | [update-account.ts:61](../../../actions/crm/accounts/update-account.ts#L61) → embed-* fns | yes (recompute) |
| FX rates fetch (ECB) | scheduled + on issue | [lib/invoices/fx.ts](../../../lib/invoices/fx.ts), inngest/functions/ecb | n/a |
| IMAP mailbox sync | scheduled | [inngest/functions/emails/](../../../inngest/functions/emails/) | re-syncable |
| Audit rows | every CRM mutation | [audit-log.ts:66-82](../../../lib/audit-log.ts#L66-L82) | append-only by design |

## Reliability concepts, as this repo implements them

**Idempotency** — an operation you can run twice with the effect of once:
- *By construction*: webhook status updates guard with current-state checks (`if (send.status === "sent")`, `if (!send.opened_at)` — [resend route:34-67](../../../app/api/campaigns/webhooks/resend/route.ts#L34-L67)). Redelivered webhooks are harmless.
- *By memoization*: `step.run("send-email", ...)` ([send-step.ts:40](../../../inngest/functions/campaigns/send-step.ts#L40-L51)) — Inngest persists each step's result; a retry after the send **replays the recorded result instead of re-sending**. This is the entire reason the function is written as steps.
- *By windowing*: the 7-day enrichment dedup ([enrich-contact.ts:91-106](../../../inngest/functions/enrich-contact.ts#L91-L106)) is really a cost-control idempotency key with a very long TTL.
- *Missing*: the Resend webhook has no event-id dedup (fine today because updates are conditional; becomes a bug the day someone adds a counter increment).

**Retries** — `retries: 3` on enrichment ([enrich-contact.ts:37](../../../inngest/functions/enrich-contact.ts#L32-L38)). Because that function does **not** use `step.run`, every retry re-executes the paid agent call and every DB write. Retry design rule: *steps make retries cheap; retries without steps make failures expensive.*

**Ordering & TOCTOU** — the paused check happens once at load ([send-step.ts:32](../../../inngest/functions/campaigns/send-step.ts#L32)); a pause landing mid-flight doesn't stop the current email. Acceptable (one email), worth knowing. Same shape as the numbering counter, solved there by Serializable, unsolved (and low-stakes) here.

**Partial failure policy** — three explicit stances worth quoting in interviews:
1. *Tolerate*: PDF generation failure never blocks legal issuance ([issue-invoice.ts:204-207](../../../actions/invoices/issue-invoice.ts#L204-L207)) — comment says "Do NOT fail".
2. *Swallow*: audit-log failure never blocks the mutation ([audit-log.ts:78-81](../../../lib/audit-log.ts#L78-L81)) — availability over completeness of the trail (defensible; also means audit gaps are invisible — no metric).
3. *Fire-and-forget*: embedding refresh events `void`-ed ([update-account.ts:61](../../../actions/crm/accounts/update-account.ts#L61)) — search staleness accepted; **no outbox**, so a crash between commit and send loses the event permanently.

**The missing pattern: transactional outbox.** State changes and their events are written in separate steps everywhere (`update` then `inngest.send`). An outbox (event row written in the same transaction, relayed by a poller) would make events exactly-as-durable as data. For this app's stakes, the simplicity trade is defensible — but you should be able to *name* the pattern being declined. That's the [refactor kata](../06-contribution-practice/04-refactor-and-design-katas.md).

**Backpressure/timeouts** — enrichment rate limiting is per-IP fixed-window via Upstash ([rate-limit.ts:18-23](../../../lib/enrichment/rate-limit.ts#L18-L23)); bulk enrichment fans out one event per row (see [enrich-contacts-bulk.ts](../../../inngest/functions/enrich-contacts-bulk.ts)); Inngest provides the queue between trigger and execution. No explicit HTTP timeouts on outbound fetches (investigate — default fetch has none; a hung Firecrawl call holds a function slot).

## Failure visibility

"How would I know it broke?" — mostly `console.error` with grep-able tags (`[ISSUE_INVOICE]`, `[AUDIT_LOG_WRITE_FAILED]`, `[UPDATE_ACCOUNT]`), enrichment status rows visible in UI, campaign send statuses on the campaign page, Inngest's own dashboard for function failures. No metrics, no alerting, no Sentry. Full treatment in [05-quality-engineering/06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md).

## Drills

1. List every write in `campaignSendStep` and mark: memoized / conditional / unconditional. Then answer: after a crash between `send-email` and `update-send-record`, what does the DB say vs what did the world see? (DB: status unchanged; world: email sent. Reconciliation: the delivered webhook will *not* fix status because it requires `status === "sent"` — [resend route:36](../../../app/api/campaigns/webhooks/resend/route.ts#L36). Confirmed gap chain — trace it yourself.)
2. Design the outbox version of `updateAccount`'s embedding event on paper: table shape, relay, delete-vs-mark, and what new failure mode you introduced (relay lag).

## Interview angle

- "How do you guarantee an email is sent exactly once?" → you can't; at-least-once + memoized step + conditional status writes = effectively-once. [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md) variation 3.
- "What's an outbox and when is it overkill?" → the fire-and-forget embedding event is your worked example of *declining* it rationally.
