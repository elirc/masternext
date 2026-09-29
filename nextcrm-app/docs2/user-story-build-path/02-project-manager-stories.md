# Project Manager User Stories

These stories are written from a project manager's perspective: they describe product outcomes, acceptance expectations, and the engineering level best suited to own the work. The split is 13 junior-oriented stories and 12 senior-oriented stories.

## Junior Software Engineer Stories

## Story 1: Clarify Empty Contacts Table Copy

**Owner Level:** Junior
**The Story:** As a project manager, I want the contacts table empty state to clearly explain that no contacts are available so that new users know the CRM is not broken.
**Acceptance Criteria:**
- The empty state on the contacts page uses clear, user-friendly language.
- The copy appears only when the contacts list is empty.
- No table behavior changes.

## Story 2: Improve Add Contact Button Label

**Owner Level:** Junior
**The Story:** As a project manager, I want the add-contact action to be easier to recognize so that users can create contacts without guessing what the plus button means.
**Acceptance Criteria:**
- The contact creation trigger has an accessible label.
- The visible UI remains consistent with the existing design.
- Existing `data-testid` behavior remains intact.

## Story 3: Add Contact Position to the Contacts Table

**Owner Level:** Junior
**The Story:** As a project manager, I want users to see each contact's position in the contacts table so that they can quickly identify decision makers.
**Acceptance Criteria:**
- A `Position` column appears in the contacts table.
- Empty positions render cleanly.
- The change uses the existing contact `position` field.

## Story 4: Rename Misleading Contact Table Types

**Owner Level:** Junior
**The Story:** As a project manager, I want internal contact table types to use contact-oriented names so that future engineers can work faster and make fewer mistakes.
**Acceptance Criteria:**
- Contact table schema names no longer refer to opportunities.
- Imports are updated.
- Runtime behavior remains unchanged.

## Story 5: Fix Contacts Page Component Name

**Owner Level:** Junior
**The Story:** As a project manager, I want the contacts page component to be named after contacts so that code reviews are less confusing.
**Acceptance Criteria:**
- The contacts page component name reflects the contacts domain.
- The route still renders correctly.
- No user-facing copy changes.

## Story 6: Show a Friendly Unassigned Label

**Owner Level:** Junior
**The Story:** As a project manager, I want unassigned contacts to show a consistent label so that users understand assignment status at a glance.
**Acceptance Criteria:**
- Contacts without assigned users show `Unassigned`.
- Contacts without assigned accounts show `Unassigned`.
- Existing assigned-user and assigned-account display still works.

## Story 7: Add Loading Text to Contact Submit Button

**Owner Level:** Junior
**The Story:** As a project manager, I want users to see that contact creation is in progress so that they do not submit the form multiple times.
**Acceptance Criteria:**
- The submit button is disabled while submitting.
- The button displays saving text while the form is submitting.
- Existing validation behavior remains unchanged.

## Story 8: Standardize Contact Table Column Labels

**Owner Level:** Junior
**The Story:** As a project manager, I want contact table labels to use polished wording so that the CRM feels professional.
**Acceptance Criteria:**
- `Sure name` is corrected to `Surname` or `Last name`.
- Other contact table labels are reviewed for consistency.
- Sorting and row links still work.

## Story 9: Add a Contact Website Column

**Owner Level:** Junior
**The Story:** As a project manager, I want the contacts table to show website information so that sales users can quickly research contacts.
**Acceptance Criteria:**
- A website column appears in the contacts table.
- Empty websites do not display broken links.
- Existing contact row navigation remains unchanged.

## Story 10: Improve Contact Form Error Visibility

**Owner Level:** Junior
**The Story:** As a project manager, I want server errors in the contact form to be easy to notice so that users know why a save failed.
**Acceptance Criteria:**
- Server errors remain visible near the submit area.
- Error text uses existing destructive/error styling.
- Successful submission still closes the form.

## Story 11: Add Contact Count to Contacts View Header

**Owner Level:** Junior
**The Story:** As a project manager, I want users to see how many contacts are currently shown so that they understand the size of the list.
**Acceptance Criteria:**
- The contacts view displays the number of contacts passed to the table.
- The count updates when the page receives new data.
- Empty state still appears when there are no contacts.

## Story 12: Make Bulk Selection Summary More Helpful

**Owner Level:** Junior
**The Story:** As a project manager, I want selected contacts to be summarized clearly so that users feel confident before starting bulk enrichment.
**Acceptance Criteria:**
- The selected count remains visible.
- The summary includes at least one selected contact name when available.
- The existing bulk enrich modal still opens correctly.

## Story 13: Add User-Friendly Report Export Error Copy

**Owner Level:** Junior
**The Story:** As a project manager, I want invalid report export requests to return clear error messages so that users and support staff can understand what went wrong.
**Acceptance Criteria:**
- Invalid report category errors remain JSON responses.
- Unknown format errors remain JSON responses.
- Error messages are consistent and readable.

