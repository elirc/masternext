# Tier 1: Junior Missions

### Mission 1: The App's Heartbeat

**Tier:** Junior
**Time Estimate:** 25 minutes
**Goal:** Identify how this app installs, runs, builds, seeds, and tests.
**The Concept:** A codebase heartbeat is the set of commands that keeps the CRM alive: install dependencies, generate Prisma, migrate the database, seed lookup data, run the app, and run tests.
**Design Intent Before You Read the Code:** `package.json` owns scripts and engines; README owns human setup steps; Docker owns dependency orchestration. Poor setup creates onboarding friction and fake bugs.
**Find It In The Code:** Open `package.json:5-24`, `README.md:287-324`, and `docker-compose.yml:61-123`.

```json
// package.json:5-24
"engines": {
  "node": ">=22.12.0",             // Runtime expectation.
  "pnpm": ">=9.0.0"                // Package manager expectation.
},
"scripts": {
  "dev": "next dev",               // Local developer server.
  "build": "prisma generate && prisma migrate deploy && next build",
  "test": "jest",                  // Unit/integration tests through Jest.
  "test:e2e": "playwright test"    // Browser tests through Playwright.
},
"prisma": {
  "seed": "ts-node ./prisma/seeds/seed.ts" // Seed entrypoint.
}
```

**The Aha Moment:** Before you debug app behavior, prove the heartbeat commands and environment are coherent.
**Socratic Checkpoint:**
1. Which command starts the dev server?
2. Which command runs Prisma before build?
3. Where is the seed script defined?
4. Which runtime versions are expected?
5. What does Docker provide that manual setup does not?

How to self-grade: Strong answers cite `package.json:5-24`, `README.md:313-324`, and `docker-compose.yml:61-123`, and explain why database setup is part of app setup.
**Connects To:** Mission 2, because scripts are useful only after you can map the folders they operate on.

### Mission 2: The Folder Mental Map

**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Build a first folder-level map of frontend, backend, data, auth, and tests.
**The Concept:** A CRM office has rooms. `app/` is the front office, `actions/` is staff work, `lib/` is infrastructure, and `prisma/` is the filing cabinet.
**Design Intent Before You Read the Code:** Folders should reveal ownership. If they do not, you slow down and follow imports.
**Find It In The Code:** Open `app/[locale]/(routes)/layout.tsx:5-14`, `actions/crm/get-contacts.ts:1-7`, `lib/prisma.ts:1-4`, `prisma/schema.prisma:4-10`, `jest.config.ts:3-15`, and `playwright.config.ts:15-83`.

```ts
// app/[locale]/(routes)/layout.tsx:5-14
import Header from "./components/Header";          // App shell UI.
import Footer from "./components/Footer";
import { AppSidebar } from "./components/app-sidebar";
import { AvatarProvider } from "@/context/avatar-context";  // Cross-app state.
import { CurrencyProvider } from "@/context/currency-context";
import { getEnabledCurrencies, getDefaultCurrency } from "@/lib/currency"; // Server helper.
```

**The Aha Moment:** Imports are a map of who owns what.
**Socratic Checkpoint:**
1. Which folder contains App Router pages?
2. Which folder contains server actions?
3. Which folder owns the Prisma singleton?
4. Where are UI primitives?
5. Where are E2E tests configured?

How to self-grade: Strong answers cite the folder paths plus `jest.config.ts:3-15` and `playwright.config.ts:15-83`.
**Connects To:** Mission 3 and Mission 4, because folder awareness lets you find types and components.

### Mission 3: TypeScript Is a Contract

**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** See how types define promises between code pieces.
**The Concept:** In a CRM, a contact card has expected fields. TypeScript is the checklist that says which fields must exist before the card can move between desks.
**Design Intent Before You Read the Code:** Form schemas, props, and Prisma fields should describe the same shape. Mismatches cause runtime surprises.
**Find It In The Code:** Open `tsconfig.json:2-34`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75`, `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, and `prisma/schema.prisma:446-504`.

