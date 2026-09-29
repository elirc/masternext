# Validation, Auth, and Permissions

Vocabulary first (all interview words): **authentication** = who are you; **authorization** = what may you do; **object-level authorization** = may you do it *to this row* (its absence is BOLA/IDOR); **isolation** here means per-user data scoping — this app has **no tenant model**: one shared CRM namespace where roles + ownership fields do all the work (stated plainly in the repo's own [audit](../../../docs/2026-05-01-bola-idor-security-audit.md)).

## The validation map (where input is checked)

| Layer | Mechanism | Example | Gaps |
| --- | --- | --- | --- |
| Client forms | react-hook-form + Zod resolver | NewAccountForm and siblings | UX only — never a security boundary |
| Server actions (new style) | `raw: unknown` + `schema.parse` | [create-invoice.ts:23](../../../actions/invoices/create-invoice.ts#L23) | — |
| Server actions (legacy style) | TS parameter types only | [update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33) | **no runtime validation** |
| API routes | manual checks + allowlists | field map at [contacts/[id]/route.ts:10-20](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L10-L20); content-type/folder allowlists at [presigned-url/route.ts:9-37](../../../app/api/upload/presigned-url/route.ts#L9-L37) | ad-hoc per route |
| MCP tools | Zod schemas per tool | [mcp route:8](../../../app/api/mcp/%5Btransport%5D/route.ts#L8) | — |
| DB | types, uniques, FKs | schema | no CHECK constraints for status machines |

## The authorization map

**Roles**: `admin`, `manager`, `user` ([lib/authz/roles.ts](../../../lib/authz/roles.ts)); legacy strings mapped by `mapLegacyRole`. Admin/manager are treated identically in almost every scope (`role === "admin" || role === "manager"` — [crm.ts:230](../../../lib/authz/scopes/crm.ts#L229-L232)); the distinction is currently more aspiration than policy (investigate: find any scope where manager ≠ admin).

**User-level ownership**, per entity: assigned_to / createdBy / watchers for accounts ([accountUserScopeOR, crm.ts:218-224](../../../lib/authz/scopes/crm.ts#L216-L224)); linked-account traversal for contacts/leads/opportunities/contracts ([crm.ts:275-332](../../../lib/authz/scopes/crm.ts#L275-L332)) — you can read a contact if you could read an account it belongs to; creator-only for targets ([crm.ts:373-381](../../../lib/authz/scopes/crm.ts#L373-L381)); multi-faceted union for documents including `visibility: "public"` ([crm.ts:402-419](../../../lib/authz/scopes/crm.ts#L402-L419)); owner-or-privileged for invoices ([permissions.ts:18-23](../../../lib/invoices/permissions.ts#L18-L23)).

**Three enforcement idioms**, strongest to weakest:
1. **Scoped write** — authorization inside the `where` of the write itself: [tryScopedUpdateContact](../../../lib/authz/scopes/crm.ts#L33-L43). Atomic; no TOCTOU; returns count for 404-folding.
2. **Assert-then-act** — `assertCanWriteAccount(user, id)` then a separate write: [create-invoice.ts:26](../../../actions/invoices/create-invoice.ts#L26). Tiny TOCTOU window; fine for ownership that doesn't change mid-request.
3. **Session-only** — `getSession()` and hope: [update-account.ts:34-35](../../../actions/crm/accounts/update-account.ts#L34-L35), [delete-account.ts:8-9](../../../actions/crm/accounts/delete-account.ts#L8-L9). **This is the IDOR pattern the advisory was about**, surviving in server actions after the API routes were fixed.

## What a junior misses vs what a senior checks

| Junior assumption | Senior check |
| --- | --- |
| "It has auth" (session exists) | object-level: is *this row* mine? (idiom 1 or 2 present?) |
| "The UI only shows my records" | server actions are public RPC; UI filtering is not authz |
| "403 for forbidden" | this repo returns 404 (`notFoundOrForbiddenResponse`, [authz/route.ts](../../../lib/authz/route.ts)) to prevent ID enumeration — deliberate |
| "Middleware protects the API" | [proxy.ts](../../../proxy.ts#L31-L39) checks cookie *presence* only; every handler must re-authorize |
| "Roles come from the session" | role re-read from DB every call ([session.ts:16-22](../../../lib/authz/session.ts#L16-L22)) → instant demotion, at a query's cost |
| "Background jobs inherit the trigger's authz" | Inngest workers trust event payloads ([enrich-contact.ts:40-45](../../../inngest/functions/enrich-contact.ts#L40-L45)); the signing key is the fence |

## Non-session credentials

- **API tokens** (MCP): hashed at rest, prefix-displayed, revocable, capped at 10 ([api-tokens.ts](../../../lib/api-tokens.ts)).
- **Provider keys**: AES-256-GCM encrypted ([email-crypto.ts](../../../lib/email-crypto.ts)) because they must be decrypted for use; 3-tier resolution env→system→user ([api-keys.ts:21-46](../../../lib/api-keys.ts#L21-L46)).
- **Capability URLs**: per-send unsubscribe tokens ([unsubscribe/route.ts](../../../app/api/campaigns/unsubscribe/route.ts)) — the URL *is* the credential; hence must be unguessable and shouldn't leak via Referer.
- **Webhook signatures**: HMAC over raw body ([resend route:5-11](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L11)); Inngest signing key ([proxy.ts:21-24](../../../proxy.ts#L21-L24) passes it through untouched).

## Drills

1. Audit one entity end-to-end: pick **leads**. For each of read-list, read-one, create, update, delete, find which idiom (1/2/3) is used (`ls actions/crm/leads`, read each file). Produce a 5-row table. Self-grade: Strong = you also checked the MCP lead tools and API routes for the same entity and noted inconsistencies.
2. Write the threat story for the session-only `deleteAccount`: concrete request an authenticated low-privilege user sends to soft-delete a colleague's account, and the blast radius (account hidden from every scoped list; recoverable via [restore-account.ts](../../../actions/crm/accounts/restore-account.ts), but who'd notice?).

## Interview angle

- "What is IDOR/BOLA and how do you prevent it?" — the single best-prepared answer this repo gives you: real CVE, scoped-where remediation, residual server-action gap, 404-vs-403 choice. [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q1-Q3.
- "AuthN vs AuthZ" — answer with the three-idiom table, not definitions.
