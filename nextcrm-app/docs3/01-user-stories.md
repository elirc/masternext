# 25 More User Stories

These stories continue the earlier `docs2` build path. They deliberately move across CRM, reports, invoices, documents, enrichment, API tooling, and platform quality so you practice the full surface area of a production SaaS app.

Each story has three layers:

- Product outcome: what the user gets.
- Engineering outcome: what the system learns to do reliably.
- Growth outcome: the senior-level skill being trained.

## Story 1: Dashboard Metric Drill-Down Links

**Level:** Junior to Mid
**Story:** As a sales manager, I want dashboard metric cards to link to the filtered list behind the number so that I can move from summary to action quickly.
**Acceptance Criteria:**
- Dashboard cards for contacts, accounts, opportunities, invoices, and documents navigate to the matching route.
- The destination route preserves the user's locale.
- The card remains keyboard accessible.
- No dashboard count query changes are introduced.
**Growth Outcome:** Learn to trace app shell routing and improve UX without widening backend scope.

## Story 2: Contact Quick Note

**Level:** Mid
**Story:** As a sales user, I want to add a quick note from a contact detail page so that relationship context is captured without leaving the page.
**Acceptance Criteria:**
- Contact detail displays existing notes.
- Authenticated users with contact write access can add a note.
- Empty notes are rejected.
- Note creation updates UI state or refreshes route data.
- Unauthorized writes fail on the server.
**Growth Outcome:** Practice frontend state, route handlers or server actions, authz, and persistence.

## Story 3: Account Watcher Digest

**Level:** Mid
**Story:** As an account owner, I want watched-account updates summarized in one digest area so that I can follow changes without opening each account manually.
**Acceptance Criteria:**
- Watched accounts can be listed for the current user.
- Recent audit or activity entries are shown per watched account.
- Deleted accounts do not appear unless explicitly restored.
- The query respects CRM read scope.
**Growth Outcome:** Learn relation traversal, scoped reads, and product-friendly aggregation.

## Story 4: Product Import Validation Preview

**Level:** Mid
**Story:** As an operations user, I want to preview product CSV import errors before committing rows so that bad data does not enter the catalog.
**Acceptance Criteria:**
- CSV rows are parsed into a preview result.
- Invalid rows report row number, field, and message.
- Preview mode writes no database records.
- Confirmed import uses the same validation rules.
**Growth Outcome:** Separate validation from mutation and design safer bulk workflows.

## Story 5: Opportunity Forecast Confidence

**Level:** Mid
**Story:** As a sales leader, I want opportunities to show forecast confidence based on stage and expected revenue so that pipeline reviews are more realistic.
**Acceptance Criteria:**
- Opportunity rows show a confidence label derived from existing fields.
- Decimal money values serialize correctly across server/client boundaries.
- Empty or unknown stages fall back safely.
- Tests cover the confidence calculation.
**Growth Outcome:** Practice domain logic extraction, Decimal handling, and deterministic tests.

## Story 6: Invoice Partial Payment Guardrails

**Level:** Mid
**Story:** As a finance user, I want partial payments to prevent overpayment and invalid negative values so that invoice balances remain correct.
**Acceptance Criteria:**
- Adding a payment rejects zero, negative, and over-balance amounts.
- Validation happens server-side before persistence.
- The invoice detail UI shows a clear error.
- Existing payment totals still calculate correctly.
**Growth Outcome:** Own money invariants and learn to protect them with tests.

## Story 7: Invoice PDF Regeneration Audit

**Level:** Mid
**Story:** As an administrator, I want invoice PDF regeneration to be audited so that finance artifacts have a traceable history.
**Acceptance Criteria:**
- Regeneration writes an audit entry with actor, invoice id, and timestamp.
- Failed regeneration does not create a success audit entry.
- Existing PDF regeneration behavior remains unchanged.
- A test proves the audit entry is written after success.
**Growth Outcome:** Practice side-effect ordering and auditability.

## Story 8: Campaign Unsubscribe Analytics

**Level:** Mid
**Story:** As a marketer, I want campaign reports to show unsubscribe counts so that I can evaluate message quality.
**Acceptance Criteria:**
- Campaign report data includes unsubscribe totals.
- Webhook or unsubscribe route behavior remains compatible.
- The report page renders zero cleanly.
- Tests cover campaigns with and without unsubscribes.
**Growth Outcome:** Connect route events, reporting data, and UI presentation.

