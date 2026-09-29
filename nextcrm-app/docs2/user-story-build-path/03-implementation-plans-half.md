# Implementation Plans for Half of Each PM Story Set

This file expands half of each project-manager story group from `02-project-manager-stories.md`: Junior Stories 1-7 and Senior Stories 14-19. The plans are written as implementation guidance, not finished code.

## Scope Selection

- Junior set: 13 stories total, planning first 7.
- Senior set: 12 stories total, planning first 6.
- Total planned stories: 13.

## Junior Story 1: Clarify Empty Contacts Table Copy

**Goal:** Improve the contacts empty state without changing table behavior.

**Primary code paths:**
- `app/[locale]/(routes)/crm/components/ContactsView.tsx:80-87`
- `app/[locale]/(routes)/crm/contacts/page.tsx:15-23`

**Implementation plan:**

1. Open `ContactsView`.
2. Locate the conditional branch that currently renders `t("contacts.empty")` when `data` is missing or empty.
3. Decide whether the improved copy should live in translation files or remain behind the existing translation key.
4. Prefer updating the translation value for `contacts.empty` rather than hard-coding text in the component.
5. Verify the empty branch still renders only when `!data || data.length === 0`.
6. Do not change the `ContactsDataTable` branch.

**Validation checklist:**
- With zero contacts, the improved copy appears.
- With one or more contacts, the table appears.
- No changes are made to filtering, sorting, selection, or table columns.

**Risk notes:**
- If the copy is hard-coded in `ContactsView`, it may bypass the app's `next-intl` pattern.
- If the conditional changes, the table could disappear for valid empty arrays or render incorrectly for undefined data.

## Junior Story 2: Improve Add Contact Button Label

**Goal:** Make the add-contact action clearer while preserving test hooks and layout.

**Primary code paths:**
- `app/[locale]/(routes)/crm/components/ContactsView.tsx:55-72`
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:41-44`

**Implementation plan:**

1. Open `ContactsView`.
2. Find the `SheetTrigger` button with `data-testid="add-contact-btn"`.
3. Keep the `data-testid` unchanged.
4. Keep the button compact, but improve accessibility with a descriptive `aria-label`.
5. If adding visible text, confirm it does not crowd the header on small screens.
6. Ensure the `Sheet` still receives `open` and `onOpenChange`.
7. Ensure `NewContactForm` still receives `accounts` and `onFinish`.

**Validation checklist:**
- Clicking the button opens the contact sheet.
- `data-testid="add-contact-btn"` remains available for E2E tests.
- Screen readers can identify the button purpose.
- The sheet closes after successful contact creation.

**Risk notes:**
- Replacing the `SheetTrigger asChild` pattern can break sheet opening.
- Removing the test id can break `tests/e2e/contact-*` style workflows.

## Junior Story 3: Add Contact Position to the Contacts Table

**Goal:** Display each contact's `position` field in the contacts table.

**Primary code paths:**
- `prisma/schema.prisma:462-468`
- `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`
- `actions/crm/get-contacts.ts:18-58`

**Implementation plan:**

1. Confirm `position` exists on `crm_Contacts` in Prisma.
2. Open the contact table row schema.
3. Confirm `position` is already represented, or add it as nullable if needed.
4. Open `columns.tsx`.
5. Add a new column near `first_name`, `last_name`, or `email`.
6. Use the existing `DataTableColumnHeader` pattern.
7. Render a clean fallback when `row.getValue("position")` is null or empty.
8. Do not add a new Prisma query; `getContacts()` already returns full contact rows and related includes.

**Validation checklist:**
- Contacts with a position show the value.
- Contacts without a position do not crash the row.
- Existing name link still navigates to `/crm/contacts/[id]`.
- Sorting behavior is either intentionally enabled or disabled.

**Risk notes:**
- Typing `position` as required in Zod would fail rows where the database value is null.
- Adding the column to the wrong table file could affect leads or accounts instead of contacts.

## Junior Story 4: Rename Misleading Contact Table Types

**Goal:** Rename contact table types so they no longer refer to opportunities.

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:9-16`
- `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:16-44`

**Implementation plan:**

