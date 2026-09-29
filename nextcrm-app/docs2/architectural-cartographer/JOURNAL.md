# Journal

## First-pass mental model

This repository is a single Next.js application named `nextcrm-app`, using `pnpm`, Next.js scripts, Prisma generation/migration, Jest, and Playwright from `package.json:1-24`. The primary product shape is a CRM workspace with modules for CRM entities, campaigns, invoices, reports, documents, projects, email, and admin functions, visible in the authenticated app shell translation object at `app/[locale]/(routes)/layout.tsx:72-91` and the dashboard card links at `app/[locale]/(routes)/page.tsx:113-192`.

## Inspection discoveries

- The top-level route system is the Next.js App Router because routes live under `app/`, with locale segments and route groups such as `app/[locale]/(routes)/layout.tsx:45-136`.
- The public root layout loads global CSS, Google font, `next-intl`, theme provider, and toast providers at `app/[locale]/layout.tsx:1-14` and `app/[locale]/layout.tsx:61-71`.
- The authenticated shell checks `getSession()`, redirects unauthenticated, pending, or inactive users, then provides sidebar, avatar, and currency state at `app/[locale]/(routes)/layout.tsx:50-67` and `app/[locale]/(routes)/layout.tsx:106-135`.
- Database access centralizes through a Prisma singleton with a Postgres adapter and development global cache at `lib/prisma.ts:1-46`.
- Auth is Better Auth backed by Prisma with email OTP, Google provider, admin plugin, role fields, and first-user admin promotion at `lib/auth.ts:12-128`.
- Authorization is not only role based. CRM read access also depends on assignment, creator, and linked account scope at `lib/authz/scopes/crm.ts:289-302`.
- The strongest end-to-end teaching path is CRM contacts: the page fetches CRM lookup data and contacts at `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, the client view opens a sheet and renders a table at `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`, server data comes from scoped Prisma queries at `actions/crm/get-contacts.ts:9-60`, and creation writes contact data, sends optional assignment email, writes audit log, emits an Inngest event, and revalidates the path at `actions/crm/contacts/create-contact.ts:33-94`.

## Why these files became teaching anchors

- `app/[locale]/(routes)/layout.tsx` is the "front door security desk" of the CRM: it decides whether a user enters the building and which shell providers wrap the workspace (`app/[locale]/(routes)/layout.tsx:50-67`, `app/[locale]/(routes)/layout.tsx:106-135`).
- `actions/crm/get-contacts.ts` is small enough to read but shows real production concerns: authentication, authz scope, relation includes, and a database return shape (`actions/crm/get-contacts.ts:9-60`).
- `actions/crm/contacts/create-contact.ts` is a good "business transaction" file because it turns form data into database data and triggers side effects (`actions/crm/contacts/create-contact.ts:47-94`).
- `app/api/crm/contacts/enrich/route.ts` is a senior-level anchor because it combines auth, validation, API keys, streaming, persistence, abort handling, and cancellation (`app/api/crm/contacts/enrich/route.ts:24-140`, `app/api/crm/contacts/enrich/route.ts:144-179`).
- `prisma/schema.prisma` is the source of truth for entity relationships, especially contact fields, user relationships, enrichments, and soft delete indexes (`prisma/schema.prisma:446-504`, `prisma/schema.prisma:115-133`, `prisma/schema.prisma:922-987`).

## Where a junior might get confused

- The contacts table type is named `Opportunity`, even though it describes contact columns. That mismatch is visible in `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20` and then imported by the contact columns at `app/[locale]/(routes)/crm/contacts/table-components/columns.tsx:9-16`.
- The contacts page component is named `AccountsPage` even though it renders contacts at `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`.
- Some CRM model names preserve older spelling and migration history, such as `cratedAt`, `crate_by_user`, and `crated_contacts` at `prisma/schema.prisma:454-482` and `prisma/schema.prisma:952-953`.
- Server Components and Client Components mix in one route: the contacts page fetches on the server at `app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, while `ContactsView` starts with `"use client"` and owns sheet state at `app/[locale]/(routes)/crm/components/ContactsView.tsx:1-40`.

## Where a mid-level engineer should slow down

- Data access is not uniformly scoped. `getContacts()` applies `contactReadScopeWhere(user)` at `actions/crm/get-contacts.ts:18-20`, while `getAllCrmData()` pulls many lookup lists and entity lists with only `deletedAt: null` filters at `actions/crm/get-crm-data.ts:23-43`.
- Server actions such as `updateContact()` call `getSession()` but do not call the same authz helper before updating by `id`; compare `actions/crm/contacts/update-contact.ts:33-66` with the safer scoped helpers at `lib/authz/scopes/crm.ts:91-108`.
- The dashboard awaits many counts sequentially at `app/[locale]/(routes)/page.tsx:60-74`, while `getAllCrmData()` already demonstrates `Promise.all` batching at `actions/crm/get-crm-data.ts:6-43`.
- Prisma Decimal serialization exists because Decimal objects cannot cross the server-to-client boundary safely, as documented in code at `lib/serialize-decimals.ts:1-25` and used for opportunities/contracts in `actions/crm/get-crm-data.ts:45-65`.

## Where a senior engineer should be skeptical

- The authenticated app shell builds `metadataBase` with a non-null assertion before an `||` fallback, so a missing `NEXT_PUBLIC_APP_URL` can still be risky; see `app/[locale]/(routes)/layout.tsx:16-19`.
- The contact enrichment route stores live abort controllers in process memory; the file itself warns this only works correctly in single-process deployments at `app/api/crm/contacts/enrich/route.ts:20-22`.
- `createContact()` and `updateContact()` use `as any` around Prisma data writes at `actions/crm/contacts/create-contact.ts:47-62` and `actions/crm/contacts/update-contact.ts:52-66`, which weakens TypeScript discipline in a high-value write path.
- The docs should not hide gaps. There is no root `middleware.ts` in the inspected tree; auth is enforced mainly by layouts, route handlers, and helpers such as `app/[locale]/(routes)/layout.tsx:50-67`, `app/api/reports/export/route.ts:73-96`, and `lib/authz/session.ts:11-23`.

## How to use checkpoints

Answer each checkpoint as if you were explaining to an engineer joining the project tomorrow. A strong answer should cite the files, name the data boundary, mention what could break, and explain how you would verify the claim. The self-grade rubrics in this suite tell you what a strong answer includes immediately after the questions.

