# Implementation Plans: Stories 1-9

These first stories build fluency. The pair-programming stance is: I can drive the keyboard, but you should narrate what file we are in, what boundary we are crossing, and what test would prove the behavior.

## Story 1: Dashboard Metric Drill-Down Links

**Primary code paths:**
- `app/[locale]/(routes)/page.tsx`
- `actions/dashboard/get-*-count.ts`
- `app/[locale]/(routes)/components/ui/Container.tsx`

**Implementation plan:**
1. Inspect the dashboard page and identify the component rendering metric cards.
2. Map each card to a route: contacts, accounts, opportunities, invoices, and documents.
3. Use the current locale-aware routing pattern from nearby links or navigation components.
4. Wrap each card in a semantic link or make the existing link area fill the card.
5. Preserve count fetching exactly as-is.
6. Add accessible labels that name both metric and destination.
7. Validate keyboard focus and hover states do not break the card layout.

**Tests and checks:**
- Add or update a component test only if dashboard tests already render the card component.
- Otherwise run lint and manually inspect the route behavior.

**Pairing prompts:**
- Which file owns the route path and which file owns the count?
- Why is this a UI-routing story instead of a data-fetching story?
- What would break if we hard-coded `/crm/contacts` without locale awareness?

**Senior rubric:** A senior keeps the scope tight: no query changes, no new abstraction, and no route strings duplicated if the repo already has a route helper.

## Story 2: Contact Quick Note