1. In `table-data/schema.tsx`, rename `opportunitySchema` to `contactRowSchema`.
2. Rename exported type `Opportunity` to `ContactRow`.
3. Update `columns.tsx` to import `ContactRow`.
4. Change `ColumnDef<Opportunity>[]` to `ColumnDef<ContactRow>[]`.
5. Update `data-table-row-actions.tsx` to import and use `contactRowSchema`.
6. Keep the schema fields exactly the same for this story.
7. Run TypeScript or lint after the rename if available.

**Validation checklist:**
- No imports still reference `Opportunity` from the contacts schema.
- Row action parsing still works.
- Columns render the same UI as before.
- No Prisma schema or database change is introduced.

**Risk notes:**
- This should be a rename-only change. Mixing in schema behavior changes makes review harder.
- If one import is missed, compile/typecheck should catch it.

## Junior Story 5: Fix Contacts Page Component Name

**Goal:** Rename the contacts page component from `AccountsPage` to a contacts-specific name.

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`

**Implementation plan:**

1. Open `app/[locale]/(routes)/crm/contacts/page.tsx`.
2. Rename `const AccountsPage = async () => { ... }` to `const ContactsPage = async () => { ... }`.
3. Update `export default AccountsPage` to `export default ContactsPage`.
4. Do not rename the file or route folder.
5. Do not change imports, data fetching, or JSX.

**Validation checklist:**
- The contacts route still compiles.
- The page still fetches `getAllCrmData()` and `getContacts()`.
- The route still renders `ContactsView`.
- No visible UI text changes.

**Risk notes:**
- Renaming the route folder would change URL behavior and is outside scope.
- This is a readability refactor; keep it isolated.

## Junior Story 6: Show a Friendly Unassigned Label

**Goal:** Ensure unassigned contacts have consistent labels for assigned user and account.

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:50-76`
- `actions/crm/get-contacts.ts:20-35`

**Implementation plan:**

1. Open `columns.tsx`.
2. Review the `assigned_to_user` column.
3. Confirm it already falls back to `Unassigned` when the related user is missing.
4. Review the assigned account column.
5. Ensure it also falls back to exactly `Unassigned` when `assigned_accounts?.name` is missing.
6. If labels are duplicated, consider a tiny local constant in the file.
7. Do not alter Prisma includes unless the relation data is missing.

**Validation checklist:**
- Contact with assigned user shows user name.
- Contact without assigned user shows `Unassigned`.
- Contact with assigned account shows account name.
- Contact without assigned account shows `Unassigned`.

**Risk notes:**
- The assigned account column currently accesses nested data through `row.original`; changing the row schema may require a typed accessor later.
- Avoid adding new fetches in a table cell.

## Junior Story 7: Add Loading Text to Contact Submit Button

**Goal:** Confirm and polish submit loading feedback in the contact form.

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120`
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:562-575`

**Implementation plan:**

1. Open `NewContactForm`.
2. Inspect `form.formState.isSubmitting` in the submit button area.
3. Confirm the button is disabled while submitting.
4. Confirm saving text is displayed while submitting.
5. If copy needs improvement, update the translation key value rather than hard-coding text.
6. Ensure `onSubmit` still sets root server errors on failure.
7. Ensure successful submission still toasts, resets, and calls `onFinish()`.

**Validation checklist:**
- Button is disabled during submission.
- Saving text appears during submission.
- Server errors appear on failed submission.
- Successful submission resets the form and closes the sheet.

**Risk notes:**
- If the button is not `type="submit"`, the form may stop submitting.
- If `onFinish()` runs before the server action returns, failed submissions may close the form incorrectly.

## Senior Story 14: Enforce Scoped Authorization on Contact Updates

**Goal:** Prevent users from updating contacts they are not allowed to modify.

**Primary code paths:**
- `actions/crm/contacts/update-contact.ts:33-82`
- `lib/authz/session.ts:11-23`
- `lib/authz/scopes/crm.ts:102-108`
- `lib/authz/scopes/crm.ts:289-302`
- `prisma/schema.prisma:446-504`

**Implementation plan:**

