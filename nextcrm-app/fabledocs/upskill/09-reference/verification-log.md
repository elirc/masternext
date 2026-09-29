# Verification Log

Running record of what was inspected while writing this curriculum. Date: **2026-07-11**. Environment: Windows 11, repo at `masternext/nextcrm-app`, working tree with no git commits (fresh clone/copy — `git log` reports no commits on `main`).

## Commands run

| Command | Result |
| --- | --- |
| `rg --files \| wc -l` | 1404 files |
| `git log --oneline` | "branch 'main' does not have any commits yet" — no git history to consult |
| Directory listings (`ls`) of app/, actions/, lib/, inngest/, prisma/, tests/, docs/ | recorded in module docs |
| `grep -n "^model\|^enum" prisma/schema.prisma` | ~80 models/enums; schema is 1909 lines |

**No install/build/test commands were executed.** `pnpm install`, `pnpm dev`, `pnpm test`, `pnpm test:e2e` are all marked __inferred__ throughout, read from [package.json](../../../package.json#L9-L21), [jest.config.ts](../../../jest.config.ts), [playwright.config.ts](../../../playwright.config.ts), [docker-compose.yml](../../../docker-compose.yml).

## Files read in full (line anchors written from these reads)

- package.json, jest.config.ts, playwright.config.ts (head), docker-compose.yml (head)
- proxy.ts; lib/auth.ts; lib/auth-server.ts (partial); lib/authz/{index,session}.ts; lib/authz/scopes/crm.ts (L1–420 of 656)
- lib/create-safe-action.ts; lib/prisma.ts; lib/serialize-decimals.ts; lib/audit-log.ts; lib/api-keys.ts; lib/api-tokens.ts; lib/email-crypto.ts; lib/enrichment/rate-limit.ts; lib/campaigns/merge-tags.ts; lib/mcp/auth.ts
- lib/invoices/{numbering,totals,permissions}.ts
- actions/invoices/{create-invoice,issue-invoice}.ts; actions/crm/get-accounts.ts; actions/crm/accounts/{update-account,delete-account}.ts; actions/fulltext/unified-search.ts (head)
- app/api/crm/contacts/[id]/route.ts; app/api/campaigns/webhooks/resend/route.ts; app/api/campaigns/unsubscribe/route.ts; app/api/mcp/[transport]/route.ts; app/api/upload/presigned-url/route.ts
- inngest/client.ts; inngest/functions/enrich-contact.ts; inngest/functions/campaigns/send-step.ts
- app/[locale]/(routes)/crm/accounts/page.tsx (head); app/[locale]/(routes)/crm/components/AccountsView.tsx (head)
- tests/e2e/invoices.spec.ts (head); __tests__/lib/invoices/{fx,numbering,permissions}.test.ts (heads)
- docs/2026-05-01-bola-idor-security-audit.md (head); docs/soft-delete-gaps.md (head); README.md (head)

## Key findings verified in code

1. **Scoped authz layer** exists and is real: `accountReadScopeWhere` etc. in lib/authz/scopes/crm.ts; used by reads like actions/crm/get-accounts.ts:7-8 and by the contacts PATCH API (tryScopedUpdateContact).
2. **Residual authz gap (confirmed)**: `updateAccount` (update-account.ts:34-49) and `deleteAccount` (delete-account.ts:7-16) check only `getSession()` — no object-level scope check — while `createInvoice` (create-invoice.ts:26) calls `assertCanWriteAccount`. Consistent with the repo's own audit doc (docs/2026-05-01-…md), which covered server actions; the API routes have since been remediated.
3. **Serializable invoice numbering**: issue-invoice.ts:56-129 wraps `consumeNextNumber` in `$transaction(..., { isolationLevel: "Serializable" })`. The helper itself (numbering.ts:13-30) is read-modify-write and is only safe under that isolation level — an invariant owned by the caller.
4. **Webhook HMAC**: resend webhook (route.ts:5-11) compares HMAC with `===`, not `crypto.timingSafeEqual` — possible timing-attack surface (low practical risk, flagged not asserted as exploitable).
5. **Merge tags unescaped**: merge-tags.ts:17-23 substitutes target fields into campaign HTML without escaping — possible stored-HTML injection into outbound email. Labeled "possible risk" (email client rendering varies).
6. **Unsubscribe is a state-changing GET**: app/api/campaigns/unsubscribe/route.ts:4-24. Confirmed; consequence (prefetch auto-unsubscribe) is inferred behavior of mail scanners, labeled as such.
7. **Rate-limit IP extraction trusts `x-forwarded-for`** (rate-limit.ts:27-41) — spoofable unless a trusted proxy strips it; labeled possible risk, deployment-dependent.
8. **First user becomes admin**: lib/auth.ts:109-124, count===1 check (race window is theoretical, labeled).
9. **MCP auth**: bearer tokens `nxtc__…` SHA-256-hashed at rest (api-tokens.ts:8-10, 47-65); dev-only session fallback (lib/mcp/auth.ts:22-28).
10. **Soft delete** via `deletedAt` across CRM models per docs/soft-delete-gaps.md; `deleteAccount` sets deletedAt rather than deleting.

## Uncertainties / not covered

- Did not run the app; all runtime behavior (redirect flows, revalidation UX, Inngest dev server) is inferred from code.
- lib/authz/scopes/crm.ts read to line ~420; board/task scope helpers (L420-656) skimmed via exports only.
- Email IMAP sync (inngest/functions/emails/), documents processing, reports scheduling, e2b sandbox enrichment, and the projects/boards UI were surveyed by directory listing only.
- Line anchors are exact as of 2026-07-11; the repo has no git history here, so future drift cannot be diffed against a commit.
- prisma/schema.prisma read via model-name grep + targeted sections, not all 1909 lines.
