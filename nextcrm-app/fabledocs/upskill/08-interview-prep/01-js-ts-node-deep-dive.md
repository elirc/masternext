# JS / TS / Node Deep Dive — Question Cards

15 cards. Format: question → what it tests → repo anchor → junior/mid/senior answer → follow-ups → drill.

---

## Q1: Explain the event loop and where `await` yields control.
Round: JS deep-dive. Tests: async fundamentals.
Repo anchor: [issue-invoice.ts:22-54](../../../actions/invoices/issue-invoice.ts#L22-L54) — network I/O awaited outside the transaction.
Junior: "await pauses the function until the promise resolves." Mid: adds that the function yields to the event loop at each await, so nothing between two awaits is atomic — cites keeping FX fetch outside the transaction so the DB isn't holding locks while the event loop services other work. Senior: distinguishes microtask (promise) vs macrotask (timer/I-O) queues, notes that a CPU-bound loop between awaits still blocks everything, and that "outside the transaction" matters because an awaited network call inside would hold a DB connection + locks for the round-trip.
Follow-ups: microtask vs macrotask ordering; what starves the loop. Drill: say Q1 in 90s citing the anchor.

## Q2: What's the difference between serial and parallel awaits, and when does it matter here?
Tests: latency reasoning. Anchor: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) — `getAllCrmData()` then `getAccounts()` serially.
Junior: "you can use Promise.all to run them together." Mid: identifies these two as independent → `Promise.all` halves latency; but a dependent pair must stay serial. Senior: notes parallelism only helps I/O-bound independent work, warns that `Promise.all` fails fast (one rejection rejects all — use `allSettled` when partial results are acceptable), and that inside a single Prisma transaction "parallel" queries still serialize on one connection.
Drill: find one serial-await pair in `app/**/page.tsx` and justify the refactor.

