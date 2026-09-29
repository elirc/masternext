# Junior Engineer Guide

## Local setup

The repo declares Node `>=22.12.0` and `pnpm >=9.0.0` in `package.json:5-8`. Development runs with `pnpm run dev`, builds with Prisma generate/migrate plus Next build, and tests run through Jest and Playwright scripts in `package.json:9-20`.

Manual setup reconstructed from existing files:

```sh
# package manager and runtime are specified in package.json:5-8
pnpm install

# env files are documented in README.md:287-311
cp .env.example .env
cp .env.local.example .env.local

# Prisma setup is documented in README.md:313-324
pnpm prisma generate
pnpm prisma migrate deploy
pnpm prisma db seed

# local app command is documented in README.md:326-332
pnpm run dev
```

Docker setup exists and is more complete for dependencies: the README says Docker Compose bundles app, PostgreSQL with pgvector, MinIO, and Inngest at `README.md:336-350`, and `docker-compose.yml` defines those services at `docker-compose.yml:1-68`.

Important environment requirements:

- `DATABASE_URL` is required for Prisma (`.env.example:7-8`).
- Better Auth needs `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL` (`.env.example:10-12`).
- Optional integrations include Resend, OpenAI, Firecrawl, MinIO, SMTP/IMAP, cron secret, and email encryption key (`.env.example:18-54`).
- Docker injects app database, auth, MinIO, Inngest, and placeholder external service values at `docker-compose.yml:69-123`.

## Folder orientation

- `app/`: Next.js App Router pages, layouts, and API route handlers. The locale root is `app/[locale]/layout.tsx:52-74`, authenticated app routes live under `app/[locale]/(routes)/layout.tsx:45-136`, and API handlers live under `app/api/*`, such as `app/api/reports/export/route.ts:73-127`.
- `actions/`: server-side functions used by pages and client components. CRM reads and writes are here, including `actions/crm/get-contacts.ts:9-60` and `actions/crm/contacts/create-contact.ts:9-99`.
- `components/`: shared UI, CRM widgets, report widgets, skeletons, and form pieces. UI primitives include the table components consumed by contacts at `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:19-33`.
- `context/`: React contexts. The app shell mounts `AvatarProvider` and `CurrencyProvider` at `app/[locale]/(routes)/layout.tsx:106-135`.
- `hooks/`: reusable client hooks. `useAction()` manages server action state (`hooks/use-action.ts:15-63`), and `useDebounce()` delays a changing value (`hooks/useDebounce.tsx:3-14`).
- `lib/`: infrastructure and domain helpers such as Prisma (`lib/prisma.ts:1-46`), auth (`lib/auth.ts:12-128`), authz (`lib/authz/session.ts:11-23`), Decimal serialization (`lib/serialize-decimals.ts:5-25`), and report/invoice/enrichment logic.
- `prisma/`: schema, migrations, and seed logic. The schema declares Postgres at `prisma/schema.prisma:4-10`, contact fields at `prisma/schema.prisma:446-504`, and the seed imports CRM lookup data at `prisma/seeds/seed.ts:9-19`.
- `tests/` and `__tests__/`: Playwright E2E tests and Jest tests. Jest config is `jest.config.ts:3-15`; Playwright config starts the dev server at `playwright.config.ts:77-83`.

## One level deeper

Important frontend folders:

- `app/[locale]/(routes)/components/`: app shell navigation and header. The route layout imports `Header`, `Footer`, and `AppSidebar` at `app/[locale]/(routes)/layout.tsx:5-14`.
- `app/[locale]/(routes)/crm/contacts/`: contact page, forms, schemas, columns, table toolbar, and row actions. The page composes `Container`, `ContactsView`, `getContacts`, and `getAllCrmData` at `app/[locale]/(routes)/crm/contacts/page.tsx:1-21`.
- `components/ui/`: reusable design-system primitives. Contacts use `Card`, `Sheet`, `Button`, table primitives, form fields, select, input, and switch in `app/[locale]/(routes)/crm/components/ContactsView.tsx:7-26` and `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:8-29`.

Important backend folders:

- `actions/crm/`: server-side CRM reads and mutations. The contact list read is `actions/crm/get-contacts.ts:9-60`; contact create/update are `actions/crm/contacts/create-contact.ts:9-99` and `actions/crm/contacts/update-contact.ts:8-82`.
- `app/api/`: HTTP route handlers. Simple route example: `app/api/crm/targets/[id]/contacts/route.ts:12-52`. Complex streaming example: `app/api/crm/contacts/enrich/route.ts:24-140`.
- `lib/authz/`: role and scope helpers. `requireAuthenticated()` creates a compact authz user at `lib/authz/session.ts:11-23`, while `contactReadScopeWhere()` builds CRM contact visibility filters at `lib/authz/scopes/crm.ts:289-302`.

## Frontend entry point walkthrough

