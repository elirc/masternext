# User Stories

## Story 1: Rename the Contacts Page Component

**Difficulty:** Easy
**Estimated Time:** 0.5 hours
**Skills You'll Practice:** reading routes, safe rename, explaining intent
**The Story:** As a product owner, I want internal component names to match the page domain so that engineers are less confused when tracing contacts.
**Acceptance Criteria:**
- [ ] `app/[locale]/(routes)/crm/contacts/page.tsx` no longer names the default component `AccountsPage`.
- [ ] The default export still works for the contacts route.
- [ ] No user-visible behavior changes.
**Files You'll Likely Touch:** `app/[locale]/(routes)/crm/contacts/page.tsx` because the route component is currently named `AccountsPage` at `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`.
**High-Level Implementation Plan:**
1. In `app/[locale]/(routes)/crm/contacts/page.tsx`, rename `AccountsPage` to `ContactsPage`.
2. Update the default export in the same file.
3. Run or reason through TypeScript import impact; this is a local symbol only.
**Tips:**
- The page already imports contact-specific data through `getContacts()` at `app/[locale]/(routes)/crm/contacts/page.tsx:7-14`.
- Keep `Container`, `Suspense`, and `ContactsView` unchanged at `app/[locale]/(routes)/crm/contacts/page.tsx:15-23`.
- This story is about reducing cognitive load, not changing UX.
**What Could Go Wrong:**
- You rename the file instead of the component and break routing.
- You change translated title keys at `app/[locale]/(routes)/crm/contacts/page.tsx:16-18` and accidentally alter UI copy.
**Stretch Goal:** Add a short code comment only if you find a non-obvious reason for the route shape.
**Connects To:** Story 2 because both clean up confusing contact table naming.

## Story 2: Rename the Contact Row Schema

**Difficulty:** Easy
**Estimated Time:** 1 hour
**Skills You'll Practice:** TypeScript rename, Zod schema reading, import tracing
**The Story:** As an engineer, I want the contacts table schema to be named after contacts so that table code is easier to review.
**Acceptance Criteria:**
- [ ] `opportunitySchema` is renamed to a contact-specific name.
- [ ] `Opportunity` type is renamed to a contact-specific type.
- [ ] Imports in contact table columns and row actions still compile.
**Files You'll Likely Touch:** `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx` because the current contact row schema is named `opportunitySchema` at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`; `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx` because it imports `Opportunity` at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:9-16`; `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx` because it parses `row.original` with `opportunitySchema` at `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:16-44`.
**High-Level Implementation Plan:**
1. Rename `opportunitySchema` to `contactRowSchema` in `table-data/schema.tsx`.
2. Rename `Opportunity` to `ContactRow`.
3. Update imports and usages in `columns.tsx`.
4. Update imports and parsing in `data-table-row-actions.tsx`.
**Tips:**
- Do not change field names yet; keep the same schema shape from `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:7-18`.
- The contact model includes corresponding name, email, phone, website, position, and status fields at `prisma/schema.prisma:459-468`.
- This is a rename-only story.
**What Could Go Wrong:**
- You change schema behavior while renaming and introduce runtime parse failures.
- You miss one import and TypeScript fails.
**Stretch Goal:** Add fields to the schema only after confirming they exist in `prisma/schema.prisma:446-504`.
**Connects To:** Story 3 because the next display change depends on clean row naming.

## Story 3: Display Contact Position in the Contacts Table

**Difficulty:** Easy
**Estimated Time:** 1 hour
**Skills You'll Practice:** table columns, Prisma field verification, UI display
**The Story:** As a sales user, I want to see a contact's position in the contacts table so that I can identify decision makers faster.
**Acceptance Criteria:**
- [ ] Contacts table shows a `Position` column.
- [ ] The column uses the existing `position` field.
- [ ] Empty positions render cleanly without crashing.
**Files You'll Likely Touch:** `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx` because contact row fields are validated there at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`; `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx` because columns are created there at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`.
**High-Level Implementation Plan:**
1. Confirm `position` exists on contacts in `prisma/schema.prisma:462-468`.
2. Ensure the table row schema includes `position`.
3. Add a column in `createColumns()` near the name/email fields in `columns.tsx`.
4. Render a fallback such as an empty string or `Unspecified`.
**Tips:**
- The table already displays email and phone using `row.getValue()` at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:100-115`.
- Use the same `DataTableColumnHeader` pattern used by existing columns at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:78-90`.
- Do not fetch new data; `getContacts()` returns full contact rows at `actions/crm/get-contacts.ts:18-58`.
**What Could Go Wrong:**
- The Zod schema rejects a nullable position if you type it as required.
- The new column is added after actions and becomes hard to scan.
**Stretch Goal:** Add a column visibility option if the toolbar supports hiding columns.
**Connects To:** Story 4 because it prepares you to add richer table display using existing data.

