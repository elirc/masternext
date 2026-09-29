# Tooling and Build System

## pnpm as security tooling

The [package.json `pnpm.overrides`](../../../package.json#L155-L192) block is ~35 forced version pins (tar, minimatch, axios, fast-xml-parser…). This is what **dependency-CVE response** looks like in practice: instead of waiting for every transitive dependency to update, overrides force safe versions across the whole graph. `onlyBuiltDependencies` ([package.json:193-206](../../../package.json#L193-L206)) is pnpm's supply-chain guard: only listed packages (prisma, bcrypt, sharp, esbuild…) may run install scripts — everything else's postinstall is blocked.

Transferable: in interviews, "how do you handle a CVE in a transitive dep?" → overrides/resolutions + lockfile audit, and know that overrides are *tech debt with an expiry* (they mask what upstream actually ships).

## Build pipeline

`pnpm build` = `prisma generate && prisma migrate deploy && next build` ([package.json:11](../../../package.json#L11)). Three consequences:

1. Generated Prisma client is a build artifact — schema edits without `generate` give phantom type errors (explains the `(prismadb as any)` cast in [audit-log.ts:69](../../../lib/audit-log.ts#L68-L70)).
2. Migrations at build time mean the build machine has prod DB credentials, and a bad migration fails the deploy *after* CI is green. Fine for small teams; a scale-up splits migrate into a release step.
3. No `typecheck` script and no test CI ([.github/workflows/](../../../.github/workflows/) has only release-please) — `eslint --max-warnings=0` is the only scripted gate. Finding this gap by reading config, not docs, is exactly the "senior inspects first" skill.

## Styling: Tailwind 4 + shadcn/ui

Tailwind 4 via PostCSS ([postcss.config.js](../../../postcss.config.js), `@tailwindcss/postcss` in deps). Components in [components/ui/](../../../components/ui/) are **vendored shadcn** — copied in, not imported from a package ([components.json](../../../components.json) configures the generator). Implication: you own them; fixing a Button quirk is a normal edit, not a fork. Radix primitives underneath supply accessibility. `cva` (class-variance-authority) + `tailwind-merge` handle variant props — read [components/ui/button.tsx](../../../components/ui/button.tsx) once and you've read them all.

## Test tooling

- **Jest** ([jest.config.ts](../../../jest.config.ts)): ts-jest preset, `testEnvironment: node`, `@/` path alias mapped, **`e2b` mocked globally** via `__mocks__/e2b.ts` (the sandbox SDK would otherwise hit the network at import time). `setupFiles: jest.env.setup.ts` seeds env vars — the pattern for testing env-dependent code like [email-crypto.ts](../../../lib/email-crypto.ts#L7-L15).
- **Playwright** ([playwright.config.ts](../../../playwright.config.ts)): a `setup` project runs [tests/auth.setup.ts](../../../tests/auth.setup.ts) first (login once, save storage state), then 5 browser projects depend on it. `webServer` boots `pnpm dev` automatically. The OTP for login is captured by better-auth's `testUtils` plugin ([lib/auth.ts:90-93](../../../lib/auth.ts#L90-L93)) — a deliberate test seam in production code, NODE_ENV-fenced.

## Lint

Flat config ([eslint.config.mjs](../../../eslint.config.mjs)) with `eslint-config-next`; `--max-warnings=0` turns every warning into a merge blocker. When you contribute, run `pnpm lint` before pushing — there is no CI to catch it for you (inferred).

## Release automation

[release-please.yml](../../../.github/workflows/release-please.yml) + [release-please-config.json](../../../release-please-config.json): conventional commits drive CHANGELOG and version bumps. Practical rule for your PRs here: use `feat:`/`fix:` prefixes or your change is invisible to the changelog.

## Drills

1. `rg "tar@" package.json` — count the tar pins. Why would one package need three range-specific overrides? (Different transitive consumers pin different major ranges; each range needs its own safe floor.)
2. Explain to a duck why mocking `e2b` in jest.config (module-level) beats `jest.mock()` in each test file for a network-touching SDK.

## Interview angle

- "How does your team handle vulnerable transitive dependencies?" → overrides block, [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q12.
- "Describe your ideal CI pipeline" → describe *this repo's gaps* (no test/typecheck CI) and what you'd add first: cheapest-signal-first ordering (lint → typecheck → unit → e2e).
- "Monolithic build vs split migrate step" → the `pnpm build` tradeoff above.
