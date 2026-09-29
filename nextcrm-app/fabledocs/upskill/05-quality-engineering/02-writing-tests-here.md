# Writing Tests Here

Seven recipes using the repo's real conventions. Commands are __inferred__ (`pnpm test -- <pattern>`).

## Recipe 1 — Pure logic (happy path + table)

Model: [__tests__/lib/invoices/permissions.test.ts](../../../__tests__/lib/invoices/permissions.test.ts). Style: `it.each` over states.

```ts
// Illustrative fake code: not from this repo — add to __tests__/lib/invoices/totals.test.ts
import { computeLineTotal } from "@/lib/invoices/totals";
import { Decimal } from "decimal.js";

it("applies discount before VAT, rounds per line", () => {
  const t = computeLineTotal({
    quantity: new Decimal(3), unitPrice: new Decimal("10"),
    discountPercent: new Decimal(10), taxRate: new Decimal(21),
  });
  expect(t.lineSubtotal.toString()).toBe("27");     // 30 - 10%
  expect(t.lineVat.toString()).toBe("5.67");        // 27 * 21%
  expect(t.lineTotal.toString()).toBe("32.67");
});
```
Run: `pnpm test -- totals`. Why it matters: pins the discount-then-VAT-then-round order that legal invoices require.

## Recipe 2 — Validation failure

Assert a bad input is rejected *before* any DB call. For a Zod-guarded action, feed `raw` that violates the schema and expect `fieldErrors`/throw — no Prisma mock needed because parse fails first.
```ts
// Illustrative fake code: not from this repo
await expect(createInvoice({ accountId: "", lineItems: [] }))
  .rejects.toThrow();  // empty accountId + no lines fails createInvoiceSchema.parse
```
Run: `pnpm test -- create-invoice`. Teaches: the cheapest test is the one that never reaches the DB.

## Recipe 3 — Permission failure (pure)

```ts
// Illustrative fake code: not from this repo
expect(canIssueInvoice({ status: "ISSUED", createdBy: "u1" }, { id: "u1", role: "user" }))
  .toBe(false);   // already issued → not issuable, even by owner
```
This is the model already in [permissions.test.ts](../../../__tests__/lib/invoices/permissions.test.ts). Note how testing the *pure* guard replaces a slow integration test of the whole issue path for the authz dimension.

## Recipe 4 — Scope-where snapshot (fills a real gap)

The scope builders are pure functions returning objects — untested today. A snapshot catches accidental broadening (the scariest authz regression):
```ts
// Illustrative fake code: not from this repo — new __tests__/lib/authz/scopes.test.ts
import { accountReadScopeWhere } from "@/lib/authz/scopes/crm";
it("user role is restricted to ownership OR", () => {
  const w = accountReadScopeWhere({ id: "u1", role: "user" });
  expect(w).toEqual({ deletedAt: null, OR: [
    { assigned_to: "u1" }, { createdBy: "u1" },
    { watchers: { some: { user_id: "u1" } } },
  ]});
});
it("admin sees all non-deleted", () => {
  expect(accountReadScopeWhere({ id: "a", role: "admin" })).toEqual({ deletedAt: null });
});
```
Run: `pnpm test -- scopes`. If someone later drops the `OR` (widening user access to all rows), this test screams. **This is a genuinely useful contribution — see [good-first-tickets](../06-contribution-practice/01-good-first-tickets.md).**

## Recipe 5 — Async side-effect helper

Model: [__tests__/inngest/](../../../__tests__/inngest/) testing exported helpers rather than the whole function.
```ts
// Illustrative fake code: not from this repo
import { shouldSkipBulkEnrichment } from "@/inngest/functions/enrich-contact";
it("skips within 7 days, runs after", () => {
  expect(shouldSkipBulkEnrichment(new Date(Date.now() - 3*864e5))).toBe(true);
  expect(shouldSkipBulkEnrichment(new Date(Date.now() - 8*864e5))).toBe(false);
  expect(shouldSkipBulkEnrichment(null)).toBe(false);
});
```
Teaches: extract the decision, test the decision. Don't try to test the whole Inngest function in Jest — test its logic and E2E the wiring.

## Recipe 6 — Crypto round-trip

Model: [__tests__/lib/email-crypto.test.ts](../../../__tests__/lib/email-crypto.test.ts). Env var seeded by [jest.env.setup.ts](../../../jest.env.setup.ts).
```ts
// Illustrative fake code: not from this repo
it("round-trips and rejects tampering", () => {
  const ct = encrypt("sk-secret");
  expect(decrypt(ct)).toBe("sk-secret");
  const tampered = Buffer.from(ct, "base64"); tampered[20] ^= 1;
  expect(() => decrypt(tampered.toString("base64"))).toThrow();  // GCM auth tag catches it
});
```
Run: `pnpm test -- email-crypto`. Teaches: authenticated encryption should be tested for the *rejection*, not just the round-trip.

## Recipe 7 — E2E denial path (Playwright)

Most E2E specs test happy paths. The high-value missing case is a *denial*. Sketch (needs a second seeded low-priv user):
```ts
// Illustrative fake code: not from this repo — tests/e2e/authz.spec.ts
test("user cannot open another user's account detail", async ({ page }) => {
  await page.goto(`/en/crm/accounts/${OTHER_USERS_ACCOUNT_ID}`);
  await expect(page.getByText(/not found|forbidden/i)).toBeVisible();  // 404-fold
});
```
Run: `pnpm test:e2e -- authz`. Note this depends on the read path being scoped (it is); the same test against `updateAccount` would currently **fail to deny** — which is how a test proves the bug in [risk #1](../09-reference/risk-register.md).

## General workflow

1. Find the nearest existing test as a template (`ls __tests__` + colocated dirs).
2. Prefer testing an extracted pure function over mocking Prisma.
3. Run the single file: `pnpm test -- <substring>`.
4. For anything authz/money/status, add both the positive and the *negative* case — the negative is where bugs hide.