```ts
// app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75
const formSchema = z.object({
  last_name: z.string().min(1, t("lastNameRequired")), // UI-level required field.
  email: z.union([z.string().email(t("emailInvalid")), z.literal("")]).optional(),
  status: z.boolean(),
  assigned_account: z.string().optional(),
});

type NewAccountFormValues = z.infer<typeof formSchema>; // Type comes from validation schema.
```

**The Aha Moment:** The best types remove duplicate truth; weak types create multiple versions of the same contact.
**Socratic Checkpoint:**
1. Is strict TypeScript enabled?
2. Which contact field is required by the form?
3. Which contact field is required by Prisma?
4. What is suspicious about `Opportunity` in the contact schema file?
5. Why is `z.infer` useful?

How to self-grade: Strong answers cite `tsconfig.json:9-12`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75`, `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, and `prisma/schema.prisma:462-464`.
**Connects To:** Mission 6, because props are also typed contracts.

### Mission 4: Your First React Component

**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Read a real Client Component and explain its state and composition.
**The Concept:** `ContactsView` is the reception desk for contact work: it can open a new-contact sheet or show the contact table.
**Design Intent Before You Read the Code:** Client Components own browser interaction. They should receive server data as props and keep UI state local when possible.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/components/ContactsView.tsx:1-94`.

```tsx
// app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90
const ContactsView = ({ data, crmData }: ContactsViewProps) => {
  const [open, setOpen] = useState(false);         // Sheet open/closed state belongs here.
  const t = useTranslations("CrmPage");            // UI text comes from translations.
  const { accounts, contactTypes } = crmData;      // Lookup data was fetched by the server page.

  return (
    <Card>
      <Sheet open={open} onOpenChange={setOpen}>
        <NewContactForm accounts={accounts} onFinish={() => setOpen(false)} />
      </Sheet>
      {!data || data.length === 0
        ? t("contacts.empty")
        : <ContactsDataTable data={data} columns={createColumns(contactTypes)} />}
    </Card>
  );
};
```

**The Aha Moment:** A good component has a small job: own interaction, compose children, pass data down.
**Socratic Checkpoint:**
1. Why is this a Client Component?
2. What state does it own?
3. Which child creates contacts?
4. Which child displays contacts?
5. What data comes from `crmData`?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/components/ContactsView.tsx:1-17`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`.
**Connects To:** Mission 7, because the component receives data fetched before it renders.

### Mission 5: Your First Node.js Route

**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Read a simple API route from auth to response.
**The Concept:** A route handler is the CRM service window: validate the visitor, validate the request, perform work, return a receipt.
**Design Intent Before You Read the Code:** Route handlers should authenticate early, authorize specific records, validate input, write/read data, and return consistent responses.
**Find It In The Code:** Open `app/api/crm/targets/[id]/contacts/route.ts:12-52`.

```ts
// app/api/crm/targets/[id]/contacts/route.ts:12-52
export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id: targetId } = await params;           // Dynamic URL segment.
  const user = await requireAuthenticated();       // Caller must be logged in.
  await assertCanWriteTarget(user, targetId);      // Caller must be allowed to modify this target.

  const { name, email, phone, linkedinUrl } = await request.json();
  if (!name && !email) {
    return new NextResponse("name or email required", { status: 400 });
  }

  const contact = await prismadb.crm_Target_Contact.create({ /* database write */ });
  return NextResponse.json(contact);               // Created row is the response.
}
```

**The Aha Moment:** API routes are contracts, and every contract needs identity, permission, input, work, and output.
**Socratic Checkpoint:**
1. Which URL parameter is used?
2. Which auth helper is called?
3. Which authorization helper is called?
4. What input is minimally required?
5. What database model is created?

How to self-grade: Strong answers cite `app/api/crm/targets/[id]/contacts/route.ts:16-28`, `app/api/crm/targets/[id]/contacts/route.ts:32-52`.
**Connects To:** Mission 12, because larger APIs follow the same contract pattern.

### Mission 6: Props Are a Typed Contract

**Tier:** Junior
**Time Estimate:** 25 minutes
**Goal:** Explain how parent and child components agree on data.
**The Concept:** Props are the handoff sheet between CRM staff. If the sheet says `accounts`, the receiving desk should know exactly what each account contains.
**Design Intent Before You Read the Code:** Props should be narrow and named by domain. Loose props make refactors risky.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/components/ContactsView.tsx:28-36` and `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:31-44`.