## Q3: What is an unhandled promise rejection and how does this repo handle fire-and-forget?
Tests: async error hygiene. Anchor: [update-account.ts:61](../../../actions/crm/accounts/update-account.ts#L61) (`void inngest.send`), [api-tokens.ts:60-62](../../../lib/api-tokens.ts#L59-L62) (`void Promise.resolve(...).catch(()=>{})`).
Junior: "you should always await promises." Mid: explains `void` signals intentional non-awaiting and the `.catch` prevents an unhandled rejection (which crashes Node ≥15); the trade is losing the event on crash. Senior: names this the missing-outbox problem — fire-and-forget events aren't durable; proposes an outbox if the event carried money, but defends declining it for search-embedding refresh.
Drill: explain why removing the `.catch` at api-tokens.ts:62 is a latent crash.

## Q4: Explain closures with a real example.
Tests: fundamentals. Anchor: [lib/prisma.ts:35-46](../../../lib/prisma.ts#L35-L46) module singleton; [create-safe-action.ts:13-28](../../../lib/create-safe-action.ts#L13-L28) returns a closure over `schema`+`handler`.
Junior: "a function that remembers variables from its scope." Mid: `createSafeAction` returns an async function closing over `schema` and `handler` — each call re-parses without re-passing them; that's a closure as a currying/config device. Senior: connects to the dev Prisma singleton (`global.cachedPrisma` closed over across hot reloads) and warns about the classic stale-closure bug in React effects (Q from frontend set).
Drill: rewrite `createSafeAction` from memory.

## Q5: How do JS modules and the Node/edge/browser split constrain this codebase?
Tests: module system + runtime boundaries. Anchor: [proxy.ts:8-9](../../../proxy.ts#L8-L9) (middleware can't hit DB), [lib/prisma.ts](../../../lib/prisma.ts) (Node-only).
Junior: "some code runs on the server, some on the client." Mid: middleware runs in the edge runtime → no Prisma/Node crypto, which is exactly why [proxy.ts](../../../proxy.ts) only checks cookie presence and defers roles to the server. Senior: notes the module graph *enforces* this — importing `pg` from a `"use client"` file is a build error; `NEXT_PUBLIC_` is the only env that crosses to the browser.
Drill: predict three modules that would break if imported into middleware.

## Q6: TypeScript — `unknown` vs `any`, and where does this repo use each?
Tests: type discipline. Anchor: [create-invoice.ts:15,23](../../../actions/invoices/create-invoice.ts#L15-L23) (`raw: unknown` + parse) vs `data: any[]` on [AccountsView](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>).
Junior: "any turns off type checking." Mid: `unknown` forces a narrowing/parse before use (good at boundaries); `any` silently propagates — the `any[]` props mean the column defs are unchecked. Senior: "parse, don't validate" — Zod turns `unknown` into a typed value; cites that `any` blocks safe refactors (can't rename a field and trust the compiler).
Drill: `rg ": any" lib actions app | wc -l` and pick the most dangerous one to defend removing.

## Q7: Describe a race condition you understand deeply.
Tests: concurrency. Anchor: [numbering.ts:13-30](../../../lib/invoices/numbering.ts#L13-L30) + [issue-invoice.ts:128](../../../actions/invoices/issue-invoice.ts#L128).
Junior: "two things happen at once and cause a bug." Mid: read-modify-write on the counter; two concurrent issuances read the same value → duplicate invoice numbers; solved by Serializable isolation. Senior: the invariant is caller-owned (a future caller without Serializable reopens it); better to relocate the lock into the function (`SELECT FOR UPDATE`); notes Serializable aborts need a retry loop the code lacks.
Follow-up: Serializable vs FOR UPDATE tradeoffs. Drill: this is [debugging scenario 2](../05-quality-engineering/03-systematic-debugging.md) — narrate the reproduction.

## Q8: How do you validate untrusted input in a TS backend?
Tests: boundary thinking. Anchor: Zod in [create-invoice.ts:23](../../../actions/invoices/create-invoice.ts#L23); missing in [update-account.ts:8-33](../../../actions/crm/accounts/update-account.ts#L8-L33).
Junior: "TypeScript types check it." Mid: TS is compile-time only; at an RPC boundary (server actions are public POST endpoints) you need runtime parsing — Zod. Cites the legacy action that trusts its typed parameter as the anti-pattern. Senior: parse-don't-validate (output type ≠ input type), and notes server actions are attack surface regardless of who "should" call them.
Drill: write the Zod schema for the update-account field list.

## Q9: Explain a TS generic you could write from scratch.
Tests: generics fluency. Anchor: [create-safe-action.ts:3-28](../../../lib/create-safe-action.ts#L3-L28) — `FieldErrors<T>` mapped type + `createSafeAction<TInput,TOutput>`.
Junior: reads it. Mid: explains the mapped type `{[K in keyof T]?: string[]}` and how two type params thread schema→handler→result. Senior: contrasts with the deliberately-dishonest `serializeDecimals<T>(x:T):T` ([serialize-decimals.ts:5](../../../lib/serialize-decimals.ts#L5)) that lies for ergonomics, and can argue both sides.
Drill: whiteboard `FieldErrors<T>` and a `Result<T,E>` type.

## Q10: What does `satisfies` / `as const` / narrowing buy you? (TS)
Tests: modern TS. Anchor: `ALLOWED_FOLDERS = [...] as const` ([presigned-url:9-10](../../../app/api/upload/presigned-url/route.ts#L9-L10)); duck-typed narrowing at [serialize-decimals.ts:10-16](../../../lib/serialize-decimals.ts#L10-L16).
Junior: partial. Mid: `as const` makes the folder list a readonly tuple of literals so `AllowedFolder` is a precise union; the `in` + `typeof` checks narrow `unknown` before calling `toNumber`. Senior: `satisfies` would validate a config object against a type without widening it; explains why `includes` on an `as const` tuple needs a cast (readonly array `includes` narrowing quirk).
Drill: find where the folder union is used for the allowlist check.

## Q11: Errors as values vs exceptions — how does this repo do both?
Tests: error modeling. Anchor: [create-safe-action.ts](../../../lib/create-safe-action.ts) envelope vs thrown `AuthenticationError` ([session.ts:11-23](../../../lib/authz/session.ts#L11-L23)) translated at boundaries.
Junior: "try/catch." Mid: domain throws typed errors; boundaries translate — actions to `{error}` returns or thrown `Error("Unauthorized")` ([create-invoice.ts:17-30](../../../actions/invoices/create-invoice.ts#L17-L30)), routes to 401/404. Senior: notes the *two* error dialects (envelope vs ad-hoc `{error}`) as a real inconsistency that can drop errors in the UI.
Drill: trace an `AuthorizationError` from throw to HTTP status.

## Q12: How do you handle a CVE in a transitive dependency?
Tests: supply chain. Anchor: [package.json overrides:155-192](../../../package.json#L155-L192), `onlyBuiltDependencies:193-206`.
Junior: "run npm audit fix." Mid: pnpm `overrides` force safe versions across the whole graph without waiting for upstream; `onlyBuiltDependencies` blocks install scripts from untrusted packages. Senior: overrides are debt with an expiry (they mask what upstream ships); you track and remove them as real fixes land.
Drill: explain the three `tar@` range pins.

## Q13: Explain AES-GCM vs hashing for storing secrets.
Tests: applied crypto. Anchor: [email-crypto.ts](../../../lib/email-crypto.ts) (AES-256-GCM) vs [api-tokens.ts:8-10](../../../lib/api-tokens.ts#L8-L10) (SHA-256).
Junior: "encrypt the secret." Mid: hash what you only *verify* (API tokens — you never need the plaintext back), encrypt what you must *replay* (OpenAI keys — you send them to OpenAI). GCM is authenticated (tamper→throws). Senior: why SHA-256 (not bcrypt) is fine for 24-random-byte tokens (no dictionary to defend against), and why IV must never repeat with GCM.
Drill: explain why bcrypt on the API token would be pure overhead.

## Q14: Why can't authorization live in Next.js middleware here?
Tests: framework+runtime. Anchor: [proxy.ts:8-9,31-39](../../../proxy.ts#L8-L39) + [session.ts:16-22](../../../lib/authz/session.ts#L16-L22).
Junior: "middleware checks the cookie." Mid: middleware runs on the edge with no DB access, so it can only check cookie *presence*; the real role check re-reads the user row server-side every request. Senior: this buys instant revocation (role from DB, not JWT) at the cost of a query, and means every handler must independently authorize — the middleware is not a security boundary for data.
Drill: what happens when an admin demotes a logged-in user mid-session?

## Q15: How does a durable-execution engine (Inngest) change how you write async code?
Tests: modern async/workflow. Anchor: [send-step.ts:20-68](../../../inngest/functions/campaigns/send-step.ts#L20-L68) (steps) vs [enrich-contact.ts](../../../inngest/functions/enrich-contact.ts) (no steps).
Junior: "it's a job queue." Mid: `step.run` persists each step's result; on retry, completed steps replay from storage instead of re-executing — so the email isn't re-sent after a crash in a later step. Senior: contrasts with the enrichment function that skips steps and thus re-pays the AI bill on every retry; notes step results are JSON (Date→string on replay) and code between steps must be deterministic.
Drill: rewrite enrich-contact as steps and mark which retries stop costing money.