## Story 4: Add Assigned Account Filtering to Contacts

**Difficulty:** Medium-Easy
**Estimated Time:** 2 hours
**Skills You'll Practice:** table filtering, nested row data, existing relation display
**The Story:** As a sales manager, I want to filter contacts by assigned account so that I can focus on one customer organization.
**Acceptance Criteria:**
- [ ] Contacts table can filter by assigned account.
- [ ] The filter uses existing included account data.
- [ ] Contacts without an assigned account remain understandable.
**Files You'll Likely Touch:** `actions/crm/get-contacts.ts` because assigned account data is included at `actions/crm/get-contacts.ts:33-34`; `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx` because assigned account display is at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:64-76`; `app/[locale]/(routes)/crm/contacts/table-components/data-table-toolbar.tsx` because toolbar controls belong near table filters; `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx` because `columnFilters` state is owned there at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74`.
**High-Level Implementation Plan:**
1. Inspect how `DataTableToolbar` filters existing columns.
2. Confirm the assigned account column has a stable accessor/filter value.
3. Add a faceted filter or select for account names.
4. Ensure unassigned contacts can be included or selected.
**Tips:**
- The assigned account currently reads `(row.original as any).assigned_accounts?.name` at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:69-72`.
- Avoid new server queries; account relation data is already included by `getContacts()`.
- If filtering nested objects is awkward, introduce a derived accessor in the column.
**What Could Go Wrong:**
- Filtering on a nested object instead of a string gives confusing results.
- Unassigned contacts disappear without a clear way to recover them.
**Stretch Goal:** Persist the filter in the URL so refresh keeps it.
**Connects To:** Story 5 because both extend table UX without changing the database.

## Story 5: Add Bulk Selection Summary for Contacts

**Difficulty:** Medium-Easy
**Estimated Time:** 2 hours
**Skills You'll Practice:** local state, selected rows, conditional UI
**The Story:** As a sales user, I want a clearer summary of selected contacts so that bulk enrichment feels safer.
**Acceptance Criteria:**
- [ ] When rows are selected, the table shows selected count.
- [ ] The summary includes at least the first selected contact name.
- [ ] The existing bulk enrich button still opens the modal.
**Files You'll Likely Touch:** `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx` because row selection and bulk enrichment UI live at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:101-121`; `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx` because selected row names depend on row fields at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:7-18`.
**High-Level Implementation Plan:**
1. In `ContactsDataTable`, derive `selectedRows` from `table.getSelectedRowModel().rows`.
2. Compute a display name from `first_name` and `last_name`.
3. Add the display name next to the existing selected count.
4. Keep `BulkEnrichModal` props unchanged.
**Tips:**
- Existing selected count is rendered at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:102-115`.
- Contact row links use `last_name` at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:87-95`.
- Keep this UI local; selected rows are table-specific state.
**What Could Go Wrong:**
- You assume `first_name` is always present, but schema allows nullable at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:7-10`.
- You recompute selected rows in a way that breaks the modal's contact id mapping at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:117-121`.
**Stretch Goal:** Show "and N more" after the first selected name.
**Connects To:** Story 6 because selection state can feed an API-backed action.

## Story 6: Add a Contact Notes API Endpoint

**Difficulty:** Medium
**Estimated Time:** 4 hours
**Skills You'll Practice:** route handler, authz, Prisma update, JSON contract
**The Story:** As a sales user, I want to add a note to a contact so that important context stays attached to the person.
**Acceptance Criteria:**
- [ ] A new API endpoint accepts a contact id and note text.
- [ ] The endpoint requires authentication.
- [ ] The endpoint verifies the user can write the contact.
- [ ] The endpoint appends to the existing `notes` array.
- [ ] Error responses are JSON.
**Files You'll Likely Touch:** `prisma/schema.prisma` because contacts already have `notes String[]` at `prisma/schema.prisma:478-479`; `lib/authz/scopes/crm.ts` because `assertCanWriteContact()` exists at `lib/authz/scopes/crm.ts:102-108`; `app/api/crm/contacts/[id]/route.ts` or a new nested route because API handlers live under `app/api`; `app/api/crm/targets/[id]/contacts/route.ts` because it is a simple authz route example at `app/api/crm/targets/[id]/contacts/route.ts:12-52`.
**High-Level Implementation Plan:**
1. Create a new route under `app/api/crm/contacts/[id]/notes/route.ts`.
2. Read `id` from params and note text from JSON.
3. Call `requireAuthenticated()` and `assertCanWriteContact()`.
4. Validate that note text is non-empty.
5. Fetch current notes or use Prisma array update if appropriate.
6. Return the updated note list or contact id.
**Tips:**
- Use JSON errors like `app/api/reports/export/route.ts:86-87`, not plain text.
- Follow auth error handling patterns from `app/api/crm/targets/[id]/contacts/route.ts:18-30`.
- Remember contact notes already exist in the model; no migration is needed for this basic story.
**What Could Go Wrong:**
- You update by id without authz and create a record-access bug.
- You overwrite existing notes instead of appending.
- Empty notes are accepted and pollute contact history.
**Stretch Goal:** Write a Jest route test for unauthorized, forbidden, validation, and success cases.
**Connects To:** Story 7 because the UI will need state around the new API behavior.

