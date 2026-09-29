# Fake-Code Contrasts

Ten pairs. Every snippet below is **illustrative fake code: not from this repo** unless it's a link. For each: spot what's wrong with A before reading B, then find the real pattern at the anchor.

## 1. Session check ≠ authorization

```ts
// Illustrative fake code: not from this repo (pattern A — the novice instinct)
export async function updateDeal(id: string, data: DealInput) {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");
  return db.deals.update({ where: { id }, data });
}
```
```ts
// Illustrative fake code: not from this repo (pattern B)
export async function updateDeal(id: string, raw: unknown) {
  const user = await requireAuthenticated();
  const data = dealSchema.parse(raw);
  const { count } = await db.deals.updateMany({
    where: { id, ...dealWriteScopeWhere(user) }, data,
  });
  if (count === 0) throw new NotFoundError();
}
```
Real: A is literally [update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49); B is the [tryScopedUpdateContact](../../../lib/authz/scopes/crm.ts#L33-L43) idiom. This repo contains both — that's the lesson.

## 2. Money as floats

```ts
// Illustrative fake code: not from this repo
const lineTotal = qty * price * (1 - discount / 100) * (1 + vat / 100);
grandTotal += Math.round(lineTotal * 100) / 100;
```
Better: Decimal end-to-end with per-line `toDecimalPlaces(2)` — [totals.ts:18-25](../../../lib/invoices/totals.ts#L18-L25). Failure the fake produces: `0.1 + 0.2` class drift and sum-then-round disagreeing with the printed line totals by a cent — an accountant *will* find it.

## 3. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// Client component
const res = await fetch(`/api/accounts`);
setRows(await res.json()); // renders prisma rows, Decimal fields explode
```
Better: server component fetches via scoped action, serializes at the boundary ([get-accounts.ts](../../../actions/crm/get-accounts.ts) + [serialize-decimals.ts](../../../lib/serialize-decimals.ts)), client receives plain data as props. The DB row type stops at the boundary on purpose.

## 4. Check-then-write counter

```ts
// Illustrative fake code: not from this repo
const s = await db.series.findUnique({ where: { id } });
await db.series.update({ where: { id }, data: { counter: s.counter + 1 } });
return format(s.counter + 1);
```
This is [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) *without* the caller's Serializable isolation ([issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128)). Two concurrent issuances → same number on two legal documents. The repo's version is only correct as a pair; the fake shows exactly what the pair prevents.

## 5. Webhook that parses before verifying

```ts
// Illustrative fake code: not from this repo
export async function POST(req: Request) {
  const event = await req.json();          // parsed untrusted bytes
  if (event.secret !== process.env.SECRET) return err();  // attacker supplies the field
  await handle(event);
}
```
Better: raw text → HMAC over exact bytes → parse only after verification — [resend route:13-21](../../../app/api/campaigns/webhooks/resend/route.ts#L13-L21). Also note what B still lacks (timing-safe compare, replay dedup) — even the good example has a next level.

## 6. Swallowed error, no policy

```ts
// Illustrative fake code: not from this repo
try { await audit(entry); } catch {}   // silent, undocumented
```
Better: same swallow, but logged with a grep-able tag and a comment stating the *policy* ("Never rethrow — audit failures must not block CRM mutations") — [audit-log.ts:78-81](../../../lib/audit-log.ts#L78-L81). The difference between a bug and a decision is the sentence explaining it.

## 7. Storing tokens raw

```ts
// Illustrative fake code: not from this repo
await db.apiToken.create({ data: { token: rawToken, userId } });  // DB leak = every token leaks
```
Better: store SHA-256, display a prefix, look up by hash — [api-tokens.ts:29-44](../../../lib/api-tokens.ts#L29-L44). And know the sibling rule: keys you must *replay* to a third party can't be hashed — encrypt those ([api-keys.ts:30-42](../../../lib/api-keys.ts#L29-L44)).

## 8. N+1 authorization

```ts
// Illustrative fake code: not from this repo
const allowed = [];
for (const id of contactIds) {
  if (await canRead(user, id)) allowed.push(id);   // one query per id
}
```
Better: one query, ids + scope in the where — [filterAuthorizedContactIds, crm.ts:126-136](../../../lib/authz/scopes/crm.ts#L126-L136). Same policy, O(1) queries. Bulk endpoints are where per-row authz helpers go to die.

## 9. Stale-closure state update

```tsx
// Illustrative fake code: not from this repo
const [items, setItems] = useState(initial);
async function add(item) {
  await createItem(item);
  setItems([...items, item]);   // `items` captured at render time — concurrent adds lose data
}
```
Better in this repo's architecture: don't mirror server state in client state at all — mutate via server action, `revalidatePath`, let the RSC re-render ([update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62)). Where local state is unavoidable, use the functional form `setItems(prev => [...prev, item])`.

## 10. Casual public-contract change

```ts
// Illustrative fake code: not from this repo (a "harmless" PR)
- const TOKEN_PREFIX = "nxtc__";
+ const TOKEN_PREFIX = "ncrm_";   // "cleaner!"
```
Every issued token now fails the prefix check at [api-tokens.ts:48](../../../lib/api-tokens.ts#L47-L48) — all MCP integrations break with no error trail but "Invalid token". Contract surfaces (prefixes, event names like `"enrich/contact.run"`, tool names, unsubscribe URLs) look like constants and behave like APIs. Before renaming any string literal, grep for who consumes it *outside* the repo.

## Self-grade

Basic = spotted the flaw in ≥7 of 10 before reading the fix. Solid = named the repo anchor from memory for ≥5. Strong = for three pairs, articulated the *test* that would catch the bad version (e.g., #4: parallel issuance integration test asserting distinct numbers).
