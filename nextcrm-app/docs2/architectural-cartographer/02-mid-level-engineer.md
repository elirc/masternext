# Mid-Level Engineer Guide

## Full-stack architecture diagram

```text
Browser
  |
  | renders Client Components and submits forms
  v
Next.js App Router
  |
  | root providers: app/[locale]/layout.tsx:61-71
  | authenticated shell: app/[locale]/(routes)/layout.tsx:50-136
  v
Server Components
  |
  | contacts page fetches CRM lookup data and contacts:
  | app/[locale]/(routes)/crm/contacts/page.tsx:11-21
  v
Server actions and route handlers
  |
  | actions/crm/get-contacts.ts:9-60
  | actions/crm/contacts/create-contact.ts:9-99
  | app/api/reports/export/route.ts:73-127
  v
Auth and authorization helpers
  |
  | lib/auth-server.ts:7-11
  | lib/authz/session.ts:11-23
  | lib/authz/scopes/crm.ts:289-302
  v
Prisma Client
  |
  | lib/prisma.ts:1-46
  v
PostgreSQL
  |
  | prisma/schema.prisma:4-10
  | contact model: prisma/schema.prisma:446-504
```

## TypeScript type system deep dive

TypeScript is used strongly at project settings level: `strict` is enabled, `noEmit` is set, JSX uses `react-jsx`, and `@/*` aliases point to repo root at `tsconfig.json:2-34`. In practice, discipline varies by feature.

Strong pattern:

```ts
// hooks/use-action.ts:5-18
type Action<TInput, TOutput> = (
  data: TInput
) => Promise<ActionState<TInput, TOutput>>;       // Callers decide input/output shapes.

interface UseActionOptions<TOutput> {
  onSuccess?: (data: TOutput) => void;            // Success callback is tied to output type.
  onError?: (error: string) => void;
  onComplete?: () => void;
}

export const useAction = <TInput, TOutput>(
  action: Action<TInput, TOutput>,
  options: UseActionOptions<TOutput> = {}
) => { /* state machine continues at hooks/use-action.ts:19-63 */ };
```

Weak pattern:

```ts
// actions/crm/contacts/create-contact.ts:47-62
const contact = await prismadb.crm_Contacts.create({
  data: {
    v: 0,
    createdBy: userId,
    updatedBy: userId,
    accountsIDs: assigned_account ?? undefined,
    assigned_to: assigned_to ?? undefined,
    contact_type_id: contact_type_id ?? undefined,
    birthday: birthday_day && birthday_month && birthday_year
      ? birthday_day + "/" + birthday_month + "/" + birthday_year
      : null,
    ...rest,
  } as any,                                      // This bypasses Prisma's generated input type.
});
```

Mid-level rule: use `as any` as a flare, not a blanket. Here it hides whether `rest` includes fields Prisma accepts, whether empty strings should be `null`, and whether `contact_type_id` is a UUID or a legacy display value. The database model says `contact_type_id` is a UUID relation field at `prisma/schema.prisma:476-477`, while `NewContactForm` currently uses literal ids `"Customer"`, `"Partner"`, and `"Vendor"` at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:104-108`.

## State management deep dive

The contacts table keeps table behavior local because selection, sorting, visibility, filtering, and pagination belong to that table instance, not the whole app.

```tsx
// app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-74
export function ContactsDataTable<TData, TValue>({ columns, data }: DataTableProps<TData, TValue>) {
  const [rowSelection, setRowSelection] = React.useState({});             // Drives bulk actions.
  const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({});
  const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([]);
  const [sorting, setSorting] = React.useState<SortingState>([]);
  const [bulkEnrichOpen, setBulkEnrichOpen] = React.useState(false);      // Modal state near trigger.

  const table = useReactTable({
    data,
    columns,
    state: { sorting, columnVisibility, rowSelection, columnFilters },
    enableRowSelection: true,
    onRowSelectionChange: setRowSelection,
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onColumnVisibilityChange: setColumnVisibility,
    getCoreRowModel: getCoreRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });
}
```

Contrast with app-wide state. The authenticated layout provides avatar and currency contexts around the entire app shell at `app/[locale]/(routes)/layout.tsx:106-135`; currency is initialized from cookies and database-backed settings at `app/[locale]/(routes)/layout.tsx:93-102`.

## API contract map

- Better Auth endpoint: `app/api/auth/[...all]/route.ts:1-4` exports `GET` and `POST` from Better Auth.
- Report export: `GET /api/reports/export?category=sales&format=csv|pdf` authenticates, validates category, applies role/report scope, and returns CSV/PDF or errors at `app/api/reports/export/route.ts:73-127`.
- Manual target contact creation: `POST /api/crm/targets/[id]/contacts` accepts `name`, `email`, `phone`, and `linkedinUrl`, requires target write access, and creates `crm_Target_Contact` at `app/api/crm/targets/[id]/contacts/route.ts:12-52`.
- Contact enrichment: `POST /api/crm/contacts/enrich` validates `contactId` and `fields`, requires write access, checks API keys, creates an enrichment record, and returns an SSE stream at `app/api/crm/contacts/enrich/route.ts:24-140`.
- Contact enrichment cancellation: `DELETE /api/crm/contacts/enrich?sessionId=...` validates session ownership through stored enrichment id, aborts, updates status, and returns success at `app/api/crm/contacts/enrich/route.ts:144-179`.

## Component interaction map

```text
app/[locale]/(routes)/crm/contacts/page.tsx:11-21
  fetches t, crmData, contacts
  passes crmData and contacts
    |
    v
