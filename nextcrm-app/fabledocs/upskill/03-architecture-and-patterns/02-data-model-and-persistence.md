# Data Model and Persistence

## Shape of the schema

[prisma/schema.prisma](../../../prisma/schema.prisma) — 1909 lines, ~80 models. Three generations visible in the naming:

1. **Mongo-era models**: `crm_Accounts`, `crm_Contacts`, snake_case fields, stray `v` version field (stamped `v: 0` at [update-account.ts:45](../../../actions/crm/accounts/update-account.ts#L45)), duplicated ownership spellings (`createdBy`/`created_by`/`created_by_user`).
2. **Explicit junction tables**: `DocumentsToAccounts`, `ContactsToOpportunities`, `AccountWatchers`, `TargetsToTargetLists` (schema L1098-L1305) — many-to-many made first-class so rows can carry metadata and scopes can traverse them.
3. **Modern modules**: `Invoices`/`Invoice_*` (L1720-1899), `crm_campaigns`/`crm_campaign_sends` (L287-392), `ApiToken` (L1382), `crm_Embeddings_*` with pgvector (L1305-1365) — camelCase, enums, snapshot columns.

Reading the [migrations/](../../../prisma/migrations/) directory names in order is the project's real changelog: pgvector → api tokens → campaigns → soft delete → audit log → better-auth → currency.

## Key entities and relationships

```
crm_Accounts ─┬─< crm_Contacts (assigned_accounts)
              ├─< crm_Opportunities >── sales stages/types (lookup tables)
              ├─< crm_Contracts
              ├─< Invoices ──< Invoice_LineItems >── Invoice_TaxRates
              │        └──< Invoice_Payments / Invoice_Activity
              ├─< AccountWatchers >── Users
              └─< DocumentsToAccounts >── Documents
crm_Targets >── TargetsToTargetLists ──< crm_TargetLists ── campaigns
Invoices ── Invoice_Series (numbering counter)   Users ──< ApiToken / ApiKeys
```

## Consistency choices worth teaching

- **Money is Decimal, serialized as string.** Totals computed in `decimal.js` with per-line 2dp rounding ([totals.ts:18-25](../../../lib/invoices/totals.ts#L18-L25)) then written as strings ([create-invoice.ts:72-76](../../../actions/invoices/create-invoice.ts#L72-L76)). Rounding **per line, then summing** is the legally-expected behavior for VAT (line totals must match what's printed); summing-then-rounding gives different results by a cent. That cent is an interview story.
- **Snapshots denormalize on purpose.** `billingSnapshot` and `taxRateSnapshot` ([issue-invoice.ts:60-89](../../../actions/invoices/issue-invoice.ts#L60-L89)) freeze master data into the invoice at issue time. Invariant: *an issued invoice renders identically forever*, regardless of later edits to accounts or tax rates. Normalization is for things that should change together; legal documents shouldn't.
- **Counters as rows, not sequences.** `Invoice_Series.counter` + yearly reset ([numbering.ts:18-28](../../../lib/invoices/numbering.ts#L18-L28)) instead of a Postgres sequence — because sequences leave gaps on rollback and can't do per-series templates/resets. Cost: needs Serializable (or row locking) to be safe.
- **Soft delete everywhere.** `deletedAt`/`deletedBy` per [docs/soft-delete-gaps.md](../../../docs/soft-delete-gaps.md); every read scope starts `{ deletedAt: null }` ([crm.ts:229-237](../../../lib/authz/scopes/crm.ts#L229-L237)). Failure mode to check in review: **writes** that forget the filter — [update-account.ts:42-48](../../../actions/crm/accounts/update-account.ts#L42-L48) will happily edit a soft-deleted row (confirmed).
- **Transaction boundaries.** Exactly one place uses explicit isolation: invoice issuance ([issue-invoice.ts:56-129](../../../actions/invoices/issue-invoice.ts#L56-L129)). Multi-write actions elsewhere (e.g., create invoice + lines + activity) get atomicity via **nested creates** in one Prisma call ([create-invoice.ts:55-98](../../../actions/invoices/create-invoice.ts#L55-L98)) — a single statement is atomic without a $transaction. Recognizing "nested create = free transaction" is a good Prisma-specific interview point.

## How to change the schema safely here

1. Edit `schema.prisma`; run `pnpm exec prisma migrate dev --name your_change` (inferred) — never edit applied migrations.
2. Additive first (new nullable column → backfill → tighten). `pnpm build` runs `migrate deploy`, so a migration that locks a hot table for minutes blocks the deploy: prefer `NOT VALID`-style phased constraints for big tables (raw SQL migration).
3. Check every **scope builder** in [lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts) that touches the model — scopes encode schema assumptions (ownership columns, junction names) that the compiler only partially checks.
4. Check `serializeDecimals` call sites if you added a Decimal column, and seeds ([prisma/seeds/](../../../prisma/seeds/)).
5. Rollback story: there is none scripted (no down migrations in Prisma). Your plan is roll-forward; say so in the PR.

## Drills

1. Find the unique constraint that makes `validateApiToken`'s `findUnique({ where: { tokenHash } })` possible ([api-tokens.ts:51-53](../../../lib/api-tokens.ts#L51-L53)) — locate `tokenHash` in the schema and confirm `@unique`. What attack does a *non*-unique hash column enable? (None directly — but lookup becomes findFirst and timing/behavior changes; uniqueness here is also collision insurance.)
2. Sketch the migration plan to unify `createdBy`/`created_by` spellings on one model. Steps, backfill SQL, scope-builder edits, and what you'd grep to find every consumer. Self-grade: Strong = includes a compatibility window where both columns are written.

## Interview angle

- "How do you model money?" → Decimal + string serialization + per-line rounding, with the VAT-cent story. [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q5.
- "Soft delete vs hard delete?" → this schema, plus the write-path gap as the failure mode. Q6.
- "When do you denormalize?" → billing snapshots. Q7.
