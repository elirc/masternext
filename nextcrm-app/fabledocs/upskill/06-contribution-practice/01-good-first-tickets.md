# Good First Tickets

18 junior tickets spread across invoices, authz, campaigns, enrichment, uploads, and tests. Each is small, follows an existing pattern, and is testable. Full template used for the first three; the rest are compressed but carry the same fields.

---

## Ticket 1: Reject unknown taxRateId in createInvoice
Difficulty: Easy — ~1h. Skills: validation, Zod, invoice domain.
Story: As a user, I want an error when I reference a non-existent tax rate, so I don't silently create a 0% VAT invoice.
Why good: closes a real validation hole ([create-invoice.ts:49-51](../../../actions/invoices/create-invoice.ts#L49-L51) maps unknown ids to Decimal(0)); tiny blast radius; mirrors existing error style.
Acceptance: [ ] unknown `taxRateId` → thrown/returned error, not silent 0%. [ ] unit test. [ ] known ids unaffected.
Read first: [create-invoice.ts:32-53](../../../actions/invoices/create-invoice.ts#L32-L53), [totals.ts](../../../lib/invoices/totals.ts).
Files: create-invoice.ts (compare fetched `rateMap` keys against requested ids, error on missing).
Plan: after building `rateMap`, assert every non-null requested `taxRateId` is present; else throw `Error("Unknown tax rate")`.
What could go wrong: null `taxRateId` is legitimately "no tax" — don't reject those.
Review Qs: does it break existing invoices with null rates? Is the message user-safe?
Interview story potential: "I found and fixed a silent-default bug that could under-charge VAT."

## Ticket 2: Add deletedAt guard to updateAccount
Difficulty: Easy — ~1h. Skills: soft delete, Prisma where.
Story: As an admin, I want edits to soft-deleted accounts rejected, so tombstoned data stays frozen.
Why good: [update-account.ts:42-48](../../../actions/crm/accounts/update-account.ts#L42-L48) updates by id without `deletedAt: null` — the `before` read already filters it ([:41](../../../actions/crm/accounts/update-account.ts#L41)) but the write doesn't; consistent, low-risk.
Acceptance: [ ] update of a soft-deleted account → no-op/error. [ ] test. [ ] normal update unaffected.
Read first: [delete-account.ts](../../../actions/crm/accounts/delete-account.ts), [soft-delete-gaps.md](../../../docs/soft-delete-gaps.md).
Plan: change `where: { id }` to `updateMany({ where: { id, deletedAt: null }, ... })` + count check, or guard on the `before` value.
Interview story potential: "I tightened a soft-delete invariant that the write path was ignoring."
Note: do NOT also fix the missing authz here — that's Ticket 12, a separate PR. Small PRs.

## Ticket 3: Snapshot test for accountReadScopeWhere
Difficulty: Easy — ~1h. Skills: unit testing, authz.
Story: As a maintainer, I want the scope-where shapes pinned, so an accidental widening of user access fails CI.
Why good: fills the highest-value test gap (authz correctness is untested); pure function, no mocks.
Acceptance: [ ] tests for admin/manager/user roles asserting exact `where`. [ ] `pnpm test -- scopes` green.
Read first: [recipe 4](../05-quality-engineering/02-writing-tests-here.md), [crm.ts:229-237](../../../lib/authz/scopes/crm.ts#L229-L237).
Interview story potential: "I added regression tests around authorization scopes that had zero coverage."

## Ticket 4: timingSafeEqual for the Resend webhook
Easy — ~1h. Swap `signature === expected` at [resend route:10](../../../app/api/campaigns/webhooks/resend/route.ts#L5-L11) for `crypto.timingSafeEqual` (with length guard). Follows the crypto style already in the repo. Test: valid/invalid signature. Story potential: "closed a timing side-channel on a webhook verifier."

## Ticket 5: Escape merge-tag values
Easy-Medium — ~2h. HTML-escape resolved values in [merge-tags.ts:17-23](../../../lib/campaigns/merge-tags.ts#L17-L23) using the already-present `entities` dep; add a `{ escape }` option so subject lines ([send-step.ts:44](../../../inngest/functions/campaigns/send-step.ts#L44)) opt out. Test with a `<script>` name. (This is [review kata 9](../04-code-reading-gym/04-review-katas.md) as a real PR.) Story potential: "fixed a stored-HTML injection into outbound email."

## Ticket 6: Boot-time env validation for EMAIL_ENCRYPTION_KEY
Easy — ~1h. Add a startup check (or a `lib/env.ts` assertion) so a missing/short key fails fast instead of on first crypto call ([email-crypto.ts:7-15](../../../lib/email-crypto.ts#L7-L15)). Story potential: "moved a config failure from runtime to boot."

## Ticket 7: Add take/pagination cap to getAccounts
Easy-Medium — ~2h. [get-accounts.ts](../../../actions/crm/get-accounts.ts) has no `take`; add a sane cap or cursor to bound the payload. Coordinate with the client table's pagination. Story potential: "bounded an unbounded query before it became a production incident."

## Ticket 8: groupBy for campaign open counts
Easy-Medium — ~2h. Implement the campaign-list open-rate column with one `groupBy` instead of per-row counts (prevents the N+1 in [review kata 8](../04-code-reading-gym/04-review-katas.md)). Story potential: "shipped a metric column without introducing an N+1."

## Ticket 9: Magic-byte check on uploads
Medium — ~3h. Beyond the declared content-type allowlist ([presigned-url:29-37](../../../app/api/upload/presigned-url/route.ts#L29-L37)), validate actual file magic bytes post-upload (or restrict presign by extension+type pairing). Story potential: "hardened file uploads against content-type spoofing."

## Ticket 10: Consistent audit log on the contacts PATCH route
Easy-Medium — ~2h. The API PATCH ([contacts route](../../../app/api/crm/contacts/%5Bid%5D/route.ts)) writes no audit entry, unlike the server-action path ([update-account.ts:54-60](../../../actions/crm/accounts/update-account.ts#L54-L60)). Add `writeAuditLog` for parity. Story potential: "unified the audit trail across two write paths."

## Ticket 11: HTTP timeout on enrichment fetches
Medium — ~3h. Add an `AbortController` timeout to outbound Firecrawl/OpenAI calls in the enrichment strategy so a hung scrape doesn't hold an Inngest slot. Story potential: "added timeouts that stopped background workers from stalling."

## Ticket 12: Scope check on deleteAccount
Medium — ~2h. Route `deleteAccount` ([delete-account.ts:13-16](../../../actions/crm/accounts/delete-account.ts#L13-L16)) through `assertCanWriteAccount` (exists: [crm.ts:251-267](../../../lib/authz/scopes/crm.ts#L251-L267)) or a scoped `updateMany`. **This touches the advisory class — coordinate with maintainers, one entity per PR.** Story potential: "closed an IDOR on account deletion, mirroring the documented remediation pattern."

## Ticket 13: Unit-test the Resend webhook status transitions
Easy-Medium — ~2h. Cover delivered/bounced/opened/clicked and the `status !== "sent"` skip path ([resend route:34-68](../../../app/api/campaigns/webhooks/resend/route.ts#L34-L68)) — also documents the reconciliation gap from [debugging scenario 4](../05-quality-engineering/03-systematic-debugging.md). Story potential: "added tests that surfaced a webhook ordering race."

## Ticket 14: Fill one skipped E2E invoice spec
Easy-Medium — ~3h. Implement the "creates a new draft invoice" TODO in [invoices.spec.ts](../../../tests/e2e/invoices.spec.ts) using the auth setup + data-testids already in the UI. Story potential: "converted a skeleton E2E into real coverage for the invoice flow."

## Ticket 15: currency formatting edge cases
Easy — ~1h. Extend [__tests__/lib/currency.test.ts](../../../__tests__/lib/currency.test.ts) for zero, negative, and locale-specific separators against [lib/currency-format.ts](../../../lib/currency-format.ts). Story potential: "hardened money formatting for negatives and locales."

## Ticket 16: Replace `(prismadb as any)` in audit-log
Easy — ~1h. Investigate why the cast exists at [audit-log.ts:69](../../../lib/audit-log.ts#L68-L70) (likely stale generated client); regenerate Prisma and remove the `any` if the model is present. Story potential: "removed an `any` cast by fixing the underlying type-generation gap."

## Ticket 17: Deny-attempt logging on scoped writes
Easy-Medium — ~2h. When `tryScopedUpdateContact`/`Target` returns count 0 ([crm.ts:42](../../../lib/authz/scopes/crm.ts#L33-L55)), log a structured deny event (no PII) so authz failures are observable (ties to [observability gaps](../05-quality-engineering/06-observability-and-operations.md)). Story potential: "added visibility into denied authorization attempts."

## Ticket 18: Document the three auth entry points
Easy — ~1h (docs, but in-repo). Add a short `lib/authz/README.md` explaining getSession vs requireAuthenticated vs getUser and when to use which, pointing at the winner. Reduces the strata cost. Story potential: "wrote the onboarding doc that stopped new code from picking the wrong auth helper."

---

## How to pick

Fresh to the repo → 3, 15, 16 (pure, no product risk). Want a security story → 4, 5, 12. Want a full-stack touch → 14, 8. Every one of these is a real improvement a maintainer could accept; none is a drive-by refactor.
