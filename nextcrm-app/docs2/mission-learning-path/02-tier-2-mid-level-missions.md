# Tier 2: Mid-Level Missions

### Mission 9: State Has a Home and a Reason

**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Decide why contact table state belongs in the table component.
**The Concept:** In the CRM, table state is the temporary arrangement of folders on one desk. It should not become company policy unless multiple desks need it.
**Design Intent Before You Read the Code:** Local UI state belongs near the interaction. Shared app state belongs in providers such as avatar/currency mounted by the app shell.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-74` and `app/[locale]/(routes)/layout.tsx:93-135`.

```tsx
// app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74
const [rowSelection, setRowSelection] = React.useState({});     // Specific to this table.
const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({});
const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([]);
const [sorting, setSorting] = React.useState<SortingState>([]);

const table = useReactTable({
  data,
  columns,
  state: { sorting, columnVisibility, rowSelection, columnFilters },
  onRowSelectionChange: setRowSelection,          // Table library controls state transitions.
});
```

**The Aha Moment:** State belongs at the lowest component that owns the user's intent.
**Socratic Checkpoint:**
1. Which table state would be wrong to put in global context?
2. Which currency state is shared app-wide?
3. Why does row selection live beside the bulk enrichment modal?
4. What would happen if sorting state lived in the route layout?
5. How would URL search params change the state home?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:43-74` and `app/[locale]/(routes)/layout.tsx:93-102`.
**Connects To:** Mission 10 and Mission 14, because state and data flow meet in real features.

### Mission 10: The Custom Hook Ecosystem

**Tier:** Mid-Level
**Time Estimate:** 35 minutes
**Goal:** Learn what custom hooks exist and what gap remains.
**The Concept:** Hooks are reusable habits. In a CRM, `useAction` is the checklist for "submit work and track result"; `useDebounce` is "wait until the search request settles."
**Design Intent Before You Read the Code:** Hooks should hide repeated state machines, not business rules that need server authority.
**Find It In The Code:** Open `hooks/use-action.ts:1-63`, `hooks/useDebounce.tsx:1-16`, and `components/ui/account-search-combobox.tsx:44-52`.

```ts
// hooks/use-action.ts:26-51
const execute = useCallback(async (input: TInput) => {
  setIsLoading(true);                              // Start user-visible pending state.
  try {
    const result = await action(input);            // Run server action contract.
    setFieldErrors(result.fieldErrors);            // Preserve validation errors.
    if (result.error) options.onError?.(result.error);
    if (result.data) options.onSuccess?.(result.data);
  } finally {
    setIsLoading(false);                           // Always end loading.
    options.onComplete?.();
  }
}, [action, options]);
```

**The Aha Moment:** A hook should make the next component simpler without hiding security decisions.
**Socratic Checkpoint:**
1. What state does `useAction` standardize?
2. What state does `useDebounce` standardize?
3. Which hook would you use for a form submit?
4. Which hook would you use for search typing?
5. What would a missing custom hook for table URL state look like?

How to self-grade: Strong answers cite `hooks/use-action.ts:15-63`, `hooks/useDebounce.tsx:3-14`, and one real component using local search state such as `components/ui/account-search-combobox.tsx:44-52`.
**Connects To:** Mission 11, because hooks often wrap side effects.

### Mission 11: Side Effects Are Promises to the System

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Identify all side effects in contact creation.
**The Concept:** Creating a contact is not just putting a card in the CRM. It can notify the owner, write audit history, start enrichment work, and refresh the desk.
**Design Intent Before You Read the Code:** Side effects should be intentional, idempotent when possible, and tested around failure boundaries.
**Find It In The Code:** Open `actions/crm/contacts/create-contact.ts:33-99`.

```ts
// actions/crm/contacts/create-contact.ts:47-94
const contact = await prismadb.crm_Contacts.create({ data: { /* contact fields */ } });

if (assigned_to && assigned_to !== userId) {
  const notifyRecipient = await prismadb.users.findFirst({ where: { id: assigned_to } });
  if (notifyRecipient) await sendEmail({ /* assignment notification */ }); // External side effect.
}

await writeAuditLog({ entityType: "contact", entityId: contact.id, action: "created" }); // History.
void inngest.send({ name: "crm/contact.saved", data: { record_id: contact.id } });       // Background work.
revalidatePath("/[locale]/crm/contacts", "page");                                       // UI cache refresh.
return { data: contact };
```

**The Aha Moment:** The database write is only the center of the workflow, not the whole workflow.
**Socratic Checkpoint:**
1. Which side effects happen after the Prisma create?
2. Which side effect is external email?
3. Which side effect is background processing?
4. Which line refreshes stale UI?
5. What side effect would you test first?

