# Type System and Contracts

## The rule this repo half-follows: `unknown` at boundaries, inference inside

**Good shape** — [create-invoice.ts:15,23](../../../actions/invoices/create-invoice.ts#L15-L23): `raw: unknown` → `createInvoiceSchema.parse(raw)` → fully typed. The Zod schema in [types/invoice.ts](../../../types/invoice.ts) is the single source of truth; TS types are *derived* (`z.infer`), so validation and types can't drift.

**Bad shape** — [update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33): a 25-field inline TS type as the parameter. At an RPC boundary this is a promise nobody keeps: the wire can carry anything. No runtime check means `annual_revenue: {"$gt": ""}`-style garbage flows straight into Prisma (Prisma will throw on wrong shapes, but the *error path* becomes your validator — 500s instead of 400s).

Transferable rule: **a type annotation is a contract only within the compiled program; at process boundaries you need a parser.** Zod's `parse` is "parse, don't validate" — output type differs from input type (`unknown` → `T`).

## Generics worth reading

- [create-safe-action.ts:13-28](../../../lib/create-safe-action.ts#L13-L28): `createSafeAction<TInput, TOutput>` — a higher-order function threading two type params through schema, handler, and `ActionState`. `FieldErrors<T>` is a mapped type (`[K in keyof T]?: string[]`). Small, real, and a perfect whiteboard example. (Investigate: `rg "createSafeAction" actions` to see how consistently it's actually adopted — many actions bypass it.)
- [serialize-decimals.ts:5](../../../lib/serialize-decimals.ts#L5-L21): `<T>(obj: T): T` is a **lie the codebase accepts knowingly** — the function changes Decimal fields to number, so the output is *not* T. It's ergonomic (callers keep their types) at the cost of honesty. A senior can defend or attack this tradeoff; know both sides.
- Deriving a where-type from the client: [crm.ts:5-10](../../../lib/authz/scopes/crm.ts#L5-L10) — `NonNullable<Parameters<typeof prismadb.crm_Contacts.updateMany>[0]>["where"]`. Utility-type composition to stay aligned with generated Prisma types instead of hand-writing them.
- Return-type inference as API: `type CrmData = Awaited<ReturnType<typeof getAllCrmData>>` ([AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>)) — the client view's prop type tracks the server function automatically.

## Narrowing and its evasions

- Honest narrowing: [serialize-decimals.ts:10-16](../../../lib/serialize-decimals.ts#L10-L16) duck-types `toNumber` with `in` + `typeof` checks before calling.
- Evasion: `data: any[]` on [AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) props, and the `(prismadb as any).crm_AuditLog` cast in [audit-log.ts:69](../../../lib/audit-log.ts#L68-L70) (comment suggests generated-client lag). Each `any` is a hole where refactors stop being checked. `rg ": any" actions app lib | wc -l` gives you the repo's honesty score — try it.
- Enum-ish unions: `InvoiceStatus` as a string-literal union ([permissions.ts:3-5](../../../lib/invoices/permissions.ts#L3-L5)) mirrors, but is not derived from, the Prisma enum — drift risk between the two definitions (investigate: compare with schema's invoice status enum).

## Where contracts actually live

| Contract | Enforced by | File |
| --- | --- | --- |
| Invoice action inputs | Zod | [types/invoice.ts](../../../types/invoice.ts) |
| Enrichment PATCH fields | allowlist map | [contacts/[id]/route.ts:10-20](../../../app/api/crm/contacts/%5Bid%5D/route.ts#L10-L20) |
| MCP tool args | Zod (`tool.schema.shape`) | [mcp/[transport]/route.ts:8](../../../app/api/mcp/%5Btransport%5D/route.ts#L8) |
| DB shape | Prisma schema + migrations | [prisma/schema.prisma](../../../prisma/schema.prisma) |
| MCP error codes | string prefix convention | [mcp/[transport]/route.ts:15-27](../../../app/api/mcp/%5Btransport%5D/route.ts#L15-L27) — stringly-typed; a `const enum`-style error object would be sturdier |

## Pitfall checklist

- [ ] Server action parameter typed but not parsed? Boundary hole.
- [ ] `as` cast near Prisma or JSON? Find out what invariant makes it safe, write it down.
- [ ] Union duplicated from a Prisma enum? Derive or add a drift test.
- [ ] `any[]` props on a heavy component? The columns file next to it is silently unchecked.

## Drills

1. Write (on paper) `updateAccountSchema` in Zod for the field list at [update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33), then refactor the action signature to `raw: unknown`. What breaks at callsites? (Answer: nothing if callers already send that shape — that's the point.)
2. Explain the difference between `z.Schema<TInput>` in create-safe-action and `z.infer<typeof schema>` in invoice types. When does each direction (schema-from-type vs type-from-schema) fight you?

## Interview angle

- "How do you validate input in a TS backend?" → parse-don't-validate with the create-invoice anchor. [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q8–Q10.
- "Generics question" → walk `createSafeAction` from memory; it's 15 lines.
- "What's wrong with `any`?" → cite a concrete refactor this repo can't safely do because of `data: any[]`.