## Senior Software Engineer Stories

## Story 14: Enforce Scoped Authorization on Contact Updates

**Owner Level:** Senior
**The Story:** As a project manager, I want contact updates to enforce record-level authorization so that users cannot modify contacts they do not own or manage.
**Acceptance Criteria:**
- Contact update checks authenticated user scope before writing.
- Unauthorized users cannot update another user's contact.
- Tests cover allowed and denied update cases.

## Story 15: Replace Loose Contact Mutation Types

**Owner Level:** Senior
**The Story:** As a project manager, I want contact create and update mutations to avoid unsafe `any` casts so that data bugs are caught before production.
**Acceptance Criteria:**
- Contact create uses a typed Prisma input mapping.
- Contact update uses a typed Prisma input mapping.
- Existing form submissions continue to work.
- Tests or type checks cover the main mutation shape.

## Story 16: Add Server-Side Contacts Pagination

**Owner Level:** Senior
**The Story:** As a project manager, I want contacts to paginate from the server so that large CRM datasets remain fast.
**Acceptance Criteria:**
- Contacts are fetched in pages rather than as one full list.
- Authorization scope is applied on the server.
- Pagination state is reflected in the UI.
- Tests verify scoped pagination behavior.

## Story 17: Move Enrichment Cancellation Out of Process Memory

**Owner Level:** Senior
**The Story:** As a project manager, I want enrichment cancellation to work in multi-instance deployments so that production users can reliably stop long-running enrichment jobs.
**Acceptance Criteria:**
- Cancellation state is stored outside a local in-memory map.
- Cancel requests work even when handled by a different server instance.
- Enrichment status is updated consistently.
- Failure modes are tested or documented.

## Story 18: Parallelize Dashboard Metric Loading

**Owner Level:** Senior
**The Story:** As a project manager, I want the dashboard to load independent metrics in parallel so that users reach the overview faster.
**Acceptance Criteria:**
- Independent dashboard count queries run concurrently.
- Displayed metrics remain unchanged.
- Errors are handled at least as safely as before.
- Performance improvement is measurable in local or test instrumentation.

## Story 19: Standardize API Error Response Shapes

**Owner Level:** Senior
**The Story:** As a project manager, I want API errors to use a consistent JSON format so that frontend code and tests can handle failures predictably.
**Acceptance Criteria:**
- Route handlers return JSON errors for validation failures.
- Existing status codes are preserved unless intentionally changed.
- Tests cover at least one converted endpoint.

## Story 20: Add Structured Contact Notes

**Owner Level:** Senior
**The Story:** As a project manager, I want contact notes to store author and timestamp so that relationship history is accountable.
**Acceptance Criteria:**
- A structured contact note model exists.
- Notes are associated with contacts and users.
- Users can add notes only to contacts they can write.
- Contact detail displays note text, author, and timestamp.

## Story 21: Audit Contact Note Changes

**Owner Level:** Senior
**The Story:** As a project manager, I want contact note changes to appear in audit history so that admins can trace sensitive relationship updates.
**Acceptance Criteria:**
- Creating a note writes an audit log entry.
- Updating or deleting a note writes an audit log entry if those actions exist.
- Audit entries include acting user and contact context.
- Tests verify audit behavior.

## Story 22: Add Saved Contacts Table Views

**Owner Level:** Senior
**The Story:** As a project manager, I want users to save contacts table filters and columns so that repeated workflows are faster.
**Acceptance Criteria:**
- Users can save a named table view.
- Saved views restore filters and visible columns.
- Views are scoped to the current user unless explicitly shared.
- The data model supports future sharing.

## Story 23: Add Contact Import Validation Preview

**Owner Level:** Senior
**The Story:** As a project manager, I want users to preview contact import validation results before committing data so that bad imports do not pollute the CRM.
**Acceptance Criteria:**
- Users can upload or select import data for preview.
- The preview identifies invalid rows and missing required fields.
- No contacts are created during preview.
- A separate confirmation step performs the import.

## Story 24: Add Role-Aware Report Export Coverage

**Owner Level:** Senior
**The Story:** As a project manager, I want report export permissions covered by tests so that sensitive reports are not exposed by future changes.
**Acceptance Criteria:**
- Basic users cannot export restricted user reports.
- Managers or admins can export permitted reports.
- Invalid categories return the expected error.
- CSV and PDF branches are covered where practical.

## Story 25: Design a Contact Activity Timeline

**Owner Level:** Senior
**The Story:** As a project manager, I want contact detail pages to show a unified timeline of notes, activities, enrichment, and audit events so that users can understand relationship history in one place.
**Acceptance Criteria:**
- Timeline design identifies source records and ordering rules.
- The implementation respects contact read scope.
- The UI distinguishes notes, activities, enrichment events, and audit events.
- The data access pattern avoids loading unnecessary full tables.
- Tests cover at least one timeline data aggregation case.

