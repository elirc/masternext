# File Reading Order

28 files, ordered so each one makes the next easier. Per file: why it matters, what to look for, what to skip.

## Junior path (1–14): make the shape stick

| # | File | Look for | Ignore |
| --- | --- | --- | --- |
| 1 | [package.json](../../../package.json#L9-L21) | scripts; `build` runs migrations — deploy implication | the 40-line pnpm `overrides` block (CVE pinning) |
| 2 | [proxy.ts](../../../proxy.ts#L18-L55) | cookie-presence-only check; the comment "role checked server-side" | matcher regex details |
| 3 | [i18n/routing.ts](../../../i18n/routing.ts) | locales and the `[locale]` URL prefix that explains every path | — |
| 4 | [app/[locale]/(routes)/layout.tsx](<../../../app/%5Blocale%5D/(routes)/layout.tsx>) | where the authenticated shell lives; what it fetches | styling |
| 5 | [app/[locale]/(routes)/crm/accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) | async RSC page → server-action reads → client view | skeleton components |
| 6 | [actions/crm/get-accounts.ts](../../../actions/crm/get-accounts.ts#L5-L40) | `cache()`, `requireAuthenticated`, scope-where, `include` shape | — |
| 7 | [lib/authz/session.ts](../../../lib/authz/session.ts#L11-L33) | throws typed errors; DB re-check of the user row | — |
| 8 | [lib/authz/roles.ts](../../../lib/authz/roles.ts) | `AppRole`, `mapLegacyRole` — legacy strings mapped forward | — |
| 9 | [lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts#L216-L302) | `accountUserScopeOR`, read-scope builders returning Prisma `where` fragments | the long tail of per-entity variants |
| 10 | [app/[locale]/(routes)/crm/components/AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) | `"use client"`, Sheet + form composition, `type CrmData = Awaited<ReturnType<…>>` trick | table plumbing |
| 11 | [app/[locale]/(routes)/crm/accounts/table-components/data-table.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/table-components/data-table.tsx>) | TanStack Table wiring — repeated for every entity | — |
| 12 | [actions/crm/accounts/update-account.ts](../../../actions/crm/accounts/update-account.ts#L34-L67) | what's *missing* (scope check, Zod), audit diff, Inngest event, revalidatePath | field list |
| 13 | [lib/audit-log.ts](../../../lib/audit-log.ts#L36-L82) | JSON.stringify diffing; write failures swallowed on purpose | — |
| 14 | [prisma/schema.prisma](../../../prisma/schema.prisma#L12-L71) | `crm_Accounts` model: soft-delete cols, `assigned_to`/`createdBy` ownership fields, watcher junction | all 81 models (+23 enums) |

## Mid path (15–22): follow the money and the queue

| # | File | Look for |
| --- | --- | --- |
| 15 | [types/invoice.ts](../../../types/invoice.ts) | Zod schemas as the input contract for invoice actions |
| 16 | [lib/invoices/totals.ts](../../../lib/invoices/totals.ts#L18-L74) | Decimal math, per-line rounding, VAT buckets |
| 17 | [lib/invoices/permissions.ts](../../../lib/invoices/permissions.ts#L10-L48) | status machine as pure functions — testable without DB |
| 18 | [actions/invoices/create-invoice.ts](../../../actions/invoices/create-invoice.ts#L15-L101) | full authn→parse→authz→compute→nested-create pipeline |
| 19 | [actions/invoices/issue-invoice.ts](../../../actions/invoices/issue-invoice.ts#L17-L210) | Serializable transaction; network calls kept outside; PDF failure tolerated |
| 20 | [inngest/client.ts](../../../inngest/client.ts) + [app/api/inngest/route.ts](../../../app/api/inngest/route.ts) | event bus setup; signing key |
| 21 | [inngest/functions/enrich-contact.ts](../../../inngest/functions/enrich-contact.ts#L32-L147) | status transitions, 7-day dedup, apply-only-to-empty-fields |
| 22 | [inngest/functions/campaigns/send-step.ts](../../../inngest/functions/campaigns/send-step.ts#L14-L71) | `step.run` memoization; paused check; unsubscribe header |

## Senior path (23–28): boundaries and contracts

| # | File | Look for |
| --- | --- | --- |
| 23 | [app/api/crm/contacts/[id]/route.ts](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L22-L55) | the CVE fix: field allowlist + scoped `updateMany`; 404-vs-403 folding |
| 24 | [docs/2026-05-01-bola-idor-security-audit.md](../../../docs/2026-05-01-bola-idor-security-audit.md) | how a real advisory was triaged; which findings are still open |
| 25 | [lib/api-tokens.ts](../../../lib/api-tokens.ts#L8-L65) | hash-at-rest tokens, prefix display, fire-and-forget lastUsedAt |
| 26 | [lib/mcp/auth.ts](../../../lib/mcp/auth.ts#L8-L31) + [app/api/mcp/[transport]/route.ts](../../../app/api/mcp/%5Btransport%5D/route.ts#L5-L40) | per-tool-call auth; error-code mapping at the boundary |
| 27 | [app/api/campaigns/webhooks/resend/route.ts](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L71) | HMAC verify (spot the non-timing-safe compare); conditional status transitions |
| 28 | [scripts/migrate-mongo-to-postgres.ts](../../../scripts/migrate-mongo-to-postgres.ts) | what a real data migration looks like; validation counterpart in scripts/validate-migration.ts |

## Pause-and-predict

Before opening file 12, predict from files 6–9: will `updateAccount` use `assertCanWriteAccount`? After reading, write one sentence on why read and write paths diverged (hint: the scope layer's comments say "Phase B1/D2" — it was retrofitted incrementally). That sentence is an interview answer about **incremental remediation under a live advisory**.
