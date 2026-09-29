# Tier 3: Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions

**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Infer why the app uses layouts, actions, route handlers, authz helpers, and Prisma this way.
**The Concept:** A senior reads the CRM building and asks why each desk exists where it does.
**Design Intent Before You Read the Code:** Architecture decisions should reveal tradeoffs: developer speed, auth consistency, data locality, and future scaling.
**Find It In The Code:** Open `app/[locale]/layout.tsx:52-74`, `app/[locale]/(routes)/layout.tsx:45-136`, `actions/crm/get-contacts.ts:9-60`, `app/api/crm/contacts/enrich/route.ts:24-179`, and `lib/prisma.ts:1-46`.

```text
// Senior reading order
layout -> authenticated shell -> server read action -> API route for streaming -> Prisma singleton
app/[locale]/layout.tsx:52-74
app/[locale]/(routes)/layout.tsx:45-136
actions/crm/get-contacts.ts:9-60
app/api/crm/contacts/enrich/route.ts:24-179
lib/prisma.ts:1-46
```

**The Aha Moment:** Architecture is the pattern of boundaries the team repeats.
**Socratic Checkpoint:**
1. Why might a layout be used for auth gating?
2. Why might contacts use server actions instead of HTTP routes for basic CRUD?
3. Why does enrichment use an API route instead of a server action?
4. Why cache Prisma globally in development?
5. Which repeated boundary is least consistent?

How to self-grade: Strong answers cite each file range and name one tradeoff per boundary.
**Connects To:** Mission 19 and Mission 21.

### Mission 19: Find the Bugs Before They Happen

**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Predict defects from naming, type, auth, and data-shape mismatches.
**The Concept:** A senior can smell a misfiled contact folder before the sales team reports it missing.
**Design Intent Before You Read the Code:** Bugs often start where names, types, and data contracts disagree.
**Find It In The Code:** Open `app/[locale]/(routes)/crm/contacts/page.tsx:11-27`, `app/[locale]/(routes)/crm/contacts/table-data/schema.tsx:5-20`, `app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:104-112`, `actions/crm/contacts/update-contact.ts:33-66`, and `prisma/schema.prisma:476-477`.

```ts
// app/[locale]/(routes)/crm/contacts/components/NewContactForm.tsx:104-112
const contactType = [
  { name: t("customer"), id: "Customer" },        // UI sends display-like values.
  { name: t("partner"), id: "Partner" },
  { name: t("vendor"), id: "Vendor" },
];

const result = await createContact({ ...rest, contact_type_id: type || undefined });
// prisma/schema.prisma:476-477 says contact_type_id is a UUID relation field.
```

**The Aha Moment:** Cross-file disagreement is one of the best bug predictors.
**Socratic Checkpoint:**
1. Which component name is misleading?
2. Which schema name is misleading?
3. Which form value may not match the database field type?
4. Which update action should add scoped authz?
5. Which bug is most likely user-visible?

How to self-grade: Strong answers cite all five ranges and explain symptom, cause, and test idea.
**Connects To:** Mission 20 and Mission 23.

### Mission 20: The Bug Injection Challenge

**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Design tests from user-visible symptoms without changing production code.
**The Concept:** Senior debugging starts with the sales rep's symptom, then traces the CRM paperwork backward.
**Design Intent Before You Read the Code:** Good tests reproduce behavior at the boundary where the user observes failure.
**Find It In The Code:** Open `actions/crm/contacts/update-contact.ts:33-82`, `app/api/crm/contacts/enrich/route.ts:20-22`, `app/api/crm/contacts/enrich/route.ts:144-179`, `app/[locale]/(routes)/page.tsx:60-74`, and `app/[locale]/(routes)/crm/contacts/table-components/data-table-row-actions.tsx:38-66`.

```ts
// actions/crm/contacts/update-contact.ts:51-66
const before = await prismadb.crm_Contacts.findUnique({ where: { id, deletedAt: null } });
const contact = await prismadb.crm_Contacts.update({
  where: { id },                                  // Symptom test: can another user update this id?
  data: { updatedBy: userId, ...rest } as any,
});
```

**The Aha Moment:** Bug injection is disciplined imagination tied to exact code paths.
**Socratic Checkpoint:**
1. What symptom would expose missing scoped write auth?
2. What symptom would expose multi-instance enrichment cancel?
3. What symptom would expose dashboard serial loading?
4. What symptom would expose row schema mismatch?
5. Which test should be unit vs E2E?

How to self-grade: Strong answers cite route/action line ranges and define expected observable behavior.
**Connects To:** Mission 23 and Mission 24.

### Mission 21: Performance X-Ray

**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Identify high-impact performance risks and propose code-level improvements.
**The Concept:** Performance is CRM throughput: how many folders can the team move before the front desk gets backed up.
**Design Intent Before You Read the Code:** Look for serial I/O, full-table client loads, expensive imports, and missing pagination.
**Find It In The Code:** Open `app/[locale]/(routes)/page.tsx:60-74`, `actions/crm/get-crm-data.ts:6-43`, `app/[locale]/(routes)/crm/contacts/table-components/data-table.tsx:34-74`, and `app/api/reports/export/route.ts:108-123`.

