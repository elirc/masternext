# Implementation Plans: Stories 18-25

These stories are senior because they force you to reason about operational constraints, shared abstractions, data lifecycle, and engineering quality. They should be implemented in small commits on `dev`, with focused tests after each boundary.

## Story 18: Enrichment Budget And Rate-Limit Dashboard

**Primary code paths:**
- `lib/enrichment/rate-limit.ts`
- `lib/enrichment/config/enrichment.ts`
- `app/api/crm/contacts/enrich/route.ts`
- `app/api/crm/targets/enrich/route.ts`
- `lib/enrichment/services/openai.ts`
- `__tests__/enrichment`

**Implementation plan:**
1. Identify stored enrichment status models for contacts and targets.
2. Build an aggregation helper for counts by entity type, status, and time window.
3. Read rate-limit state from existing rate-limit helpers without exposing secrets.
4. Add an admin-only dashboard panel under reports, settings, or CRM admin depending on current navigation.
5. Distinguish pending, running, completed, failed, and skipped states.
6. Add tests for aggregation and permission denial.
7. Document operational interpretation: what failed vs skipped means.

**Tests and checks:**
- Enrichment aggregation unit tests.
- Authz tests for admin/manager visibility if the panel is restricted.

**Senior checkpoint:** Operational dashboards should explain system state without leaking provider keys, prompts, or private enrichment payloads.

## Story 19: Saved CRM Table Views

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx`
- `app/[locale]/(routes)/crm/accounts/table-components` if present
- `app/[locale]/(routes)/crm/opportunities/table-components`
- `app/[locale]/(routes)/crm/contacts/table-components/data-table-toolbar.tsx`
- `prisma/schema.prisma`

**Implementation plan:**
1. Inspect the TanStack table state currently owned in each CRM table.
2. Design a generic saved-view payload for filters, sorting, column visibility, and target table key.
3. Add a Prisma model scoped to user id and table key.
4. Validate saved-view payloads with a schema before persistence and application.
5. Start with one table, preferably contacts, then extract only if a second table uses the same code.
6. Add toolbar controls for save current view, choose view, reset, rename, and delete.
7. Ensure invalid or stale column ids are ignored rather than crashing.
8. Add action tests for user scoping and payload validation.

**Tests and checks:**
- Unit tests for payload parser.
- Action tests for user-private saved views.
- Manual table state check with filters, sorting, and hidden columns.

**Senior checkpoint:** Do not over-abstract before the second table proves the shape. Senior code is reusable because the boundary is real, not because it is premature.

## Story 20: Multi-Entity Timeline

**Primary code paths:**
- `actions/crm/activities/get-activities-by-entity.ts`
- `actions/crm/audit-log/get-audit-log-by-entity.ts`
- `app/[locale]/(routes)/crm/contacts/[contactId]/components/HistoryTab.tsx`
- `app/[locale]/(routes)/crm/accounts/[accountId]` if present
- `lib/authz/scopes/crm.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Define normalized event types: activity, audit, note, enrichment, invoice, campaign, document.
2. Start with one entity page, likely contact detail.
3. Apply read-scope authorization before querying source tables.
4. Query each source with limits and project only fields needed for the timeline.
5. Normalize source rows into a common `TimelineEvent` type.
6. Sort by event time with stable tie-breakers.
7. Render source-specific icons, labels, actor, timestamp, and summary.
8. Add tests for normalization, ordering, and permission denial.

**Tests and checks:**
- Pure normalization tests.
- Action tests for scoped timeline access.

**Senior checkpoint:** A good timeline is an anti-corruption layer. The UI should not know every source table's quirks.

## Story 21: Soft-Delete Restore Center