## Story 9: Email Sync Health Badge

**Level:** Mid
**Story:** As a user, I want email accounts to show sync health so that I know which inboxes need attention.
**Acceptance Criteria:**
- Email account list shows healthy, syncing, failed, or disconnected state.
- The badge is derived from existing sync/account data where possible.
- Failure details are visible without exposing secrets.
- The UI handles accounts with no sync history.
**Growth Outcome:** Build operational UX from existing state without leaking sensitive data.

## Story 10: Document Duplicate Review Drawer

**Level:** Mid to Senior
**Story:** As a document manager, I want possible duplicate uploads shown in a review drawer so that I can decide whether to keep or skip them.
**Acceptance Criteria:**
- Duplicate check results open a review UI before upload confirmation.
- The user can skip duplicates and continue with unique documents.
- Server-side duplicate detection remains authoritative.
- Bulk upload state remains recoverable after closing the drawer.
**Growth Outcome:** Coordinate client workflow with server truth.

## Story 11: Fulltext Search Saved Filters

**Level:** Mid to Senior
**Story:** As a power user, I want to save fulltext search filters so that recurring searches are one click away.
**Acceptance Criteria:**
- Users can save a named filter set.
- Saved filters are scoped to the current user.
- Applying a saved filter updates search state and URL where appropriate.
- Invalid saved filter payloads are ignored or rejected safely.
**Growth Outcome:** Design persisted user preferences and defensive parsing.

## Story 12: Report Export Background Job

**Level:** Senior
**Story:** As a manager exporting large reports, I want long exports to run in the background so that the request does not time out.
**Acceptance Criteria:**
- Small exports still return synchronously.
- Large exports create a job and return job status.
- Job completion stores a downloadable artifact or clear result.
- Authorization is checked at request and download time.
**Growth Outcome:** Learn async job design, idempotency, and user-facing progress.

## Story 13: Role-Aware Report Templates

**Level:** Senior
**Story:** As an administrator, I want report templates to respect roles so that users only see relevant report options.
**Acceptance Criteria:**
- Report templates declare required scope or role.
- UI hides unavailable templates.
- Server routes still enforce permissions.
- Tests prove hidden templates cannot be exported directly.
**Growth Outcome:** Understand defense in depth between UI gating and server policy.

## Story 14: API Token Last-Used Details

**Level:** Senior
**Story:** As a developer, I want API tokens to show last-used timestamp and route family so that I can identify stale or risky tokens.
**Acceptance Criteria:**
- Token usage updates last-used metadata without storing raw token values.
- Profile developer tab displays last-used details.
- Revoked tokens cannot update usage metadata.
- Tests cover token creation, use, revocation, and display mapping.
**Growth Outcome:** Build security-sensitive telemetry with privacy constraints.

## Story 15: Project Task SLA Board

**Level:** Senior
**Story:** As a team lead, I want project tasks grouped by SLA risk so that I can triage overdue work quickly.
**Acceptance Criteria:**
- Tasks are categorized as on-track, due-soon, overdue, or blocked.
- The board respects project/task read permissions.
- Date calculations are deterministic in tests.
- Existing kanban behavior remains intact.
**Growth Outcome:** Add a new product view without destabilizing existing boards.

## Story 16: CRM Activity Conflict Guard

**Level:** Senior
**Story:** As a sales user, I want activity edits to detect stale updates so that two users do not overwrite each other's notes.
**Acceptance Criteria:**
- Activity update payload includes the version or `updatedAt` value the user edited.
- Server rejects stale writes with a conflict response.
- UI offers a refresh path after conflict.
- Tests cover fresh update and stale update branches.
**Growth Outcome:** Learn optimistic concurrency and conflict UX.

## Story 17: Target Import Mapping Profiles

**Level:** Senior
**Story:** As a growth operator, I want reusable target import mappings so that recurring vendor files can be imported consistently.
**Acceptance Criteria:**
- Users can save a mapping profile after previewing an import.
- Saved profiles are user or organization scoped by design.
- Applying a profile preselects column mappings.
- Invalid profile references fail safely.
**Growth Outcome:** Persist reusable workflow configuration and protect bulk operations.

