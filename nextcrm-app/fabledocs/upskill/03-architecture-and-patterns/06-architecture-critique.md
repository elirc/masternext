# Architecture Critique

An opinionated review, written as if handing the codebase to a new owner. Doubles as system-design interview material — cross-linked from [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices

1. **The authz scope layer** ([lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts)) — policy as composable query fragments, born from a real advisory. Centralized, auditable, atomic with the queries it guards. The phase comments (B1, D2, E3) show disciplined incremental rollout.
2. **Invoice issuance** ([issue-invoice.ts](../../../actions/invoices/issue-invoice.ts)) — network I/O outside the transaction, Serializable numbering, snapshots for legal immutability, tolerated PDF failure with regeneration path. The best 200 lines in the repo.
3. **Pure-function domain policy** ([lib/invoices/permissions.ts](../../../lib/invoices/permissions.ts), [totals.ts](../../../lib/invoices/totals.ts)) — testable without infrastructure, and actually tested.
4. **Credential hygiene by purpose** — hashed bearer tokens ([api-tokens.ts](../../../lib/api-tokens.ts)), AES-GCM for replayable provider keys ([email-crypto.ts](../../../lib/email-crypto.ts)), capability URLs for anonymous actions. Someone here knew the differences.
5. **Deliberate test seams** — better-auth `testUtils` OTP capture NODE_ENV-fenced in prod code ([lib/auth.ts:90-93](../../../lib/auth.ts#L90-L93)); `shouldSkipBulkEnrichment` exported for unit tests ([enrich-contact.ts:11-18](../../../inngest/functions/enrich-contact.ts#L11-L18)).

## Risks and tradeoffs (prioritized)

| # | Risk | Evidence | Impact / likelihood | Confidence |
| --- | --- | --- | --- | --- |
| 1 | **Object-level authz missing in legacy server actions** (update/delete account and siblings) | [update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49); [delete-account.ts:8-16](../../../actions/crm/accounts/delete-account.ts#L8-L16); repo's own [audit](../../../docs/2026-05-01-bola-idor-security-audit.md) | any authenticated user mutates any record / high | confirmed |
| 2 | **No test/typecheck CI** — only release-please | [.github/workflows/](../../../.github/workflows/) | regressions land silently / certain over time | confirmed |
| 3 | **Numbering safety is caller-owned** | [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) needs [Serializable at :128](../../../actions/invoices/issue-invoice.ts#L128); no retry on serialization abort | duplicate legal numbers or user-facing failures under concurrency / medium | confirmed mechanism; load-dependent |
| 4 | **Unescaped merge tags into campaign HTML** | [merge-tags.ts:17-23](../../../lib/campaigns/merge-tags.ts#L17-L23) | HTML injection into outbound mail (phishing vector) / low-medium | possible risk — depends on target-data provenance |
| 5 | **Legacy actions skip runtime validation** | [update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33) | garbage-in → 500s, injection of unexpected fields prevented only by explicit field list | confirmed |
| 6 | **State-changing GET unsubscribe** | [unsubscribe/route.ts](../../../app/api/campaigns/unsubscribe/route.ts#L4-L24) | scanner-triggered unsubscribes deflate campaigns / medium | confirmed shape; consequence inferred |
| 7 | **Enrichment retries re-spend** (no step memoization) + lost-update on concurrent edits | [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts#L109-L138) | cost + silent data overwrite / medium | confirmed |
| 8 | Non-timing-safe webhook compare; XFF-trusting rate-limit IP | [resend route:10](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L11); [rate-limit.ts:27-41](../../../lib/enrichment/rate-limit.ts#L27-L41) | narrow attacks / low | confirmed code, exploitability deployment-dependent |
| 9 | Naming/idiom strata (3 auth entry points, 3 createdBy spellings, partial safe-action adoption) | throughout | onboarding cost, review burden / certain | confirmed |
| 10 | PDF may render live account data instead of the snapshot | [issue-invoice.ts:133-141](../../../actions/invoices/issue-invoice.ts#L131-L141) | regenerated PDFs could differ from issued facts | **investigate** — trace regenerate-pdf.ts before claiming |

## What I'd change owning this for 3 months

**Month 1 — stop the bleeding.** Add CI (lint, `tsc --noEmit`, jest; then Playwright nightly). Sweep server actions for idiom-3 (session-only) mutations; migrate each to `requireAuthenticated` + scope check — mechanical, reviewable, one entity per PR. Test strategy: for each migrated action, a unit test that a non-owner `user` role gets an error (mock prisma or hit a test DB), mirroring [__tests__ patterns](../../../__tests__/).
**Month 2 — harden the boundaries.** Zod-parse every mutating action (`raw: unknown`); make `consumeNextNumber` own its lock (accept only a Serializable tx or take `FOR UPDATE` internally) with a retry-on-conflict wrapper; escape merge-tag values; POST-based one-click unsubscribe (keep GET as a confirm page). Each change is small-blast-radius with an existing pattern to follow.
**Month 3 — pay down strata.** Pick the winning auth idiom and lint-ban the others (`no-restricted-imports` on `getSession` outside lib/); document the decision as an ADR. Consider extracting the per-entity table components into one generic component only if a third divergence bug has actually occurred (don't refactor on aesthetics).

**Migration path discipline** for each: additive first, compatibility window, one entity per PR, test proving the new invariant, changelog entry via conventional commit.

## What I would *not* change

- No outbox: fire-and-forget embedding events lose only search freshness; the operational cost of a relay isn't justified yet. Revisit if events start carrying money.
- No RLS/multi-tenancy retrofit: the audit doc is explicit that there's no tenant column; bolting RLS onto ownership-OR semantics would be a rewrite. State it as a known ceiling.
- The Mongo-era naming: renames churn every query for zero behavior. Freeze the idiom for new code instead.

## Interview angle

This file *is* the answer to "walk me through a codebase you've evaluated: what was good, what was risky, what would you do first?" Practice delivering the three sections in 4 minutes: 3 strengths with anchors → top 3 risks with evidence → 90-day plan with sequencing logic (safety → boundaries → consistency). That sequencing logic — *why* CI before refactors, why authz before validation — is the senior signal.
