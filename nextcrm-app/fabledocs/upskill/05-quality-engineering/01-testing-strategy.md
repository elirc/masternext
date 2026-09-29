# Testing Strategy

## The layers as they exist

| Layer | Tooling | Where | What it actually covers |
| --- | --- | --- | --- |
| Unit (pure logic) | Jest + ts-jest | [__tests__/lib/](../../../__tests__/lib/), colocated `__tests__/` next to actions/routes | invoice totals/numbering/permissions, api tokens, crypto, currency, campaign/enrichment helpers |
| Route/action tests | Jest (node env) | e.g. [contacts route tests](../../../app/api/crm/contacts/%5Bid%5D/__tests__/route.test.ts), [reports export tests](../../../app/api/reports/export/__tests__/) | handler logic with mocked deps |
| E2E | Playwright, 5 browser projects | [tests/e2e/](../../../tests/e2e/) — 24 specs | real app: auth, CRM CRUD, campaigns, products, sales flow; **several specs are TODO skeletons** ([invoices.spec.ts](../../../tests/e2e/invoices.spec.ts)) |
| CI | — | [.github/workflows/](../../../.github/workflows/) | **nothing runs tests automatically** (release-please only) — the single biggest quality gap |

## Where the repo draws the unit-test boundary (and why it's smart)

The tested code is deliberately **infrastructure-free**: [permissions.ts](../../../lib/invoices/permissions.ts) takes context objects, [totals.ts](../../../lib/invoices/totals.ts) takes Decimals, `diffObjects` takes plain objects, `shouldSkipBulkEnrichment` was *extracted and exported specifically to be testable* ([enrich-contact.ts:11-18](../../../inngest/functions/enrich-contact.ts#L11-L18)). The design rule to steal: **when logic is hard to test, move the logic, not the test** — pull decisions out of handlers into pure functions and test those.

What's consequently *not* unit-covered: everything Prisma-touching (scope builders' actual filtering behavior, the numbering counter under concurrency, soft-delete write paths). The scope helpers construct `where` objects — a test could at least snapshot those objects per role (cheap, catches accidental broadening — see recipe 4).

## Test seams built into production code

- better-auth `testUtils` OTP capture, NODE_ENV-fenced ([lib/auth.ts:90-93](../../../lib/auth.ts#L90-L93)) — lets [tests/auth.setup.ts](../../../tests/auth.setup.ts) log in without a real mailbox, then every E2E project reuses the storage state.
- Global `e2b` mock in [jest.config.ts](../../../jest.config.ts) (`__mocks__/e2b.ts`) — network SDKs neutralized at config level.
- [jest.env.setup.ts](../../../jest.env.setup.ts) seeds env vars so env-validating code ([email-crypto.ts:7-15](../../../lib/email-crypto.ts#L7-L15)) can run.
- Fixtures in [tests/fixtures/](../../../tests/fixtures/) and deterministic seeds ([prisma/seeds/](../../../prisma/seeds/)).

## What belongs at which layer (this repo's implicit policy, made explicit)

- Money math, status machines, formatting, diffing → unit. Fast, exhaustive, `it.each` tables ([permissions.test.ts](../../../__tests__/lib/invoices/permissions.test.ts) shows the style).
- Authorization *decisions* → unit on the pure part (scope-where shape) + one route test proving the wiring.
- Cross-layer behavior (form → action → DB → revalidate) → E2E, one happy path + one denial path per entity.
- Don't E2E-test what a unit test can kill: VAT bucket math through the browser is waste.

## Flake discipline

Playwright config: `retries: CI ? 2 : 0`, `workers: CI ? 1 : undefined`, trace/screenshot/video on failure ([playwright.config.ts](../../../playwright.config.ts)). Time and randomness: token generation uses `crypto.randomBytes` (untestable by equality — tests assert *properties*: prefix, length, hash match); dates flow as parameters (`consumeNextNumber(tx, seriesId, now)` — [numbering.ts:13-17](../../../lib/invoices/numbering.ts#L13-L17) takes `now` injectable — a small, excellent choice).

## Interview angle

- "How do you decide unit vs integration vs E2E?" → the boundary-drawing story above, with `shouldSkipBulkEnrichment` as the extract-to-test example. [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
- "How do you test auth flows in E2E?" → OTP capture seam, NODE_ENV fencing, storage-state reuse.
- "Your CI takes 40 minutes — what do you do?" → this repo inverted: CI takes 0 minutes; describe building the pyramid cheapest-first.
