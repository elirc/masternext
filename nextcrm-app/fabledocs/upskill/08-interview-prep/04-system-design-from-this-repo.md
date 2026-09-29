# System Design From This Repo

The prompt: **"Design a CRM with invoicing from scratch."** Walk it as a whiteboard exercise, and at each step show what NextCRM actually chose (with anchors) plus a stronger/simpler alternative. This turns your code study into a design answer no memorized template can match.

Practice out loud, 30–40 minutes end to end. At each step, know the junior/mid/senior version of the answer.

## Step 1: Requirements (2–3 min)

Functional: manage accounts/contacts/leads/opportunities; issue invoices with tax + numbering; send cold-email campaigns; multi-user with roles. Non-functional: multi-user data isolation, correct money, auditability, background processing.
- Junior: lists CRUD entities. Mid: surfaces the hard constraints — money correctness, gap-free invoice numbers, per-user visibility, irreversible emails. Senior: asks the scoping question that shapes everything — **single shared workspace or true multi-tenant?** NextCRM chose shared-namespace-with-ownership ([no tenant column](../../../docs/2026-05-01-bola-idor-security-audit.md)); that decision cascades into every authz choice.

## Step 2: API sketch (5 min)

- App-driven mutations → server actions (typed, no client fetch): `createInvoice`, `updateAccount`. External/machine → REST routes: webhooks (Resend), presigned uploads, PDF download, MCP tools. NextCRM: [actions/](../../../actions/) vs [app/api/](../../../app/api/).
- Mid: notes both surfaces must authorize independently. Senior: flags the duplication risk (a route and an action doing the same write with different rigor) and picks one home per operation.
- Alternative: a single tRPC/GraphQL layer for uniform typing; simpler contract story but loses server-actions' zero-client-fetch ergonomics.

## Step 3: Data model (7 min)

- Core: Accounts as the hub; Contacts/Leads/Opportunities/Contracts referencing it; Invoices → LineItems → TaxRates; junction tables for many-to-many (watchers, doc links). NextCRM: [schema.prisma](../../../prisma/schema.prisma) (~80 models).
- Money: NUMERIC + decimal.js, per-line rounding, snapshots on issue. Numbering: counter row + Serializable, not a sequence. Soft delete via `deletedAt`.
- Mid: explains snapshots (immutability contract) and Decimal (no floats). Senior: names the numbering isolation requirement and where it should live (in the helper, not the caller); discusses index needs on ownership columns for scope queries.
- Alternative: event-sourced invoices (full history free) — heavier; NextCRM's snapshot+audit-log is the pragmatic middle.

## Step 4: AuthZ (5 min)

- Roles (admin/manager/user) + ownership fields; policy as composable `where` fragments intersected into every query; bulk ops filter ids in one query. NextCRM: [lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts).
- Mid: object-level checks in the query, not per-route ifs; 404-for-forbidden. Senior: the enforcement problem (nothing forces usage → the residual IDOR gap), and app-level scoping vs DB RLS given no tenant key.
- Alternative: Postgres RLS keyed on a tenant/owner column — strong default-deny, but requires a tenancy model NextCRM doesn't have; retrofitting is a rewrite.

## Step 5: Async / side effects (5 min)

- Durable-execution engine (Inngest) for enrichment, campaign sends, embeddings, scheduled reports; step-memoized so retries don't repeat side effects; webhooks reconcile external state. NextCRM: [inngest/functions/](../../../inngest/functions/).
- Mid: idempotency by memoization + conditional writes; the email-exactly-once impossibility (at-least-once + memoized step ≈ effectively once). Senior: the missing transactional outbox (fire-and-forget events lost on crash) and when it's worth adding (money yes, search-refresh no).
- Alternative: a raw queue (SQS/BullMQ) + hand-rolled idempotency — more control, more code; durable-execution trades some control for correctness-by-default.

## Step 6: Scaling concerns (5 min)

- Reads: index ownership columns (every scoped read touches them); paginate with cursors (NextCRM's activity feed does, its account list doesn't yet). Writes: the series counter is a contention point under bulk issuance. Bundle: heavy client deps (PDF/tiptap/primereact) server-only or dynamic.
- Mid: identifies the counter and the unbounded `getAccounts` as first bottlenecks. Senior: measures before optimizing, and notes the numbering lock choice (FOR UPDATE vs Serializable+retry) *is* a throughput decision at month-end.

## Step 7: Tradeoffs summary (2 min)

State the three defining choices and their costs: shared-namespace authz (simple, but every mutation must scope), monolith with build-time migrations (one thing to deploy, but migrate+deploy coupled and roll-forward-only), durable execution over raw queues (correctness by default, less control). A senior closes by naming what they'd revisit first at 10x scale.

---

## Variation prompts (practice each in 5 min)

1. **"Now add true multi-tenancy."** Introduce `organizationId` on every model; switch scope-wheres to tenant-keyed; consider Postgres RLS now that a tenant key exists; migration is expand/contract across ~80 models — huge, sequence it. Contrast with today's ownership-OR model.
2. **"Now add real-time (live invoice/campaign updates)."** Add a pub/sub (WebSocket/SSE) layer; the send-status rows and webhook updates become the event source; watch out for authorizing subscriptions (a socket must respect the same scopes).
3. **"Now handle 10x campaign volume."** Throttle sends (Inngest concurrency / token bucket in the Upstash Redis already present); batch webhook processing; the exactly-once story stays (memoized steps) but you add replay dedup and per-campaign rate caps.
4. **"Guarantee no duplicate invoice numbers under heavy concurrency."** This is the numbering hardening — relocate the lock into `consumeNextNumber`, add retry-on-serialization-abort, load-test with N parallel issuances.

## What junior/mid/senior sound like overall

- Junior: correct entities and CRUD, misses money/concurrency/authz depth.
- Mid: correct choices *with* tradeoffs and failure modes; cites concrete mechanisms.
- Senior: leads with the one decision that shapes the rest (tenancy), names invariants and who owns them, sequences migrations to stay safe, and says what they'd revisit at scale — and when the standard answer (e.g., "just use RLS") doesn't fit the constraints.

Cross-reference: [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) is your source material; deliver its three sections (strengths, risks, 90-day plan) as the backbone of this walkthrough.
