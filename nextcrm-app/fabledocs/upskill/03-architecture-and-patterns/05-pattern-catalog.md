# Pattern Catalog

16 cards. Goal: recognition in *any* repo, not name-dropping. Each: problem → shape → real anchors → failure modes → when to use/avoid → interview angle → drill.

---

## Pattern 1: Authorization as a query fragment (scoped where)

Problem it solves: object-level authorization that can't drift from the query it guards.
General shape: policy function returns a `where` fragment; every read/write spreads it in.
Real example: [accountReadScopeWhere, crm.ts:229-237](../../../lib/authz/scopes/crm.ts#L229-L237) used at [get-accounts.ts:7-8](../../../actions/crm/get-accounts.ts#L7-L8).
Second example: [contactReadScopeWhere, crm.ts:290-302](../../../lib/authz/scopes/crm.ts#L290-L302).
Why this works: authz executes atomically with the query; composes into `findMany`/`count`/`updateMany`; one file to audit.
Failure modes: a query that forgets to spread it (nothing forces usage — see updateAccount); scope drifting from schema when ownership columns change.
Use when: app-level policy over shared tables. Avoid when: DB offers RLS and you can push policy down (this app has no tenant column to key RLS on).
Interview angle: "How do you prevent IDOR at scale?" — this, plus enforcement.
Drill: write `taskReadScopeWhere` for the Tasks model from schema alone, then compare with the board-scope exports in [authz/index.ts:72-79](../../../lib/authz/index.ts#L72-L79).

## Pattern 2: Scoped write (authorize-in-the-update)

Problem: check-then-write leaves a TOCTOU gap and two round trips.
Shape: `updateMany({ where: {id, ...ownershipOR}, data })`; `count === 0` → 404.
Real example: [tryScopedUpdateContact, crm.ts:33-43](../../../lib/authz/scopes/crm.ts#L33-L43) via [contacts PATCH route:51-52](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L51-L52).
Second example: [tryScopedUpdateTarget, crm.ts:45-55](../../../lib/authz/scopes/crm.ts#L45-L55).
Failure modes: can't distinguish "not found" from "forbidden" (here that's a feature); doesn't return the updated row (needs a follow-up read).
Interview angle: atomicity of authorization — few candidates can articulate it.
Drill: convert `deleteAccount` ([delete-account.ts:13-16](../../../actions/crm/accounts/delete-account.ts#L13-L16)) to this pattern on paper.

## Pattern 3: Status machine as pure functions

Problem: lifecycle rules scattered across handlers.
Shape: `can<Verb>(entityCtx, userCtx): boolean` pure module; callers gate on it.
Real example: [lib/invoices/permissions.ts:22-48](../../../lib/invoices/permissions.ts#L22-L48); consumed at [issue-invoice.ts:30-37](../../../actions/invoices/issue-invoice.ts#L30-L37).
Second example: `isInvoiceImmutable` ([permissions.ts:10-12](../../../lib/invoices/permissions.ts#L10-L12)).
Failure modes: nothing *forces* callers to consult it; DB has no CHECK backing it, so a raw update can create illegal states.
Use when: >2 states with role-dependent transitions. Avoid when: a boolean flag suffices.
Interview angle: "how would you model an order lifecycle" — answer with this file's shape.
Drill: draw the full status graph from the `PAYMENT_ALLOWED` set ([permissions.ts:41-43](../../../lib/invoices/permissions.ts#L41-L48)) and find one state no code can currently reach (candidates: DISPUTED, REFUNDED — investigate).

## Pattern 4: Serializable-transaction counter

Problem: gap-free, template-formatted legal numbering.
Shape: counter row + read-modify-write inside `$transaction({isolationLevel:"Serializable"})`.
Real example: [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) + [issue-invoice.ts:56-58,128](../../../actions/invoices/issue-invoice.ts#L56-L58).
Second example: none — single instance (that's part of the lesson: rare tools stay unfamiliar).
Failure modes: caller forgets the isolation option (compiles fine, duplicates under load); serialization conflicts throw and there's no retry wrapper.
Use when: legal/gap-free sequences. Avoid when: gaps are fine — use a DB sequence and stop paying the serialization tax.
Interview angle: "why not just SERIAL?" — rollback gaps, per-series templates, yearly reset.
Drill: write the `SELECT ... FOR UPDATE` variant and compare failure behavior (blocking vs abort-retry).

## Pattern 5: Snapshot-on-transition (denormalize at the boundary)

Problem: mutable master data vs immutable documents.
Shape: copy the fields you legally froze into the row at the state transition.
Real example: [billingSnapshot, issue-invoice.ts:60-71](../../../actions/invoices/issue-invoice.ts#L60-L71).
Second example: [taxRateSnapshot, issue-invoice.ts:82-89](../../../actions/invoices/issue-invoice.ts#L82-L89).
Failure modes: forgetting one field (renders wrong forever); snapshotting too early (draft edits lost).
Interview angle: "normalize or denormalize?" — answer: by mutability contract, not dogma.
Drill: find what the PDF renders from — snapshot or live account? ([issue-invoice.ts:133-141](../../../actions/invoices/issue-invoice.ts#L131-L141) uses `result.account.*` — live! Investigate whether that's a bug given the snapshot exists.)

## Pattern 6: Best-effort side channel (swallow-and-log)

Problem: secondary writes must not break primary mutations.
Shape: try/catch around the side write; log; never rethrow.
Real example: [writeAuditLog, audit-log.ts:66-82](../../../lib/audit-log.ts#L66-L82).
Second example: PDF post-step ([issue-invoice.ts:204-207](../../../actions/invoices/issue-invoice.ts#L204-L207)).
Failure modes: silent decay (no metric on failure count); people *assume* the audit trail is complete in incident review.
Use when: side channel is genuinely optional. Avoid when: compliance requires the trail — then it belongs in the transaction.
Interview angle: "what should fail a request?" — blast-radius reasoning.
Drill: list the three stances (tolerate/swallow/void) from [04-side-effects](04-side-effects-async-and-reliability.md) with one anchor each, from memory.

## Pattern 7: Step-memoized workflow (durable execution)

Problem: multi-step jobs where retries must not repeat completed side effects.
Shape: wrap each effect in `step.run("name", fn)`; engine persists results and replays.
Real example: [send-step.ts:20-68](../../../inngest/functions/campaigns/send-step.ts#L20-L68) — load, send, record as three steps.
Second example (counter-example): [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts) — no steps; retries re-pay the AI bill.
Failure modes: step results are JSON (Dates→strings on replay); non-deterministic code between steps breaks replay assumptions.
Interview angle: "how do you make a workflow resumable?" — durable execution is the current-generation answer to sagas.
Drill: rewrite enrich-contact with steps on paper; name each step and what memoization saves.

## Pattern 8: Field allowlist at an update boundary

Problem: mass assignment (client setting columns it shouldn't).
Shape: explicit map of permitted input→column; iterate the map, never the input.
Real example: [FIELD_MAP, contacts route:10-20](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L10-L20).
Second example: [contactFieldMap, enrich-contact.ts:20-30](../../../inngest/functions/enrich-contact.ts#L20-L30).
Failure modes: allowlist drifts from product needs (too strict = silent no-ops — this route returns 400 on zero valid fields to surface it).
Interview angle: classic Rails-era question, still asked; `...spread` of request bodies into ORMs is the modern sin.
Drill: find one action that spreads client input into Prisma unfiltered (`rg "\.\.\.rest" actions` — [update-account.ts:47](../../../actions/crm/accounts/update-account.ts#L46-L48) qualifies).

## Pattern 9: Hash-at-rest credentials

Problem: DB leak must not leak usable API tokens.
Shape: store SHA-256(token); look up by hash; show only a prefix.
Real example: [api-tokens.ts:8-10,29-44](../../../lib/api-tokens.ts#L8-L44).
Second example (contrast): reversible AES-GCM for provider keys ([email-crypto.ts](../../../lib/email-crypto.ts), [api-keys.ts](../../../lib/api-keys.ts)) — encrypt what you must replay, hash what you only verify.
Failure modes: unsalted fast hash is fine *only* because tokens are 24 random bytes (no dictionary); prefix display must come from a separate column, not the hash.
Interview angle: "how do you store passwords vs API keys?" — three-way contrast: bcrypt (low-entropy), SHA-256 (high-entropy tokens), AES-GCM (replayable secrets).
Drill: explain why bcrypt here would be pure overhead (rate of legit validations × entropy of input).

## Pattern 10: Capability URL

Problem: authorize an action for someone with no account (email recipient).
Shape: unguessable token in the link *is* the credential.
Real example: unsubscribe token ([send-step.ts:48](../../../inngest/functions/campaigns/send-step.ts#L47-L49) → [unsubscribe route:11-24](../../../app/api/campaigns/unsubscribe/route.ts#L11-L24)).
Second example: presigned S3 PUT URLs ([presigned-url/route.ts:51](../../../app/api/upload/presigned-url/route.ts#L50-L56)) — time-boxed capability.
Failure modes: leaks via Referer/logs; state-changing **GET** lets prefetchers fire it (this repo's confirmed shape); no expiry on unsubscribe tokens (acceptable — unsubscribing twice is idempotent).
Interview angle: "design a password-reset flow" — same pattern with expiry + single-use added; be able to say *why* reset needs both and unsubscribe needs neither.
Drill: classify which of the two failure modes applies to each of: unsubscribe link, presigned PUT, invite link.

## Pattern 11: Layered key resolution (env → system → user)

Problem: one deployment serves ops-provided, admin-provided, and user-provided API keys.
Shape: ordered fallback chain, first hit wins.
Real example: [getApiKey, api-keys.ts:21-46](../../../lib/api-keys.ts#L21-L46).
Second example: FROM-address fallbacks ([issue-invoice.ts:146-149](../../../actions/invoices/issue-invoice.ts#L144-L156) settings→env→literal).
Failure modes: precedence surprises (user sets a key, env silently wins — confirmed order here: env beats user); no per-tier telemetry of which tier served.
Interview angle: config layering — same idea as CSS cascade / 12-factor config; show you can reason about precedence bugs.
Drill: argue for and against flipping tiers 1 and 3 for the USER scope.

## Pattern 12: Soft delete with scope-enforced visibility

Problem: recoverability + referential integrity vs "gone" semantics.
Shape: `deletedAt` timestamp; every read scope filters it; restore = null it.
Real example: [delete-account.ts:13-16](../../../actions/crm/accounts/delete-account.ts#L13-L16) + `{deletedAt: null}` in every scope builder.
Second example: [restore-account.ts](../../../actions/crm/accounts/restore-account.ts); inventory table in [docs/soft-delete-gaps.md](../../../docs/soft-delete-gaps.md).
Failure modes: writes not filtered (edit-after-delete, confirmed here); uniques colliding with tombstones (unique email + soft-deleted duplicate); tables that must purge for GDPR anyway.
Interview angle: classic. This repo adds the rarely-mentioned *write-path* gap.
Drill: find one unique constraint in the schema that a tombstone could block from reuse.

## Pattern 13: Webhook signature verification over raw body

Problem: authenticating push events from a third party.
Shape: read raw text → HMAC with shared secret → compare → then parse.
Real example: [resend route:13-19](../../../app/api/campaigns/webhooks/resend/route.ts#L13-L19).
Second example: Inngest handles its own signing (passed through at [proxy.ts:21-24](../../../proxy.ts#L21-L24)).
Failure modes: parsing before verifying; comparing with `===` (timing side channel — present here, low practical risk over network jitter but `timingSafeEqual` is free); missing replay protection (no event-id store).
Interview angle: "secure a webhook" — raw body, constant-time compare, replay window, idempotent handlers: four points, you have anchors for all.
Drill: write the `crypto.timingSafeEqual` patch including the length-check it requires.

## Pattern 14: Per-request dedup with React cache()

Problem: several server components need the same query in one render pass.
Shape: wrap the fetch in `cache(fn)`; call freely.
Real example: [get-accounts.ts:5](../../../actions/crm/get-accounts.ts#L5).
Second example: `rg "^import { cache }" actions` — most `get-*` actions follow it.
Failure modes: assuming it caches across requests (it doesn't — request-scoped); memoizing a function with side effects.
Interview angle: distinguish request-dedup vs data cache vs router cache in Next — most candidates blur them.
Drill: explain why `requireAuthenticated` inside the cached function is still safe.

## Pattern 15: Typed action result envelope

Problem: server actions can't throw rich errors across the wire ergonomically.
Shape: `{ fieldErrors?, error?, data? }` discriminated-ish envelope + schema-validating wrapper.
Real example: [create-safe-action.ts:7-28](../../../lib/create-safe-action.ts#L7-L28).
Second example (divergence): ad-hoc `{ error: string } | { data }` returns in [update-account.ts:35,63,66](../../../actions/crm/accounts/update-account.ts#L34-L67) — the wrapper exists but adoption is partial.
Failure modes: two error dialects in one codebase → UI handles one and silently drops the other.
Interview angle: "errors as values vs exceptions" with an RPC twist.
Drill: count adopters: `rg -l "createSafeAction" actions`.

## Pattern 16: Vendored UI kit (shadcn)

Problem: design-system velocity without library lock-in.
Shape: generator copies component source into the repo; you own it.
Real example: [components/ui/](../../../components/ui/) + [components.json](../../../components.json).
Second example: table components duplicated per entity (accounts/leads/... `table-components/`) — the same philosophy applied to feature code, with the same tradeoff (drift between copies).
Failure modes: upstream fixes don't arrive; copies diverge; "just patch it" erodes consistency.
Interview angle: "build vs buy vs vendor" for UI kits.
Drill: diff accounts vs contacts `data-table.tsx` (`diff` or side-by-side) and list divergences — then argue whether extracting a shared generic table is worth it (see [refactor katas](../06-contribution-practice/04-refactor-and-design-katas.md)).
