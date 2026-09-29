# Annotation Drills

For each excerpt: open the anchor, and annotate **(a)** inputs and their trust level, **(b)** outputs, **(c)** dependencies, **(d)** invariants it maintains or assumes, **(e)** side effects, **(f)** failure modes. Write it down; then self-grade with the rubric at the bottom. Key observations follow each drill — cover them until done.

## Drill 1 — [lib/invoices/numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) `consumeNextNumber`

Key observations: input `tx` is *typed* as any client but *assumed* Serializable (invariant lives at the caller); yearly reset couples correctness to `now` (UTC — a New-Year's-Eve issuance in UTC+1 gets the old year's counter reset behavior worth pondering); side effect = counter row update; failure mode = serialization abort (no retry) and `findUniqueOrThrow` on a deleted series.

## Drill 2 — [lib/audit-log.ts:36-56](../../../lib/audit-log.ts#L36-L56) `diffObjects`

Key observations: JSON.stringify equality — `Date` vs ISO string compare unequal→ phantom diffs if `before`/`after` sources differ in type; `undefined` values vanish in stringify (`{a: undefined}` vs `{}` compare equal — fine); key union via two passes + `seen` set; INTERNAL_FIELDS blocklist contains a typo'd legacy field `cratedAt` ([:33](../../../lib/audit-log.ts#L31-L34)) that documents real schema history; pure function, no side effects — that's *why* it's the testable half of the audit feature.

## Drill 3 — [lib/api-tokens.ts:47-65](../../../lib/api-tokens.ts#L47-L65) `validateApiToken`

Key observations: input = attacker-controlled string; prefix check is a cheap filter, hash lookup is the real gate; expiry compared against `new Date()` per call; the fire-and-forget update means the function's *type* (`Promise<string>`) hides a background write; failure mode = all invalid cases collapse to one generic error (deliberate — no oracle for attackers).

## Drill 4 — [inngest/functions/enrich-contact.ts:120-138](../../../inngest/functions/enrich-contact.ts#L120-L138) apply-updates block

Key observations: merges into fields *empty at read time*, not write time — the invariant "never overwrite user data" holds only if no human edits during the agent's multi-second run; `String(enrichment.value)` coerces agent output (could stringify an object into a column); the update lacks a scoped where (background context — authorization was the trigger's job).

## Drill 5 — [app/api/upload/presigned-url/route.ts:16-41](../../../app/api/upload/presigned-url/route.ts#L16-L41)

Key observations: `path.basename` kills traversal; folder allowlist with silent fallback (invalid folder → "uploads", not 400 — debatable); content-type allowlist checks the *declared* type — the actual uploaded bytes are enforced by S3 signature to match ContentType header, but nothing sniffs magic bytes; key is UUID-based → uploads are unguessable and collision-free; extension derived from filename with "bin" fallback.

## Drill 6 — [lib/authz/scopes/crm.ts:402-424](../../../lib/authz/scopes/crm.ts#L402-L424) `documentReadScopeWhere`

Key observations: the OR union is the *whole* sharing model (owner, legacy owner, assignee, public, or linked-entity access); every branch is a potential performance cost (deep nested `some` joins); adding a new link type means remembering this file — invariant maintained by convention; note the comment discipline ("legacy duplicate").

## Drill 7 — [actions/invoices/create-invoice.ts:32-53](../../../actions/invoices/create-invoice.ts#L32-L53) tax-rate resolution

Key observations: fetches only the tax rates referenced by input (`in` filter — no N+1); unknown `taxRateId` silently becomes rate 0 ([:49-51](../../../actions/invoices/create-invoice.ts#L49-L51)) rather than 400 — a validation hole worth a ticket (an invoice with a typo'd rate id computes 0% VAT); Decimal built from `.toString()` to avoid float contamination.

## Drill 8 — [proxy.ts:31-51](../../../proxy.ts#L31-L51)

Key observations: order matters — inngest/auth pass-throughs precede cookie checks; `path.includes(p)` for auth pages is substring matching (a route named `/x-sign-in-y` would pass — harmless today, sloppy tomorrow); API routes not in ADMIN_ONLY_PATHS get **no middleware auth at all** — every API handler is on its own (which is the design, but you must *know* it).

## Drill 9 — [lib/email-crypto.ts:21-42](../../../lib/email-crypto.ts#L21-L42) encrypt/decrypt

Key observations: AES-256-GCM = authenticated encryption (tamper → decrypt throws); fresh random IV per encryption (never reuse with GCM — catastrophic); output layout iv‖tag‖ciphertext must match between the two functions (implicit shared invariant); key validated as 64 hex chars at *use* time, so a bad env var fails on first crypto op, not at boot (investigate: is there a boot-time env check anywhere?).

## Drill 10 — [inngest/functions/campaigns/send-step.ts:34-51](../../../inngest/functions/campaigns/send-step.ts#L34-L51)

Key observations: `resolveMergeTags` output goes into `html` unescaped (see risk register); from-address is env-fixed with display-name from campaign — replies go to `reply_to` only if set; the List-Unsubscribe header uses `NEXTAUTH_URL` env (legacy name — better-auth migration leftover, investigate whether it's set in .env.example); the send itself is inside `step.run` = retry-safe.

## Self-grading rubric (per drill)

- **Basic**: named inputs/outputs and one failure mode.
- **Solid**: identified the invariant AND who maintains it (this function, its caller, or convention).
- **Strong**: found something not listed in the key observations, or correctly disputed one — and could say what test would pin the behavior.