**Primary code paths:**
- `actions/crm/contacts/restore-contact.ts`
- `actions/crm/accounts/restore-account.ts`
- `actions/crm/contracts/restore-contract/index.ts`
- `actions/crm/*/delete-*`
- `docs/soft-delete-gaps.md`
- `lib/audit-log.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Read `docs/soft-delete-gaps.md` and list which models support `deletedAt`.
2. Inventory restore actions that already exist.
3. Create a restore-center read action that returns only supported soft-deleted records.
4. Apply admin or entity-specific restore permissions before listing and restoring.
5. Add UI grouped by entity type with deleted time, deleted by, and record name.
6. Call existing restore actions where possible instead of duplicating logic.
7. Write audit entries for restore events if existing restore actions do not already do it.
8. Add tests for supported entities, unsupported entities, and permission denial.

**Tests and checks:**
- Restore action tests.
- Restore center read tests.

**Senior checkpoint:** Recovery tooling is powerful. Keep supported scope explicit and do not imply hard-deleted records can be restored.

## Story 22: Decimal Serialization Hardening Sweep

**Primary code paths:**
- `lib/serialize-decimals.ts`
- `actions/invoices`
- `actions/crm/get-opportunities*`
- `actions/crm/contracts`
- `actions/crm/products`
- `app/[locale]/(routes)/invoices`
- `prisma/schema.prisma`

**Implementation plan:**
1. Search Prisma schema for `Decimal` fields.
2. Search server actions and Server Components returning models with those fields.
3. Classify each path: server-only, server-to-client props, server action return, API JSON response.
4. Apply `serializeDecimals()` or `serializeDecimalsList()` at the boundary where Decimal-bearing data leaves the server-only layer.
5. Do not narrow returned records to `{ id }` to dodge serialization.
6. Add a regression test for one high-risk action, preferably invoice or opportunity detail.
7. Document the audited paths and remaining low-risk paths in this story's PR body.

**Tests and checks:**
- Targeted tests for the updated action.
- Lint/typecheck.
- Manual UI check for invoice or opportunity page using money fields.

**Senior checkpoint:** Cross-cutting sweeps should be systematic and documented. The hard part is controlling blast radius.

## Story 23: Audit Log Diff Viewer

**Primary code paths:**
- `actions/crm/audit-log/get-audit-log-by-entity.ts`
- `actions/crm/audit-log/get-audit-log-admin.ts`
- `lib/audit-log.ts`
- `app/[locale]/(routes)/crm/*/[id]/components/HistoryTab.tsx`
- `prisma/schema.prisma`

**Implementation plan:**
1. Inspect current audit metadata shape and historical records.
2. Create a formatter that accepts old and new audit metadata shapes.
3. Redact sensitive keys such as tokens, passwords, secrets, API keys, and email credentials.
4. Format dates, booleans, nulls, arrays, and Decimal-like values consistently.
5. Update history tab UI to show field-level before/after diffs.
6. Preserve a fallback raw summary for records that cannot be diffed safely.
7. Add tests for formatter behavior and redaction.

**Tests and checks:**
- Pure formatter tests.
- Existing audit-log scope tests.

**Senior checkpoint:** Admin visibility does not mean unlimited raw data display. Redaction still matters.

## Story 24: MCP CRM Tool Permission Smoke Tests

**Primary code paths:**
- `lib/mcp/tools/crm-contacts.ts`
- `lib/mcp/tools/crm-accounts.ts`
- `lib/mcp/tools/crm-opportunities.ts`
- `lib/mcp/helpers.ts`
- `lib/mcp/auth.ts`
- `lib/authz/scopes/crm.ts`

**Implementation plan:**
1. Pick representative MCP tools: one read list, one read detail, one mutation if available.
2. Verify each tool uses the same auth/authz helpers as app actions.
3. Add smoke tests with allowed user, denied user, and unauthenticated context.
4. Assert denied responses do not include private record fields.
5. Keep mocks small and local to the tool handler.
6. Document MCP tools as production API surface in the test name or helper comment.

**Tests and checks:**
- New MCP tool permission tests.
- Existing CRM authz tests if the tool delegates to shared scope helpers.

**Senior checkpoint:** Agent-facing tools are not "internal" once an agent can call them. Treat them like API endpoints.

## Story 25: Observability Quality Gate

**Primary code paths:**
- `package.json`
- `jest.config.ts`
- `playwright.config.ts`
- `CONTRIBUTING.md`
- `.github/workflows` if CI exists
- `docs3`

**Implementation plan:**
1. Define high-risk categories: authz, money, data export, background jobs, enrichment, MCP tools, migrations.
2. Map each category to required checks: unit, action, route, E2E, manual dev deploy, or migration review.
3. Add a lightweight doc, script, or package command for targeted local suites.
4. Keep trunk-based flow in mind: the gate should help `dev`, not create a heavyweight branch process.
5. Add story-template language requiring high-risk stories to cite the gate.
6. If CI workflow changes are needed, make the smallest additive check first.
7. Verify commands exist and do not require unavailable services unless clearly documented.

**Tests and checks:**
- Run the new script or at least `pnpm test -- --runInBand` equivalent for the targeted pattern if supported.
- Validate package scripts remain valid JSON.

**Senior checkpoint:** Senior engineers improve the system's ability to catch future mistakes, not only the current feature's behavior.

