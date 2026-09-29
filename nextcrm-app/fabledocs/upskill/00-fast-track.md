# Fast Track — One Weekend

Goal: by Sunday night you can run NextCRM locally, trace two end-to-end flows aloud, and have made one safe change with a passing test.

## 1. Install and run (Saturday morning)

All commands are **inferred** from [package.json](../../package.json#L9-L21), [docker-compose.yml](../../docker-compose.yml), and [.env.example](../../.env.example) — they were not executed while writing these docs. Node ≥ 22.12 and pnpm ≥ 9 are required ([package.json](../../package.json#L5-L8)).

```bash
# Infra: Postgres (pgvector image) + MinIO
docker compose up -d postgres minio        # inferred

cp .env.example .env                        # then fill DATABASE_URL, BETTER_AUTH_SECRET,
                                            # EMAIL_ENCRYPTION_KEY (openssl rand -hex 32), etc.
pnpm install                                # inferred
pnpm exec prisma migrate deploy             # inferred — migrations in prisma/migrations/
pnpm exec prisma db seed                    # inferred — seed entry in package.json:22-24
pnpm dev                                    # inferred — Next.js dev server on :3000
```

Sign-in: email+password is **disabled** ([lib/auth.ts](../../lib/auth.ts#L65-L67)); auth is Google OAuth or email OTP. The **first user to register becomes admin** automatically ([lib/auth.ts](../../lib/auth.ts#L109-L124)) — that's how you bootstrap a local admin.

Tests:

```bash
pnpm test          # inferred — Jest unit tests in __tests__/ and colocated __tests__/ dirs
pnpm lint          # inferred — eslint . --max-warnings=0
pnpm test:e2e      # inferred — Playwright; needs a running seeded app (playwright.config.ts webServer)
```

## 2. First 10 files to open, in order

| # | File | Why |
| --- | --- | --- |
| 1 | [package.json](../../package.json#L9-L21) | Scripts = the repo's verbs. Note `build` runs `prisma migrate deploy`. |
| 2 | [proxy.ts](../../proxy.ts#L18-L55) | The middleware: cookie presence check + next-intl routing. Auth *decisions* are NOT made here. |
| 3 | [lib/auth.ts](../../lib/auth.ts#L12-L126) | better-auth config: OTP plugin, admin roles, first-user-admin callback. |
| 4 | [lib/authz/session.ts](../../lib/authz/session.ts#L11-L33) | `requireAuthenticated` / `requireRole` — the server-side auth gate everything should use. |
| 5 | [lib/authz/scopes/crm.ts](../../lib/authz/scopes/crm.ts#L229-L237) | `accountReadScopeWhere` — authorization as a Prisma `where` fragment. The repo's crown jewel. |
| 6 | [app/[locale]/(routes)/crm/accounts/page.tsx](<../../app/[locale]/(routes)/crm/accounts/page.tsx>) | A server component page: fetch via server action, render client view. |
| 7 | [actions/crm/get-accounts.ts](../../actions/crm/get-accounts.ts#L5-L40) | Scoped read: `requireAuthenticated()` + scope where + React `cache()`. |
| 8 | [actions/crm/accounts/update-account.ts](../../actions/crm/accounts/update-account.ts#L34-L49) | Scoped? Look closely. Session check only — compare with #5. This gap is real and documented in [docs/2026-05-01-bola-idor-security-audit.md](../../docs/2026-05-01-bola-idor-security-audit.md). |
| 9 | [actions/invoices/issue-invoice.ts](../../actions/invoices/issue-invoice.ts#L56-L129) | Serializable transaction, invoice numbering, billing snapshot. The best server-side code in the repo. |
| 10 | [inngest/functions/enrich-contact.ts](../../inngest/functions/enrich-contact.ts#L32-L147) | A background job: dedup window, AI enrichment, apply-to-empty-fields merge. |

## 3. Two flows to trace aloud

**Flow A — Accounts list to edit.** Browser hits `/en/crm/accounts` → [proxy.ts](../../proxy.ts#L42-L51) checks the session cookie exists → server component [page.tsx](<../../app/[locale]/(routes)/crm/accounts/page.tsx>) awaits `getAccounts()` → [get-accounts.ts](../../actions/crm/get-accounts.ts#L5-L8) calls `requireAuthenticated()` then queries with `accountReadScopeWhere(user)` (admins/managers see everything not soft-deleted; users see assigned/created/watched — [crm.ts](../../lib/authz/scopes/crm.ts#L216-L237)) → rows render into `AccountsView` (client component) → an edit calls the `updateAccount` server action → [update-account.ts](../../actions/crm/accounts/update-account.ts#L40-L62) writes, records an audit diff, fires an Inngest event, and calls `revalidatePath`.

**Flow B — Issue an invoice.** UI calls `issueInvoice` server action → [issue-invoice.ts](../../actions/invoices/issue-invoice.ts#L17-L54) authenticates, Zod-parses, checks `canIssueInvoice` (pure function: [permissions.ts](../../lib/invoices/permissions.ts#L31-L34)), fetches FX rate *outside* the transaction → a `$transaction(..., { isolationLevel: "Serializable" })` consumes the next invoice number ([numbering.ts](../../lib/invoices/numbering.ts#L13-L30)), snapshots billing data and tax rates, flips status to ISSUED → after commit, PDF generation runs in a try/catch that deliberately never fails the issuance ([issue-invoice.ts](../../actions/invoices/issue-invoice.ts#L131-L207)).

Teach-back exercise: explain Flow B to a rubber duck as if an interviewer asked *"walk me through a transaction you'd call well-designed."* You should hit: why FX fetch is outside the transaction, why Serializable, what "snapshot" protects, and why PDF failure is tolerated.

## 4. One small safe change

Add a unit test for `formatNumber` covering a template with **both** `{YYYY}` twice, e.g. `"{YYYY}-{YYYY}-{##}"`, in [__tests__/lib/invoices/numbering.test.ts](../../__tests__/lib/invoices/numbering.test.ts). Pure function, zero blast radius, exercises the regex-global-replace behavior at [numbering.ts](../../lib/invoices/numbering.ts#L3-L9). Run `pnpm test -- numbering` (inferred).

## 5. What the fast path skips

Campaigns and the Resend webhook, the MCP server, email IMAP sync, embeddings/semantic search, the Mongo→Postgres migration scripts, i18n plumbing, and all of the projects/boards module. They're covered in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md) and later modules.
