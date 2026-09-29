# Reading Map

## Mental model

Think of NextCRM as a guarded CRM office. The locale root layout sets up the building utilities such as fonts, translations, theme, and toasts (`app/[locale]/layout.tsx:52-74`). The authenticated route layout is the front desk that checks whether the visitor has an active session and then wraps the workspace with navigation, avatar, and currency context (`app/[locale]/(routes)/layout.tsx:50-67`, `app/[locale]/(routes)/layout.tsx:106-135`). Inside the office, pages such as CRM contacts fetch server data (`app/[locale]/(routes)/crm/contacts/page.tsx:11-21`), client components manage interaction state (`app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`), server actions and route handlers mutate/query data (`actions/crm/contacts/create-contact.ts:47-94`, `app/api/reports/export/route.ts:73-127`), and Prisma maps those operations to Postgres models (`lib/prisma.ts:1-46`, `prisma/schema.prisma:446-504`).

## Top 10 files to read in order

1. `package.json:1-24`
   - Why now: identifies project type, engines, scripts, and seed command.
   - Know before: this repo uses `pnpm`.
   - Explain after: what command starts dev, builds, tests, and seeds.

2. `app/[locale]/layout.tsx:52-74`
   - Why now: it is the public root of every localized page.
   - Know before: App Router layouts wrap child routes.
   - Explain after: which providers are global and which are not.

3. `app/[locale]/(routes)/layout.tsx:45-136`
   - Why now: this is the authenticated app shell.
   - Know before: `redirect()` stops rendering for invalid sessions.
   - Explain after: how pending, inactive, and unauthenticated users are handled.

4. `lib/auth.ts:12-128`
   - Why now: defines the auth system and role fields.
   - Know before: Better Auth owns auth routes through an adapter.
   - Explain after: how email OTP, Google auth, user status, and admin promotion work.

5. `lib/authz/session.ts:11-23`
   - Why now: converts a session into a smaller authorization user.
   - Know before: authn answers "who are you"; authz answers "what may you do".
   - Explain after: why the role is reloaded from the database.

6. `lib/prisma.ts:1-46`
   - Why now: every database query depends on this client.
   - Know before: Next dev reloads modules often.
   - Explain after: why the repo caches Prisma globally in development.

7. `prisma/schema.prisma:446-504`
   - Why now: contacts are a central CRM entity.
   - Know before: Prisma relation fields explain how includes work.
   - Explain after: which fields power assignment, account linking, enrichment, and soft delete.

8. `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`
   - Why now: small route-level Server Component.
   - Know before: Server Components can call server functions directly.
   - Explain after: what data the route needs before rendering the client view.

9. `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-94`
   - Why now: shows client state and composition.
   - Know before: `"use client"` moves rendering and state to the browser.
   - Explain after: how the add-contact sheet and table are connected.

10. `actions/crm/contacts/create-contact.ts:9-99`
    - Why now: complete write path.
    - Know before: server actions can be called from forms/client code.
    - Explain after: how form data becomes a contact, audit log, Inngest event, and revalidated page.

## The 3 most important data flows

1. Authenticated page render:
   `getSession()` reads Better Auth session headers (`lib/auth-server.ts:7-11`), the route layout redirects invalid users (`app/[locale]/(routes)/layout.tsx:50-67`), then wraps children in app providers (`app/[locale]/(routes)/layout.tsx:106-135`).

2. Contact list read:
   The contacts route fetches CRM data and contacts (`app/[locale]/(routes)/crm/contacts/page.tsx:11-21`), `getContacts()` requires auth and applies contact scope (`actions/crm/get-contacts.ts:9-20`), then Prisma includes related users, accounts, opportunities, and documents (`actions/crm/get-contacts.ts:20-58`).

3. Contact create:
   `NewContactForm` validates and submits (`app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120`), `createContact()` writes the row (`actions/crm/contacts/create-contact.ts:47-62`), optionally emails the assignee (`actions/crm/contacts/create-contact.ts:64-83`), writes audit history and emits enrichment work (`actions/crm/contacts/create-contact.ts:85-94`), and the UI closes the sheet on success (`app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:113-119`).

## Pre-reading checklist

1. Can I name the framework and package manager from `package.json:1-24`?
2. Can I explain why `app/[locale]/layout.tsx:62-68` wraps children in `NextIntlClientProvider`?
3. Can I describe the difference between `app/[locale]/layout.tsx` and `app/[locale]/(routes)/layout.tsx`?
4. Can I identify where an unauthenticated user is redirected (`app/[locale]/(routes)/layout.tsx:50-56`)?
5. Can I find the Prisma singleton (`lib/prisma.ts:35-46`)?
6. Can I find the contact model (`prisma/schema.prisma:446-504`)?
7. Can I find a simple server action (`actions/dashboard/get-contacts-count.ts:1-6`)?
8. Can I find a complex route handler (`app/api/crm/contacts/enrich/route.ts:24-140`)?
9. Can I explain what `contactReadScopeWhere()` protects (`lib/authz/scopes/crm.ts:289-302`)?
10. Can I point to a real test config (`jest.config.ts:3-15`, `playwright.config.ts:15-83`)?

## Red flags checklist for review and architecture reading

- A write path updates by `id` without an authorization scope check; compare `actions/crm/contacts/update-contact.ts:51-66` with `lib/authz/scopes/crm.ts:102-108`.
- A Prisma Decimal result crosses to a Client Component without `serializeDecimals()` or `serializeDecimalsList()` from `lib/serialize-decimals.ts:5-25`.
- A Server Component fetches many independent values sequentially as in `app/[locale]/(routes)/page.tsx:60-74`.
- A client component accepts `any[]` for core business data, as `ContactsViewProps` does at `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`.
- A schema or type name describes the wrong domain, as `opportunitySchema` does for contact rows at `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`.
- A route relies on in-memory process state for active sessions; the enrichment route notes that limitation at `app/api/crm/contacts/enrich/route.ts:20-22`.
- A required environment variable is non-null asserted in a layout, as `NEXT_PUBLIC_APP_URL!` is at `app/[locale]/layout.tsx:29`.
- A mutation uses `as any` on Prisma data in a core write path, as contact create/update do at `actions/crm/contacts/create-contact.ts:47-62` and `actions/crm/contacts/update-contact.ts:52-66`.
- User-visible text appears hard-coded instead of translated, as some contact table labels do at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:37-123`.
- A "soft delete" model query forgets `deletedAt: null`; the contact model has `deletedAt` indexed at `prisma/schema.prisma:492-504`.