1. Open `update-contact.ts`.
2. Replace the raw `getSession()`-only authorization approach with an authz helper path.
3. Use `requireAuthenticated()` to get `{ id, role }`.
4. Before calling `prismadb.crm_Contacts.update`, call `assertCanWriteContact(user, id)`.
5. Use `user.id` for `updatedBy`.
6. Decide how to translate `AuthenticationError` and `AuthorizationError` into server-action return values.
7. Preserve existing audit log, diff, Inngest event, and `revalidatePath` behavior.
8. Add tests that prove:
   - admin/manager can update,
   - assigned or created user can update,
   - unrelated user cannot update,
   - unauthenticated user gets an error.

**Validation checklist:**
- Unauthorized updates are blocked before Prisma update.
- Existing successful update path still writes audit logs.
- UI receives a useful error response.
- No contact data leaks through authorization errors.

**Risk notes:**
- Throwing errors directly from server actions may break existing UI assumptions if the form expects `{ error }`.
- Do not remove audit logging while refactoring the auth path.

## Senior Story 15: Replace Loose Contact Mutation Types

**Goal:** Remove unsafe `as any` casts from contact create and update mutations.

**Primary code paths:**
- `actions/crm/contacts/create-contact.ts:9-99`
- `actions/crm/contacts/update-contact.ts:8-82`
- `prisma/schema.prisma:446-504`
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120`

**Implementation plan:**

1. Define explicit input types for create and update, preferably close to the actions.
2. Import Prisma types if useful, for example generated `Prisma.crm_ContactsCreateInput` or unchecked input variants if relations require scalar IDs.
3. Create a mapper function that converts form fields into Prisma data.
4. Normalize empty strings to `undefined` or `null` deliberately.
5. Convert birthday fields in one helper so create and update behave consistently.
6. Replace `data: { ... } as any` in create.
7. Replace `data: { ... } as any` in update.
8. Verify `contact_type_id` behavior against `prisma/schema.prisma:476-477`; if the UI sends display values instead of UUIDs, split that into a separate bug fix or map through real contact type ids.
9. Add focused unit tests for the mapper if the action is difficult to test directly.

**Validation checklist:**
- TypeScript accepts create/update data without `as any`.
- Existing form fields still save.
- Empty optional fields behave consistently.
- Contact type values are not silently corrupted.

**Risk notes:**
- Prisma relation fields may require either nested relation syntax or unchecked scalar input. Choose deliberately.
- This change can surface existing hidden bugs, especially around `contact_type_id`.

## Senior Story 16: Add Server-Side Contacts Pagination

**Goal:** Load contacts in pages from the server while preserving authorization scope.

**Primary code paths:**
- `actions/crm/get-contacts.ts:9-60`
- `lib/authz/scopes/crm.ts:289-302`
- `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`
- `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-179`
- `app/[locale]/(routes)/crm/contacts/table-components/data-table-pagination.tsx`

**Implementation plan:**

1. Design a return type like `{ rows, totalCount, page, pageSize }`.
2. Add a new server function, for example `getContactsPage(input)`, rather than immediately deleting `getContacts()`.
3. In the server function, call `requireAuthenticated()`.
4. Build `where` with `contactReadScopeWhere(user)`.
5. Apply `skip`, `take`, and deterministic `orderBy`.
6. Fetch `rows` and `totalCount`, preferably with `Promise.all`.
7. Update the contacts page to read `searchParams` for `page`, `pageSize`, and optional search/sort.
8. Pass paginated data to the table.
9. Convert the table to manual pagination mode or create a separate paginated table wrapper.
10. Update pagination controls to change URL params.
11. Define bulk selection semantics clearly: current page only unless a larger selection model is designed.
12. Add tests for scoped pagination and count leakage.

**Validation checklist:**
- A normal user cannot infer unauthorized contact counts.
- Page navigation updates visible contacts.
- Refresh keeps pagination state if URL params are used.
- Existing row actions still work.

**Risk notes:**
- Total counts can leak data if calculated without the same auth scope.
- Mixing client-side and server-side pagination can produce double pagination bugs.

## Senior Story 17: Move Enrichment Cancellation Out of Process Memory

**Goal:** Make contact enrichment cancellation reliable in multi-instance deployments.

**Primary code paths:**
- `app/api/crm/contacts/enrich/route.ts:20-22`
- `app/api/crm/contacts/enrich/route.ts:68-140`
- `app/api/crm/contacts/enrich/route.ts:144-179`
- `prisma/schema.prisma:115-133`

**Implementation plan:**

1. Document the current lifecycle: POST creates an enrichment row, stores `sessionId` in `activeSessions`, streams progress, and DELETE aborts if the session exists locally.
2. Choose a durable cancellation mechanism:
   - database-backed `cancelRequested`/status field, or
   - Redis/session store if infrastructure already exists.
3. Prefer database status if the enrichment worker can check cancellation between steps.
4. Add schema support if needed, such as a cancellation timestamp or status transition.
5. Update POST to persist `sessionId` or a durable cancellation token with the enrichment record.
6. Update DELETE to look up the enrichment record durably, validate `assertCanCancelContactEnrichment`, and mark cancellation requested.
7. Update the streaming process to check cancellation state and stop safely.
8. Ensure status transitions are explicit: `RUNNING`, `COMPLETED`, `FAILED`, cancelled representation.
9. Add tests or a documented manual test for POST-on-instance-A and DELETE-on-instance-B behavior.

**Validation checklist:**
- Cancel request does not depend on a local `Map`.
- Unauthorized cancel attempts fail.
- Cancelled enrichment records are no longer left as running.
- Disconnect abort behavior remains safe.

**Risk notes:**
- AbortController cannot be shared across processes; the design must shift from "remote abort" to "cooperative cancellation."
- Streaming code must avoid writing `COMPLETED` after cancellation.

## Senior Story 18: Parallelize Dashboard Metric Loading

**Goal:** Reduce dashboard latency by running independent data fetches concurrently.

**Primary code paths:**
- `app/[locale]/(routes)/page.tsx:44-74`
- `actions/crm/get-crm-data.ts:6-43`
- `actions/dashboard/get-contacts-count.ts:1-6`

**Implementation plan:**

1. Open the dashboard page.
2. Identify all independent awaited calls from `getLeadsCount()` through `getUsersTasksCount(userId)`.
3. Confirm none of these calls depend on the result of another metric call.
4. Preserve earlier dependencies: session, user id, cookies, default/display currency, translations.
5. Replace serial awaits with a single `Promise.all`.
6. Keep destructuring order readable and aligned with the Promise list.
7. Confirm the rendered cards still receive the same variables.
8. Consider error behavior: one rejected promise will reject the whole batch, matching the existing likely page failure behavior but faster.
9. Optionally add lightweight timing instrumentation during local validation, then remove it before commit.

**Validation checklist:**
- Dashboard renders the same cards and numbers.
- Expected revenue still uses `displayCurrency`.
- User-specific tasks still use `userId`.
- No debug timing logs remain.

**Risk notes:**
- Misordered destructuring can put the wrong metric into the wrong card.
- Do not parallelize work that depends on session or cookies before those values exist.

## Senior Story 19: Standardize API Error Response Shapes

**Goal:** Make validation and failure responses consistently JSON across API routes.

**Primary code paths:**
- `app/api/crm/targets/[id]/contacts/route.ts:36-38`
- `app/api/reports/export/route.ts:86-127`
- `lib/authz/route.ts:3-12`
- `app/api/crm/contacts/enrich/route.ts:33-66`

**Implementation plan:**

1. Inventory route handlers that return plain text or inconsistent error bodies.
2. Start with the small target-contact route.
3. Replace `new NextResponse("name or email required", { status: 400 })` with `NextResponse.json({ error: "name or email required" }, { status: 400 })`.
4. Preserve status codes.
5. Compare with report export's JSON error patterns.
6. If several routes need this, consider a tiny helper, but do not over-abstract after one endpoint.
7. Add or update route tests for the changed endpoint.
8. Document the expected error shape for future API work.

**Validation checklist:**
- Validation failure returns JSON.
- Existing auth failures still use shared authz responses.
- Clients expecting status codes still receive the same status.
- Tests assert both status and body.

**Risk notes:**
- Existing frontend code may read text responses from one endpoint. Search call sites before changing widely.
- Avoid changing success response shapes during an error-shape cleanup.