```tsx
// app/[locale]/layout.tsx:52-74
export default async function RootLayout(props: Props) {
  const params = await props.params;              // App Router params are async in this codebase.
  const { locale } = params;                      // Locale controls html lang and translations.
  const { children } = props;                     // Child route tree is rendered inside providers.
  const messages = await getMessages();           // next-intl messages are loaded before render.

  return (
    <html lang={locale} suppressHydrationWarning>
      <body className={inter.className + " min-h-screen"}>
        <NextIntlClientProvider locale={locale} messages={messages}>
          <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
            {children}                            // Actual pages mount here.
          </ThemeProvider>
        </NextIntlClientProvider>
        <Toaster />                               // Toast systems are global UI services.
        <SonnerToaster />
      </body>
    </html>
  );
}
```

## Backend entry point walkthrough

```ts
// app/api/reports/export/route.ts:73-127
export async function GET(request: NextRequest) {
  let user;
  try {
    user = await requireAuthenticated();          // Authenticate before reading report data.
  } catch (e) {
    if (e instanceof AuthenticationError) return unauthorizedResponse();
    throw e;
  }

  const { searchParams } = request.nextUrl;       // URL is the API contract input.
  const category = searchParams.get("category") ?? "sales";
  const format = searchParams.get("format") ?? "csv";

  if (!REPORT_CATEGORIES.includes(category as ReportCategory)) {
    return NextResponse.json({ error: "Invalid category" }, { status: 400 });
  }

  if (category === "users" && user.role === "user") {
    return forbiddenResponse();                   // Role-specific rule.
  }

  const filters = parseSearchParamsToFilters(searchParams);
  const scope = getReportScope(user);             // Visibility scope travels to data functions.

  if (format === "csv") {                         // Response shape depends on requested format.
    const { data, headers } = await getReportData(category, filters, scope);
    const csv = generateCSV(data, headers);
    return new Response(csv, {
      headers: {
        "Content-Type": "text/csv",
        "Content-Disposition": `attachment; filename="${category}-report.csv"`,
      },
    });
  }

  return NextResponse.json({ error: "Unknown format" }, { status: 400 });
}
```

## TypeScript orientation

- Props define component contracts. `ContactsViewProps` says the component expects `data`, `crmData`, and optional `accountId` at `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-36`.
- Inferred types reduce duplication. `NewContactForm` defines `formSchema` and derives `NewAccountFormValues` with `z.infer` at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75`.
- Generic hooks preserve input/output types. `useAction<TInput, TOutput>()` accepts a typed action and exposes typed `execute`, `data`, `fieldErrors`, and loading state at `hooks/use-action.ts:5-63`.
- Gaps matter. The contacts table uses `Opportunity` as the row type for contact data at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, and `ContactsViewProps.data` is `any[]` at `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`.

## React component anatomy 1: simple dashboard card

```tsx
// app/[locale]/(routes)/page.tsx:200-224
const DashboardCard = ({ href, title, IconComponent, content }: {
  href?: string;                                  // Optional link target.
  title: string;                                  // Display label from translations.
  IconComponent: any;                             // Icon type is loose here; could be improved.
  content: number;                                // Count displayed in the card.
}) => (
  <Link href={href || "#"}>                       // Card is navigable.
    <Suspense fallback={<LoadingBox />}>          // Local fallback, though content is already awaited.
      <Card>
        <CardHeader>
          <CardTitle>{title}</CardTitle>
          <IconComponent className="w-4 h-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-medium">{content}</div>
        </CardContent>
      </Card>
    </Suspense>
  </Link>
);
```

## React component anatomy 2: contacts view

```tsx
// app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90
const ContactsView = ({ data, crmData }: ContactsViewProps) => {
  const [open, setOpen] = useState(false);        // Local UI state for the add-contact sheet.
  const t = useTranslations("CrmPage");           // Client-side translation namespace.
  const { accounts, contactTypes } = crmData;     // Lookup data from server page.

  return (
    <Card>
      <Sheet open={open} onOpenChange={setOpen}>
        <SheetTrigger asChild>
          <Button data-testid="add-contact-btn">+</Button>
        </SheetTrigger>
        <SheetContent>
          <NewContactForm
            accounts={accounts}                   // Form needs account choices.
            onFinish={() => setOpen(false)}       // Successful create closes the sheet.
          />
        </SheetContent>
      </Sheet>

      {!data || data.length === 0
        ? t("contacts.empty")                     // Empty state.
        : <ContactsDataTable data={data} columns={createColumns(contactTypes)} />}
    </Card>
  );
};
```

## React component anatomy 3: contacts table state

```tsx
// app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-74
export function ContactsDataTable<TData, TValue>({ columns, data }: DataTableProps<TData, TValue>) {
  const [rowSelection, setRowSelection] = React.useState({});
  const [columnVisibility, setColumnVisibility] = React.useState<VisibilityState>({});
  const [columnFilters, setColumnFilters] = React.useState<ColumnFiltersState>([]);
  const [sorting, setSorting] = React.useState<SortingState>([]);

  const table = useReactTable({
    data,
    columns,
    state: { sorting, columnVisibility, rowSelection, columnFilters },
    enableRowSelection: true,                     // Enables bulk enrichment selection.
    onRowSelectionChange: setRowSelection,
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onColumnVisibilityChange: setColumnVisibility,
    getCoreRowModel: getCoreRowModel(),           // TanStack derives visible rows from plugins.
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });
}
```

## Backend anatomy 1: simple query action

```ts
// actions/dashboard/get-contacts-count.ts:1-6
import { prismadb } from "@/lib/prisma";          // Shared Prisma singleton.

