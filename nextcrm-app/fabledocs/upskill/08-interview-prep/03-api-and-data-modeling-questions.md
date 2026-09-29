# API and Data Modeling — Question Cards

13 cards, anchored to NextCRM's real API, schema, and transactions.

---

## Q1: What is IDOR/BOLA and how do you prevent it?
Tests: the #1 API security question. Anchor: [contacts route:51-52](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L51-L52) (fixed) vs [update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49) (gap); [advisory doc](../../../docs/2026-05-01-bola-idor-security-audit.md).
Junior: "check the user is logged in." Mid: authentication ≠ authorization; you must verify the user owns *this specific object*, not just that they're authenticated. Cites the scoped `updateMany` where ownership is *in the where clause*. Senior: this repo shipped a real CVE (GHSA-mg5f-m89f-4gmc) with exactly this pattern; the fix centralizes policy as query fragments; the residual gap is legacy server actions — and 404-for-forbidden prevents enumeration.
Follow-ups: 403 vs 404; how to enforce it can't regress (tests + lint). Drill: this is a whole STAR story — [06](06-behavioral-star-stories.md).

## Q2: How do you do object-level authorization at scale without an if-statement per route?
Tests: authz architecture. Anchor: [accountReadScopeWhere:229-237](../../../lib/authz/scopes/crm.ts#L229-L237), [filterAuthorizedContactIds:126-136](../../../lib/authz/scopes/crm.ts#L126-L136).
Junior: per-route checks. Mid: policy as composable `where` fragments spread into every query — authorization executes atomically with the read/write, and bulk endpoints intersect ids with the scope in one query (no N+1). Senior: contrasts app-level scoping (needed here — no tenant column) with DB RLS (would need a tenant/owner key pushed to Postgres), and notes nothing *forces* usage → needs enforcement.
Drill: write a scope-where for a new entity.

## Q3: 403 vs 404 for forbidden resources — which and why?
Tests: security nuance. Anchor: `notFoundOrForbiddenResponse` / count===0 → 404 ([contacts route:52](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L52)).
Junior: "403 forbidden." Mid: returning 404 for both missing and forbidden prevents attackers from enumerating which ids exist. Senior: tradeoff — 404-for-all can confuse legitimate clients and complicate debugging; you accept that for anti-enumeration on multi-tenant-ish data.
Drill: find where the fold happens and what a `user` sees for a colleague's contact.

## Q4: Design pagination for a large list endpoint.
Tests: API design. Anchor: activity feed uses compound cursor (`createdAt + id`) per [README](../../../README.md#L63-L72); [get-accounts.ts](../../../actions/crm/get-accounts.ts) has *no* limit (offset-less, unbounded).
Junior: "LIMIT/OFFSET." Mid: cursor pagination (keyset) avoids offset drift on inserts and is O(1) — the activities feed uses `createdAt + id` for stable ordering with ties broken by id. Senior: notes offset pagination degrades on deep pages and duplicates rows under concurrent inserts; cites that `getAccounts` ships everything (a real unbounded-query risk) and would need a cursor.
Drill: design the cursor for the accounts list.

## Q5: How do you model money in a database?
Tests: correctness. Anchor: [totals.ts:18-25](../../../lib/invoices/totals.ts#L18-L25) Decimal per-line rounding; string serialization ([create-invoice.ts:72-76](../../../actions/invoices/create-invoice.ts#L72-L76)).
Junior: "use a decimal/numeric column." Mid: never floats — decimal.js in app, NUMERIC in DB, serialized as strings across the RSC boundary; round *per line then sum* so printed line totals match the invoice total. Senior: the per-line-vs-total rounding difference is a real cent-level discrepancy accountants catch; VAT buckets ([totals.ts:45-65](../../../lib/invoices/totals.ts#L45-L65)) group by rate for legal breakdowns.
Drill: compute a 3-line invoice both rounding orders; show the cent.

## Q6: Soft delete vs hard delete — tradeoffs?
Tests: schema judgment. Anchor: [delete-account.ts:13-16](../../../actions/crm/accounts/delete-account.ts#L13-L16); [soft-delete-gaps.md](../../../docs/soft-delete-gaps.md).
Junior: "soft delete keeps the data." Mid: `deletedAt` gives recoverability + referential integrity; every read scope filters `deletedAt: null`. Senior: names the failure modes — write paths that forget the filter (this repo edits soft-deleted accounts), unique constraints colliding with tombstones, and GDPR requiring real purge anyway.
Drill: find one write path missing the filter.

## Q7: When do you denormalize?
Tests: modeling. Anchor: `billingSnapshot`/`taxRateSnapshot` ([issue-invoice.ts:60-89](../../../actions/invoices/issue-invoice.ts#L60-L89)).
Junior: "for performance." Mid: by *mutability contract* — an issued invoice must render identically forever, so you freeze (snapshot) the account's billing data and tax rates at issue time; normalized data that should change together stays normalized. Senior: notes the risk of a missed snapshot field, and the subtle bug where the PDF might render live account data despite the snapshot existing ([risk #10](../09-reference/risk-register.md)).
Drill: list what must be snapshotted for a legally-immutable invoice.

## Q8: How do you guarantee a gap-free sequence (invoice numbers)?
Tests: transactions + concurrency. Anchor: [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) + Serializable at [issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128).
Junior: "auto-increment." Mid: DB sequences leave gaps on rollback and can't do per-series templates/yearly reset; so a counter row consumed inside a Serializable transaction. Senior: the caller-owns-the-lock trap; prefers `SELECT FOR UPDATE` inside the helper; Serializable needs retry-on-abort.
Drill: [refactor kata A](../06-contribution-practice/04-refactor-and-design-katas.md).

## Q9: REST route vs server action (RPC) — when to use which here?
Tests: API style. Anchor: server actions for app-driven mutations ([actions/](../../../actions/)); API routes for webhooks/integrations/PDF/MCP ([app/api/](../../../app/api/)).
Junior: "REST for the API." Mid: server actions for same-app mutations (typed, no client fetch), REST routes for external contracts (webhooks with HMAC, presigned uploads, MCP tools, PDF download) that non-browser clients call. Senior: notes both must authorize independently, and the duplication risk when a route and an action do the same thing with different authz rigor (contacts PATCH route has scoping; some actions don't).
Drill: classify five endpoints as action-appropriate vs route-appropriate.

## Q10: How do you secure a webhook?
Tests: integration security. Anchor: [resend route:5-19](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L19).
Junior: "check a secret." Mid: HMAC over the *raw* body (parse only after verifying — attacker controls the parsed JSON), shared secret, reject on mismatch. Senior: adds constant-time compare (`timingSafeEqual` — this repo uses `===`), replay protection via event-id dedup (missing here), and idempotent handlers so redelivery is safe.
Drill: [review kata 5 + ticket 4](../06-contribution-practice/01-good-first-tickets.md).

## Q11: How do you version/change an API without breaking consumers?
Tests: contract management. Anchor: MCP tool schemas ([mcp route](../../../app/api/mcp/%5Btransport%5D/route.ts)), token prefix ([api-tokens.ts:4](../../../lib/api-tokens.ts#L4)), unsubscribe URLs already in sent emails.
Junior: "add a v2." Mid: identify what's a real contract (tool names, token format, embedded URLs, event names) vs internal; additive changes are safe, renames break external consumers silently. Senior: expand/contract, deprecation windows, and that some contracts (URLs in already-sent emails) can *never* change — you can only add new ones.
Drill: [fake-code #10](../04-code-reading-gym/03-fake-code-contrasts.md) — the token prefix rename.

## Q12: How does caching interact with authorization? (A subtle one.)
Tests: senior nuance. Anchor: React `cache()` around an authenticated fetch ([get-accounts.ts:5-8](../../../actions/crm/get-accounts.ts#L5-L8)).
Junior: unaware. Mid: `cache()` here is request-scoped, so the authenticated user is constant within the dedup window — safe. Senior: warns the danger case — a cross-request cache keyed without the user would leak one user's authorized data to another; explains why request-scoped dedup avoids that while a naive data cache wouldn't.
Drill: explain why `requireAuthenticated` inside the cached fn is still safe.

## Q13: Design the API-token/credential model for machine access.
Tests: auth for non-humans. Anchor: [api-tokens.ts](../../../lib/api-tokens.ts) (hashed, capped, revocable) vs [api-keys.ts](../../../lib/api-keys.ts) (encrypted provider keys).
Junior: "give them an API key." Mid: generate a random token, store only its SHA-256 hash, show a prefix for identification, support revocation (timestamp checked on validate) and expiry, cap per user. Senior: distinguishes hash-what-you-verify from encrypt-what-you-replay, and per-tool authorization threading the token's userId into the same scopes ([mcp route:10-11](../../../app/api/mcp/%5Btransport%5D/route.ts#L10-L11)).
Drill: [key flow 7](../01-codebase-cartography/05-key-flows.md).
