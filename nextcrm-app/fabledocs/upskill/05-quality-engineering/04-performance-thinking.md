# Performance Thinking

Rule zero: **measure before optimizing.** The anchors below are *likely* hotspots to profile, not confirmed problems — treat each as "investigate," and confirm with query logs / the network tab / React Profiler before changing anything.

## Performance domains present here

| Domain | Where it lives | How to measure |
| --- | --- | --- |
| Server render (RSC) | page.tsx data assembly | server timing logs; count awaits |
| DB query cost | Prisma includes / nested scope `some` | `log: ["query"]` ([lib/prisma.ts:17](../../../lib/prisma.ts#L17)) then read the SQL/EXPLAIN |
| Network fanout (async jobs) | Inngest bulk enrichment | Inngest dashboard, external API dashboards |
| Bundle size | client components, PDF/tiptap/primereact deps | `next build` output, bundle analyzer |
| Payload serialization | serializeDecimals over large lists | response sizes |
| Cache effectiveness | React `cache()`, revalidatePath | dedup counts in query logs |

## Likely hotspots (investigate, don't assume)

1. **Serial awaits in page loads.** [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) awaits `getAllCrmData()` then `getAccounts()` sequentially. Independent → `Promise.all` halves latency. Grep for the pattern: `rg -n "await get" app/**/page.tsx`.
2. **Nested-`some` scope queries.** [documentReadScopeWhere](../../../lib/authz/scopes/crm.ts#L402-L424) and the linked-account branches for contacts/leads/opportunities ([crm.ts:275-332](../../../lib/authz/scopes/crm.ts#L275-L332)) generate correlated subqueries. For a `user` with many accounts these can be slow; the columns they filter (`assigned_to`, `createdBy`, junction FKs) **need indexes** — verify in the schema (investigate: are `assigned_to`/`createdBy` indexed on each CRM model?).
3. **Wide includes.** [get-accounts.ts:9-34](../../../actions/crm/get-accounts.ts#L9-L34) pulls contacts, watchers-with-user, assigned-user for the whole list. Fine at 100 accounts, a payload problem at 10k; the list UI probably doesn't need every contact name eagerly.
4. **N+1 in list aggregations.** Any per-row `count`/`findMany` inside a `.map` (see [review kata 8](../04-code-reading-gym/04-review-katas.md)) — the fix is `groupBy`. Grep: `rg -n "\.map\(async" actions`.
5. **Client bundle heavies.** `@react-pdf/renderer`, `primereact`, `@tiptap/*`, `recharts`, `mammoth`, `pdf-parse` are large; ensure they're server-only or dynamically imported where used. `next build`'s per-route JS report tells the truth.
6. **Enrichment cost/latency.** Each enrichment is an OpenAI + Firecrawl round trip; bulk fans out one Inngest event per row. The 7-day dedup ([enrich-contact.ts:91-106](../../../inngest/functions/enrich-contact.ts#L91-L106)) is the main throttle; missing per-call HTTP timeouts mean a hung scrape holds a function slot (investigate).

## How to find each class

- **N+1**: `log: ["query"]`, load one list page, count near-identical SELECTs.
- **Serial async**: read for consecutive `await`s with no data dependency.
- **Expensive renders**: React DevTools Profiler on the heavy tables (TanStack Table over large datasets without virtualization).
- **Missing indexes**: EXPLAIN the scope-filtered queries; look for Seq Scan on `crm_Accounts` filtered by `assigned_to`.
- **Oversized bundles**: `next build` report; then dynamic-import or move server-side.
- **Unbounded queries**: `findMany` without `take` — [get-accounts.ts](../../../actions/crm/get-accounts.ts#L6-L38) has no limit; pagination lives in the client table, so the server ships everything (investigate the real row counts).

## The one optimization worth pre-committing

Indexes on ownership columns used by every scope builder. They're on the hot path of *every authenticated CRM read*, and a missing index there taxes the whole app uniformly. Everything else: measure first.

## Interview angle

- "How do you find an N+1?" → query logging + count identical selects; show the `groupBy` fix. [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).
- "This page is slow, walk me through it" → domain table above as a checklist: render → query → payload → bundle, measuring at each.
- "Premature optimization" → point at the wide includes and say why you'd *profile at real row counts* before touching them.
