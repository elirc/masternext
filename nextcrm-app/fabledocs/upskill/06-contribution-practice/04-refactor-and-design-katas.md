# Refactor and Design Katas

Senior-judgment exercises. Each has a prompt and self-grading criteria — do them on paper/branch, then grade.

## Kata A: Fix a boundary leak (numbering invariant)
Prompt: `consumeNextNumber` requires Serializable but its signature doesn't encode that ([numbering.ts:11-30](../../../lib/invoices/numbering.ts#L11-L30)). Refactor so the invariant is enforced by the function, not its callers.
Self-grade: **Solid** = moved to `SELECT ... FOR UPDATE` on the series row, making any isolation safe, with a test. **Strong** = compared FOR-UPDATE vs Serializable-with-retry on throughput/complexity, and documented why the chosen one; noted the existing caller must be re-tested.

## Kata B: Introduce an outbox (eventing change)
Prompt: replace one `void inngest.send(...)` ([update-account.ts:61](../../../actions/crm/accounts/update-account.ts#L61)) with a transactional outbox for the `crm/account.saved` event.
Self-grade: **Solid** = event row written in the same tx as the update; a relay emits + marks sent; consumer idempotent. **Strong** = crash-injection test proving delivery survives a process death between commit and relay; discussed at-least-once + dedup and the new failure mode (relay lag) you accepted.

## Kata C: Split a module (auth idioms)
Prompt: the repo has three auth entry points (getSession/requireAuthenticated/getUser). Design the consolidation.
Self-grade: **Solid** = chose one, wrote a migration plan (one module per PR), added a lint ban on the others. **Strong** = handled the behavioral difference (throw vs return-null) at each callsite, sequenced so no PR mixes mechanical + semantic change, and wrote the ADR.

## Kata D: Remove duplication (table components) — or refuse it
Prompt: decide whether to extract a generic CRM table ([Project 6](03-senior-build-projects.md)).
Self-grade: **Solid** = made a *decision* with a criterion (extract iff a divergence bug occurred), not a reflex. **Strong** = if extracting, scoped it behind E2E coverage and migrated two entities as proof; if refusing, quantified the regression blast radius vs the DRY payoff. Either answer can be Strong — the reasoning is graded, not the verdict.

## Kata E: Improve type safety at a boundary
Prompt: replace `data: any[]` on [AccountsView](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) with the real inferred row type from the server action.
Self-grade: **Solid** = derived the type via `Awaited<ReturnType<typeof getAccounts>>` and threaded it to the column defs. **Strong** = found what breaks (column accessors that were silently wrong), fixed them, and argued whether `serializeDecimals`'s dishonest `<T>(x:T):T` signature ([serialize-decimals.ts:5](../../../lib/serialize-decimals.ts#L5)) undermines the effort.

## Kata F: Design a migration (createdBy unification)
Prompt: unify `createdBy`/`created_by`/`created_by_user` on the Documents model.
Self-grade: **Solid** = expand/contract plan (add canonical col, backfill, dual-write window, update scopes, drop old). **Strong** = the backfill SQL, every consumer found via grep (including [documentReadScopeWhere:402-419](../../../lib/authz/scopes/crm.ts#L402-L419)), and the roll-forward-only reality (no down migration here) stated in the plan.

## Kata G: Reduce N+1
Prompt: an activities feed shows, per activity, the count of linked entities via a per-row query.
Self-grade: **Solid** = single `groupBy`/aggregate replacing the loop. **Strong** = measured with `log:["query"]` before/after, and added a test asserting query count doesn't scale with rows.

## Kata H: Write an RFC
Prompt: propose the manager/admin permission split ([M4](02-mid-level-feature-tickets.md)) as an RFC.
Self-grade: use the template in [07/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). **Strong** = enumerates every `role === "admin" || role === "manager"` site as an appendix, proposes a capability model (not per-site ifs), and has a rollout that never leaves the system in a half-migrated insecure state.

## Kata I: Review a flawed PR
Prompt: take [review kata 2](../04-code-reading-gym/04-review-katas.md) ("remove Serializable for speed") and write the full review + a counter-proposal PR description.
Self-grade: **Strong** = your review blocks with the concrete failure (duplicate numbers), explains the aborts are the feature, and your counter-proposal is Kata A — turning a bad PR into the right one.

---

## Meta-grade

Across all katas, the senior signal is the same: you named the invariant, chose an approach *with* a rejected alternative, sequenced the change to stay safe and reviewable, and specified the test that pins the new behavior. If your answer has all four, it's Strong regardless of the specific verdict.