app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90
  owns add-sheet open state
  passes accounts to NewContactForm
  creates columns with contactTypes
    |
    +--> app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120
    |    validates and calls createContact()
    |
    +--> app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-179
         owns row selection, filters, sorting, bulk enrich modal
         uses columns from:
           app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134
```

## Full-stack feature trace: create a contact

1. The Server Component route loads translations, CRM lookup data, and visible contacts at `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`.
2. `ContactsView` renders a `Sheet` and passes accounts into `NewContactForm` at `app/[locale]/(routes)/crm/components/ContactsView.tsx:56-72`.
3. `NewContactForm` defines the validation schema and form defaults at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-102`.
4. The submit handler maps `type` into `contact_type_id` and calls the server action at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120`.
5. `createContact()` gets a session and returns an unauthorized error if missing at `actions/crm/contacts/create-contact.ts:33-36`.
6. It transforms birthday fields, assignment fields, and remaining form values into a Prisma create at `actions/crm/contacts/create-contact.ts:37-62`.
7. If the contact is assigned to someone else, it looks up that user and sends an email at `actions/crm/contacts/create-contact.ts:64-83`.
8. It writes an audit log, emits `crm/contact.saved` to Inngest, revalidates the contacts page, and returns the contact at `actions/crm/contacts/create-contact.ts:85-94`.
9. The form shows a toast, resets fields, and closes the sheet through `onFinish()` at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:113-119`.
10. On the next render, `getContacts()` reloads scoped contact rows and includes user/account/opportunity/document relations at `actions/crm/get-contacts.ts:18-58`.

## Diff reading exercise

Hypothetical change: "Add a `LinkedIn URL` field to contacts."

Read the diff like this:

1. Database: Does `crm_Contacts` already have a field? It has `social_linkedin` at `prisma/schema.prisma:469-472`, so no migration should be needed for a basic display/edit change.
2. Create form: Does validation include the field? Yes, `social_linkedin` is optional at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:65-68`, and the input exists at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:490-505`.
3. Server action: Does create accept and spread it? Yes, `social_linkedin?: string` is part of input at `actions/crm/contacts/create-contact.ts:24-31`, then `...rest` is written at `actions/crm/contacts/create-contact.ts:47-62`.
4. Table: Is it displayed? No, the contact columns shown at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:37-134` do not include `social_linkedin`.
5. Tests: E2E tests exist for contact update/detail update, listed at `tests/e2e/contact-update.spec.ts` and `tests/e2e/contact-detail-update.spec.ts` from the inspected file tree, but no new test was run for this documentation task.

Review question: if the diff adds a new database field instead of reusing `social_linkedin`, ask why. The app already models social links.

## Non-obvious architectural patterns

- App shell auth instead of global middleware: no root `middleware.ts` was found during inspection. The main page gate is the route layout at `app/[locale]/(routes)/layout.tsx:50-67`, and API routes use helper-based auth at `app/api/reports/export/route.ts:73-96`.
- Soft delete as a product rule: contact queries and scopes filter `deletedAt: null` at `actions/dashboard/get-contacts-count.ts:3-5` and `lib/authz/scopes/crm.ts:290-302`, supported by an index at `prisma/schema.prisma:492-504`.
- Server-side cache for reads: `getContacts()` is wrapped in React `cache()` at `actions/crm/get-contacts.ts:1-9`, which can reduce duplicate work in one render pass.
- Lookup aggregation: `getAllCrmData()` batches multiple Prisma queries through `Promise.all` at `actions/crm/get-crm-data.ts:6-43`, then serializes Decimal-bearing lists at `actions/crm/get-crm-data.ts:45-65`.
- Event-based follow-up work: contact create/update emit `crm/contact.saved` at `actions/crm/contacts/create-contact.ts:92-93` and `actions/crm/contacts/update-contact.ts:75-76`.

## Mid-level Socratic checkpoint

1. Why does `getContacts()` need both `requireAuthenticated()` and `contactReadScopeWhere()`?
2. Where does the create-contact UI convert form state into server action input?
3. What parts of contact creation are synchronous side effects?
4. What would break if `getAllCrmData()` returned raw Decimal contracts to a Client Component?
5. Why is the dashboard performance shape different from `getAllCrmData()`?
6. What API route demonstrates role-specific forbidden responses?
7. Where is table row selection stored, and why is that a reasonable home?
8. What naming mismatch would you fix first before expanding the contacts table?

## How to self-grade

Strong answers cite `actions/crm/get-contacts.ts:9-20`, `lib/authz/scopes/crm.ts:289-302`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120`, `actions/crm/contacts/create-contact.ts:64-94`, `lib/serialize-decimals.ts:1-25`, `app/[locale]/(routes)/page.tsx:60-74`, `actions/crm/get-crm-data.ts:6-43`, `app/api/reports/export/route.ts:86-96`, `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74`, and `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.

