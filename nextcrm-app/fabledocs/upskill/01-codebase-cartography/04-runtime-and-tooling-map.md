# Runtime and Tooling Map

## Package management & scripts

pnpm ≥ 9, Node ≥ 22.12 ([package.json](../../../package.json#L5-L8)). Key scripts (all __inferred__, not run):

| Script | What it does | Notes |
| --- | --- | --- |
| `pnpm dev` | `next dev` | Turbo dev server; Inngest functions need the Inngest dev server or cloud to actually execute |
| `pnpm build` | `prisma generate && prisma migrate deploy && next build` | **Migrations run during build** — deploys are coupled to schema changes ([package.json](../../../package.json#L11)) |
| `pnpm lint` | `eslint . --max-warnings=0` | zero-warning policy; flat config in [eslint.config.mjs](../../../eslint.config.mjs) |
| `pnpm test` | Jest (ts-jest, node env) | config: [jest.config.ts](../../../jest.config.ts) — matches `**/__tests__/**/*.test.ts(x)`, maps `@/*` to root, mocks `e2b` |
| `pnpm test:e2e` | Playwright | [playwright.config.ts](../../../playwright.config.ts): auth setup project, 5 browser targets, `webServer: pnpm dev` |
| `pnpm migrate:mongo-to-postgres` | tsx script | the historical Mongo→Postgres data migration |

There is **no typecheck script**; type safety is enforced by `next build` and the editor. There is **no CI workflow for tests** — [.github/workflows/](../../../.github/workflows/) contains only `release-please.yml` (version/changelog automation). That absence is itself a finding for the architecture critique.

## Runtime boundaries

| Boundary | Runs | Watch out |
| --- | --- | --- |
| Browser | `"use client"` components, jotai atoms, react-hook-form | no secrets, no Prisma; Decimals must be serialized first ([serialize-decimals.ts](../../../lib/serialize-decimals.ts#L5-L21)) |
| Edge-ish middleware | [proxy.ts](../../../proxy.ts) | only cookie *presence* — cannot query DB; that's why role checks live server-side |
| Node server | RSC pages, server actions, API routes, Inngest handlers | Prisma via pg adapter singleton ([lib/prisma.ts](../../../lib/prisma.ts#L10-L46)) — dev hot-reload caching via `global.cachedPrisma` |
| Inngest execution | [inngest/functions/](../../../inngest/functions/) invoked over HTTP by Inngest | `step.run` results are **serialized/replayed** — Dates come back as strings on retry |
| External | Resend, IMAP, OpenAI/Firecrawl, MinIO, Upstash, ECB FX, e2b | every one has an env var; several have graceful fallbacks (rate limiter returns null in dev — [rate-limit.ts](../../../lib/enrichment/rate-limit.ts#L6-L11)) |

## Persistence toolchain

Prisma 7 with the **pg driver adapter** (not the default engine binary) — [lib/prisma.ts](../../../lib/prisma.ts#L12-L13). Postgres needs **pgvector** (docker image `pgvector/pgvector:pg17` in [docker-compose.yml](../../../docker-compose.yml)). Migrations in [prisma/migrations/](../../../prisma/migrations/) read like a project history: pgvector embeddings, api tokens, campaigns module, soft-delete columns, audit log, better-auth tables, currency support. Seeds at [prisma/seeds/seed.ts](../../../prisma/seeds/) via `ts-node`.

## Env vars (high level, no secrets)

Groups you'll actually need locally: `DATABASE_URL`; `BETTER_AUTH_SECRET`/`BETTER_AUTH_URL` + `GOOGLE_ID`/`GOOGLE_SECRET`; `EMAIL_ENCRYPTION_KEY` (64 hex chars — enforced at [email-crypto.ts](../../../lib/email-crypto.ts#L7-L15)); `RESEND_API_KEY` + `RESEND_WEBHOOK_SECRET`; MinIO creds; `INNGEST_*`; optional `OPENAI_API_KEY`/`FIRECRAWL_API_KEY` (or store per-user encrypted keys — [api-keys.ts](../../../lib/api-keys.ts#L21-L46)); optional `UPSTASH_REDIS_REST_URL/TOKEN`. See [.env.example](../../../.env.example).

## Testing toolchain

- **Unit (Jest)**: pure-logic tests dominate — invoice math, permissions, numbering, token/crypto helpers ([__tests__/lib/](../../../__tests__/lib/)). DB-touching code is mostly *not* unit tested; the boundary was drawn at pure functions.
- **E2E (Playwright)**: [tests/auth.setup.ts](../../../tests/auth.setup.ts) logs in once (using better-auth's `testUtils` OTP capture — [lib/auth.ts](../../../lib/auth.ts#L90-L93)) and shares storage state across 24 spec files. Several specs are documented skeletons with TODO bodies ([tests/e2e/invoices.spec.ts](../../../tests/e2e/invoices.spec.ts)) — read them as executable documentation, not proof of coverage.

## Interview angle

1. "Walk me through what happens when you run `pnpm build` here and why running migrations in build is a tradeoff." (Coupling deploy+migrate: simple, but a failed migration blocks rollout and rollback needs a down-path — this repo has none.)
2. "Where would a type error surface if there's no typecheck script?" → `next build` / editor; discuss CI gap.
3. "Why does the Prisma client need a singleton in dev?" → hot reload creates new module instances; `global.cachedPrisma` ([lib/prisma.ts](../../../lib/prisma.ts#L37-L44)) prevents connection exhaustion. Classic question; this repo is a clean real example.
