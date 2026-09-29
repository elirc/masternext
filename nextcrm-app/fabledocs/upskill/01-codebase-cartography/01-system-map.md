# System Map

## Shape

Single Next.js 16 App Router application (no monorepo, no workspaces). One `package.json`, one Prisma schema, one deployable unit. Background work is *logically* separate (Inngest functions) but ships in the same Next.js process behind [app/api/inngest/route.ts](../../../app/api/inngest/route.ts).

```
Browser ──► proxy.ts (middleware: cookie presence + i18n)
   │
   ├─► app/[locale]/(auth)/…      sign-in, register (public)
   ├─► app/[locale]/(routes)/…    RSC pages: crm, invoices, campaigns, projects,
   │        │                     documents, emails, reports, admin, profile
   │        └── call server actions in actions/**  ──► lib/authz ──► Prisma ──► Postgres (pgvector)
   │
   ├─► app/api/**                 REST-ish routes: crm PATCH endpoints, campaigns
   │                              webhooks/unsubscribe, invoices PDF, upload presign,
   │                              reports export, admin CRUD, og image
   ├─► app/api/mcp/[transport]    MCP server (bearer token auth) → lib/mcp/tools
   └─► app/api/inngest            Inngest webhook → inngest/functions/** (enrichment,
                                  campaigns, embeddings, email sync, reports, ECB rates)

Side services: MinIO (S3 uploads, invoice PDFs), Resend (email), IMAP (mail sync),
OpenAI/Firecrawl (enrichment agents), Upstash Redis (rate limiting), e2b (sandboxed enrichment)
```

## Ownership map

| Layer | Directory | Owns | Must NOT own |
| --- | --- | --- | --- |
| UI pages | [app/[locale]/(routes)/](../../../app/%5Blocale%5D/(routes)/) | RSC data loading, layout, page composition | business rules, raw Prisma calls |
| UI components | [components/](../../../components/), per-route `components/` + `table-components/` | rendering, client state (jotai/useState), forms (react-hook-form + Zod) | authorization decisions |
| Server actions | [actions/](../../../actions/) | mutations + reads callable from RSC/client, revalidation | rendering; skipping authz (see risk register) |
| API routes | [app/api/](../../../app/api/) | webhooks, external integrations, file/PDF endpoints, MCP | duplicating server-action logic without scoping |
| Domain logic | [lib/](../../../lib/) | authz scopes ([lib/authz/](../../../lib/authz/)), invoice math ([lib/invoices/](../../../lib/invoices/)), crypto, audit log, enrichment engine | HTTP concerns |
| Background jobs | [inngest/functions/](../../../inngest/functions/) | enrichment, campaign sends, embeddings, email sync, scheduled reports | interactive request handling |
| Persistence | [prisma/](../../../prisma/) | schema (1909 lines), migrations, seeds | — |
| Email templates | [emails/](../../../emails/) | React Email components | sending logic (lives in lib/resend.ts, lib/sendmail.ts) |
| Tests | [__tests__/](../../../__tests__/) + colocated `__tests__/`, [tests/e2e/](../../../tests/e2e/) | Jest unit, Playwright E2E | — |

## Public interfaces vs private internals

**Public (contract — changing these breaks someone):**
- Every route under [app/api/](../../../app/api/) — especially the webhook contracts ([campaigns/webhooks/resend](../../../app/api/campaigns/webhooks/resend/route.ts), Inngest signing) and [unsubscribe](../../../app/api/campaigns/unsubscribe/route.ts) links already embedded in sent emails.
- MCP tool names/schemas in [lib/mcp/tools/](../../../lib/mcp/tools/) — external AI agents depend on them.
- API token format `nxtc__…` ([lib/api-tokens.ts](../../../lib/api-tokens.ts#L4-L10)) — tokens are hashed; changing prefix/hashing invalidates every issued token.
- The Prisma schema, in the sense that migrations must be forward-safe (`pnpm build` runs `prisma migrate deploy` — [package.json](../../../package.json#L11)).
- Invoice numbers already issued (legal artifact — [numbering.ts](../../../lib/invoices/numbering.ts)).

**Private (refactor freely with tests):** everything in `lib/` not listed above, server action implementations (callers are in-repo), component trees, Inngest function internals (but not their event names — events are a contract between senders like [update-account.ts](../../../actions/crm/accounts/update-account.ts#L61) and handlers).

## What a senior inspects first

1. `lib/authz/scopes/crm.ts` — is authorization centralized or scattered? (Centralized for reads and API routes; **scattered/absent for some legacy server actions** — the repo's own audit doc [docs/2026-05-01-bola-idor-security-audit.md](../../../docs/2026-05-01-bola-idor-security-audit.md) says why.)
2. `prisma/schema.prisma` naming — `crm_Accounts`, `createdBy` vs `created_by` inconsistencies betray the MongoDB heritage; migrations tell the modernization story.
3. Where money lives — Decimal everywhere in [lib/invoices/totals.ts](../../../lib/invoices/totals.ts), string-serialized at boundaries.
4. The three docs trees (`docs/`, `docs2/`, `docs3/`) — previous audits, migration guides, learning material. Real projects accrete these; learn to mine them.

## Drill

Without opening the code, predict: where does the "convert lead to opportunity" logic live — `app/api`, `actions/crm`, or `lib/`? Then verify with `rg -l "convert" actions app/api lib`. Self-grade: **Basic** = found it after the search. **Solid** = predicted `actions/crm` from the ownership table. **Strong** = also articulated *why* a mutation initiated by the app's own UI belongs in a server action rather than an API route in this architecture.