## Story 7: Add Notes UI to Contact Detail

**Difficulty:** Medium
**Estimated Time:** 5 hours
**Skills You'll Practice:** form state, API calls, optimistic refresh, detail page tracing
**The Story:** As a sales user, I want to add and view notes on a contact detail page so that I can track relationship context.
**Acceptance Criteria:**
- [ ] Contact detail page displays existing notes.
- [ ] User can submit a new note.
- [ ] Empty note submissions are blocked.
- [ ] UI updates after successful save.
- [ ] API errors are shown to the user.
**Files You'll Likely Touch:** `app/[locale]/(routes)/crm/contacts/[contactId]/page.tsx` because contact detail route exists in the file tree; `actions/crm/get-contact.ts` because detail data is likely fetched there; your new notes route from Story 6; `components/ui/textarea.tsx` and `components/ui/button.tsx` if using existing UI primitives; `lib/authz/scopes/crm.ts` because read/write visibility rules live there at `lib/authz/scopes/crm.ts:91-108`.
**High-Level Implementation Plan:**
1. Inspect the contact detail page and identify where contact data is loaded.
2. Add a notes section that renders `contact.notes`.
3. Add a textarea and submit button.
4. Call the notes API from Story 6.
5. On success, refresh route data or update local notes state.
6. Show validation and server errors.
**Tips:**
- Reuse the form error pattern from `NewContactForm` at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120`.
- For server data refresh, compare how row actions call `router.refresh()` after delete at `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:50-66`.
- Keep authz on the server; UI checks are only user experience.
**What Could Go Wrong:**
- You trust the client and skip server authorization.
- You duplicate notes locally but forget to persist them.
- You refresh the whole app shell instead of just the route data.
**Stretch Goal:** Add timestamps by changing the data model from `String[]` to structured note records.
**Connects To:** Story 8 because structured notes lead naturally to schema and migration work.

## Story 8: Convert Contact Notes Into Structured Records

**Difficulty:** Hard
**Estimated Time:** 8 hours
**Skills You'll Practice:** Prisma migration, relations, server actions/API, UI migration thinking
**The Story:** As a sales manager, I want contact notes to store author and timestamp so that the team can trust who wrote each note and when.
**Acceptance Criteria:**
- [ ] A new Prisma model stores contact note text, author, contact id, and created time.
- [ ] Contact detail displays notes with author and timestamp.
- [ ] Creating a note writes the current user as author.
- [ ] Users can only create notes on contacts they can write.
- [ ] Existing `notes String[]` behavior is handled or intentionally migrated.
**Files You'll Likely Touch:** `prisma/schema.prisma` because contact and user relations are defined at `prisma/schema.prisma:446-504` and `prisma/schema.prisma:922-987`; `lib/authz/scopes/crm.ts` because contact write rules are at `lib/authz/scopes/crm.ts:102-108`; your notes route/action from Stories 6-7; `app/[locale]/(routes)/crm/contacts/[contactId]/page.tsx`; `prisma/migrations/*` for the generated migration.
**High-Level Implementation Plan:**
1. Design a `crm_ContactNote` model related to `crm_Contacts` and `Users`.
2. Generate a Prisma migration.
3. Update detail query to include notes and authors.
4. Update create-note API to write a note row instead of appending a string.
5. Update UI to show note text, author, and created time.
6. Decide how to display legacy `notes String[]` during transition.
**Tips:**
- Follow relation style from contact enrichment relations at `prisma/schema.prisma:115-133`.
- Users already relate to many CRM entities at `prisma/schema.prisma:943-980`.
- Soft delete may be needed if notes can be removed later; contact soft delete fields are at `prisma/schema.prisma:492-504`.
**What Could Go Wrong:**
- Migration breaks existing data because legacy notes are ignored without a plan.
- Relation names conflict with existing user/contact relations.
- UI expects old `notes: string[]` but receives note objects.
**Stretch Goal:** Add edit/delete note permissions with audit logging.
**Connects To:** Story 9 because structured records create better audit and reporting possibilities.

## Story 9: Add Audit History to Contact Notes

**Difficulty:** Hard
**Estimated Time:** 8 hours
**Skills You'll Practice:** audit logging, authz, tests, UI traceability
**The Story:** As an admin, I want contact note changes to appear in audit history so that sensitive relationship updates are traceable.
**Acceptance Criteria:**
- [ ] Creating a note writes an audit log entry.
- [ ] Updating or deleting notes, if implemented, writes audit entries.
- [ ] Audit entries identify contact id and acting user.
- [ ] Tests verify at least create-note audit behavior.
**Files You'll Likely Touch:** `lib/audit-log.ts` because audit helpers are used by contact create at `actions/crm/contacts/create-contact.ts:85-91`; your structured notes route/action; `prisma/schema.prisma` because audit model fields are at `prisma/schema.prisma:989-1004`; a new Jest test under `__tests__` or route test folder.
**High-Level Implementation Plan:**
1. Inspect `writeAuditLog()` in `lib/audit-log.ts`.
2. After successful note creation, call `writeAuditLog()`.
3. Use `entityType: "contact"` or introduce a note-specific type if the audit helper supports it.
4. Include changes that identify note text or note id without leaking sensitive excess.
5. Write a Jest test that mocks Prisma and verifies audit call/write.
**Tips:**
- Contact creation already demonstrates audit usage at `actions/crm/contacts/create-contact.ts:85-91`.
- The audit table indexes entity type/id/time at `prisma/schema.prisma:989-1004`.
- Keep authorization before auditing; denied attempts should not look like successful changes unless you intentionally add security-event auditing.
**What Could Go Wrong:**
- Audit logging happens before the note write and records changes that never persisted.
- Audit changes include too much sensitive note content.
- The test mocks the wrong layer and misses the policy.
**Stretch Goal:** Display note audit history in the contact detail page's history tab.
**Connects To:** Story 10 because audit, auth, and performance concerns shape larger architecture.

## Story 10: Add Server-Side Pagination for Contacts

**Difficulty:** Expert
**Estimated Time:** 12+ hours
**Skills You'll Practice:** architecture design, API contract, URL state, server data fetching, performance, authz
**The Story:** As a team with a growing CRM, I want the contacts table to load pages from the server so that thousands of contacts remain fast and secure.
**Acceptance Criteria:**
- [ ] Contacts list no longer requires loading every visible contact into the browser.
- [ ] Server query applies `contactReadScopeWhere()`.
- [ ] Pagination, sorting, and search are represented in the URL or an explicit request contract.
- [ ] Table UI supports next/previous pages and shows total count.
- [ ] Existing row actions and bulk selection remain understandable.
- [ ] Tests cover scoped server pagination behavior.
**Files You'll Likely Touch:** `actions/crm/get-contacts.ts` because it currently returns all scoped contacts at `actions/crm/get-contacts.ts:18-58`; `lib/authz/scopes/crm.ts` because scope must stay in the server query at `lib/authz/scopes/crm.ts:289-302`; `app/[locale]/(routes)/crm/contacts/page.tsx` because route data loading starts at `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`; `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx` because current pagination is client-side at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:54-74`; `app/[locale]/(routes)/crm/contacts/table-components/data-table-pagination.tsx`; `app/[locale]/(routes)/crm/contacts/table-components/data-table-toolbar.tsx`; `__tests__/crm` or `actions/crm/__tests__` for scoped pagination tests.
**High-Level Implementation Plan:**
1. Design a `getContactsPage({ page, pageSize, sort, search })` server function.
2. Apply `requireAuthenticated()` and `contactReadScopeWhere(user)` before `findMany`.
3. Return `{ rows, totalCount, page, pageSize }`.
4. Update the route page to read search params and pass paginated data.
5. Convert the table to manual pagination/sorting/filtering.
6. Update toolbar and pagination controls to mutate URL params.
7. Preserve row actions and rethink bulk selection as page-scoped selection.
8. Add tests that prove basic users cannot page through contacts outside scope.
**Tips:**
- Use `getAllCrmData()` as an example of batching independent queries with `Promise.all` at `actions/crm/get-crm-data.ts:6-43`.
- Keep `contactReadScopeWhere()` in the server action; never trust client filters for security.
- The current table state lives locally at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74`; manual server pagination will move some of that state to URL/search params.
- Watch Decimal serialization if the paginated query expands into contracts or opportunities later; helper is `lib/serialize-decimals.ts:5-25`.
**What Could Go Wrong:**
- Search or sorting is applied before auth scope in a way that leaks counts.
- Bulk selection silently changes from all filtered rows to current page only.
- URL state and table state drift apart, causing confusing back/forward behavior.
**Stretch Goal:** Add saved contact table views per user.
**Connects To:** This is the capstone story: it uses routing, authz, server data, table state, tests, and performance design.

