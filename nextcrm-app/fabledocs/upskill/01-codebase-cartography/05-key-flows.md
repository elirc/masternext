# Key Flows

Seven end-to-end traces. Everything else in the curriculum rotates through these. Line anchors verified 2026-07-11.

---

## Flow 1: Accounts list → edit (UI/client flow)

Why this flow matters: it's the canonical RSC → server action → revalidate loop, repeated for every CRM entity — and it contains the repo's most instructive authorization asymmetry.

Open these files first:
- [app/[locale]/(routes)/crm/accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) — async RSC page
- [actions/crm/get-accounts.ts](../../../actions/crm/get-accounts.ts#L5-L40) — scoped read
- [actions/crm/accounts/update-account.ts](../../../actions/crm/accounts/update-account.ts#L8-L68) — the write

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | middleware | [proxy.ts:42-51](../../../proxy.ts#L42-L51) | no session cookie → redirect `/sign-in`; else next-intl routing | cookie string | presence-only check |
| 2 | RSC page | accounts/page.tsx | awaits `getAllCrmData()` + `getAccounts()` | Prisma rows | two sequential awaits (serial latency) |
| 3 | server action | [get-accounts.ts:5-8](../../../actions/crm/get-accounts.ts#L5-L8) | `requireAuthenticated()` then `findMany` with `accountReadScopeWhere(user)` | `AuthzUser` → rows + includes | — |
| 4 | authz | [crm.ts:229-237](../../../lib/authz/scopes/crm.ts#L229-L237) | admin/manager: `{deletedAt: null}`; user: + OR(assigned/created/watching) | Prisma `where` | scope must stay in sync with schema |
| 5 | client | AccountsView.tsx | renders TanStack table, "+" opens Sheet form | `data: any[]` (!) | `any[]` loses the contract |
| 6 | server action | [update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49) | `getSession()` only, then `update({ where: { id } })` | typed object, **no Zod** | **no object-level authz** — any authenticated user can update any account by id |
| 7 | audit | [update-account.ts:50-60](../../../actions/crm/accounts/update-account.ts#L50-L60) | `diffObjects(before, after)` → `writeAuditLog` (best-effort) | `AuditChange[]` | audit failure swallowed by design ([audit-log.ts:78-81](../../../lib/audit-log.ts#L78-L81)) |
| 8 | async + cache | [update-account.ts:61-62](../../../actions/crm/accounts/update-account.ts#L61-L62) | `inngest.send("crm/account.saved")` (fire-and-forget), `revalidatePath` | event | `void`-ed promise: embedding refresh can silently fail |

Validation and authorization: read path fully scoped (steps 3–4); write path authenticated-only (step 6) — confirmed gap, consistent with the repo's own [BOLA/IDOR audit](../../../docs/2026-05-01-bola-idor-security-audit.md).
Persistence and side effects: Prisma update, audit row, Inngest event, Next.js cache invalidation.
Tests that cover it: E2E [tests/e2e/account-update.spec.ts](../../../tests/e2e/account-update.spec.ts), [account-detail-update.spec.ts](../../../tests/e2e/account-detail-update.spec.ts). No unit test asserts the (missing) write scope.
What juniors usually miss: `revalidatePath` is what makes the UI update — the action returns data nobody uses for rendering.
What seniors notice: the asymmetry between steps 3 and 6; `v: 0` hardcoded on update ([update-account.ts:45](../../../actions/crm/accounts/update-account.ts#L45)) — a dead Mongo version field; update not filtered by `deletedAt`, so soft-deleted accounts can be edited.
Interview angle: "Tell me about an authorization bug pattern" — you can cite a real CVE (GHSA-mg5f-m89f-4gmc), its remediation architecture, and the residual gap, in one story.
Drill: write the one-line diff that would scope step 6 (hint: `assertCanWriteAccount(user, id)` exists — [crm.ts:251-267](../../../lib/authz/scopes/crm.ts#L251-L267)).
Self-grade: Basic = named the missing check. Solid = also handled the error → return value mapping like [create-invoice.ts:25-30](../../../actions/invoices/create-invoice.ts#L25-L30). Strong = discussed why `updateMany`-with-scoped-where (as in [tryScopedUpdateContact](../../../lib/authz/scopes/crm.ts#L33-L43)) beats check-then-write under concurrency.

---

## Flow 2: PATCH /api/crm/contacts/[id] — the CVE remediation (server/API flow)

Why this flow matters: this exact endpoint was the subject of a published high-severity advisory; the current code is the fix. You get to study before/after of a real IDOR.

Open these files first:
- [app/api/crm/contacts/[id]/route.ts](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L22-L55)
- [lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts#L12-L43) — `contactScopedWhere` + `tryScopedUpdateContact`
- [lib/authz/route.ts](../../../lib/authz/route.ts) — response helpers

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | route | route.ts:22-34 | await params (Next 15+ async params), `requireAuthenticated()` | `AuthzUser` | — |
| 2 | route | route.ts:36-49 | body → `FIELD_MAP` allowlist filters columns | `Record<string,string>` | allowlist prevents mass assignment |
| 3 | authz+write | [crm.ts:33-43](../../../lib/authz/scopes/crm.ts#L33-L43) | `updateMany` where `{id, OR:[assigned_to, createdBy]}` (role-widened) | `count` | atomic authorize+write |
| 4 | route | route.ts:52 | `count === 0` → `notFoundOrForbiddenResponse()` | 404 | deliberately doesn't reveal existence |

Validation and authorization: field allowlist (step 2) + scoped write (step 3). Note values are coerced with `String(value)` — type laundering, but into text columns.
Persistence and side effects: single `updateMany`; **no audit log entry** here (unlike the server-action path — an inconsistency worth a ticket).
Tests that cover it: [app/api/crm/contacts/[id]/__tests__/route.test.ts](../../../app/api/crm/contacts/%5Bid%5D/__tests__/route.test.ts).
What juniors usually miss: why `updateMany` instead of `update` — the scope lives *in the where*, so authorization and write are one atomic statement; there is no TOCTOU window.
What seniors notice: 404-for-forbidden as an anti-enumeration choice; `updatedBy` stamped inside the helper; the enrichment-only field map means this endpoint can't touch email/name.
Interview angle: "How do you prevent IDOR?" — answer with ownership-scoped where clauses, not per-route if-statements.
Drill: diff this handler against the pre-fix shape described in the [audit doc](../../../docs/2026-05-01-bola-idor-security-audit.md) and list the three distinct defenses added.
Self-grade: Basic = found allowlist + scope. Solid = explained 404-vs-403. Strong = explained the atomicity argument for scoped `updateMany`.

---

## Flow 3: Invoice lifecycle — create → issue (persistence/transaction flow)

Why this flow matters: money, legal invariants, and the repo's only Serializable transaction. This is the flow to bring to a system-design interview.

Open these files first:
- [actions/invoices/create-invoice.ts](../../../actions/invoices/create-invoice.ts#L15-L101)
- [actions/invoices/issue-invoice.ts](../../../actions/invoices/issue-invoice.ts#L17-L210)
- [lib/invoices/numbering.ts](../../../lib/invoices/numbering.ts#L13-L30), [totals.ts](../../../lib/invoices/totals.ts#L18-L74), [permissions.ts](../../../lib/invoices/permissions.ts#L31-L34)

Trace (issue):

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | action | issue-invoice.ts:17-19 | `getUser()`, Zod parse | typed input | uses `getUser`, not `requireAuthenticated` — third auth idiom |
| 2 | guard | [permissions.ts:31-34](../../../lib/invoices/permissions.ts#L31-L34) | pure `canIssueInvoice`: DRAFT + owner/privileged | booleans | no account-scope check — owner ≠ account access |
| 3 | prep | issue-invoice.ts:22-54 | load invoice+lines+account, settings, FX rate — all **outside** tx | Decimal-bearing rows | comment says why: don't hold locks during network I/O |
| 4 | tx | [issue-invoice.ts:56-58,128](../../../actions/invoices/issue-invoice.ts#L56-L58) | `$transaction(..., { isolationLevel: "Serializable" })` | — | serialization failures will surface as errors; no retry loop |
| 5 | numbering | [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) | read counter → maybe yearly reset → +1 → write | `INV-2026-0007` | read-modify-write; **only safe because caller chose Serializable** |
| 6 | snapshot | issue-invoice.ts:60-89 | billing snapshot JSON + per-line `taxRateSnapshot` | frozen copies | protects issued invoices from later master-data edits |
| 7 | flip | issue-invoice.ts:99-124 | status ISSUED, dates, totals recomputed, activity row nested | — | totals recomputed from *pre-tx read* of lines (step 3) |
| 8 | post | issue-invoice.ts:131-207 | render PDF → MinIO upload → save storage key; failures logged, **never thrown** | Buffer | invoice legally issued even if PDF fails — explicit tradeoff |

Validation and authorization: Zod at entry; status machine as pure functions (fully unit-tested — [permissions.test.ts](../../../__tests__/lib/invoices/permissions.test.ts)).
Persistence and side effects: Serializable tx; MinIO write; PDF regeneratable via [regenerate-pdf.ts](../../../actions/invoices/regenerate-pdf.ts).
Tests that cover it: [__tests__/invoices/lifecycle.test.ts](../../../__tests__/invoices/lifecycle.test.ts), unit tests for totals/numbering/permissions; E2E skeleton [invoices.spec.ts](../../../tests/e2e/invoices.spec.ts).
What juniors usually miss: why the FX fetch (step 3) must be outside the transaction.
What seniors notice: `consumeNextNumber`'s safety is an invariant owned by its *caller*; a future caller without Serializable gets duplicate invoice numbers. Also step 7 recomputes totals from data read before the tx started — line items could theoretically change between read and tx (small window, DRAFT-only edits mitigate).
Interview angle: "Design an invoice numbering system that never skips or duplicates" — you can describe gap-free sequences, why Postgres `SERIAL` doesn't work (gaps on rollback), and what Serializable costs.
Drill: write the fake-code version using `SELECT ... FOR UPDATE` instead of Serializable and list what changes.
Self-grade: Basic = explained the number consumption. Solid = explained isolation choice + network-outside-tx. Strong = identified the caller-owned-invariant problem and proposed moving the lock into the helper.

---

## Flow 4: Contact enrichment (background/async flow)

Why this flow matters: the repo's fullest async pipeline — API trigger → event → agent → merge-back — with idempotency and partial-failure handling to critique.

Open these files first:
- [app/api/crm/contacts/enrich/route.ts](../../../app/api/crm/contacts/enrich/route.ts) — trigger
- [inngest/functions/enrich-contact.ts](../../../inngest/functions/enrich-contact.ts#L32-L147) — worker
- [lib/api-keys.ts](../../../lib/api-keys.ts#L21-L46) — 3-tier key resolution

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API route | contacts/enrich/route.ts | authz + create enrichment row (PENDING) + `inngest.send("enrich/contact.run")` | event `{contactId, enrichmentId, fields, triggeredBy}` | event payload is the *only* context the worker gets |
| 2 | worker | enrich-contact.ts:47-56 | resolve OPENAI+FIRECRAWL keys via env→system→user chain | strings | missing keys → status FAILED with actionable message |
| 3 | worker | enrich-contact.ts:59-62 | status → RUNNING | — | no state-machine guard (a cancelled row can be set RUNNING) |
| 4 | worker | enrich-contact.ts:91-106 | 7-day dedup: recent COMPLETED enrichment → SKIPPED | — | check is against `createdAt` of prior run; TOCTOU across concurrent runs |
| 5 | agent | enrich-contact.ts:109-114 | `AgentEnrichmentStrategy.enrichRow({email}, fields)` | enrichment map | external AI + scraping; `retries: 3` at function level ([:37](../../../inngest/functions/enrich-contact.ts#L37)) |
| 6 | merge | enrich-contact.ts:123-138 | apply **only to fields empty at step-4 read** | `Record<string,string>` | lost-update window: user edits during enrichment get overwritten if field was empty at read time |
| 7 | done | enrich-contact.ts:140-143 | status COMPLETED + stored result JSON | `StoredEnrichmentResult` | — |

Validation and authorization: enforced at the trigger (step 1); the worker **trusts the event payload** — anyone who can emit Inngest events can enrich anything (Inngest signing key is the boundary).
Persistence and side effects: enrichment row status transitions; contact field writes; third-party API spend (real money — the dedup window is a cost control, not just politeness).
Tests that cover it: [__tests__/inngest/](../../../__tests__/inngest/) + `shouldSkipBulkEnrichment` exported specifically for unit testing ([enrich-contact.ts:11-18](../../../inngest/functions/enrich-contact.ts#L11-L18)).
What juniors usually miss: the whole function is retried by Inngest; every step must tolerate re-execution — which is why "apply only to empty fields" doubles as an idempotency strategy.
What seniors notice: no `step.run` wrapping here (unlike [send-step.ts](../../../inngest/functions/campaigns/send-step.ts)) — so a retry after the agent call re-runs the *paid* agent call too. Wrapping steps would memoize results across retries.
Interview angle: "How do you make a background job safe to retry?" — this function is a worked example with one deliberate gap to propose fixing.
Drill: rewrite steps 2–7 as `step.run` blocks on paper and mark which retries stop costing money.
Self-grade: Basic = traced statuses. Solid = named the lost-update window. Strong = explained step memoization + cost-aware retry design.

---

## Flow 5: Campaign send → webhook → unsubscribe (outbound integration flow)

Why this flow matters: covers signed webhooks, external-state reconciliation, and a state-changing GET — three interview staples in one module.

Open these files first:
- [inngest/functions/campaigns/send-step.ts](../../../inngest/functions/campaigns/send-step.ts#L14-L71)
- [app/api/campaigns/webhooks/resend/route.ts](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L71)
- [app/api/campaigns/unsubscribe/route.ts](../../../app/api/campaigns/unsubscribe/route.ts#L4-L33)
- [lib/campaigns/merge-tags.ts](../../../lib/campaigns/merge-tags.ts#L17-L23)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | worker | send-step.ts:20-32 | `step.run("load-send-record")` — campaign paused? skip | send + campaign + step + target | paused check happens once; pause *after* this step won't stop the send |
| 2 | template | [merge-tags.ts:17-23](../../../lib/campaigns/merge-tags.ts#L17-L23) | `{{first_name}}` etc. replaced with target fields, **unescaped** | HTML string | possible stored-HTML injection into outbound email (labeled risk) |
| 3 | send | send-step.ts:40-51 | Resend send with `List-Unsubscribe` header carrying per-send token | Resend result | — |
| 4 | record | send-step.ts:53-68 | status → sent/failed + `resend_message_id` | — | if step 4 crashes after step 3, retry **re-sends the email**? No — `step.run` memoized step 3's result. This is why the steps exist. |
| 5 | webhook | resend/route.ts:13-19 | HMAC-SHA256 verify of raw body | — | `===` compare, not `timingSafeEqual` (possible risk); no event-id replay dedup |
| 6 | webhook | resend/route.ts:34-68 | delivered/bounced/opened/clicked → conditional field updates | — | conditional writes (`if (!send.opened_at)`) give idempotency-by-construction |
| 7 | unsubscribe | unsubscribe/route.ts:4-24 | **GET** with token → set `unsubscribed_at` | HTML page | state-changing GET: mail scanners/prefetchers that follow links can unsubscribe recipients silently (inferred consequence; RFC 8058 one-click expects POST) |

Validation and authorization: webhook = HMAC; unsubscribe = unguessable per-send token (capability URL). No session on either — correct, but the *token* is the entire authz story.
Persistence and side effects: email to a real human (the least reversible side effect in the repo), send-status rows.
Tests that cover it: [__tests__/campaigns/](../../../__tests__/campaigns/), E2E campaign specs.
What juniors usually miss: why the webhook reads the **raw body** (`req.text()`) before JSON parse — HMAC is computed over exact bytes.
What seniors notice: step-level memoization is precisely what makes "send exactly once" survive retries; the GET-unsubscribe tradeoff (deliverability tooling compatibility vs HTTP semantics).
Interview angle: "How do you secure a webhook?" and "Why shouldn't GET mutate state?" — both answerable with these exact files.
Drill: write the `timingSafeEqual` version of the signature check and explain what attack class it closes.
Self-grade: Basic = traced statuses. Solid = raw-body + memoization. Strong = articulated the unsubscribe GET tradeoff with the prefetcher failure mode.

---

## Flow 6: Authentication & roles (auth/security boundary)

Why this flow matters: three cooperating layers (middleware, better-auth, DB-backed authz) that a junior usually conflates into "the auth."

Open these files first:
- [lib/auth.ts](../../../lib/auth.ts#L12-L126) — better-auth config
- [proxy.ts](../../../proxy.ts#L18-L55) — middleware
- [lib/authz/session.ts](../../../lib/authz/session.ts#L11-L33) — server-side gate

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | better-auth | auth.ts:58-99 | Google OAuth or email OTP (password auth disabled: [auth.ts:65-67](../../../lib/auth.ts#L65-L67)); OTP mailed via Resend; `testUtils` captures OTPs outside production for E2E | OTP capture must never ship enabled in prod (guarded by NODE_ENV) |
| 2 | signup | [auth.ts:109-124](../../../lib/auth.ts#L109-L124) | first user (count===1 after create) → admin+ACTIVE; others → PENDING + admin notification | count-based bootstrap has a theoretical signup race (labeled) |
| 3 | middleware | proxy.ts:31-51 | cookie presence → allow; admin paths additionally require cookie; role NOT checked here | comment documents the contract: "role checked server-side" |
| 4 | server | [session.ts:11-23](../../../lib/authz/session.ts#L11-L23) | `requireAuthenticated`: session → **re-fetch user row** → `mapLegacyRole` | DB round-trip per call; guarantees fresh role/status |
| 5 | roles | [session.ts:25-33](../../../lib/authz/session.ts#L25-L33) | `requireRole([...])` throws `AuthorizationError` | callers must catch and map to 401/403 — helpers in [authz/route.ts](../../../lib/authz/route.ts) |

What juniors usually miss: middleware cannot do DB role checks (edge constraint) — hence the split. What seniors notice: session cookie 7-day expiry with 24h refresh ([auth.ts:22-25](../../../lib/auth.ts#L22-L25)); role read from DB (step 4) not from JWT — revocation is immediate at the cost of a query.
Interview angle: "Session vs JWT?" — this repo picked DB-backed sessions and re-reads the role every request; explain what that buys (instant revocation, fresh role) and costs (latency).
Drill: trace what happens when an admin demotes a logged-in manager mid-session. Self-grade: Strong = notes the next `requireAuthenticated` call picks up the new role because role comes from the Users row, not the session token.

---

## Flow 7: MCP server & API tokens (machine-to-machine boundary)

Why this flow matters: an AI-agent-facing API with its own credential system — the newest kind of public interface, and this repo does credentials correctly (hash-at-rest).

Open these files first:
- [app/api/mcp/[transport]/route.ts](../../../app/api/mcp/%5Btransport%5D/route.ts#L5-L40)
- [lib/mcp/auth.ts](../../../lib/mcp/auth.ts#L8-L31)
- [lib/api-tokens.ts](../../../lib/api-tokens.ts#L8-L65)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | handler | route.ts:5-8 | every tool from [lib/mcp/tools/](../../../lib/mcp/tools/) registered with schema | tool schemas are a public contract |
| 2 | auth | mcp/auth.ts:16-19 | `Bearer nxtc__…` → SHA-256 hash → DB lookup ([api-tokens.ts:47-57](../../../lib/api-tokens.ts#L47-L57)) | raw token never stored; only `tokenPrefix` (8 chars) shown in UI |
| 3 | auth | mcp/auth.ts:22-28 | dev-only fallback to session cookie | NODE_ENV guard is the only fence |
| 4 | tool | route.ts:10-11 | `tool.handler(args, mcpUser.id)` — userId threads into the same authz scopes | per-tool authz depends on each handler honoring it |
| 5 | errors | route.ts:15-27 | message-string → error-code mapping (`NOT_FOUND`, `CONFLICT:` prefix…) | stringly-typed error contract; fragile but explicit |
| 6 | telemetry | [api-tokens.ts:59-62](../../../lib/api-tokens.ts#L59-L62) | fire-and-forget `lastUsedAt` update | intentionally unawaited; failures silenced |

What juniors usually miss: auth happens **per tool call** (step 2 inside the tool callback), not per connection.
What seniors notice: token cap (10 active/user — [api-tokens.ts:17-27](../../../lib/api-tokens.ts#L17-L27)) counted with check-then-create (benign race); revocation is a timestamp, and validation checks it — so revocation is immediate.
Interview angle: "How do you store API keys?" — contrast this file's hashing with the *encrypted* (reversible, because they must be replayed to OpenAI) provider keys in [api-keys.ts](../../../lib/api-keys.ts): hash what you only verify, encrypt what you must reuse.
Drill: explain in 60 seconds why `ApiToken` uses SHA-256 but `ApiKeys` uses AES-256-GCM. Self-grade: Strong = the verify-vs-reuse distinction plus where each key lives.