```ts
// app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:31-44
type AccountOption = {
  id: string;                                      // Select value.
  name: string;                                    // Select label.
};

type NewContactFormProps = {
  accounts: AccountOption[];                      // Parent must supply options.
  onFinish: () => void;                            // Child can tell parent it is done.
};
```

**The Aha Moment:** A prop type is a promise about what the child can rely on.
**Socratic Checkpoint:**
1. What shape does `NewContactForm` require for accounts?
2. What does `onFinish` allow?
3. Which prop in `ContactsViewProps` is too loose?
4. What type derives CRM lookup data?
5. What would happen if accounts were missing `name`?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/components/ContactsView.tsx:28-36` and `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:31-44`.
**Connects To:** Mission 9, because prop boundaries affect where state should live.

### Mission 7: Following Data Into the App

**Tier:** Junior
**Time Estimate:** 40 minutes
**Goal:** Trace contact list data from page load to table render.
**The Concept:** Data enters the CRM like a stack of contact folders: the server fetches them, the view routes them, and the table displays them.
**Design Intent Before You Read the Code:** Server Components should fetch data close to the route, and Client Components should receive only what they need.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, `actions/crm/get-contacts.ts:9-60`, and `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:39-74`.

```tsx
// app/[locale]/(routes)/crm/contacts/page.tsx:11-21
const AccountsPage = async () => {
  const t = await getTranslations("CrmPage");     // Server-side translations.
  const crmData = await getAllCrmData();           // Lookup lists and CRM reference data.
  const contacts = await getContacts();            // Scoped contact rows.
  return (
    <Container title={t("contacts.pageTitle")} description={t("contacts.pageDescription")}>
      <ContactsView crmData={crmData} data={contacts} />
    </Container>
  );
};
```

**The Aha Moment:** The route is the data-loading boundary for the contacts screen.
**Socratic Checkpoint:**
1. Which function fetches contacts?
2. Which function fetches lookup data?
3. Where are contacts passed into the client?
4. Where does authorization filter the contacts?
5. Where does the table receive the final array?

How to self-grade: Strong answers cite `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, `actions/crm/get-contacts.ts:18-20`, and `app/[locale]/(routes)/crm/components/ContactsView.tsx:84-87`.
**Connects To:** Mission 14, because this is the read half of a full-stack trace.

### Mission 8: Navigation Is the App's Skeleton

**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Understand how the authenticated app shell organizes product modules.
**The Concept:** Navigation is the CRM floor plan. It tells users where accounts, contacts, campaigns, reports, invoices, and documents live.
**Design Intent Before You Read the Code:** App-wide navigation should be built once in the shell and receive translated labels and session context.
**Find It In The Code:** Open `app/[locale]/(routes)/layout.tsx:68-117` and `app/[locale]/(routes)/page.tsx:113-192`.

```tsx
// app/[locale]/(routes)/layout.tsx:68-117
const dict = await getTranslations("ModuleMenu"); // Shell-level labels.
const translations = {
  dashboard: dict("dashboard"),
  crm: { contacts: dict("crm.contacts"), leads: dict("crm.leads") },
  reports: dict("reports"),
  documents: dict("documents"),
  invoices: dict("invoices"),
};

<AppSidebar dict={translations} session={session} /> // Sidebar gets labels and user state.
<Header id={session.user.id as string} lang={session.user.userLanguage as string} />
```

**The Aha Moment:** The app shell gives every module the same authenticated frame.
**Socratic Checkpoint:**
1. Where are module labels translated?
2. Which component receives the sidebar dictionary?
3. What user fields does the header receive?
4. Which dashboard link leads to contacts?
5. What user statuses redirect away from the app shell?

How to self-grade: Strong answers cite `app/[locale]/(routes)/layout.tsx:60-91`, `app/[locale]/(routes)/layout.tsx:109-117`, and `app/[locale]/(routes)/page.tsx:149-154`.
**Connects To:** Mission 13, because navigation sits behind the auth chain.

