# User Story Build Path

Use these stories as real practice tickets. Start at Story 1 even if it feels small. The progression is deliberate: first you change existing UI, then existing data display, then state/API behavior, then database-backed and architectural changes.

How to know a story is done:

- Acceptance criteria are independently verifiable.
- You can name every file you touched and why.
- You can explain the data flow using file and line references.
- You ran the smallest relevant check you can justify.
- You did not widen the scope beyond the story.

How to get unstuck without asking for the solution:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Use this repo's actual files and patterns when guiding me.
```

Difficulty progression:

- Easy stories touch one or two files and train you to read UI and types.
- Medium-Easy stories add UI features using existing data.
- Medium stories introduce API or state ownership.
- Hard stories touch frontend, backend, and database.
- Expert asks you to design around performance, auth, or a new domain concept.

Core code paths used by these stories include contacts (`app/[locale]/(routes)/crm/contacts/page.tsx:11-27`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-94`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120`, `actions/crm/contacts/create-contact.ts:9-99`), CRM authz (`lib/authz/scopes/crm.ts:289-302`), report export (`app/api/reports/export/route.ts:73-127`), dashboard metrics (`app/[locale]/(routes)/page.tsx:60-192`), and Prisma models (`prisma/schema.prisma:446-504`).