How to self-grade: Strong answers cite `actions/crm/contacts/create-contact.ts:47-94` and name the failure mode of each side effect.
**Connects To:** Mission 14 and Mission 23, because full-stack traces and tests must account for side effects.

### Mission 12: The Full API Contract

**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Read a route handler with validation, role guard, format branching, and streamed file responses.
**The Concept:** Report export is the CRM print room. You choose a report category and output format, then receive a CSV or PDF only if your role allows it.
**Design Intent Before You Read the Code:** A route contract should define accepted inputs, forbidden combinations, output formats, and error shapes.
**Find It In The Code:** Open `app/api/reports/export/route.ts:23-127`.

```ts
// app/api/reports/export/route.ts:73-127
const category = searchParams.get("category") ?? "sales";       // Input default.
const format = searchParams.get("format") ?? "csv";

if (!REPORT_CATEGORIES.includes(category as ReportCategory)) {
  return NextResponse.json({ error: "Invalid category" }, { status: 400 });
}

if (category === "users" && user.role === "user") {
  return forbiddenResponse();                                   // Role guard.
}

if (format === "csv") {
  const { data, headers } = await getReportData(category, filters, scope);
  return new Response(generateCSV(data, headers), { headers: { "Content-Type": "text/csv" } });
}

if (format === "pdf") {
  const { generatePDF } = await import("@/actions/reports/export-pdf"); // Lazy heavy path.
  return new Response(new Uint8Array(buffer), { headers: { "Content-Type": "application/pdf" } });
}
```

**The Aha Moment:** API clarity is about every branch, not just the happy path.
**Socratic Checkpoint:**
1. What are the query params?
2. What category is forbidden for basic users?
3. Where are filters parsed?
4. Why is PDF imported lazily?
5. What happens for unknown format?

How to self-grade: Strong answers cite `app/api/reports/export/route.ts:82-127`.
**Connects To:** Mission 13 and Mission 22, because auth and security are route contract concerns.

### Mission 13: The Middleware Chain

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Understand the repo's auth chain even without root middleware.
**The Concept:** This app does not rely on one hallway guard. It uses guards at the app layout door, API service windows, and record-specific desks.
**Design Intent Before You Read the Code:** If no root `middleware.ts` exists, every protected path needs explicit gates in layouts, handlers, or helpers.
**Find It In The Code:** Open `app/[locale]/(routes)/layout.tsx:50-67`, `lib/auth-server.ts:7-11`, `lib/authz/session.ts:11-23`, and `lib/authz/scopes/crm.ts:91-108`.

```ts
// lib/authz/session.ts:11-23
export async function requireAuthenticated(): Promise<AuthzUser> {
  const session = await getSession();              // Authn from Better Auth.
  const userId = session?.user?.id;
  if (!userId) throw new AuthenticationError();

  const dbUser = await prismadb.users.findUnique({
    where: { id: userId },
    select: { id: true, role: true },              // Reload role from database.
  });
  if (!dbUser) throw new AuthenticationError();

  return { id: dbUser.id, role: mapLegacyRole(dbUser.role) };
}
```

**The Aha Moment:** Middleware is a pattern, not only a file named `middleware.ts`.
**Socratic Checkpoint:**
1. Where are app pages guarded?
2. Where is the Better Auth session read?
3. Why does authz reload user role?
4. What helper checks contact write access?
5. What is the impact of not having root middleware?

How to self-grade: Strong answers cite `app/[locale]/(routes)/layout.tsx:50-67`, `lib/auth-server.ts:7-11`, `lib/authz/session.ts:11-23`, and `lib/authz/scopes/crm.ts:102-108`.
**Connects To:** Mission 14 and Mission 22.

### Mission 14: End-to-End Feature Trace

**Tier:** Mid-Level
**Time Estimate:** 60 minutes
**Goal:** Trace contact creation from button to database to UI refresh.
**The Concept:** A user story travels like a contact moving from reception to records, notifications, audit, and back to the visible table.
**Design Intent Before You Read the Code:** Full-stack tracing should identify UI event, validation, server boundary, auth, data write, side effects, response, and UI update.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/components/ContactsView.tsx:56-72`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120`, `actions/crm/contacts/create-contact.ts:33-94`, and `actions/crm/get-contacts.ts:18-58`.

```tsx
// app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:110-120
const onSubmit = async (data: NewAccountFormValues) => {
  const { type, ...rest } = data;                  // UI field is adapted for backend shape.
  const result = await createContact({ ...rest, contact_type_id: type || undefined });
  if (result?.error) {
    form.setError("root.serverError", { message: result.error });
  } else {
    toast.success(t("createSuccess"));             // Response drives UI feedback.
    form.reset();
    onFinish();                                    // Parent sheet closes.
  }
};
```

