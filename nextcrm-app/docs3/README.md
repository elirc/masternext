# Docs3 Senior Growth Build Path

This folder contains 25 additional user stories for `nextcrm-app`. The goal is not only to ship features; it is to use each feature as deliberate practice for moving from junior execution to senior software engineering judgment.

Use these docs like pair-programming tickets. I am expected to help build the implementation, but you should pause at the checkpoints, predict the next code path, and explain the tradeoff before we make the edit.

## Reading Order

1. `01-user-stories.md` gives the full product backlog.
2. `02-implementation-plans-01-09.md` covers foundation and mid-level delivery tickets.
3. `03-implementation-plans-10-17.md` covers cross-feature product work and policy-aware implementation.
4. `04-implementation-plans-18-25.md` covers senior architecture, platform, and quality systems.
5. `05-pair-programming-playbook.md` explains how to use the stories to level up.

## Skill Progression

- Stories 1-5: trace existing flows, make localized UI/data changes, and validate with focused tests.
- Stories 6-12: connect frontend, server actions, route handlers, Prisma, and authorization.
- Stories 13-19: design durable product behavior across roles, data volume, async work, and user preferences.
- Stories 20-25: own system boundaries, observability, operational safety, test strategy, and architecture.

## Definition Of Done

- The story's acceptance criteria are independently verifiable.
- The implementation follows existing repo patterns before inventing a new abstraction.
- Data crossing a React Server Action or Client Component boundary serializes Prisma Decimal values with `serializeDecimals()` or `serializeDecimalsList()` when relevant.
- Server mutations authenticate and authorize before writing.
- Tests cover the smallest meaningful behavior surface.
- You can explain the data flow from UI to database and back using file paths.
- You can name the most likely production failure mode and how the implementation handles it.

## Repo Anchors

These plans intentionally reuse current code areas:

- CRM contacts: `app/[locale]/(routes)/crm/contacts`, `actions/crm/contacts`, `actions/crm/get-contact.ts`
- Accounts and ownership: `actions/crm/accounts`, `lib/authz/scopes/crm.ts`
- Activities and audit log: `actions/crm/activities`, `actions/crm/audit-log`, `lib/audit-log.ts`
- Reports: `app/[locale]/(routes)/reports`, `actions/reports`, `app/api/reports/export/route.ts`
- Invoices: `app/[locale]/(routes)/invoices`, `actions/invoices`, `lib/invoices`
- Documents: `app/[locale]/(routes)/documents`, `actions/documents`
- Enrichment: `app/api/crm/contacts/enrich`, `app/api/crm/targets/enrich`, `lib/enrichment`
- MCP tools: `lib/mcp/tools`
- Tests: `__tests__`, `actions/**/__tests__`, `app/api/**/__tests__`, `tests/e2e`

