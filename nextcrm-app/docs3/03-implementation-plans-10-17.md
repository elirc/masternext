# Implementation Plans: Stories 10-17

These stories train cross-feature ownership. Expect to touch UI, server actions, route handlers, Prisma models, tests, and documentation.

## Story 10: Document Duplicate Review Drawer

**Primary code paths:**
- `app/[locale]/(routes)/documents/components/bulk-upload-modal.tsx`
- `app/[locale]/(routes)/documents/components/modal-dropzone.tsx`
- `actions/documents/check-duplicate.ts`
- `actions/documents/create-document.ts`
- `actions/documents/__tests__/check-duplicate-scope.test.ts`

**Implementation plan:**
1. Trace the current upload flow from file selection to document creation.
2. Ensure duplicate checking happens before final create.
3. Introduce a review state that stores duplicate candidates and unique files.
4. Build a drawer or modal section using existing document UI primitives.
5. Let users skip duplicate files while preserving unique files.
6. Confirm final create still calls the authoritative server action.
7. Add tests around duplicate action scope and client state mapping if practical.

**Tests and checks:**
- Existing duplicate-scope tests.
- E2E or manual bulk upload flow with one duplicate and one unique file.

**Senior checkpoint:** The UI can guide the user, but the server action remains the trust boundary.

## Story 11: Fulltext Search Saved Filters

**Primary code paths:**
- `app/[locale]/(routes)/fulltext-search/page.tsx`
- `app/[locale]/(routes)/fulltext-search/search/page.tsx`
- `app/[locale]/(routes)/fulltext-search/search/components/ResultPage.tsx`
- `actions/fulltext/unified-search.ts`
- `actions/fulltext/__tests__/unified-search-scope.test.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Identify the current search filter state and URL parameters.
2. Design a small saved-filter model: id, userId, name, payload, createdAt, updatedAt.
3. Validate payload shape with a schema before saving and before applying.
4. Add actions to list, create, update, delete, and apply saved filters.
5. Scope all saved filters to the authenticated user.
6. Add UI for save current filter and select saved filter.
7. Applying a filter updates URL/search state without forcing unrelated state resets.
8. Test payload validation and user scoping.

**Tests and checks:**
- Action tests for user scoping.
- Pure schema tests for invalid payloads.

**Senior checkpoint:** Persisted UI state is untrusted input after it leaves memory; parse it like API data.

## Story 12: Report Export Background Job

**Primary code paths:**
- `app/api/reports/export/route.ts`
- `actions/reports/export-csv.ts`
- `actions/reports/export-pdf.ts`
- `actions/reports/schedule.ts`
- `inngest`
- `app/api/reports/export/__tests__/route.test.ts`

**Implementation plan:**
1. Establish the threshold for synchronous vs background export: row count, report type, or explicit request flag.
2. Design an export job record with status, requester, report category, format, parameters, artifact key, and error.
3. Keep the current route behavior for small exports.
4. For large exports, validate permissions and create a pending job.
5. Dispatch an Inngest job or existing background mechanism to generate the artifact.
6. Add a status endpoint or action that returns safe job state for the requester.
7. Add a download endpoint that checks the requester can access the job and artifact.
8. Make job execution idempotent enough to tolerate retry.

**Tests and checks:**
- Route tests for small sync export and large async response.
- Job tests for success, failure, and unauthorized download.

**Senior checkpoint:** Authorization must happen at request time and retrieval time because permissions can change while the job runs.

## Story 13: Role-Aware Report Templates

**Primary code paths:**
- `app/[locale]/(routes)/reports/page.tsx`
- `actions/reports/types.ts`
- `actions/reports/config.ts`
- `app/api/reports/export/route.ts`
- `lib/authz/scopes/report-scope.ts`
- `actions/reports/__tests__`

**Implementation plan:**
1. Locate report type/config definitions.
2. Add required scope metadata to each report template.
3. Build a helper that filters templates for a user/session.
4. Use the helper in report UI so unavailable templates are hidden or disabled.
5. Reuse the same policy helper, or a stricter server-side equivalent, in export routes/actions.
6. Add tests proving a hidden template cannot be exported by direct API call.
7. Confirm admin and manager roles still see intended templates.

**Tests and checks:**
- Report config scope tests.
- Export route permission test.

**Senior checkpoint:** UI filtering is a convenience; server policy is the control.

## Story 14: API Token Last-Used Details

**Primary code paths:**
- `lib/api-tokens.ts`
- `lib/api-keys.ts`
- `__tests__/lib/api-tokens.test.ts`
- `__tests__/lib/api-keys.test.ts`
- `app/[locale]/(routes)/profile/components/tabs/DeveloperTabContent.tsx`
- `app/[locale]/(routes)/profile/actions/api-keys.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Inspect token storage and verification paths.
2. Add fields for last-used timestamp and route family if they do not exist.
3. Update usage metadata after successful token verification only.
4. Avoid storing raw tokens, request bodies, or sensitive route params.
5. Ensure revoked tokens cannot update last-used metadata.
6. Return safe token metadata to the developer profile tab.
7. Display "Never used" clearly for new tokens.
8. Add tests for create, verify, revoke, and last-used update.

