# Senior Build Projects

Six projects, 2 days–4 weeks. Each is scoped like a real proposal a maintainer might accept, with the sections a senior would write before touching code.

---

## Project 1: Authorization enforcement layer (2–3 weeks)

**Problem**: object-level authz is present on reads and API routes but bypassable in legacy server actions ([risk #1](../09-reference/risk-register.md)); nothing *forces* an action to authorize.
**Product value**: closes the advisory class permanently and prevents regressions.
**Design checklist**: enumerate every mutating server action (`rg -l "use server" actions | xargs rg -l "prismadb.*\.(update|delete|create)"`); classify by current idiom; decide the target idiom (scoped `updateMany` where possible, `assertCan*` otherwise).
**Architecture decisions**: enforce via (a) a lint rule banning `getSession` outside lib/, and/or (b) a thin `authorizedAction(schema, scope, handler)` wrapper all mutations must use. Pick one; justify.
**Likely files**: all of `actions/crm/**`, `actions/invoices/**`, new `lib/authz/enforce.ts`, eslint config.
**Migration plan**: one entity per PR; each PR migrates + tests + keeps behavior. Compatibility: none needed (tightening).
**Test plan**: per action, a denial test (non-owner `user` → error) and an allow test. This *creates* the authz test suite the repo lacks.
**Security/perf/rollout**: rollout is incremental and reversible per PR; perf cost is one extra query for assert-then-act paths (or zero for scoped-updateMany).
**Open questions**: manager vs admin split (depends on M4); do read-only actions need the wrapper?
**Stretch**: generate the enforcement wrapper types from Prisma model names.
Interview story: "led the remediation of an IDOR advisory class across ~40 write paths with zero regressions."

## Project 2: Test + CI foundation (1–2 weeks)

**Problem**: no automated test/typecheck CI ([risk #2](../09-reference/risk-register.md)); coverage is real but unguarded.
**Value**: every future PR gets a safety net; this unblocks Projects 1, 3, 5.
**Design checklist**: GitHub Actions workflow — lint → `tsc --noEmit` → jest → (nightly) Playwright; a test Postgres+MinIO in services; seed + migrate in CI; artifact the Playwright trace.
**Decisions**: cheapest-signal-first ordering; which E2E run per-PR vs nightly (they're slow and some are skeletons).
**Likely files**: `.github/workflows/ci.yml`, a `typecheck` script in package.json, jest CI config, docker services in the workflow.
**Rollout/rollback**: start non-blocking (report-only) for a week, then required.
**Open questions**: secrets for E2E (OTP capture removes the mailbox need — reuse [auth.setup.ts](../../../tests/auth.setup.ts)).
Interview story: "stood up the first CI pipeline for a project that had none, sequenced to give fast signal."

## Project 3: Transactional outbox for domain events (2–3 weeks)

**Problem**: state changes and their Inngest events are separate steps (`update` then `void inngest.send`) — a crash between loses the event ([risk in 04-side-effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
**Value**: search embeddings and future event consumers become as durable as the data.
**Design checklist**: an `outbox` table written *in the same transaction* as the mutation; a relay (Inngest scheduled fn or Postgres LISTEN) that emits and marks sent; at-least-once semantics + idempotent consumers.
**Decisions**: mark-sent vs delete; relay cadence vs latency; which events migrate first (embeddings — lowest stakes).
**Migration**: additive table; dual-write (old `inngest.send` + outbox) during a compatibility window; cut over per event type.
**Test plan**: crash-injection test (write tx commits, process dies before relay) → event still delivered.
**Open questions**: do we need ordering guarantees? (Embeddings: no.)
Interview story: "introduced a transactional outbox to stop losing domain events on crash, migrated incrementally."

## Project 4: Numbering correctness hardening (2–4 days)

**Problem**: gap-free invoice numbering is correct only because *every* caller remembers Serializable ([risk #3](../09-reference/risk-register.md)); no retry on serialization abort.
**Value**: makes duplicate legal numbers impossible regardless of caller, and removes user-facing serialization errors.
**Design checklist**: move the lock into `consumeNextNumber` (use `SELECT ... FOR UPDATE` on the series row so any isolation level is safe), or wrap issuance in a retry-on-serialization-failure loop. Compare both.
**Decisions**: FOR UPDATE (blocking, simpler) vs Serializable+retry (more concurrent, more code).
**Likely files**: [numbering.ts](../../../lib/invoices/numbering.ts), [issue-invoice.ts](../../../actions/invoices/issue-invoice.ts).
**Test plan**: concurrency integration test — N parallel issuances → N distinct numbers, zero errors.
Interview story: "eliminated a duplicate-invoice-number race by relocating a lock into the function that owns the invariant."

## Project 5: Observability baseline (1–2 weeks)

**Problem**: log-and-pray; swallowed failures are invisible ([05/06](../05-quality-engineering/06-observability-and-operations.md)).
**Value**: incidents become detectable before users report them.
**Design checklist**: structured JSON logging with request/trace ids; counters on the two intentional-swallow paths (audit-write failure, PDF-gen failure); `/api/health`; Inngest failure → alert channel.
**Decisions**: minimal (structured logs + a few counters) vs full (OpenTelemetry). Recommend minimal first.
**Rollout**: adopt the logger in invoice + campaign paths first, expand.
Interview story: "gave a blind service enough observability to detect its own failures, starting with the highest-risk swallowed errors."

## Project 6: Generic CRM entity table (3–4 weeks, higher risk)

**Problem**: `table-components/` are duplicated per entity ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)); fixes must be applied N times.
**Value**: one place to fix table behavior; consistency.
**Design checklist**: extract a `<CrmDataTable columns config>` generic over row type; migrate accounts + contacts first as proof; keep column defs per entity.
**Decisions — and a warning**: only pursue if a *concrete* divergence bug has occurred; otherwise this is refactor-for-aesthetics with real regression risk across every list view. A senior *declining* this with that reasoning is also a strong answer.
**Test plan**: E2E per migrated entity list (sort, filter, paginate, row actions) — the safety net Project 2 provides is a prerequisite.
Interview story: "consolidated duplicated table code — and scoped it to migrate incrementally behind E2E coverage" *or* "argued against a tempting refactor because the blast radius outweighed the DRY benefit."

---

## Using these

Pick one, write the full design doc *before* code, and treat the "open questions" as things to resolve with a maintainer — that conversation is itself the senior skill. Projects 2 → 1/4 → 3/5 is a sensible dependency order.