**The Aha Moment:** A feature trace is a chain of contracts, and each link can fail differently.
**Socratic Checkpoint:**
1. What user action opens the form?
2. Where is input validated?
3. Where is auth checked?
4. Where is the database write?
5. Where is the page revalidated?

How to self-grade: Strong answers cite all four ranges in "Find It In The Code" and name one failure at each boundary.
**Connects To:** Mission 15 and Mission 23.

### Mission 15: Read the Diff Like an Engineer

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Practice reviewing a realistic contacts table change.
**The Concept:** A diff is a proposed new workflow in the CRM. You ask whether every desk it touches agrees on the new rule.
**Design Intent Before You Read the Code:** A display change might still require schema, action, type, translation, and test review.
**Find It In The Code:** Open `prisma/schema.prisma:446-504`, `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`, and `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.

```tsx
// app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:87-95
{
  accessorKey: "last_name",                        // Must exist in row data and schema.
  header: ({ column }) => (
    <DataTableColumnHeader column={column} title="Sure name" />
  ),
  cell: ({ row }) => (
    <Link href={`/crm/contacts/${row.original.id}`} data-testid="contact-row-name">
      <div>{row.getValue("last_name")}</div>       // User-visible navigation target.
    </Link>
  ),
}
```

**The Aha Moment:** Review the contract, not just the changed lines.
**Socratic Checkpoint:**
1. If a new column is added, what three files might need updates?
2. Where does the row schema live?
3. What typo in the header should a reviewer notice?
4. How does the row link know the contact id?
5. Which model field proves `last_name` exists?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:87-95`, `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, and `prisma/schema.prisma:462-464`.
**Connects To:** Mission 19.

### Mission 16: Composition Over Inheritance

**Tier:** Mid-Level
**Time Estimate:** 35 minutes
**Goal:** See how the app builds screens by composing small parts.
**The Concept:** The CRM page is not one giant clerk. It is a route, a container, a card, a sheet, a form, a table, columns, and row actions.
**Design Intent Before You Read the Code:** Composition keeps ownership visible and makes future feature additions easier.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/contacts/page.tsx:15-23`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:44-90`, and `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:68-118`.

```tsx
// app/[locale]/(routes)/crm/components/ContactsView.tsx:67-87
<NewContactForm accounts={accounts} onFinish={() => setOpen(false)} />
<ContactsDataTable
  data={data}
  columns={createColumns(contactTypes)}            // Columns are injected, not hardcoded in table.
/>
```

**The Aha Moment:** Composition lets each component own one decision.
**Socratic Checkpoint:**
1. Which component owns the page title?
2. Which component owns sheet open state?
3. Which component owns row actions?
4. Which component owns TanStack table state?
5. Where are columns created?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/contacts/page.tsx:15-23`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`, `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-74`, and `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:68-118`.
**Connects To:** Mission 18.

### Mission 17: TypeScript's Hidden Work

**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Notice where TypeScript protects you and where it is bypassed.
**The Concept:** TypeScript is the CRM form validation clerk. When someone writes `as any`, they let a folder skip the desk.
**Design Intent Before You Read the Code:** Generated Prisma types, Zod inference, and generic hooks reduce risk. `any` should be temporary and explained.
**Find It In The Code:** Open `hooks/use-action.ts:5-63`, `actions/crm/contacts/create-contact.ts:47-62`, `actions/crm/contacts/update-contact.ts:52-66`, and `lib/authz/scopes/crm.ts:5-10`.

```ts
// lib/authz/scopes/crm.ts:5-10
type ContactWhere = NonNullable<
  Parameters<typeof prismadb.crm_Contacts.updateMany>[0]
>["where"];                                        // Type extracted from Prisma API.

// This means scope helpers track Prisma's expected where shape.
```

**The Aha Moment:** TypeScript quietly carries architecture decisions when you let it.
**Socratic Checkpoint:**
1. Where is Prisma's update type reused?
2. Where is TypeScript bypassed in contact create?
3. Where is TypeScript bypassed in contact update?
4. Where does Zod create a form type?
5. What risk does `any[]` create in `ContactsView`?

How to self-grade: Strong answers cite `lib/authz/scopes/crm.ts:5-10`, `actions/crm/contacts/create-contact.ts:47-62`, `actions/crm/contacts/update-contact.ts:52-66`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75`, and `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`.
**Connects To:** Mission 19 and Mission 23.

