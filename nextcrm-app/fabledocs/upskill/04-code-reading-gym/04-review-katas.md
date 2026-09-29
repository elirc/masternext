# Review Katas

Nine fake PRs against this repo. For each: write your review (comments you'd actually post), classify findings **Blocking / Important / Optional**, then check the expected findings. All diffs are **illustrative fake code: not from this repo**.

Review in layers: does it work → is it correct → will it stay correct → does it fit the codebase → is it kind to future maintainers.

---

## Kata 1: "Add quick-edit endpoint for opportunities"

Author intent: PATCH endpoint like the contacts one, for opportunity fields.
Fake diff summary: new `app/api/crm/opportunities/[id]/route.ts` that calls `requireAuthenticated()` then `prismadb.crm_Opportunities.update({ where: { id }, data: body })`.
Files this resembles: [contacts/[id]/route.ts](../../../app/api/crm/contacts/%5Bid%5D/route.ts) — the template it *should* have copied.
Expected findings — **Blocking**: no object-level scope (reintroduces the exact advisory pattern; point at `tryScopedUpdateContact` and ask for an opportunity equivalent); raw `body` spread = mass assignment (no FIELD_MAP). **Important**: no `deletedAt` guard; missing test mirroring [contacts route tests](../../../app/api/crm/contacts/%5Bid%5D/__tests__/route.test.ts). **Optional**: return 404-style folding for consistency.
Good review comment example:
> This reopens the pattern from GHSA-mg5f-m89f-4gmc that the contacts route was fixed for — can we add a `tryScopedUpdateOpportunity` to lib/authz/scopes/crm.ts and route the write through it? Happy to pair on the scope OR-clause; `opportunityReadScopeWhere` (crm.ts:305) has the ownership branches to mirror.

## Kata 2: "Speed up issueInvoice by removing Serializable"

Author intent: "Serializable causes occasional retry errors under load; default isolation fixes them."
Fake diff summary: deletes `{ isolationLevel: "Serializable" }` at [issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128).
Expected findings — **Blocking**: duplicate invoice numbers under concurrency ([numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) is read-modify-write); the "errors" are the isolation *working*. Correct fix: retry-on-conflict wrapper, or row lock. **Important**: no load test exists to prove either claim — ask for one before any isolation change.

## Kata 3: "Show toast when account update fails"

Fake diff summary: client change: `const res = await updateAccount(data); if (res.error) toast.error(res.error);` plus removes `revalidatePath` from the action "since the client now handles feedback."
Expected findings — **Blocking**: removing `revalidatePath` ([update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62)) makes every accounts list stale after edit — feedback and cache invalidation are unrelated. **Optional**: the toast itself is fine; suggest keeping error strings generic (the action already returns sanitized messages).

## Kata 4: "Add companyLogo to invoice PDF"

Fake diff summary: adds `logoUrl` to `Invoice_Settings`, fetches it in the PDF template with `fetch(logoUrl)` at render time.
Expected findings — **Blocking**: server-side fetch of an admin-controlled URL = **SSRF** (internal metadata endpoints); require upload-to-MinIO instead, reusing [presigned-url](../../../app/api/upload/presigned-url/route.ts) flow. **Important**: PDF render happens post-transaction in issue + in [regenerate-pdf.ts](../../../actions/invoices/regenerate-pdf.ts) — both paths need the change; migration adds a nullable column (fine). **Optional**: cache the logo bytes.

## Kata 5: "Bulk delete contacts"

Fake diff summary: server action `deleteContacts(ids: string[])` → `updateMany({ where: { id: { in: ids } }, data: { deletedAt: new Date() } })` after `requireAuthenticated()`.
Expected findings — **Blocking**: no per-id authorization — must intersect with scope (`filterAuthorizedContactIds` exists for exactly this: [crm.ts:126-136](../../../lib/authz/scopes/crm.ts#L126-L136)) or put the scope in the updateMany where. **Important**: no audit log entries (single-delete path writes them); silent partial success needs a return contract (which ids were denied?). **Optional**: cap the batch size.

## Kata 6: "Retry enrichment on failure"

Fake diff summary: wraps the agent call in `for (let i = 0; i < 3; i++) try {...}` inside [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts).
Expected findings — **Important**: Inngest already retries the whole function (`retries: 3`, [:37](../../../inngest/functions/enrich-contact.ts#L32-L38)) — inner loop multiplies to 9 paid attempts; the real fix is `step.run` granularity so retries skip completed work. **Blocking** only if cost controls matter to the org — argue it. This kata teaches: *know the platform's retry semantics before adding your own.*

## Kata 7: "Unify auth helpers"

Fake diff summary: mechanical replace of every `getSession()` in actions/ with `requireAuthenticated()`, 40 files, no other changes.
Expected findings — **Important**: behavior change hiding in "mechanical": `requireAuthenticated` throws typed errors while callers `return { error }` on null session — every callsite's error path needs rework, or the wrapper catches ([session.ts:11-23](../../../lib/authz/session.ts#L11-L23) vs [update-account.ts:34-35](../../../actions/crm/accounts/update-account.ts#L34-L35)); also adds a DB query per call (fine, but say it). **Blocking**: 40 files in one PR is unreviewable — request per-module split. Teaches: large mechanical PRs hide the one non-mechanical line.

## Kata 8: "Add open-rate column to campaign list"

Fake diff summary: campaign list action adds, per campaign, `await prismadb.crm_campaign_sends.count({ where: { campaignId, opened_at: { not: null } } })` in a `.map`.
Expected findings — **Important**: N+1 (two counts × N campaigns); use a single `groupBy` on `campaignId`. **Optional**: opened_at semantics (only first open recorded — [resend route:51-58](../../../app/api/campaigns/webhooks/resend/route.ts#L51-L58) — so "opens" means "unique opens"; label the column honestly).

## Kata 9: "Escape merge tags" (a *good* PR to practice approving)

Fake diff summary: [merge-tags.ts](../../../lib/campaigns/merge-tags.ts) gains HTML-entity escaping of resolved values via the `entities` package (already a dependency).
Expected findings — **Approve with comments**: correct fix for the injection risk; **Important**: subject lines must NOT be HTML-escaped ([send-step.ts:44](../../../inngest/functions/campaigns/send-step.ts#L40-L51) resolves tags into the subject too — needs a `{ html: boolean }` mode); ask for a test with a `<script>`-bearing target name; check existing templates don't intentionally use HTML in fields. Teaches: reviewing a fix means checking *both* call sites and over-correction.

---

## Review language guide

State impact, cite the in-repo precedent, offer a path: *"X reopens/violates Y (link). Z in this repo does it safely — can we mirror that? Happy to pair."* Blocking = correctness/security/data loss. Important = will bite within a quarter. Optional = taste, marked as such.

Self-grade across katas: Basic = caught the headline issue in 6/9. Solid = correct B/I/O triage in 6/9. Strong = your comments named in-repo precedents (file paths) the author should copy — that's the reviewer who makes codebases converge instead of sprawl.
