# Additional High-Level Plans

This file adds 10 junior-oriented and 10 senior-oriented high-level implementation plans. These are intentionally lighter than `03-implementation-plans-half.md`: they name the likely code areas, outcome, and broad approach without detailed implementation steps.

## Junior High-Level Plans

## Junior Plan 1: Standardize Contact Table Column Labels

**Related PM Story:** Story 8
**Outcome:** Contact table labels should read professionally and consistently.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:37-123`

**High-level plan:**
- Review all visible contact table labels.
- Correct awkward wording such as `Sure name`.
- Keep accessor keys and sorting behavior unchanged.
- Prefer existing translation conventions if table labels are already localized elsewhere.

## Junior Plan 2: Add a Contact Website Column

**Related PM Story:** Story 9
**Outcome:** Sales users can see contact website values directly in the contacts table.
**Likely code paths:**
- `prisma/schema.prisma:462-468`
- `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`

**High-level plan:**
- Confirm the `website` field exists on contacts.
- Ensure the table row schema accepts nullable website values.
- Add a website column using the existing column pattern.
- Render empty websites safely.

## Junior Plan 3: Improve Contact Form Error Visibility

**Related PM Story:** Story 10
**Outcome:** Users can clearly see why contact creation failed.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120`
- `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:562-568`

**High-level plan:**
- Keep the current root server error behavior.
- Improve styling or placement only where the submit area already handles errors.
- Preserve successful submit behavior: toast, reset, and close.
- Avoid changing validation rules in this UI-only story.

## Junior Plan 4: Add Contact Count to Contacts View Header

**Related PM Story:** Story 11
**Outcome:** Users can quickly see how many contacts are currently displayed.
**Likely code paths:**
- `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`

**High-level plan:**
- Use the `data` prop already passed into `ContactsView`.
- Display a small count near the contacts title or table header.
- Keep the empty state behavior unchanged.
- Do not introduce a new database count query for this local displayed-count feature.

## Junior Plan 5: Make Bulk Selection Summary More Helpful

**Related PM Story:** Story 12
**Outcome:** Users get clearer confirmation before starting bulk enrichment.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:101-121`
- `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`

**High-level plan:**
- Use `table.getSelectedRowModel().rows`.
- Keep the existing selected count.
- Add a lightweight name preview from selected contact rows.
- Preserve the existing `BulkEnrichModal` contract.

## Junior Plan 6: Add User-Friendly Report Export Error Copy

**Related PM Story:** Story 13
**Outcome:** Invalid report export responses are easier for users and support to understand.
**Likely code paths:**
- `app/api/reports/export/route.ts:82-127`

**High-level plan:**
- Review existing JSON error messages.
- Improve error text without changing status codes.
- Keep CSV and PDF success responses unchanged.
- Add or update route tests only if existing tests already cover this route.

## Junior Plan 7: Add Contact Detail Back Link

**Related PM Story:** New junior extension
**Outcome:** Users can return from a contact detail page to the contacts list easily.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/[contactId]/page.tsx`
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:87-95`

**High-level plan:**
- Locate the contact detail page.
- Add a simple back link to `/crm/contacts`.
- Keep route parameters and data loading unchanged.
- Use existing button/link styling patterns.

## Junior Plan 8: Add Contact Email Mailto Link

**Related PM Story:** New junior extension
**Outcome:** Users can click a contact email address to start an email.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:100-107`

**High-level plan:**
- Update the email column cell renderer.
- Render plain empty text when no email exists.
- Render a `mailto:` link when an email exists.
- Preserve sorting and column visibility settings.

## Junior Plan 9: Add Mobile-Friendly Contact Column Defaults

**Related PM Story:** New junior extension
**Outcome:** The contacts table is less crowded on smaller screens.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74`
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`

**High-level plan:**
- Identify columns that are lower priority on mobile.
- Set initial column visibility in the table state if supported by the current table setup.
- Keep users able to restore hidden columns through existing view options.
- Do not remove data from the server response.

## Junior Plan 10: Add Contact Row Status Badge

