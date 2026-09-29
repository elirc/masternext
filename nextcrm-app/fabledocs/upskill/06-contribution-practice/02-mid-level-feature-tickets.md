# Mid-Level Feature Tickets

12 cross-layer features (schema/API/UI/tests). Each **requires a design note before code** and includes risk + rollback. These are the tickets that produce mid-level interview stories.

Design-note template (write before coding): problem, chosen approach + one rejected alternative, schema change (if any) + migration + rollback, authz impact, test plan, blast radius.

---

## Ticket M1: Invoice reminders for overdue invoices
Layers: schema (reminder log) + Inngest scheduled fn + email template + settings.
Approach: a scheduled Inngest function scans ISSUED/PARTIALLY_PAID invoices past `dueDate`, sends a reminder via Resend (mirror [ScheduledReport](../../../emails/ScheduledReport.tsx) email pattern), records a reminder row to avoid re-sending (idempotency key = invoice+date).
Risk/rollback: emails are irreversible — gate behind a settings flag, dry-run mode first; rollback = disable flag. Reuse [send-scheduled report fn](../../../inngest/functions/reports/send-scheduled.ts) as structural template.
Design note must cover: dedup so a reminder isn't sent twice if the job retries (step.run + reminder-log row).
Interview story: "designed an idempotent scheduled email job."

## Ticket M2: Bulk assign contacts to a user
Layers: server action + authz + UI (table selection) + audit.
Approach: `assignContacts(ids, userId)` scoping writes via `filterAuthorizedContactIds` ([crm.ts:126-136](../../../lib/authz/scopes/crm.ts#L126-L136)); return which ids were denied. Audit each.
Risk: partial success semantics — the return contract is the hard part. Rollback: reassignment is reversible.
Interview story: "handled partial-success authorization in a bulk operation."

## Ticket M3: Invoice PDF uses the billing snapshot, not live account
Layers: lib/invoices/pdf + issue + regenerate.
Approach: make the PDF render from `billingSnapshot`/`taxRateSnapshot` ([issue-invoice.ts:60-89](../../../actions/invoices/issue-invoice.ts#L60-L89)) rather than `result.account.*` ([:133-141](../../../actions/invoices/issue-invoice.ts#L131-L141)) — **first confirm this is actually a bug** ([risk #10](../09-reference/risk-register.md)); the design note is half investigation.
Risk: changes what issued PDFs show — must match the legal snapshot; test both issue and regenerate paths. Rollback: revert; no data migration.
Interview story: "found that regenerated legal PDFs could drift from issued facts and fixed the data source."

## Ticket M4: Manager vs admin permission split
Layers: authz across scopes.
Approach: introduce one real capability difference (e.g., only admin manages invoice settings/currencies; manager manages CRM). Today they're near-identical ([crm.ts:230](../../../lib/authz/scopes/crm.ts#L229-L232)). Design note must enumerate every `role === "admin" || role === "manager"` site and decide each.
Risk: broad blast radius — this is the design-heavy one; ship behind tests per scope. Rollback: revert, but any data created under new perms stays.
Interview story: "designed a role capability model and migrated a codebase that conflated two roles."

## Ticket M5: Campaign send throttling / rate control
Layers: Inngest fan-out + config.
Approach: cap sends/hour per campaign to protect deliverability; use Inngest concurrency/throttle controls or a token-bucket in Redis (Upstash already present). Design note: where the limit is enforced (fan-out vs per-send) and how pause interacts.
Interview story: "added deliverability-protecting throttling to a cold-email engine."

## Ticket M6: Soft-delete restore UI + audit
Layers: UI (trash view) + existing restore actions + authz.
Approach: a "Deleted" tab listing tombstoned records (scope: who can restore?), wired to [restore-account.ts](../../../actions/crm/accounts/restore-account.ts) and siblings. Design note: who may see/restore deleted records (probably manager+).
Interview story: "shipped a recoverability feature end-to-end."

## Ticket M7: Webhook replay protection
Layers: schema (processed-event table) + webhook route.
Approach: store Resend event ids; ignore duplicates ([resend route](../../../app/api/campaigns/webhooks/resend/route.ts)). Makes handlers safe even if a future one isn't conditional. Rollback: drop table + guard.
Interview story: "added idempotency keys to a webhook pipeline."

## Ticket M8: Enrichment as memoized steps
Layers: Inngest.
Approach: rewrite [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts) with `step.run` around key-fetch, agent-call, and DB-write so retries don't re-pay the AI bill (see [pattern 7](../03-architecture-and-patterns/05-pattern-catalog.md)). Design note: which steps are memoizable, Date-serialization caveats.
Interview story: "cut AI spend by making a background job's retries idempotent."

## Ticket M9: Consistent Zod validation across legacy actions
Layers: actions + types.
Approach: introduce Zod schemas for the account/contact/lead/opportunity update actions (`raw: unknown` + parse), one entity per PR. Follows the invoice actions' model.
Interview story: "closed a class of unvalidated-input bugs across a service's write actions."

## Ticket M10: Multi-currency invoice display consistency
Layers: totals + UI + settings.
Approach: ensure base-currency conversion (`fxRateToBase`, [issue-invoice.ts:109-113](../../../actions/invoices/issue-invoice.ts#L109-L113)) is surfaced in list/report views consistently; add a "reporting currency" total. Design note: rounding rules for converted totals.
Interview story: "made multi-currency reporting consistent across an invoicing product."

## Ticket M11: Health check + structured logging
Layers: new route + logging util.
Approach: `/api/health` checking DB + MinIO; a `log(level, tag, meta)` helper emitting JSON with a request id, adopted first in the invoice + campaign paths. Ties to [observability gaps](../05-quality-engineering/06-observability-and-operations.md).
Interview story: "bootstrapped observability in a service that had only console.log."

## Ticket M12: Deny-and-audit on all scoped writes
Layers: authz + audit.
Approach: standardize that every scoped write logs denials and successes, closing the audit inconsistency between routes and actions (Tickets 10 + 17 generalized). Design note: PII-safe deny logging, volume concerns.
Interview story: "standardized authorization auditing across an application."

---

## Grading your design note (before writing code)

- **Weak**: jumps to implementation; no alternative considered; no rollback.
- **Solid**: states the approach, one rejected alternative with *why*, migration + rollback, test plan.
- **Strong**: also names the blast radius precisely (which files/users), sequences the change to stay reviewable (one entity/PR), and identifies the one place it could go wrong under concurrency or partial failure.
