# Architectural Cartographer

This suite is a top-down onboarding map for `nextcrm-app`. It teaches the system as a CRM cockpit first, then drills into the Next.js route tree, authenticated app shell, Prisma persistence layer, typed UI contracts, API routes, authorization helpers, tests, and review habits.

Use it in this order:

1. `00-reading-map.md` gives the first mental model and the file order.
2. `01-junior-engineer.md` teaches setup, folders, entry points, components, routes, and vocabulary.
3. `02-mid-level-engineer.md` traces a real feature from UI to database and back.
4. `03-senior-engineer.md` critiques the architecture and gives ownership exercises.
5. `04-reference-suite.md` is the permanent field manual for reviews, debugging, changes, and interviews.

Every factual claim about this repo points to a concrete file and line range. When you hit a Socratic checkpoint, answer it out loud or in notes, then immediately use the included self-grade section to compare your reasoning against a strong answer.

Teaching anchors used throughout this suite:

- App and package identity: `package.json:1-24`, `package.json:98-129`.
- Root layout and providers: `app/[locale]/layout.tsx:52-74`.
- Authenticated shell: `app/[locale]/(routes)/layout.tsx:45-136`.
- CRM contacts feature: `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-94`, `actions/crm/get-contacts.ts:9-60`, `actions/crm/contacts/create-contact.ts:9-99`.
- Persistence and auth: `lib/prisma.ts:1-46`, `lib/auth.ts:12-128`, `lib/authz/scopes/crm.ts:289-302`, `prisma/schema.prisma:446-504`.

