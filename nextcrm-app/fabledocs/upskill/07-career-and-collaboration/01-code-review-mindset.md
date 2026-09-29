# Code Review Mindset

Review in five ascending layers. Most juniors stop at layer 1; mid-level lives at 2–3; senior owns 4–5.

1. **Does it work?** Runs, does the thing.
2. **Is it correct?** Edge cases, error paths, concurrency. (The invoice race, the lost-update window — layer-2 findings.)
3. **Will it stay correct?** Tests, invariants that can't silently break, no caller-owned traps.
4. **Does it fit?** Uses this repo's patterns (scope-where, safe-action envelope, pure-function policy) instead of inventing a parallel one.
5. **Is it kind to the next maintainer?** Named invariants, a comment where behavior is surprising, small reviewable diffs.

## Repo-specific review checklist

Paste this into your review notes for any PR here:

- [ ] **Authz**: does every mutation check object-level ownership (scope-where / `assertCan*`), not just a session? (The #1 recurring defect — [risk register](../09-reference/risk-register.md).)
- [ ] **Validation**: `raw: unknown` + Zod on new actions? Any `...spread` of client input into Prisma?
- [ ] **Money/status**: Decimal not float; status transitions go through [permissions.ts](../../../lib/invoices/permissions.ts)-style guards?
- [ ] **Transactions**: multi-write atomic (nested create or `$transaction`)? Any read-modify-write needing isolation?
- [ ] **Soft delete**: writes filter `deletedAt: null`?
- [ ] **Side effects**: unawaited promises carry `void` + `.catch`? Retryable jobs idempotent / step-memoized?
- [ ] **Revalidation**: mutations call `revalidatePath` with the right segment?
- [ ] **Contracts**: any changed string literal that's actually a public contract (token prefix, event name, tool name, URL)?
- [ ] **Tests**: does the PR add the test that would catch its own regression? Negative/denial case included?
- [ ] **Size**: is this reviewable, or should it be split per entity?

## Calibrating severity

- **Blocking**: correctness, security, data loss, contract break. (Missing authz, removed Serializable, renamed token prefix.)
- **Important**: will bite within a quarter. (N+1, missing test on a money path, inconsistent audit.)
- **Optional/nit**: taste, naming, micro-perf. *Mark it as optional* so the author knows they can ignore it.

## Example comments (from this repo's real seams)

Good (specific, cites precedent, offers a path):
> This `update({ where: { id } })` authorizes by session only — same shape as GHSA-mg5f-m89f-4gmc. `tryScopedUpdateContact` (lib/authz/scopes/crm.ts:33) shows the scoped-`updateMany` pattern; want to add an account equivalent? I can pair on the OR-clause.

Good (asks, doesn't assume):
> Is `taxRateId` guaranteed to exist here? create-invoice maps unknown ids to Decimal(0) (create-invoice.ts:49) — if the same is true on this path, a typo silently zeroes VAT. Worth a guard + test?

Unkind (avoid):
> This is wrong.

Better:
> This breaks under concurrent issuance — two callers can read the same counter (numbering.ts:13). The Serializable wrapper at the issue call site is what prevents it; this new path doesn't have it. Can we route through the same transaction?

## The reviewer's prime directive

A review should make the *codebase* converge, not just the diff. Every time you cite an in-repo precedent ("do it like X does"), you're pulling the code toward one idiom. That's how you fight the strata problem this repo already has (three auth helpers, three createdBy spellings).

## Interview angle

"How do you approach code review?" → the five layers + "make the codebase converge" + a real example of a kind, specific, precedent-citing comment. Most candidates describe layer 1–2 only; naming layers 4–5 is the mid/senior signal. See [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