**Primary code paths:**
- `app/[locale]/(routes)/crm/contacts/[contactId]/page.tsx`
- `app/[locale]/(routes)/crm/contacts/[contactId]/components/BasicView.tsx`
- `actions/crm/get-contact.ts`
- `app/api/crm/contacts/[id]/route.ts`
- `lib/authz/scopes/crm.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Inspect how contact detail data is loaded and where the page renders editable detail sections.
2. Decide whether the first version uses the existing `notes String[]` field or a new structured note model. For this story, prefer the existing field unless the current UI already expects structured notes.
3. Add a create-note route handler or server action near the current contact mutation pattern.
4. Authenticate the user and call the contact write-scope helper before mutation.
5. Validate note text on the server: trim, require non-empty, set a length limit.
6. Append the note without overwriting existing notes.
7. Add a small client component in the contact detail page for textarea, submit, loading, and error states.
8. On success, refresh route data or update local state from the server response.

**Tests and checks:**
- Route/action test for unauthenticated, forbidden, validation error, and success.
- UI smoke check that empty submit stays local and a server error is visible.

**Pairing prompts:**
- What data crosses from server to client?
- Which authorization check proves the current user can write this contact?
- What is the data migration cost if we later move from `String[]` to note rows?

**Senior rubric:** A senior does not trust the client, avoids swallowing errors, and leaves a clear path to structured notes later.

## Story 3: Account Watcher Digest

**Primary code paths:**
- `actions/crm/accounts/watch-account.ts`
- `actions/crm/accounts/unwatch-account.ts`
- `actions/crm/accounts/get-account-by-id.ts`
- `actions/crm/audit-log/get-audit-log-by-entity.ts`
- `actions/crm/activities/get-activities-by-entity.ts`
- `prisma/schema.prisma`

**Implementation plan:**
1. Trace the account watcher relation in Prisma and the current watch/unwatch actions.
2. Create a read action for the current user's watched accounts.
3. Use `requireAuthenticated()` and the CRM read-scope helpers before returning records.
4. Include recent activity or audit summaries with a strict limit per account.
5. Exclude soft-deleted accounts by default.
6. Add a small digest panel to an existing dashboard or account area.
7. Render empty, loading, and no-recent-updates states distinctly.
8. Keep the first version read-only.

**Tests and checks:**
- Action tests for current user only, soft-deleted exclusion, and scope denial.
- If UI tests are expensive, test the data-shaping helper directly.

**Pairing prompts:**
- What is the difference between "watched by me" and "visible to me"?
- Where should the result limit live?
- How could this query become slow as audit logs grow?

**Senior rubric:** A senior recognizes aggregation risk and limits data early rather than rendering an unbounded digest.

## Story 4: Product Import Validation Preview

**Primary code paths:**
- `app/[locale]/(routes)/crm/products/components/ImportProductsDialog.tsx`
- `actions/crm/targets/import-targets.ts` as an import-flow reference
- `tests/fixtures/products-import.csv`
- `tests/e2e/product-import.spec.ts`
- `actions/crm/products` if present in current checkout

**Implementation plan:**
1. Inspect the existing product import dialog and E2E import tests.
2. Extract validation into a pure helper that accepts parsed rows and returns valid rows plus row-level errors.
3. Ensure the preview path calls validation but performs no writes, audit logs, events, or revalidation.
4. Display a preview table grouped by valid and invalid rows.
5. Add a confirm step that submits only valid rows or blocks until errors are resolved, depending on product decision.
6. Ensure confirmed import reuses the same validation helper.
7. Preserve existing CSV parsing and accepted fixture behavior.

**Tests and checks:**
- Unit tests for validation helper.
- E2E update for preview before confirm if the import test already covers the flow.

**Pairing prompts:**
- Which part is pure validation and which part is mutation?
- How do we prove preview mode writes nothing?
- What should happen if the CSV parser accepts a row the database rejects?

**Senior rubric:** A senior makes the safe path the default and avoids duplicating validation rules between preview and commit.

## Story 5: Opportunity Forecast Confidence

**Primary code paths:**
- `app/[locale]/(routes)/crm/opportunities`
- `actions/crm/get-opportunities*`
- `lib/serialize-decimals.ts`
- `prisma/schema.prisma`
- `__tests__/lib/currency.test.ts` as a money-test style reference

**Implementation plan:**
1. Identify opportunity list columns and the shape returned by opportunity actions.
2. Create a pure forecast helper that accepts stage, probability if available, expected revenue, and close date if useful.
3. Return a small enum-like result such as `low`, `medium`, `high`, or `unknown`.
4. Ensure any Prisma Decimal values returned to client components are serialized with `serializeDecimalsList()`.
5. Render the confidence as a restrained badge in the opportunity table.
6. Use safe fallbacks for missing stage and missing expected revenue.
7. Add tests for the helper with representative stage/revenue inputs.

**Tests and checks:**
- Unit tests for confidence helper.
- Type or runtime check for serialized Decimal payload if the table is client-rendered.

**Pairing prompts:**
- Why should forecast calculation live outside JSX?
- What happens when a Decimal reaches a Client Component un-serialized?
- Is this a business rule, display rule, or both?

**Senior rubric:** A senior separates calculation from rendering and protects the server-client serialization boundary.

## Story 6: Invoice Partial Payment Guardrails

**Primary code paths:**
- `actions/invoices/add-payment.ts`
- `lib/invoices/totals.ts`
- `__tests__/lib/invoices/totals.test.ts`
- `__tests__/invoices/lifecycle.test.ts`
- `app/[locale]/(routes)/invoices/[invoiceId]/components/add-payment-dialog.tsx`

**Implementation plan:**
1. Inspect how invoice totals and current balance are calculated.
2. Add server-side validation before payment creation.
3. Reject zero, negative, non-finite, and over-balance amounts.
4. Use Decimal-safe comparisons if the current code uses Prisma Decimal or decimal.js.
5. Return a structured error compatible with the current dialog.
6. Update the dialog to display the server validation message.
7. Add regression tests for valid partial payment, exact remaining balance, overpayment, zero, and negative amounts.

**Tests and checks:**
- Run invoice unit tests and any payment lifecycle tests.
- Manually inspect UI error state if the dialog is not covered.

**Pairing prompts:**
- Where is the authoritative balance: UI state or server calculation?
- Why is client-side validation helpful but insufficient?
- What should happen with concurrent payments?

**Senior rubric:** A senior treats money validation as a server invariant and thinks about concurrency even if the first patch only closes common cases.

## Story 7: Invoice PDF Regeneration Audit

**Primary code paths:**
- `actions/invoices/regenerate-pdf.ts`
- `lib/invoices/pdf/render.tsx`
- `lib/invoices/storage.ts`
- `lib/audit-log.ts`
- `app/[locale]/(routes)/invoices/[invoiceId]/components/invoice-actions.tsx`

**Implementation plan:**
1. Trace the regenerate action from UI click to PDF render and storage update.
2. Identify the point at which regeneration is definitely successful.
3. Write an audit entry after that success point.
4. Include actor id, invoice id, action name, and safe metadata such as previous file key if available.
5. Ensure errors thrown before success do not write a misleading audit entry.
6. Add a test that mocks successful PDF generation/storage and asserts audit creation.
7. Add a failure test if the action already has mockable failure branches.

**Tests and checks:**
- Focused action test with mocked render/storage/audit dependencies.
- Invoice detail smoke check that regenerate still returns the expected result.

**Pairing prompts:**
- What side effect is the source of truth for success?
- What audit metadata helps without leaking private invoice content?
- Why should the audit happen after storage, not before?

**Senior rubric:** A senior orders side effects deliberately and tests the order where production accountability matters.

## Story 8: Campaign Unsubscribe Analytics

**Primary code paths:**
- `app/api/campaigns/unsubscribe/route.ts`
- `app/api/campaigns/webhooks/resend/route.ts`
- `actions/reports/campaigns.ts`
- `app/[locale]/(routes)/reports/campaigns/page.tsx`
- `__tests__/reports/campaigns.test.ts`
- `__tests__/campaigns/api/unsubscribe.test.ts`

**Implementation plan:**
1. Inspect how unsubscribe events are stored today.
2. Add or expose an unsubscribe count in the campaign report action.
3. Keep the unsubscribe route behavior backward compatible.
4. Update the campaign report page/table to render the count.
5. Use `0` as an explicit display value, not an empty fallback.
6. Add tests for campaigns with unsubscribes, without unsubscribes, and unauthorized report access if not already covered.
7. Verify export paths include or intentionally omit the new field.

**Tests and checks:**
- Campaign report unit test.
- Existing unsubscribe route test.
- Report export test if campaign exports include report columns.

**Pairing prompts:**
- Is unsubscribe count event data, report data, or both?
- What should the report show for older campaigns with no tracked events?
- How do we keep route behavior stable while expanding reporting?

**Senior rubric:** A senior preserves ingestion compatibility and adds reporting without assuming historical data is complete.

## Story 9: Email Sync Health Badge

**Primary code paths:**
- `app/[locale]/(routes)/profile/components/tabs/EmailAccountsTabContent.tsx`
- `app/[locale]/(routes)/profile/components/EmailAccountsList.tsx`
- `actions/emails/accounts.ts`
- `actions/emails/sync.ts`
- `app/[locale]/(routes)/emails/components/account-switcher.tsx`

**Implementation plan:**
1. Inspect email account fields and sync action return values.
2. Define a sync-health mapper: healthy, syncing, failed, disconnected, unknown.
3. Keep provider secrets and tokens out of the returned UI model.
4. Add a badge in the email account list and optionally the account switcher.
5. Display failure summary text only when it is already safe to expose.
6. Handle accounts with no sync history as `unknown` or `not synced yet`.
7. Test the mapper independently.

**Tests and checks:**
- Unit test for health mapper.
- Manual check for list layout with long email addresses.

**Pairing prompts:**
- Which fields are safe to render?
- Should "no history" be failure or unknown?
- Where does operational state become user-facing product state?

**Senior rubric:** A senior protects secrets, avoids false precision, and keeps health classification testable.

