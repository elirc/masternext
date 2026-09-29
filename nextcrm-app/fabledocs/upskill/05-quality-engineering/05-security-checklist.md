# Security Checklist

Mapped to real code. Confidence labels: **confirmed** (read in code), **possible risk** (plausible, deployment/data-dependent), **investigate** (unverified).

## Authorization / IDOR / BOLA

- Object-level authz on **reads**: present via scope-where ([crm.ts:229-237](../../../lib/authz/scopes/crm.ts#L229-L237)) — **confirmed good**.
- Object-level authz on **API writes**: present (scoped `updateMany`) — [contacts route](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L51-L52) — **confirmed good**.
- Object-level authz on **legacy server-action writes**: **missing** — [update-account.ts](../../../actions/crm/accounts/update-account.ts#L34-L49), [delete-account.ts](../../../actions/crm/accounts/delete-account.ts#L8-L16) — **confirmed gap**, matches [GHSA advisory](../../../docs/2026-05-01-bola-idor-security-audit.md).
- Error responses fold 403 into 404 to prevent enumeration ([authz/route.ts](../../../lib/authz/route.ts)) — **confirmed, deliberate**.
- Background jobs trust event payloads (no per-record authz) — signing key is the fence ([enrich-contact.ts:40-45](../../../inngest/functions/enrich-contact.ts#L40-L45)) — **confirmed, acceptable if Inngest signing enforced**.

## Input validation / mass assignment

- Zod parse on new actions + MCP tools — **confirmed good**.
- TS-type-only on legacy actions — **confirmed gap** ([update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33)).
- Field allowlists prevent mass assignment on enrichment routes — **confirmed good** ([contacts route:10-20](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L10-L20)); but `...rest` spread in [update-account.ts:47](../../../actions/crm/accounts/update-account.ts#L46-L48) has no allowlist — **confirmed, bounded by the typed field list only at compile time**.

## Injection

- SQL: Prisma parameterizes; raw SQL used for pgvector search ([inngest/lib/embedding-utils.ts](../../../inngest/lib/embedding-utils.ts) `toVectorLiteral`, [unified-search.ts](../../../actions/fulltext/unified-search.ts)) — **investigate** that vector literals and any `$queryRaw` interpolate only numbers/parameters, never user strings.
- HTML/XSS: campaign merge tags substituted **unescaped** into email HTML ([merge-tags.ts:17-23](../../../lib/campaigns/merge-tags.ts#L17-L23)) — **possible risk** (stored-HTML injection into outbound mail; impact depends on where target names originate). React escapes app UI by default; check any `dangerouslySetInnerHTML` (`rg "dangerouslySetInnerHTML"`) — **investigate**.

## SSRF

- Enrichment fetches attacker-influenceable URLs (Firecrawl scrapes target websites) — that's the feature, but ensure it can't be pointed at internal hosts — **investigate** the enrichment URL handling.
- Any admin-supplied URL fetched server-side (e.g., a future logo feature) is SSRF — **[review kata 4](../04-code-reading-gym/04-review-katas.md)** exists to drill this. Currently **no confirmed SSRF**; flagged as a class to guard.

## CSRF

- Server actions use Next's action protocol (same-origin, action-id) — **confirmed mitigated** for the action surface.
- Webhook/unsubscribe are unauthenticated by design; unsubscribe as a **state-changing GET** ([unsubscribe/route.ts](../../../app/api/campaigns/unsubscribe/route.ts#L4-L24)) can be triggered by any GET (mail scanners, prefetch) — **confirmed shape, consequence possible risk**.

## Secrets & crypto

- Provider keys AES-256-GCM encrypted at rest ([email-crypto.ts](../../../lib/email-crypto.ts)) — **confirmed good**.
- API tokens SHA-256 hashed at rest, prefix-only display ([api-tokens.ts](../../../lib/api-tokens.ts#L8-L44)) — **confirmed good**.
- Encryption key validated as 64-hex at use ([email-crypto.ts:7-15](../../../lib/email-crypto.ts#L7-L15)) — **confirmed**; no boot-time validation (fails on first crypto op) — **investigate/minor**.
- Only `NEXT_PUBLIC_*` reaches the browser — **confirmed by framework**.

## Webhooks

- HMAC-SHA256 over raw body ([resend route:5-19](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L19)) — **confirmed good**, but `===` not `timingSafeEqual` (**possible risk**, low over network jitter) and no replay/event-id dedup (**confirmed gap**, currently benign due to conditional writes).

## Uploads

- Path traversal killed via `path.basename` ([presigned-url:19](../../../app/api/upload/presigned-url/route.ts#L18-L23)); folder + content-type allowlists; UUID keys; 10-min presign ([presigned-url:9-51](../../../app/api/upload/presigned-url/route.ts#L9-L51)) — **confirmed good**. Content type is the *declared* type, not sniffed — **possible risk** (a .exe declared as image/png uploads; mitigated by it being served from object storage, not executed). Session-gated — **confirmed**.

## Rate limiting

- Per-IP fixed-window via Upstash, prod-only ([rate-limit.ts](../../../lib/enrichment/rate-limit.ts#L6-L24)); IP from `x-forwarded-for` first entry ([:27-41](../../../lib/enrichment/rate-limit.ts#L27-L41)) — spoofable unless a trusted proxy overwrites XFF — **possible risk, deployment-dependent**. Coverage: enrichment endpoints; **investigate** whether auth/OTP and other abuse-prone endpoints are limited (they may not be).

## Auth specifics

- Password auth disabled; OAuth + OTP only ([auth.ts:65-67](../../../lib/auth.ts#L65-L67)) — **confirmed**.
- First-user-becomes-admin ([auth.ts:109-124](../../../lib/auth.ts#L109-L124)) — **confirmed**; theoretical signup race (**possible risk**, one-time window).
- Session role re-read from DB every request → instant revocation — **confirmed good**.
- OTP capture test plugin NODE_ENV-fenced ([auth.ts:90-93](../../../lib/auth.ts#L90-L93)) — **confirmed**; verify prod never sets NODE_ENV≠production — **investigate/deploy-check**.

## Dependency risk

- ~35 forced version overrides for CVE'd transitive deps ([package.json:155-192](../../../package.json#L155-L192)); install-script allowlist ([:193-206](../../../package.json#L193-L206)) — **confirmed active management**.

## Pre-merge security checklist (use on every PR)

- [ ] Does this mutation check *object-level* ownership, not just a session? (scope-where or assertCan)
- [ ] Is input parsed with Zod (`raw: unknown`), not just typed?
- [ ] Any client input spread into Prisma without an allowlist?
- [ ] Any user/target string rendered into HTML or email without escaping?
- [ ] Any server-side `fetch` of a user/admin-supplied URL? (SSRF)
- [ ] New secret: hashed if verify-only, encrypted if replayed, never logged?
- [ ] New webhook: raw-body HMAC, constant-time compare, idempotent handler?
- [ ] New public interface (route/tool/event/URL token): is the credential story explicit?
- [ ] Does the change keep the 404-for-forbidden behavior?
