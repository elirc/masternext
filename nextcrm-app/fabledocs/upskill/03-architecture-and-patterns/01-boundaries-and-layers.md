# Boundaries and Layers

A **boundary** is a line where responsibility changes hands and assumptions must be re-verified. An **invariant** is a fact that must stay true no matter which code path runs. Layers exist to give each invariant one owner.

## The layer stack, with owners

| Layer | Owns (its invariants) | Must not own |
| --- | --- | --- |
| Middleware ([proxy.ts](../../../proxy.ts)) | "unauthenticated browsers don't reach app pages"; locale routing | role decisions (edge: no DB) |
| RSC pages | composition, per-request data assembly | business rules, raw Prisma |
| Server actions ([actions/](../../../actions/)) | one use-case per file: validate → authorize → mutate → side-effects → revalidate | rendering, cross-entity policy (belongs in lib) |
| Domain lib ([lib/](../../../lib/)) | invoice math, status machines, authz scopes, crypto — pure or near-pure policy | HTTP shapes, React |
| Persistence (Prisma/Postgres) | referential integrity, uniqueness, isolation | application policy (no RLS here — all policy is app-level) |
| Background ([inngest/functions/](../../../inngest/functions/)) | eventual side-effects, retries | interactive authorization (trusts event payloads) |

## Boundaries done well (study these)

1. **Status policy extracted to pure functions.** [lib/invoices/permissions.ts](../../../lib/invoices/permissions.ts#L10-L48) knows nothing about HTTP, Prisma, or sessions — it takes `{status, createdBy}` + `{id, role}` and returns booleans. Result: exhaustively unit-tested ([permissions.test.ts](../../../__tests__/lib/invoices/permissions.test.ts)) and reusable from actions, routes, and MCP tools alike. This is the "functional core, imperative shell" idea in miniature.
2. **Authorization as data.** [accountReadScopeWhere](../../../lib/authz/scopes/crm.ts#L229-L237) returns a Prisma `where` fragment instead of performing a check. Policy composes into any query (`findMany`, `updateMany`, `count`) and executes *atomically with* the operation it guards.
3. **Error-type translation at the edge.** Domain code throws `AuthenticationError`/`AuthorizationError` ([authz/errors.ts](../../../lib/authz/errors.ts)); routes translate to 401/404 via [authz/route.ts](../../../lib/authz/route.ts) helpers; server actions translate to thrown `Error("Unauthorized")` ([create-invoice.ts:17-30](../../../actions/invoices/create-invoice.ts#L17-L30)). Each boundary speaks its own dialect and owns the translation.

## Boundary leaks (study these harder)

1. **Policy bypassed at a mutation boundary.** [update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49) goes session → Prisma directly, skipping the authz layer entirely. The layer exists; the action predates it and was never migrated. *Lesson: a layer you can bypass is a convention, not a boundary. Enforcement (lint rule, wrapper, or code review checklist) is part of the architecture.*
2. **The transaction invariant lives in the wrong layer.** [consumeNextNumber](../../../lib/invoices/numbering.ts#L13-L30) requires Serializable isolation to be correct, but the isolation level is chosen by its caller ([issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128)). The helper's contract ("give me a tx client") does not encode the real requirement ("give me a *Serializable* tx"). A second caller with a default-isolation transaction would compile fine and duplicate invoice numbers under load.
3. **Three auth dialects.** `getSession()` (raw), `requireAuthenticated()` (throws, returns role), `getUser()` (in [actions/get-user.ts](../../../actions/get-user.ts), returns the full user row). Newer code uses the second; older code the first; invoices the third. Every dialect a codebase keeps alive is a decision a reviewer must re-make on every PR.
4. **UI type boundary erased.** `data: any[]` into [AccountsView](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) — the server's carefully-inferred row type dies at the component boundary; column definitions downstream are unchecked.

## Ownership rule of thumb

Ask of any line: *"if this is wrong, which file should the fix land in?"* If the answer is "several," you've found either a missing abstraction or a leak. Try it: invoice rounding is wrong → one file ([totals.ts](../../../lib/invoices/totals.ts)). An account edit is possible by the wrong user → today, ~30 action files individually (leak); after adopting scoped writes, one file.

## Drill

Map the **campaigns** module against the table above: where does its policy live, where do its side effects fire, and does anything leak? (Start: [actions/campaigns/](../../../actions/campaigns/) vs [inngest/functions/campaigns/](../../../inngest/functions/campaigns/) vs [lib/campaigns/merge-tags.ts](../../../lib/campaigns/merge-tags.ts).)
Self-grade: Basic = drew the table. Solid = found that pause-checking happens in the worker (send-step.ts:32), i.e., policy in the background layer. Strong = argued whether that's a leak or correct placement (it's arguably correct: the pause invariant must hold at *send time*, and only the worker exists then).

## Interview angle

"Describe a layered architecture you've worked with and a place the layering broke down." You now have a specific, honest answer: scope layer as remediation, legacy actions that bypass it, and what enforcement would close the gap. That's a senior-signal answer built entirely from reading code.