## Story 18: Enrichment Budget And Rate-Limit Dashboard

**Level:** Senior
**Story:** As an administrator, I want enrichment usage and rate-limit status visible so that AI-powered workflows stay predictable.
**Acceptance Criteria:**
- Dashboard shows recent enrichment counts by status and entity type.
- Rate-limit state is visible without exposing provider secrets.
- Failed and skipped enrichments are distinguishable.
- Tests cover aggregation logic.
**Growth Outcome:** Turn operational constraints into product-visible system health.

## Story 19: Saved CRM Table Views

**Level:** Senior
**Story:** As a CRM power user, I want to save table filters, sorting, and column visibility so that I can return to my working view.
**Acceptance Criteria:**
- Contacts, accounts, or opportunities tables can save named views.
- Views store filters, sorting, and column visibility.
- Views are scoped to the creating user.
- Invalid view state cannot crash the table.
**Growth Outcome:** Design reusable state persistence around table contracts.

## Story 20: Multi-Entity Timeline

**Level:** Senior
**Story:** As a sales manager, I want contact, account, opportunity, note, activity, enrichment, and audit events shown in one timeline so that I can understand relationship history.
**Acceptance Criteria:**
- Timeline events are normalized into a stable type.
- Query applies read scope before returning events.
- Events sort consistently by time.
- UI distinguishes event source and actor.
**Growth Outcome:** Build an aggregation boundary that hides source complexity.

## Story 21: Soft-Delete Restore Center

**Level:** Senior
**Story:** As an administrator, I want a restore center for recently deleted CRM records so that accidental deletes are recoverable.
**Acceptance Criteria:**
- Restore center lists soft-deleted contacts, accounts, leads, opportunities, products, and contracts where supported.
- Restore action checks admin or appropriate owner permissions.
- Restored records receive audit entries.
- Hard-deleted or unsupported records are not shown as restorable.
**Growth Outcome:** Own data lifecycle, permissions, and operational safety.

## Story 22: Decimal Serialization Hardening Sweep

**Level:** Senior
**Story:** As an engineer, I want all Decimal-returning server actions and Server Components hardened so that money fields never break client boundaries.
**Acceptance Criteria:**
- Decimal fields in invoices, opportunities, products, and line items are identified.
- Server actions returning Decimal-bearing records use `serializeDecimals()` or `serializeDecimalsList()`.
- Tests or type-level checks cover at least one high-risk path.
- No return value is narrowed to `{ id }` just to avoid serialization.
**Growth Outcome:** Practice cross-cutting risk reduction without broad refactor churn.

## Story 23: Audit Log Diff Viewer

**Level:** Senior
**Story:** As an administrator, I want audit log entries to show readable field diffs so that record changes are easier to review.
**Acceptance Criteria:**
- Audit detail UI shows changed fields with before/after values.
- Sensitive fields are redacted.
- Decimal and date values are formatted consistently.
- Existing audit records with older metadata still render safely.
**Growth Outcome:** Build trustworthy admin tooling over imperfect historical data.

## Story 24: MCP CRM Tool Permission Smoke Tests

**Level:** Senior
**Story:** As a platform owner, I want MCP CRM tools covered by permission smoke tests so that agent access cannot bypass app policy.
**Acceptance Criteria:**
- Tests cover allowed and denied access for representative CRM MCP tools.
- Tool handlers reuse existing authz helpers.
- Error responses do not leak private record data.
- Tests are fast and isolated from external services.
**Growth Outcome:** Treat AI/agent tooling as production API surface.

## Story 25: Observability Quality Gate

**Level:** Staff-Track Senior
**Story:** As an engineering lead, I want a quality gate for high-risk changes so that auth, money, async jobs, and data export regressions are caught before release.
**Acceptance Criteria:**
- A documented checklist maps risk type to required tests.
- CI or local scripts can run targeted suites for authz, invoices, reports, enrichment, and MCP tools.
- New high-risk stories cite the gate in their implementation plan.
- The gate is lightweight enough for trunk-based development on `dev`.
**Growth Outcome:** Move from feature ownership to engineering-system ownership.

