# Observability and Operations

The honest state: this repo is **log-and-pray** observability — structured enough to debug by grep, but no metrics, tracing, or alerting. Knowing what's missing is as interview-relevant as what's present.

## What exists

| Signal | Mechanism | Where |
| --- | --- | --- |
| Error logs | `console.error/log` with grep-able tags | `[ISSUE_INVOICE]` ([issue-invoice.ts:205](../../../actions/invoices/issue-invoice.ts#L204-L206)), `[AUDIT_LOG_WRITE_FAILED]` ([audit-log.ts:79](../../../lib/audit-log.ts#L78-L80)), `[UPDATE_ACCOUNT]` ([update-account.ts:65](../../../actions/crm/accounts/update-account.ts#L64-L66)) |
| Prisma logs | `error`/`warn` (dev), `error` (prod) | [lib/prisma.ts:17](../../../lib/prisma.ts#L17) |
| Domain status as data | enrichment status, campaign send status, invoice activity | enrichment rows, `crm_campaign_sends.status`, `Invoice_Activity` |
| Audit trail | field diffs per entity | [crm_AuditLog](../../../lib/audit-log.ts) — but writes are swallowed on failure (gaps invisible) |
| Job runs | Inngest's own dashboard | external (Inngest cloud/dev) |
| Health check | container healthchecks (Postgres/MinIO) | [docker-compose.yml](../../../docker-compose.yml) — infra only, no app `/health` route (**investigate**: `rg -ri "health" app/api`) |

## What's missing (and how you'd know it mattered)

- **No metrics** (request rate, error rate, latency, queue depth). You cannot answer "is enrichment backing up?" without opening Inngest.
- **No tracing** across the RSC → action → Inngest → external hops. A slow issuance can't be attributed to FX fetch vs PDF vs DB without manual timing.
- **No error aggregation** (Sentry/equivalent). A spike in `[ISSUE_INVOICE] PDF generation failed` is invisible until a user complains.
- **No alerting.** The swallowed audit-log failures ([audit-log.ts:78-81](../../../lib/audit-log.ts#L78-L81)) are the sharpest example: the trail can silently develop holes and nothing notices — a compliance risk hiding behind a `console.error`.

## "How would I know it broke?" per key flow

| Flow | Detection today | Gap |
| --- | --- | --- |
| Invoice issue | user sees error or missing PDF; log tag | no alert on PDF-failure rate |
| Campaign send | send-status rows go `failed`; per-send `error_message` | no aggregate "X% of a campaign bounced" alert |
| Enrichment | status row `FAILED` with reason; Inngest retries visible | no dashboard of failure reasons across runs |
| Auth | login just fails; OTP email may silently not send ([auth.ts:80-87](../../../lib/auth.ts#L80-L87) swallows in non-prod) | no metric on OTP send failures |
| Account mutation | log tag on catch | no object-level audit that a *denied* action was even attempted (there's no deny logging) |

## Deploy & rollback

- Deploy runs migrations (`pnpm build` → `prisma migrate deploy` — [package.json:11](../../../package.json#L11)). **Roll-forward only**: Prisma has no down-migrations here, so rollback = deploy a new forward migration. Say this explicitly in any schema PR.
- Backups: [scripts/db-backup.sh](../../../scripts/db-backup.sh) exists — **investigate** whether it's scheduled anywhere or manual.
- Blast radius of a bad deploy: single app + single DB, migrations coupled to build — a failing migration blocks the whole release (not just one service). The upside of a monolith is there's one thing to roll forward.

## The cheapest observability wins (if you owned it)

1. Structured logging (JSON with a `traceId` per request) — turns grep into query.
2. A counter on `writeAuditLog` failures + on PDF-generation failures — makes the two intentional-swallow paths *visible* without changing their behavior.
3. An app `/api/health` that checks DB + MinIO reachability — for load balancers.
4. Ship Inngest failure webhooks to a channel — free alerting for the async tier.

## Interview angle

- "How do you make a system observable?" → the three pillars (logs/metrics/traces), then diagnose this repo: strong domain-state-as-data, zero metrics/traces, swallowed-failure blind spots. Naming a specific blind spot (audit gaps) beats reciting the pillars. [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).
- "Design for rollback" → roll-forward-only migrations here; contrast with expand/contract migration discipline.
