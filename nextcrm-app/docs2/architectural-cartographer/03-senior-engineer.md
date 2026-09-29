# Senior Engineer Guide

## Architecture critique

Scoring: 1 means fragile, 3 means serviceable with clear risks, 5 means strong.

| Category | Score | Evidence and reasoning |
|---|---:|---|
| Scalability | 3 | Prisma singleton and pooling are reasonable (`lib/prisma.ts:9-18`), but contact enrichment keeps active sessions in process memory and warns about multi-replica limits (`app/api/crm/contacts/enrich/route.ts:20-22`). |
| TypeScript discipline | 3 | `strict` is enabled (`tsconfig.json:9-12`), but core contact writes use `as any` (`actions/crm/contacts/create-contact.ts:47-62`, `actions/crm/contacts/update-contact.ts:52-66`) and contacts table row type is misnamed (`app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`). |
| Separation of concerns | 3 | Pages, views, actions, authz, and Prisma are mostly separated (`app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, `actions/crm/get-contacts.ts:9-60`, `lib/authz/session.ts:11-23`), but contact server actions mix validation, persistence, email, audit, events, and cache revalidation (`actions/crm/contacts/create-contact.ts:33-94`). |
| Testability | 4 | Jest and Playwright are configured (`jest.config.ts:3-15`, `playwright.config.ts:15-83`), and many authz/report/enrichment tests exist in the inspected tree. Some risky contact server action behavior remains loosely typed and deserves unit tests. |
| Maintainability | 3 | App Router structure is understandable, but legacy naming (`cratedAt`, `crate_by_user`, `Opportunity` for contacts) increases mental load (`prisma/schema.prisma:454-482`, `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`). |
| Security posture | 3 | API routes use authz helpers (`app/api/reports/export/route.ts:73-96`), and scopes exist (`lib/authz/scopes/crm.ts:289-302`), but contact update writes by id without an explicit scoped write helper in that action (`actions/crm/contacts/update-contact.ts:51-66`). |
| Performance | 3 | `getAllCrmData()` batches queries (`actions/crm/get-crm-data.ts:6-43`), while dashboard counts are awaited sequentially (`app/[locale]/(routes)/page.tsx:60-74`). |

## Performance audit

Finding 1: Dashboard independent awaits are serial.

Evidence: `app/[locale]/(routes)/page.tsx:60-74` awaits each count and revenue call one after another.

Corrected snippet:

```ts
// Based on app/[locale]/(routes)/page.tsx:60-74
const [
  leads,
  tasks,
  invoices,
  campaigns,
  targets,
  storage,
  projects,
  contacts,
  contracts,
  users,
  accounts,
  revenue,
  documents,
  opportunities,
  usersTasks,
] = await Promise.all([
  getLeadsCount(),
  getTasksCount(),
  getInvoicesCount(),
  getCampaignsCount(),
  getTargetsCount(),
  getStorageSize(),
  getBoardsCount(),
  getContactCount(),
  getContractsCount(),
  getActiveUsersCount(),
  getAccountsCount(),
  getExpectedRevenue(displayCurrency),
  getDocumentsCount(),
  getOpportunitiesCount(),
  getUsersTasksCount(userId),
]);
```

Finding 2: Contact table renders all rows into client-side TanStack state.

Evidence: `ContactsDataTable` receives `data: TData[]` and uses client-side row models at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:34-74`. This is fine for small teams, but large CRM datasets need server pagination or virtualized rows.

Corrected direction:

```ts
// Shape to introduce near actions/crm/get-contacts.ts:18-58
export async function getContactsPage(input: {
  page: number;             // UI asks for one page, not the whole CRM.
  pageSize: number;
  search?: string;
}) {
  return prismadb.crm_Contacts.findMany({
    where: { deletedAt: null /* plus contactReadScopeWhere(user) */ },
    skip: input.page * input.pageSize,
    take: input.pageSize,
    orderBy: { created_on: "desc" },
  });
}
```

## Security audit

Finding 1: Contact update action should use a scoped write check.

Evidence: `updateContact()` reads session at `actions/crm/contacts/update-contact.ts:33-36`, then updates by `where: { id }` at `actions/crm/contacts/update-contact.ts:50-66`. The repo already has `assertCanWriteContact()` at `lib/authz/scopes/crm.ts:102-108`.

Corrected snippet:

```ts
// Apply before actions/crm/contacts/update-contact.ts:50-66
import { requireAuthenticated, assertCanWriteContact } from "@/lib/authz";

const user = await requireAuthenticated();
await assertCanWriteContact(user, id);       // Blocks users who do not own or manage this contact.

const contact = await prismadb.crm_Contacts.update({
  where: { id },
  data: { updatedBy: user.id, ...safeData },
});
```

Finding 2: In-memory cancellation is not multi-replica safe.

Evidence: `activeSessions` is a process-local `Map`, and the code comment says to use Redis for multi-replica deployments at `app/api/crm/contacts/enrich/route.ts:20-22`. If the next request hits another server instance, cancellation cannot find the session.

Corrected direction:

```ts
// Based on app/api/crm/contacts/enrich/route.ts:20-22
// Store sessionId -> enrichmentId in Redis, and store cancellation status in the database.
// The streaming worker should check cancellation status between agent steps.
```

Finding 3: Error handling can reveal implementation inconsistency.

Evidence: `app/api/crm/targets/[id]/contacts/route.ts:36-38` returns plain text for validation failure, while other APIs use JSON errors such as `app/api/reports/export/route.ts:86-87`. Standard JSON errors are easier for clients and tests.

Corrected snippet:

```ts
// Based on app/api/crm/targets/[id]/contacts/route.ts:36-38
if (!name && !email) {
  return NextResponse.json({ error: "name or email required" }, { status: 400 });
}
```

## TypeScript discipline review

Keep:

- Generic action hook pattern at `hooks/use-action.ts:5-63`.
- Zod-driven form type inference at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75`.
- Prisma-generated parameter extraction in authz scopes at `lib/authz/scopes/crm.ts:5-10`.

Fix:

- Replace contact form `contactType` string ids with actual database `crm_Contact_Types.id` values. The database relation expects UUID at `prisma/schema.prisma:476-477`, while the form defines `"Customer"`, `"Partner"`, and `"Vendor"` at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:104-108`.
- Rename `opportunitySchema` and `Opportunity` in the contacts table schema to `contactRowSchema` and `ContactRow`; current mismatch is `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.
- Remove `any[]` from `ContactsViewProps.data` at `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`.

## Custom abstractions inventory

- `prismadb`: shared Prisma singleton with Postgres adapter and dev cache (`lib/prisma.ts:1-46`).
- `getSession`: Better Auth session wrapper around request headers (`lib/auth-server.ts:7-11`).
- `requireAuthenticated` and `requireRole`: authz entry points (`lib/authz/session.ts:11-33`).
- CRM scope helpers: contact, lead, opportunity, target, document, campaign, board, and task scope helpers in `lib/authz/scopes/crm.ts`; contact read scope is `lib/authz/scopes/crm.ts:289-302`.
- `serializeDecimals` and `serializeDecimalsList`: server-client boundary helpers (`lib/serialize-decimals.ts:5-25`).
- `useAction`: client state machine for safe actions (`hooks/use-action.ts:15-63`).
- `useDebounce`: timer-based value stabilization (`hooks/useDebounce.tsx:3-14`).
- `getAllCrmData`: batched lookup/data aggregation for CRM views (`actions/crm/get-crm-data.ts:5-69`).

## Testing assessment and one runnable test

Current setup: Jest runs `**/__tests__/**/*.test.ts[x]` with `ts-jest` at `jest.config.ts:3-15`. Playwright uses `tests/`, a setup project, browser projects, and a dev server command at `playwright.config.ts:15-83`.

Risky behavior: the report export route rejects basic users requesting user reports. That is already a route-level pattern and demonstrates how to unit-test route security.

Runnable test to add:

```ts
// app/api/reports/export/__tests__/users-forbidden.test.ts
jest.mock("@/lib/authz", () => {
  const actual = jest.requireActual("@/lib/authz");
  return {
    ...actual,
    requireAuthenticated: jest.fn(),
    getReportScope: jest.fn(() => ({ account: {}, contact: {}, lead: {}, opportunity: {}, campaign: {} })),
  };
});

import { NextRequest } from "next/server";
import { GET } from "../route";
import { requireAuthenticated } from "@/lib/authz";

it("returns 403 when a basic user exports users report", async () => {
  (requireAuthenticated as jest.Mock).mockResolvedValue({ id: "u1", role: "user" });
  const req = new NextRequest("http://localhost:3000/api/reports/export?category=users&format=csv");

  const res = await GET(req);

  expect(res.status).toBe(403);
  await expect(res.json()).resolves.toEqual({ error: "Forbidden" });
});
```

This test targets the branch at `app/api/reports/export/route.ts:90-92`.

## Bug injection exercise

Do not change production code. Use these as scenario tests.

1. Symptom: A user can edit another user's contact by guessing an id. Test scenario: create two users and a contact assigned to user A, authenticate as user B, call the update action, and expect authorization failure. Relevant code: `actions/crm/contacts/update-contact.ts:33-66`, `lib/authz/scopes/crm.ts:102-108`.
2. Symptom: Newly created contact type is not persisted. Test scenario: submit the contact form with type and verify `contact_type_id` is a UUID. Relevant code: `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:104-112`, `prisma/schema.prisma:476-477`.
3. Symptom: Dashboard loads slowly under database latency. Test scenario: mock count calls with delay and measure serial vs parallel. Relevant code: `app/[locale]/(routes)/page.tsx:60-74`.
4. Symptom: Cancel enrichment works locally but not in production behind multiple instances. Test scenario: simulate POST on one process and DELETE on another. Relevant code: `app/api/crm/contacts/enrich/route.ts:20-22`, `app/api/crm/contacts/enrich/route.ts:144-179`.
5. Symptom: Contact table crashes when a row has a shape mismatch. Test scenario: pass row data missing `last_name` and expect Zod parse failure. Relevant code: `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:38-44`.

## Git history learning exercise

Realistic commit messages and what they imply:

1. `feat(crm): add scoped contact reads`
   - Look for new calls to `contactReadScopeWhere()` in read actions such as `actions/crm/get-contacts.ts:18-20`.
2. `fix(authz): block user report exports for basic users`
   - Look for route-level role checks like `app/api/reports/export/route.ts:90-92`.
3. `perf(dashboard): parallelize dashboard metrics`
   - Expect changes around sequential awaits in `app/[locale]/(routes)/page.tsx:60-74`.
4. `refactor(crm): replace contact row any with zod-inferred type`
   - Expect changes to `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35` and `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.
5. `fix(enrichment): persist cancellation state for multi-instance deployments`
   - Expect changes around `activeSessions` in `app/api/crm/contacts/enrich/route.ts:20-22` and cancellation at `app/api/crm/contacts/enrich/route.ts:153-178`.

## If I owned this codebase

| Item | Effort | Impact | Evidence |
|---|---:|---:|---|
| Enforce scoped writes for all CRM mutations | M | High | `updateContact()` updates by id at `actions/crm/contacts/update-contact.ts:51-66`; helpers exist at `lib/authz/scopes/crm.ts:91-108`. |
| Rename contact row schema/type and remove `any[]` | S | Medium | Misnamed `opportunitySchema` is at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`; `any[]` prop is at `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`. |
| Parallelize dashboard metrics | S | Medium | Sequential awaits are at `app/[locale]/(routes)/page.tsx:60-74`; `Promise.all` pattern exists at `actions/crm/get-crm-data.ts:6-43`. |
| Standardize API error JSON | S | Medium | Plain text error appears at `app/api/crm/targets/[id]/contacts/route.ts:36-38`; JSON pattern appears at `app/api/reports/export/route.ts:86-87`. |
| Move enrichment session/cancel state out of process memory | L | High | In-memory map warning is at `app/api/crm/contacts/enrich/route.ts:20-22`. |
| Introduce server pagination for large CRM tables | L | High | Client table receives full `data` array at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:34-42`. |
| Normalize contact create/update input mapping | M | High | `as any` writes are at `actions/crm/contacts/create-contact.ts:47-62` and `actions/crm/contacts/update-contact.ts:52-66`. |

