# Journal

## Mission design rationale

The missions are ordered from "how the app turns on" to "how a senior predicts failure." The first tier builds map-reading skill using `package.json:1-24`, root layouts (`app/[locale]/layout.tsx:52-74`, `app/[locale]/(routes)/layout.tsx:45-136`), and a real CRM contacts feature (`app/[locale]/(routes)/crm/contacts/page.tsx:11-27`). The second tier deepens into state, hooks, side effects, route contracts, authz, and full-stack tracing. The third tier asks for critique, testing, security, performance, and history interpretation.

## Why missions are ordered this way

You first learn the "building": scripts, route tree, root providers, authenticated shell, and folders. Then you learn one "desk workflow": contacts list and create. Then you learn the "back office": Prisma, authz, API routes, reports, enrichment, and tests. This mirrors how a senior enters an unfamiliar codebase: locate entry points, trace one feature, identify boundaries, then look for systemic risks.

## Chosen code paths

- App heartbeat: `package.json:5-24`, `app/[locale]/layout.tsx:52-74`.
- Authenticated shell: `app/[locale]/(routes)/layout.tsx:50-136`.
- Contact path: `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`, `app/[locale]/(routes)/crm/components/ContactsView.tsx:38-94`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:48-120`, `actions/crm/contacts/create-contact.ts:33-94`.
- Data and authorization: `actions/crm/get-contacts.ts:9-60`, `lib/authz/session.ts:11-23`, `lib/authz/scopes/crm.ts:289-302`.
- API contracts: `app/api/reports/export/route.ts:73-127`, `app/api/crm/targets/[id]/contacts/route.ts:12-52`, `app/api/crm/contacts/enrich/route.ts:24-179`.
- Persistence: `lib/prisma.ts:1-46`, `prisma/schema.prisma:446-504`, `prisma/schema.prisma:922-987`.

## Skills by tier

- Junior: identify files, name boundaries, read components, read simple actions, connect route to UI.
- Mid-Level: trace state, hooks, side effects, API contracts, middleware-like helper chains, and diffs.
- Senior: critique architectural decisions, predict bugs, audit performance/security, add missing tests, and infer evolution from commit messages.

## What a senior would do differently at checkpoints

A senior does not only answer "what happens." They answer "what invariant is protected, where could it fail, and how would I test it." For example, when reading `actions/crm/get-contacts.ts:18-20`, a senior immediately checks `lib/authz/scopes/crm.ts:289-302` and `prisma/schema.prisma:446-504` to see whether the visibility rule matches the data model.