**Related PM Story:** New junior extension
**Outcome:** Active and inactive contacts are easier to scan.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:117-127`
- `components/ui/badge.tsx`

**High-level plan:**
- Replace plain status text with the existing badge component.
- Keep the true/false mapping from the current status column.
- Use restrained variants that match existing UI.
- Do not change the underlying `status` boolean.

## Senior High-Level Plans

## Senior Plan 1: Add Structured Contact Notes

**Related PM Story:** Story 20
**Outcome:** Contact notes store text, author, timestamp, and contact relationship.
**Likely code paths:**
- `prisma/schema.prisma:446-504`
- `prisma/schema.prisma:922-987`
- `lib/authz/scopes/crm.ts:102-108`

**High-level plan:**
- Add a dedicated contact note model.
- Relate notes to contacts and users.
- Update contact detail data loading to include structured notes.
- Enforce contact write access before note creation.
- Plan migration behavior for existing `notes String[]`.

## Senior Plan 2: Audit Contact Note Changes

**Related PM Story:** Story 21
**Outcome:** Note changes appear in audit history.
**Likely code paths:**
- `actions/crm/contacts/create-contact.ts:85-91`
- `lib/audit-log.ts`
- `prisma/schema.prisma:989-1004`

**High-level plan:**
- Reuse the existing audit log helper.
- Write audit entries after successful note mutations.
- Include contact id, acting user, and safe change metadata.
- Add tests around note creation audit behavior.

## Senior Plan 3: Add Saved Contacts Table Views

**Related PM Story:** Story 22
**Outcome:** Users can save and restore contacts table filters and visible columns.
**Likely code paths:**
- `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74`
- `app/[locale]/(routes)/crm/contacts/table-components/data-table-toolbar.tsx`
- `prisma/schema.prisma:922-987`

**High-level plan:**
- Design a saved-view model scoped to a user.
- Store table state such as filters, sorting, and column visibility.
- Add UI controls to save, select, and apply views.
- Keep the first version user-private unless sharing is explicitly designed.

## Senior Plan 4: Add Contact Import Validation Preview

**Related PM Story:** Story 23
**Outcome:** Users can preview invalid contact import rows before writing data.
**Likely code paths:**
- `actions/crm/contacts/create-contact.ts:9-99`
- `prisma/schema.prisma:446-504`
- `lib/authz/session.ts:11-23`

**High-level plan:**
- Build a preview-only import path that validates rows without creating contacts.
- Reuse contact field rules where practical.
- Return row-level validation results.
- Add a separate confirmation path for actual import.
- Ensure preview does not trigger audit logs, emails, Inngest events, or revalidation.

## Senior Plan 5: Add Role-Aware Report Export Coverage

**Related PM Story:** Story 24
**Outcome:** Report export permissions are protected by automated tests.
**Likely code paths:**
- `app/api/reports/export/route.ts:73-127`
- `jest.config.ts:3-15`
- `lib/authz/session.ts:11-23`

**High-level plan:**
- Add route tests for basic user, manager/admin, invalid category, and format branches.
- Mock authentication and report data dependencies.
- Assert both status codes and response body/header behavior.
- Keep tests focused on route policy rather than report rendering internals.

## Senior Plan 6: Design a Contact Activity Timeline

**Related PM Story:** Story 25
**Outcome:** Contact detail shows notes, activities, enrichment, and audit events in one timeline.
**Likely code paths:**
- `prisma/schema.prisma:115-133`
- `prisma/schema.prisma:613-652`
- `prisma/schema.prisma:989-1004`
- `lib/authz/scopes/crm.ts:91-108`

**High-level plan:**
- Define timeline event types and ordering rules.
- Aggregate events from notes, activities, enrichment records, and audit logs.
- Apply contact read scope before returning timeline data.
- Keep the UI visually distinct by event type.
- Add tests for timeline aggregation and authorization.

## Senior Plan 7: Add Contact Duplicate Detection

**Related PM Story:** New senior extension
**Outcome:** Users get warned before creating likely duplicate contacts.
**Likely code paths:**
- `actions/crm/contacts/create-contact.ts:33-62`
- `prisma/schema.prisma:459-468`
- `components/crm/similar-records-drawer.tsx:16-33`

**High-level plan:**
- Define duplicate heuristics, starting with email and name.
- Check for likely matches before contact creation.
- Decide whether duplicate detection blocks save or only warns.
- Reuse existing similarity UI patterns where appropriate.
- Add tests for duplicate detection rules.

## Senior Plan 8: Add Contact Ownership Transfer Audit

**Related PM Story:** New senior extension
**Outcome:** Assignment changes are tracked clearly for accountability.
**Likely code paths:**
- `actions/crm/contacts/update-contact.ts:50-76`
- `lib/audit-log.ts`
- `prisma/schema.prisma:450-457`

**High-level plan:**
- Detect when `assigned_to` changes during contact update.
- Add explicit audit metadata for old owner and new owner.
- Preserve the existing generic update audit log behavior.
- Consider whether assignment notification email should be sent on transfer.

## Senior Plan 9: Add Contact Search API

**Related PM Story:** New senior extension
**Outcome:** Contact search can be reused by multiple UI surfaces without loading all contacts.
**Likely code paths:**
- `actions/crm/get-contacts.ts:9-60`
- `lib/authz/scopes/crm.ts:289-302`
- `prisma/schema.prisma:459-468`

**High-level plan:**
- Create a scoped server search function or API route.
- Search across name, email, and phone fields.
- Apply `contactReadScopeWhere()` before returning results.
- Limit result count for performance.
- Add tests for scoping and search matching.

## Senior Plan 10: Add Contact Data Quality Dashboard Card

**Related PM Story:** New senior extension
**Outcome:** Managers can see how complete contact records are.
**Likely code paths:**
- `app/[locale]/(routes)/page.tsx:60-192`
- `actions/dashboard/get-contacts-count.ts:1-6`
- `prisma/schema.prisma:459-468`

**High-level plan:**
- Define a contact completeness metric.
- Add a dashboard action that counts contacts missing important fields.
- Render a new dashboard card or extend an existing contacts card.
- Keep dashboard fetches parallel if Story 18 is implemented.
- Add tests for the metric query.

