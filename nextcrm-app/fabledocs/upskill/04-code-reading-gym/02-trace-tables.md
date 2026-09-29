# Trace Tables

Fill each table yourself with the files open, *then* compare with the completed rows in [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md) where they overlap. Columns: Step | File:lines | Value shape | Owner | Transformation | Risk.

## Trace 1 (UI → API): enrichment field save from the contact detail page

Start: user accepts an enrichment suggestion in the UI. End: contact row updated.
Path to trace: client component → `fetch PATCH /api/crm/contacts/[id]` → [route.ts](../../../app/api/crm/contacts/%5Bid%5D/route.ts) → [tryScopedUpdateContact](../../../lib/authz/scopes/crm.ts#L33-L43) → response handling.
Rows to produce: ≥6. Must include: where `enrichmentFields` gets its keys filtered, and where the 404-vs-403 decision is made.
Checkpoint questions: What happens with `{"enrichmentFields": {"email": "x"}}`? (Filtered out — email not in FIELD_MAP; if it's the *only* field → 400 "No valid fields".) What does a `user`-role caller get for a colleague's contact? (404 via count===0.)

## Trace 2 (persistence): `addPayment` → balance and status

Open [actions/invoices/add-payment.ts](../../../actions/invoices/add-payment.ts) (not excerpted in this curriculum — fresh eyes) and trace a $500 payment against a $1200 invoice.
Rows: ≥6. Must include: where `canAddPayment` gates ([permissions.ts:45-48](../../../lib/invoices/permissions.ts#L45-L48)), how `balanceDue` is computed, and which status (`PARTIALLY_PAID`?) gets written and by what rule.
Checkpoint: is the payment insert + invoice update atomic (one transaction / nested write / two calls)? Whatever you find, write the failure narrative for the non-atomic case.

## Trace 3 (auth): first admin bootstrap

Trace a brand-new deployment's first Google sign-in through [lib/auth.ts:58-124](../../../lib/auth.ts#L58-L124): OAuth callback → user created → `onUserCreated` → count===1 → role admin + ACTIVE.
Rows: ≥5. Must include: what the *second* user experiences (PENDING + [newUserNotify](../../../lib/new-user-notify.ts)) and where PENDING users get blocked (investigate: find the status check — `rg "PENDING" app lib actions` — that's the point of the drill).
Checkpoint: two users complete OAuth simultaneously on a fresh install — describe the race window at [auth.ts:110-116](../../../lib/auth.ts#L109-L124) and its worst outcome (two admins? zero? count runs after each create, so two concurrent creates can both see count===1? No — one sees 2. Worst case both see... work it out and label your confidence).

## Trace 4 (error path): issuing an invoice with a deleted series

`issueInvoice` where `invoice.seriesId` points to a series an admin deleted. Trace [issue-invoice.ts:48-58](../../../actions/invoices/issue-invoice.ts#L48-L58) → [numbering.ts:18](../../../lib/invoices/numbering.ts#L18) `findUniqueOrThrow`.
Rows: ≥4. Must include: which line throws, what the transaction does (rolls back — nothing committed), what the user sees (generic server-action error), and what the *fix* options are (validate series before tx vs FK constraint vs soft-delete series).
Checkpoint: is invoice data corrupted? (No — that's the transaction earning its keep. Say precisely why.)

## Trace 5 (async): campaign pause racing a send

At T0 the send-step function loads a running campaign ([send-step.ts:20-32](../../../inngest/functions/campaigns/send-step.ts#L20-L32)); at T1 a user pauses it ([actions/campaigns/pause-campaign.ts](../../../actions/campaigns/pause-campaign.ts)); at T2 the send step executes.
Rows: ≥5 across both timelines (two "Owner" values: worker, user).
Checkpoint: the email at T2 — sent or not? (Sent — the paused check was at T0.) How many more emails can leak after a pause? (Only in-flight executions past their load step; queued events for *this* function re-check on load.) Is that acceptable? Argue both sides in two sentences each.

## Self-grading (all traces)

- **Basic**: correct sequence of files and calls.
- **Solid**: value *shapes* correct at each hop (raw JSON vs parsed vs Prisma row vs Decimal-bearing row) and each row has an owner.
- **Strong**: the Risk column contains at least one entry per trace that this curriculum did not hand you.
