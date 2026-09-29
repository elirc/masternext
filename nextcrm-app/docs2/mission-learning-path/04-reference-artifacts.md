# Mission Reference Artifacts

### Mission 25: Write the Docs That Don't Exist

**Tier:** Senior
**Time Estimate:** 90 minutes
**Goal:** Turn code-reading knowledge into action references someone can use while working.
**The Concept:** A CRM team does not only need architecture knowledge; it needs checklists for onboarding, reviewing, debugging, changing, owning, and explaining the system.
**Design Intent Before You Read the Code:** Workflow docs should point to the exact boundary a worker will touch: app shell, server action, API route, authz helper, Prisma model, or test runner.
**Find It In The Code:** Open `package.json:5-24`, `app/[locale]/(routes)/layout.tsx:45-136`, `actions/crm/get-contacts.ts:9-60`, `actions/crm/contacts/create-contact.ts:9-99`, `app/api/reports/export/route.ts:73-127`, `lib/authz/scopes/crm.ts:289-302`, and `prisma/schema.prisma:446-504`.

```text
// Mission 25 map
package.json:5-24                                  // Setup and scripts.
app/[locale]/(routes)/layout.tsx:45-136            // App shell and auth gate.
actions/crm/get-contacts.ts:9-60                   // Scoped read pattern.
actions/crm/contacts/create-contact.ts:9-99        // Write and side-effect pattern.
app/api/reports/export/route.ts:73-127             // HTTP contract pattern.
lib/authz/scopes/crm.ts:289-302                    // CRM visibility policy.
prisma/schema.prisma:446-504                       // Contact persistence model.
```

**The Aha Moment:** Good internal docs are executable memory: they tell the next engineer what to do, where to look, and what can break.
**Socratic Checkpoint:**
1. Which reference would help on day one?
2. Which reference would help during code review?
3. Which reference would help during a production bug?
4. Which reference would help when changing a CRM feature?
5. Which reference would help you explain the system under pressure?

How to self-grade: Strong answers cite the relevant reference doc below and at least one code range from the mission map above.
**Connects To:** The user story build path, because these references become the field guides for implementing real tickets.

These are workflow-focused action references for doing work in the codebase.

## Doc 1: Junior Onboarding Checklist

- Confirm Node and pnpm versions from `package.json:5-8`.
- Run `pnpm install`.
- Copy env files as described at `README.md:287-295`.
- Configure `DATABASE_URL`, `BETTER_AUTH_SECRET`, and `BETTER_AUTH_URL` from `.env.example:7-12`.
- Run Prisma commands from `README.md:313-324`.
- Start local dev using `README.md:326-332`.
- Read `app/[locale]/layout.tsx:52-74`, `app/[locale]/(routes)/layout.tsx:45-136`, and `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`.
- First PR checklist: explain the route touched, auth boundary, data query/mutation, UI state, and test path.

## Doc 2: Architecture Guide for New Engineers

Navigate by boundaries:

- App shell boundary: `app/[locale]/layout.tsx:52-74` and `app/[locale]/(routes)/layout.tsx:45-136`.
- Server read boundary: `actions/crm/get-contacts.ts:9-60`.
- Server write boundary: `actions/crm/contacts/create-contact.ts:9-99`.
- API boundary: `app/api/reports/export/route.ts:73-127`.
- Authz boundary: `lib/authz/session.ts:11-23` and `lib/authz/scopes/crm.ts:289-302`.
- Database boundary: `lib/prisma.ts:1-46` and `prisma/schema.prisma:446-504`.

## Doc 3: Code Review Checklist

- Is the component server or client? Check for `"use client"` like `app/[locale]/(routes)/crm/components/ContactsView.tsx:1`.
- Is session required? Check layout or route auth such as `app/[locale]/(routes)/layout.tsx:50-67`.
- Is record scope enforced? Check helpers like `lib/authz/scopes/crm.ts:289-302`.
- Are Prisma Decimal values serialized before entering client props? See `lib/serialize-decimals.ts:1-25`.
- Are side effects intentional and tested? See `actions/crm/contacts/create-contact.ts:64-94`.
- Are types aligned with Prisma fields? Compare `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-75` and `prisma/schema.prisma:446-504`.

## Doc 4: Debugging Playbook

- Auth bug: inspect `lib/auth-server.ts:7-11`, `lib/auth.ts:12-128`, and app redirects at `app/[locale]/(routes)/layout.tsx:50-67`.
- Missing CRM row: inspect read scope at `actions/crm/get-contacts.ts:18-20` and `lib/authz/scopes/crm.ts:289-302`.
- Create did not appear: inspect `revalidatePath` at `actions/crm/contacts/create-contact.ts:92-94` and table data flow at `app/[locale]/(routes)/crm/components/ContactsView.tsx:80-87`.
- Export failed: inspect category and format branches at `app/api/reports/export/route.ts:82-127`.
- Enrichment stuck: inspect API key checks at `app/api/crm/contacts/enrich/route.ts:47-50` and streaming update code at `app/api/crm/contacts/enrich/route.ts:88-139`.

## Doc 5: Change Playbook

1. Start on `dev` according to repo workflow in `AGENTS.md`.
2. Identify whether the change is UI-only, server action, API route, or schema.
3. For UI, locate component and props, for example `app/[locale]/(routes)/crm/components/ContactsView.tsx:32-90`.
4. For data, locate action/route and Prisma model, for example `actions/crm/get-contacts.ts:18-58` and `prisma/schema.prisma:446-504`.
5. Apply authz checks if record access changes, using `lib/authz/session.ts:11-23` and `lib/authz/scopes/crm.ts:91-108`.
6. Add tests matching `jest.config.ts:3-15` or `playwright.config.ts:15-83`.
7. Run the smallest relevant check first, then broader test/lint as appropriate.

## Doc 6: Senior Ownership Notes

Monitor:

- Contact update authorization (`actions/crm/contacts/update-contact.ts:51-66`).
- Enrichment process-local state (`app/api/crm/contacts/enrich/route.ts:20-22`).
- Dashboard serial awaits (`app/[locale]/(routes)/page.tsx:60-74`).
- Loose contact row types (`app/[locale]/(routes)/crm/components/ContactsView.tsx:32-35`).
- Decimal serialization in server-to-client data (`lib/serialize-decimals.ts:1-25`).

Leave alone unless changing behavior:

- Prisma singleton shape in `lib/prisma.ts:1-46`.
- Better Auth route delegation in `app/api/auth/[...all]/route.ts:1-4`.
- Existing App Router route grouping under `app/[locale]/(routes)/layout.tsx:45-136`.

## Doc 7: Interview Walkthrough

Practice answer:

"I worked in a Next.js App Router CRM. The route shell first loads global providers for translations and theme (`app/[locale]/layout.tsx:52-74`), then an authenticated app layout checks Better Auth session and user status before rendering navigation (`app/[locale]/(routes)/layout.tsx:50-136`). Data access goes through Prisma with a Postgres adapter (`lib/prisma.ts:1-46`) and domain models like contacts, users, and enrichments in Prisma (`prisma/schema.prisma:446-504`, `prisma/schema.prisma:922-987`). One feature I can trace is contacts: the server page fetches scoped contacts (`app/[locale]/(routes)/crm/contacts/page.tsx:11-21`, `actions/crm/get-contacts.ts:9-60`), the client view opens a create form and table (`app/[locale]/(routes)/crm/components/ContactsView.tsx:38-90`), and the create action writes the row, sends optional email, writes audit log, emits an Inngest event, and revalidates (`actions/crm/contacts/create-contact.ts:47-94`)."
