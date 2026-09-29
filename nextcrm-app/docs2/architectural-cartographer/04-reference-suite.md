# Reference Suite

## Doc 1: Junior Onboarding Guide

Day 1 goal: run the app, trace one feature, and explain one database-backed page.

1. Confirm toolchain from `package.json:5-8`.
2. Install dependencies with `pnpm install`; scripts are listed at `package.json:9-20`.
3. Use Docker if you want bundled Postgres, MinIO, and Inngest; README documents that at `README.md:336-350`, and services are defined in `docker-compose.yml:1-68`.
4. For manual setup, copy `.env.example` and `.env.local.example` as documented at `README.md:287-295`.
5. Configure required database and auth variables from `.env.example:7-12`.
6. Generate/migrate/seed Prisma using the README commands at `README.md:313-324`.
7. Start dev with `pnpm run dev` from `README.md:326-332`.
8. Read the root layout (`app/[locale]/layout.tsx:52-74`), app shell (`app/[locale]/(routes)/layout.tsx:45-136`), contacts page (`app/[locale]/(routes)/crm/contacts/page.tsx:11-27`), and contact action (`actions/crm/contacts/create-contact.ts:9-99`).

## Doc 2: Mid-Level Architecture Guide

System design:

- Routing is App Router and locale-based: root layout is `app/[locale]/layout.tsx:52-74`.
- Authenticated app pages are inside `(routes)`, with session and user-status redirects in `app/[locale]/(routes)/layout.tsx:50-67`.
- API routes live under `app/api`, with Better Auth delegated by `app/api/auth/[...all]/route.ts:1-4` and hand-written handlers such as `app/api/reports/export/route.ts:73-127`.
- Database access goes through `prismadb` from `lib/prisma.ts:1-46`.
- Prisma models define the domain; contact fields and relationships are `prisma/schema.prisma:446-504`.
- Authz is helper-driven; `requireAuthenticated()` is `lib/authz/session.ts:11-23`, and contact visibility is `lib/authz/scopes/crm.ts:289-302`.

Data flow for contacts:

```text
page -> ContactsView -> NewContactForm -> createContact -> Prisma -> audit/Inngest/revalidate -> page refresh
app/[locale]/(routes)/crm/contacts/page.tsx:11-21
app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90
app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120
actions/crm/contacts/create-contact.ts:33-94
```

## Doc 3: Senior Ownership Guide

Technical debt register:

- Authz consistency: update actions should use scoped write helpers. Evidence: `actions/crm/contacts/update-contact.ts:51-66`, `lib/authz/scopes/crm.ts:102-108`.
- Type naming: contacts table schema is named opportunity. Evidence: `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.
- Loose data props: `ContactsViewProps.data` is `any[]`. Evidence: `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`.
- Process-local enrichment cancellation: `activeSessions` map is not multi-replica safe. Evidence: `app/api/crm/contacts/enrich/route.ts:20-22`.
- Dashboard serial loading: independent awaits are sequential. Evidence: `app/[locale]/(routes)/page.tsx:60-74`.

Upgrade path:

1. Add missing authz checks to all write actions.
2. Replace `as any` in CRM writes with Prisma input builders.
3. Normalize contact table row type.
4. Parallelize independent dashboard fetches.
5. Add server pagination for large tables.
6. Move enrichment session state to durable storage or database cancellation flags.

## Doc 4: Code Review Guide

Review in this order:

1. Boundary: Is the changed code a Server Component, Client Component, server action, or route handler? Examples: Server Component at `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`; Client Component at `app/[locale]/(routes)/crm/components/ContactsView.tsx:1-40`.
2. Auth: If data is read or written, is there session/authz enforcement? Examples: `actions/crm/get-contacts.ts:9-20`; `app/api/reports/export/route.ts:73-96`.
3. Scope: For CRM data, does the query apply role/ownership/account visibility? Example: `lib/authz/scopes/crm.ts:289-302`.
4. Types: Are Zod schemas, Prisma types, and component props aligned? Compare `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75` with `prisma/schema.prisma:446-504`.
5. Side effects: Are email, audit logs, events, and cache revalidation intentional? Example: `actions/crm/contacts/create-contact.ts:64-94`.
6. Serialization: Do Decimal values cross server-client boundary? Use `lib/serialize-decimals.ts:5-25`.
7. Tests: Unit tests should fit Jest config `jest.config.ts:3-15`; browser workflows should fit Playwright config `playwright.config.ts:15-83`.

## Doc 5: Debugging Guide

Common traces:

- User cannot enter app: inspect `getSession()` at `lib/auth-server.ts:7-11`, redirects at `app/[locale]/(routes)/layout.tsx:50-67`, and Better Auth config at `lib/auth.ts:12-128`.
- Contact missing from list: check `getContacts()` scope at `actions/crm/get-contacts.ts:9-20`, then contact fields `assigned_to`, `createdBy`, `accountsIDs`, and `deletedAt` at `prisma/schema.prisma:450-504`.
- Contact create succeeds but UI does not update: check revalidation path at `actions/crm/contacts/create-contact.ts:92-94` and form success close/reset at `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:113-119`.
- Report export returns 403: check the users category role guard at `app/api/reports/export/route.ts:90-92`.
- Enrichment cancel fails in production: check `activeSessions` limitation at `app/api/crm/contacts/enrich/route.ts:20-22`.

## Doc 6: Change Playbook

Adding a contact field:

1. Check whether the field exists in `prisma/schema.prisma:446-504`.
2. If it exists, update form schema/defaults in `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-102`.
3. Ensure create/update actions accept and persist the field at `actions/crm/contacts/create-contact.ts:9-62` and `actions/crm/contacts/update-contact.ts:8-66`.
4. Add display column if needed in `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:16-134`.
5. Verify table row schema in `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.
6. Add or update tests under `__tests__` or `tests/e2e`, matching `jest.config.ts:11-14` and `playwright.config.ts:15-83`.

## Doc 7: Interview Walkthrough

Use this concise explanation:

"NextCRM is a Next.js App Router CRM. The root locale layout sets up translation, theme, and toast providers (`app/[locale]/layout.tsx:52-74`). The authenticated app layout gates access with Better Auth sessions and user status, then renders the sidebar, header, avatar, and currency providers (`app/[locale]/(routes)/layout.tsx:50-136`). Data access goes through a Prisma singleton with Postgres adapter (`lib/prisma.ts:1-46`) and the schema models CRM entities such as contacts, accounts, activities, users, enrichments, and invoices (`prisma/schema.prisma:446-504`, `prisma/schema.prisma:922-987`). Authorization is enforced through helpers that convert sessions into `{ id, role }` and apply scoped where clauses (`lib/authz/session.ts:11-23`, `lib/authz/scopes/crm.ts:289-302`). A representative feature is contacts: the page fetches data server-side, renders a client table and create sheet, then a server action creates the contact, writes audit history, emits an event, and revalidates the page (`app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`, `actions/crm/contacts/create-contact.ts:33-94`)."