```ts
// app/[locale]/(routes)/page.tsx:60-74
const leads = await getLeadsCount();
const tasks = await getTasksCount();
const invoices = await getInvoicesCount();
// These independent calls can use Promise.all, as shown by actions/crm/get-crm-data.ts:6-43.
```

**The Aha Moment:** Performance review starts by finding where independent work is accidentally serialized.
**Socratic Checkpoint:**
1. Which dashboard calls can run in parallel?
2. Where does the repo already use `Promise.all`?
3. Which table loads all rows into client memory?
4. Which route lazily imports PDF generation?
5. What metric would prove the fix worked?

How to self-grade: Strong answers cite all four ranges and separate quick wins from architectural changes.
**Connects To:** Mission 18 and Mission 23.

### Mission 22: The Security Audit

**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Audit auth, authz, API keys, record scope, and error behavior.
**The Concept:** Security is CRM trust: the right person sees the right contact and no more.
**Design Intent Before You Read the Code:** Every read/write path should answer who, what role, which record, and what response on failure.
**Find It In The Code:** Open `lib/auth.ts:12-128`, `lib/authz/session.ts:11-23`, `lib/authz/scopes/crm.ts:289-302`, `app/api/reports/export/route.ts:73-96`, and `app/api/crm/contacts/enrich/route.ts:47-50`.

```ts
// app/api/crm/contacts/enrich/route.ts:47-50
const firecrawlApiKey = await getApiKey("FIRECRAWL", user.id); // Per-user/admin/env key path.
const openaiApiKey = await getApiKey("OPENAI", user.id);
if (!firecrawlApiKey || !openaiApiKey) {
  return NextResponse.json({ error: "NO_API_KEY" }, { status: 402 });
}
```

**The Aha Moment:** Security is a chain; the weakest unchecked record write becomes the real policy.
**Socratic Checkpoint:**
1. Which auth system is configured?
2. Where is role loaded for authz?
3. How is contact read scope built?
4. Which report category is role-restricted?
5. How does enrichment handle missing API keys?

How to self-grade: Strong answers cite all five ranges and mention one missing or risky check.
**Connects To:** Mission 23.

### Mission 23: Write the Test That Doesn't Exist

**Tier:** Senior
**Time Estimate:** 90 minutes
**Goal:** Draft a runnable test for one risky behavior.
**The Concept:** A test is a CRM audit stamp: it proves the workflow still follows policy after future changes.
**Design Intent Before You Read the Code:** Test the boundary with the highest risk and clearest expected outcome.
**Find It In The Code:** Open `jest.config.ts:3-15`, `app/api/reports/export/route.ts:86-96`, and `app/api/reports/export/__tests__/route.test.ts:1-60` if present in your checkout.

```ts
// app/api/reports/export/route.ts:86-96
if (!REPORT_CATEGORIES.includes(category as ReportCategory)) {
  return NextResponse.json({ error: "Invalid category" }, { status: 400 });
}

if (category === "users" && user.role === "user") {
  return forbiddenResponse();                     // Clear test target.
}
```

**The Aha Moment:** The best first test targets an explicit policy branch.
**Socratic Checkpoint:**
1. Which test runner handles route unit tests?
2. What branch should the test exercise?
3. What mocks are needed?
4. What response status is expected?
5. What JSON body is expected?

How to self-grade: Strong answers cite `jest.config.ts:3-15` and `app/api/reports/export/route.ts:86-96`, then provide arrange/act/assert steps.
**Connects To:** Mission 24.

### Mission 24: The Git History Tells a Story

**Tier:** Senior
**Time Estimate:** 45 minutes
**Goal:** Infer system evolution from realistic commit messages and current code.
**The Concept:** Git history is the CRM archive. It tells you why a workflow was changed, not only what it is today.
**Design Intent Before You Read the Code:** Commit messages should be interpreted against current architecture and likely migration pressure.
**Find It In The Code:** Open `prisma/migrations/20260328000002_add_crm_audit_log/migration.sql` if present, `lib/audit-log.ts:1-90`, `actions/crm/contacts/create-contact.ts:85-94`, `lib/serialize-decimals.ts:1-25`, and `actions/crm/get-crm-data.ts:45-65`.

```ts
// actions/crm/contacts/create-contact.ts:85-94
await writeAuditLog({
  entityType: "contact",                           // Current code implies audit-log feature exists.
  entityId: contact.id,
  action: "created",
  userId: session.user.id,
});
```

**The Aha Moment:** Good history reading connects migrations, helpers, and current feature behavior.
**Socratic Checkpoint:**
1. What does an audit-log migration imply?
2. What code proves contact creation uses audit logs?
3. What does a Decimal serialization helper imply about prior bugs?
4. What current code uses Decimal serialization?
5. Which future commit would you want to see next?

How to self-grade: Strong answers cite current code plus migration/helper evidence and infer user impact.
**Connects To:** Mission 25 in the reference artifacts.