export const getContactCount = async () => {
  const data = await prismadb.crm_Contacts.count({
    where: { deletedAt: null },                   // Soft-deleted contacts are excluded.
  });
  return data;                                    // Dashboard expects a number.
};
```

## Backend anatomy 2: scoped contact list

```ts
// actions/crm/get-contacts.ts:9-60
export const getContacts = cache(async () => {
  let user;
  try {
    user = await requireAuthenticated();          // Turn session into { id, role }.
  } catch (e) {
    if (e instanceof AuthenticationError) return [];
    throw e;
  }

  const data = await prismadb.crm_Contacts.findMany({
    where: { ...contactReadScopeWhere(user) },    // Role and ownership filter.
    include: {
      assigned_to_user: { select: { name: true } },
      crate_by_user: { select: { name: true } },
      assigned_accounts: true,
      opportunities: { include: { opportunity: { select: { id: true, name: true } } } },
      documents: { include: { document: { select: { id: true, document_name: true } } } },
    },
  });
  return data;
});
```

## Backend anatomy 3: API route handler

```ts
// app/api/crm/targets/[id]/contacts/route.ts:12-52
export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id: targetId } = await params;          // Dynamic route segment.

  let user = await requireAuthenticated();        // 401 if missing session.
  await assertCanWriteTarget(user, targetId);     // 404-style response if not allowed.

  const { name, email, phone, linkedinUrl } = await request.json();
  if (!name && !email) {
    return new NextResponse("name or email required", { status: 400 });
  }

  const contact = await prismadb.crm_Target_Contact.create({
    data: { targetId, name: name ?? null, email: email ?? null, phone: phone || null, linkedinUrl: linkedinUrl || null, source: "manual", enrichStatus: "PENDING" },
  });

  return NextResponse.json(contact);              // Response is the created target-contact row.
}
```

## Domain glossary

- Account: a company or organization record. Its Prisma model starts at `prisma/schema.prisma:12-40`.
- Contact: a person in the CRM, optionally assigned to a user and account. Its Prisma model is `prisma/schema.prisma:446-504`.
- Lead: a potential sales opportunity, separately modeled at `prisma/schema.prisma:72-114`.
- Opportunity: sales pipeline record; contacts can link to opportunities through `ContactsToOpportunities`, included in contact queries at `actions/crm/get-contacts.ts:35-45`.
- Target: a prospecting/enrichment object; target contact creation uses `crm_Target_Contact` at `app/api/crm/targets/[id]/contacts/route.ts:40-50` and schema fields at `prisma/schema.prisma:155-170`.
- Enrichment: AI-assisted research that fills selected fields for contacts or targets. Contact enrichment records are modeled at `prisma/schema.prisma:115-133` and streamed by `app/api/crm/contacts/enrich/route.ts:68-139`.
- Activity: calls, emails, tasks, or meetings linked to CRM entities; modeled at `prisma/schema.prisma:613-652`.
- Soft delete: rows are hidden using `deletedAt` instead of removed; contact scope includes `deletedAt: null` at `lib/authz/scopes/crm.ts:290-302`.
- Server action: server-side function called from UI, such as `createContact()` at `actions/crm/contacts/create-contact.ts:9-99`.
- Route handler: HTTP endpoint under `app/api`, such as report export at `app/api/reports/export/route.ts:73-127`.

## Junior Socratic checkpoint

1. What does the root layout provide that the authenticated layout does not?
2. Where is the first auth gate for app pages?
3. Which file owns the Prisma singleton?
4. What fields make a contact visible to a normal user?
5. What happens after a contact is created successfully?
6. Why is `serializeDecimalsList()` needed in CRM data?
7. Name one confusing type or naming mismatch in the contact feature.

## How to self-grade

Strong answers should cite `app/[locale]/layout.tsx:61-71`, `app/[locale]/(routes)/layout.tsx:50-67`, `lib/prisma.ts:35-46`, `lib/authz/scopes/crm.ts:289-302`, `actions/crm/contacts/create-contact.ts:85-94`, `lib/serialize-decimals.ts:1-25`, and `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`. The best answers also explain why each boundary exists, not just where it lives.

