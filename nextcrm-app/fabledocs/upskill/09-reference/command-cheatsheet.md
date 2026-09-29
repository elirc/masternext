# Command Cheatsheet

All commands **__inferred__** from [package.json](../../../package.json), [docker-compose.yml](../../../docker-compose.yml), and config files unless marked __verified__. None were executed while writing this curriculum except the read-only exploration in the [verification log](verification-log.md). Requires Node ≥ 22.12, pnpm ≥ 9.

## Setup

```bash
docker compose up -d postgres minio        # __inferred__ — infra (pgvector Postgres + MinIO)
cp .env.example .env                        # then fill secrets (see below)
pnpm install                                # __inferred__
pnpm exec prisma migrate deploy             # __inferred__ — apply migrations/
pnpm exec prisma db seed                    # __inferred__ — seed via prisma.seed (package.json:22-24)
```

Required secrets to fill: `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_ID`/`GOOGLE_SECRET`, `EMAIL_ENCRYPTION_KEY` (`openssl rand -hex 32`), `RESEND_API_KEY`, MinIO creds, `INNGEST_*`. Optional: `OPENAI_API_KEY`, `FIRECRAWL_API_KEY`, `UPSTASH_REDIS_REST_URL`/`TOKEN`.

## Dev / build

```bash
pnpm dev            # __inferred__ — next dev on :3000
pnpm build          # __inferred__ — prisma generate && prisma migrate deploy && next build
pnpm start          # __inferred__ — next start
pnpm lint           # __inferred__ — eslint . --max-warnings=0  (the only scripted gate)
```

There is **no** `typecheck` script — type errors surface via `pnpm build` or the editor. Run `pnpm exec tsc --noEmit` (__inferred__) to typecheck without building.

## Tests

```bash
pnpm test                       # __inferred__ — all Jest unit/route tests
pnpm test -- numbering          # __inferred__ — single suite by path substring
pnpm test -- totals             # __inferred__ — invoice math
pnpm test:e2e                   # __inferred__ — Playwright (boots pnpm dev, needs seeded DB)
pnpm test:e2e:ui                # __inferred__ — Playwright UI mode
pnpm test:e2e:headed            # __inferred__
pnpm test:e2e:debug             # __inferred__
```

Jest config: [jest.config.ts](../../../jest.config.ts) (ts-jest, node env, `@/` alias, `e2b` mocked). Playwright: [playwright.config.ts](../../../playwright.config.ts) (auth setup project + 5 browsers).

## Prisma / DB

```bash
pnpm exec prisma migrate dev --name <change>   # __inferred__ — create+apply a dev migration
pnpm exec prisma generate                      # __inferred__ — regenerate client (fixes stale-type casts)
pnpm exec prisma studio                        # __inferred__ — DB browser
pnpm exec prisma migrate deploy                # __inferred__ — apply pending migrations (prod path)
```

## Migration scripts (Mongo → Postgres, historical)

```bash
pnpm migrate:mongo-to-postgres     # __inferred__ — scripts/migrate-mongo-to-postgres.ts
pnpm validate:migration            # __inferred__ — scripts/validate-migration.ts
bash scripts/db-backup.sh          # __inferred__ — DB backup (check if scheduled)
```

## Useful exploration (read-only, safe)

```bash
rg --files | wc -l                              # file count (~1404)
rg -n "^model|^enum" prisma/schema.prisma        # list all models/enums
rg -l "use server" actions                       # find server actions
rg "consumeNextNumber"                           # find all callers of the numbering helper
rg ": any" lib actions app | wc -l               # the repo's `any` honesty score
rg -l "getSession|requireAuthenticated|getUser"  # the three auth idioms
```

## Docker (full stack)

```bash
docker compose up -d              # __inferred__ — all services ([docker-compose.yml](../../../docker-compose.yml))
docker compose logs -f postgres   # __inferred__
```

Note: Postgres/MinIO ports are commented out by default in docker-compose.yml — uncomment to expose to host.