**Tests and checks:**
- Existing API token/key tests.
- Add a migration test only if schema migration tooling exists locally.

**Senior checkpoint:** Security telemetry should increase visibility without increasing blast radius.

## Story 15: Project Task SLA Board

**Primary code paths:**
- `app/[locale]/(routes)/projects/dashboard/page.tsx`
- `app/[locale]/(routes)/projects/dashboard/components/ProjectDasboard.tsx`
- `app/[locale]/(routes)/projects/boards/[boardId]/components/Kanban.tsx`
- `actions/projects/get-user-tasks-scope.test.ts`
- `actions/projects/get-tasks-past-due-scope.test.ts`
- `actions/projects`

**Implementation plan:**
1. Inspect current task fields: due date, status, blocked markers, done state.
2. Define an SLA classifier helper with deterministic date input.
3. Categories: blocked, overdue, due-soon, on-track, no-date if needed.
4. Build a read action that returns tasks visible to the current user.
5. Group tasks by classifier result.
6. Add an SLA board to project dashboard without changing the kanban data contract.
7. Add tests for classifier and scoped read behavior.

**Tests and checks:**
- Pure classifier tests using fixed dates.
- Existing project scope tests.

**Senior checkpoint:** Time-based behavior must accept an injectable "now" in tests.

## Story 16: CRM Activity Conflict Guard

**Primary code paths:**
- `actions/crm/activities/update-activity.ts`
- `actions/crm/activities/create-activity.ts`
- `actions/crm/activities/__tests__/get-activities-by-entity-scope.test.ts`
- `app/[locale]/(routes)/crm/*/[id]/components/ActivitiesSection.tsx`
- `prisma/schema.prisma`

**Implementation plan:**
1. Inspect activity update action and activity UI edit flow.
2. Include `updatedAt` or a version field in the edit payload.
3. On update, include the expected timestamp/version in the Prisma `where` condition or compare before writing inside a transaction.
4. Return a conflict result when the record changed after the user loaded it.
5. Show a conflict message in the UI with a refresh action.
6. Preserve authorization checks before conflict details are returned.
7. Add tests for fresh update, stale update, and unauthorized update.

**Tests and checks:**
- Activity action tests.
- Manual two-tab test if UI supports editing activities.

**Senior checkpoint:** Do not reveal that a record exists to a user who cannot read or write it; authz precedes conflict messaging.

## Story 17: Target Import Mapping Profiles

**Primary code paths:**
- `actions/crm/targets/import-targets.ts`
- `actions/crm/targets/suggest-mapping.ts`
- `app/api/crm/targets/enrich/validate.ts`
- `actions/crm/__tests__/get-targets-scope.test.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Trace target import mapping and suggested mapping behavior.
2. Design a mapping profile model with owner, name, source type, payload, and timestamps.
3. Validate mapping payloads before save and before apply.
4. Add actions to save, list, apply, rename, and delete profiles.
5. In the import UI, allow applying a profile before preview/commit.
6. Make preview show which profile is active.
7. Add tests for profile ownership, invalid payloads, and import application.

**Tests and checks:**
- Action tests for CRUD and ownership.
- Existing target import tests if present.

**Senior checkpoint:** Reusable bulk-operation configuration needs strict validation because one bad profile can damage many rows.

