# Language & Runtime Model

## Event loop and async in this repo

Node's event loop means every `await` yields; anything between two awaits is not atomic. Concrete instances here:

- **Serial awaits cost latency**: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) awaits `getAllCrmData()` then `getAccounts()` sequentially. `Promise.all` would halve the wall time (they're independent). Recognizing serial-vs-parallel awaits in review is a classic mid-level signal.
- **Check-then-act is racy**: `consumeNextNumber` ([numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30)) reads the counter, computes, then writes — two round trips. Under concurrency, two callers can read the same counter. This repo neutralizes it with a Serializable transaction at the call site ([issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128)); the JS itself cannot save you.
- **Fire-and-forget with intent**: `void inngest.send(...)` ([update-account.ts:61](../../../actions/crm/accounts/update-account.ts#L61)) and the `void Promise.resolve(...).catch(() => {})` `lastUsedAt` update ([api-tokens.ts:60-62](../../../lib/api-tokens.ts#L59-L62)). The `void` marks "I know this is unawaited." Unhandled rejections crash Node ≥ 15, so the `.catch` is load-bearing, not decoration.

## Microtasks vs macrotasks (interview staple, light repo presence)

Promise callbacks run in the microtask queue and drain completely before timers/I-O. You rarely debug this directly here, but it explains why `void`-ed promises still execute before the process idles, and why `await`ing in a loop serializes I/O — see the per-line-item updates in [issue-invoice.ts:83-89](../../../actions/invoices/issue-invoice.ts#L83-L89) (`for...of` + await = serial; acceptable inside a transaction where ordering doesn't matter but the connection is single anyway).

## Serialization boundaries

Three places where "it's just an object" stops being true:

1. **Server → client component**: Prisma `Decimal` objects don't cross; hence [serialize-decimals.ts](../../../lib/serialize-decimals.ts#L5-L21) duck-types `toNumber` and converts. Note it's **shallow** — nested relations with Decimals need care; server actions return `serializeDecimals(invoice)` at every exit ([create-invoice.ts:100](../../../actions/invoices/create-invoice.ts#L100)).
2. **Inngest step memoization**: `step.run` results are JSON-serialized and replayed on retry — a `Date` loaded in [send-step.ts:20-29](../../../inngest/functions/campaigns/send-step.ts#L20-L29) is a `Date` on first run and a **string** on replay. Any `instanceof Date` check downstream is a latent retry-only bug class (none observed here, but check before adding one).
3. **Event payloads**: the enrichment worker gets exactly `{contactId, enrichmentId, fields, triggeredBy}` ([enrich-contact.ts:40-45](../../../inngest/functions/enrich-contact.ts#L40-L45)) — no closures, no session. Async boundaries force explicit contracts.

## Node vs edge vs browser

- [proxy.ts](../../../proxy.ts) runs in the middleware runtime: no Prisma, no Node crypto streams — which is *why* it only checks cookie presence and defers role checks to the server ([proxy.ts:8](../../../proxy.ts#L8-L9) comment).
- Node-only libs: `crypto` ([email-crypto.ts](../../../lib/email-crypto.ts), [api-tokens.ts](../../../lib/api-tokens.ts)), `pg` pool ([lib/prisma.ts:12](../../../lib/prisma.ts#L12)), IMAP, PDF rendering. Importing any of these from a `"use client"` file is a build error — the module graph is the boundary enforcement.
- Browser: jotai atoms, react-hook-form, TanStack Table. Nothing here may see `process.env` secrets; only `NEXT_PUBLIC_*` is inlined.

## Closures and module state

[lib/prisma.ts:35-46](../../../lib/prisma.ts#L35-L46) is the canonical "module-level singleton + dev global cache" pattern: in dev, hot reload re-evaluates modules, so the client is stashed on `global` to avoid a new connection pool per save. Explaining *why this exists* is a solid interview answer about module caching semantics.

## Pitfall checklist

- [ ] Two independent awaits in sequence? Consider `Promise.all`.
- [ ] Read-then-write to DB? Name the isolation level or use a scoped `updateMany`.
- [ ] Unawaited promise? Must carry `void` + `.catch`.
- [ ] Value crossing RSC→client or step-replay boundary? Assume JSON-only.
- [ ] `Date`/`Decimal`/`BigInt` at a boundary? Convert explicitly.

## Drills

1. Find every `Promise.all` in `actions/` (`rg "Promise.all" actions`) and classify each as latency optimization vs necessity.
2. In [issue-invoice.ts](../../../actions/invoices/issue-invoice.ts#L83-L89), estimate the round trips for a 20-line invoice and sketch the `updateMany`/`Promise.all` alternative — then explain why parallelism inside one Prisma transaction doesn't actually parallelize (single connection).

## Interview angle

- "Explain the event loop / microtasks" → anchor with the `void`-ed Inngest send. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q1–Q3.
- "What's a race condition you've seen?" → invoice counter, Q7.
- "Why can't middleware check roles?" → edge runtime constraints, Q14.
