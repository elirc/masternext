# Framework Mental Models — Next.js 16 App Router + React 19

## The core split: server components render data, client components render interaction

Every CRM page follows one shape: an async **server component** fetches via server actions and passes plain data down to a `"use client"` view.

- Server side: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) — `await getAccounts()` directly in the component body. No useEffect, no loading-state dance, no API route.
- Client side: [AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) — `useState` for the Sheet, `useTranslations`, TanStack Table. It receives `data` as props and never fetches.

Mental model: the RSC tree is a **template rendered on the server per request**; client components are **islands** hydrated in the browser. Props crossing the island boundary must be serializable — which is exactly why [serialize-decimals.ts](../../../lib/serialize-decimals.ts) exists.

## Server actions are the mutation layer

`"use server"` files in [actions/](../../../actions/) are RPC endpoints in disguise: Next generates a POST endpoint per exported function. Consequences this repo demonstrates:

1. **They are public attack surface.** Anyone with a session can invoke `updateAccount` with any id — which is why the missing scope check ([update-account.ts:34-49](../../../actions/crm/accounts/update-account.ts#L34-L49)) is a real vulnerability, not a theoretical one. "It's only called from our UI" is never true for server actions.
2. **Validate inside, not at the callsite.** [create-invoice.ts:23](../../../actions/invoices/create-invoice.ts#L23) takes `raw: unknown` and Zod-parses. Compare `updateAccount`'s trusting typed parameter — TS types are compile-time fiction at an RPC boundary.
3. **Mutation → revalidation.** `revalidatePath("/[locale]/(routes)/crm/accounts", "page")` ([update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62)) tells Next to re-render that route segment on next request. Without it the RSC cache serves stale data. Note it revalidates the *route pattern*, covering all locales.

## Caching layers you must be able to name

| Layer | Where seen | Invalidation |
| --- | --- | --- |
| React `cache()` per-request dedup | [get-accounts.ts:5](../../../actions/crm/get-accounts.ts#L5) — two components calling `getAccounts()` in one request hit the DB once | automatic, request-scoped |
| Router cache / RSC payload | every navigation | `revalidatePath` after mutations |
| `global.cachedPrisma` | [lib/prisma.ts:40-44](../../../lib/prisma.ts#L40-L44) | dev-only, process lifetime |

`cache()` also gives a subtle correctness property: `requireAuthenticated` inside a cached function runs once per request, so authz stays consistent within one render pass.

## React 19 specifics in play

- `use client` components use hooks as usual; there's no Redux — global client state is **jotai** (small atoms) plus URL/state props. Look at [context/](../../../context/) and `rg "useAtom" app components` for instances.
- Forms: react-hook-form + `@hookform/resolvers` Zod — the same Zod schema shape used server-side, giving parallel client/server validation (belt and suspenders; only the server one is a security boundary).
- Suspense: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) wraps the view in `<Suspense fallback={<CrmAccountsSkeleton/>}>`; `loading.tsx` files provide route-level fallbacks.

## i18n as a routing concern

Every page lives under `app/[locale]/…`; [proxy.ts](../../../proxy.ts#L53-L54) delegates to next-intl's middleware for locale negotiation, and server components call `getTranslations` ([accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>)) while client components call `useTranslations` ([AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>)). Adding a user-visible string means touching [locales/](../../../locales/) for en + cz.

## Failure modes to recognize

- Fetching in a client component with useEffect when an RSC could do it → waterfall + loading flicker. This repo largely avoids it; SWR appears only where interactivity demands client fetching.
- Forgetting `revalidatePath` → "my save worked but the list is stale."
- Passing non-serializable values (Decimal, Date is fine, class instances aren't) into a client component → runtime error only on the pages that hit it.
- Treating a server action's TS signature as validation → see #2 above.

## Drills

1. Trace what physically happens (network tab level) when you submit the NewAccountForm: one POST to the current route with a `Next-Action` header, not a REST call. Explain to a duck why CSRF risk is mitigated (same-origin action ids) but authz still isn't.
2. Find one page that does two independent awaits serially and write the `Promise.all` refactor. Measure nothing; just argue the latency math.

## Interview angle

- "RSC vs client components?" → [08-interview-prep/02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md) Q1–Q4, anchored to accounts page.
- "How do server actions work under the hood, and what are their security implications?" → Q5; cite `updateAccount` as the cautionary tale.
- "Explain Next.js caching" → Q6; the three-layer table above is a complete mid-level answer.
