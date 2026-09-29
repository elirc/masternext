# Frontend Framework Questions — Question Cards

12 cards on Next.js 16 App Router + React 19, anchored to real components.

---

## Q1: Server Components vs Client Components — what runs where?
Tests: RSC model. Anchor: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) (server) vs [AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>) (`"use client"`).
Junior: "server components render on the server." Mid: RSC run per-request on the server (can await data directly, no JS shipped), client components hydrate in the browser for interactivity; props crossing the boundary must be serializable. Cites the page fetching via server action and passing plain data to the client view. Senior: notes Decimals must be serialized first ([serialize-decimals.ts](../../../lib/serialize-decimals.ts)), and that pushing `"use client"` too high in the tree ships unnecessary JS.
Drill: identify the exact boundary line in the accounts feature.

## Q2: How do server actions work and what are their security implications?
Tests: the App Router mutation model. Anchor: [update-account.ts](../../../actions/crm/accounts/update-account.ts) + the missing authz.
Junior: "functions you call from the client." Mid: `"use server"` generates a POST endpoint per function invoked via a Next-Action header; therefore they're public attack surface — "only our UI calls it" is false, which is why [update-account.ts](../../../actions/crm/accounts/update-account.ts#L34-L49) missing an object-level check is a real vuln. Senior: adds that CSRF is mitigated by same-origin action ids but authz and input validation are still the developer's job, every time.
Drill: explain why UI-side filtering is never authorization.

## Q3: Explain Next.js caching layers.
Tests: the most-confused Next topic. Anchor: React `cache()` in [get-accounts.ts:5](../../../actions/crm/get-accounts.ts#L5); `revalidatePath` in [update-account.ts:62](../../../actions/crm/accounts/update-account.ts#L62).
Junior: "Next caches pages." Mid: distinguishes request-scoped dedup (`cache()` — two components, one query per request), the router/RSC cache (invalidated by `revalidatePath` after mutations), and that they're different mechanisms. Senior: notes `revalidatePath` here targets the route *pattern* (covers all locales), and that forgetting it is the classic "saved but stale" bug.
Drill: [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md).

## Q4: What's a stale closure in React and how do you avoid it?
Tests: hooks depth. Anchor: [fake-code #9](../04-code-reading-gym/03-fake-code-contrasts.md); real pattern — this repo mostly avoids client state mirroring by using server actions + revalidate.
Junior: partial. Mid: an effect/callback captures a state value from its render; if it runs later it sees the old value — fix with the functional updater `setX(prev => ...)` or correct deps. Notes NextCRM sidesteps it by not mirroring server data in client state. Senior: connects to `useEffect` dependency arrays and why "just add it to deps" can cause loops; prefers deriving over storing.
Drill: write the buggy and fixed versions.

## Q5: How does data fetching work without useEffect here?
Tests: RSC data model. Anchor: [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>) awaits in the component body; SWR ([swr](../../../package.json#L123) dep) used only for client-interactive fetches.
Junior: "useEffect + fetch." Mid: server components await data directly — no loading state, no waterfall, no client fetch; SWR/useSWR appear only where the browser must fetch (live-updating widgets). Senior: contrasts the RSC approach's elimination of client/server waterfalls with the old useEffect pattern, and when you still reach for SWR (mutable client-driven data).
Drill: find one `useSWR` usage (`rg useSWR`) and justify why it's client-fetched.

## Q6: How would you find and fix an expensive re-render in the CRM tables?
Tests: React perf. Anchor: TanStack Table in [data-table.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/table-components/data-table.tsx>); large `getAccounts` result.
Junior: "use React.memo." Mid: profile first (React DevTools), then reduce work — memoize columns, virtualize long tables, avoid recreating callbacks each render. Notes the server ships all rows ([get-accounts has no take](../../../actions/crm/get-accounts.ts)) so the table renders everything. Senior: the real fix may be server-side pagination, not client memoization — measure at real row counts before optimizing.
Drill: [performance-thinking](../05-quality-engineering/04-performance-thinking.md) hotspot list.

## Q7: How is i18n structured and what does adding a string cost?
Tests: real-world framework plumbing. Anchor: `app/[locale]/…`, [proxy.ts:53](../../../proxy.ts#L53), `getTranslations` (server) vs `useTranslations` (client).
Junior: "there's a translations file." Mid: locale is a route segment; server components use `getTranslations`, client components `useTranslations`; a new user-facing string means editing [locales/](../../../locales/) for en + cz. Senior: notes middleware handles locale negotiation and that missing keys are a runtime surprise (no compile check) — a case for typed message keys.
Drill: add a string to the accounts page and list every file you touch.

## Q8: Forms — how does validation work client and server side?
Tests: forms + the security boundary. Anchor: react-hook-form + Zod resolver in NewAccountForm; server-side Zod in invoice actions.
Junior: "react-hook-form validates." Mid: client validation is UX (instant feedback); the server-side parse is the *security* boundary — same Zod shape both sides, but only the server one is trusted. Senior: notes legacy actions skip the server parse (real gap) so client validation is doing security work it can't do.
Drill: which of the two validations can an attacker bypass, and how?

## Q9: What is Suspense doing in these pages?
Tests: React 19 streaming. Anchor: `<Suspense fallback={<CrmAccountsSkeleton/>}>` in [accounts/page.tsx](<../../../app/%5Blocale%5D/(routes)/crm/accounts/page.tsx>); `loading.tsx` files.
Junior: "shows a spinner." Mid: Suspense lets the server stream a fallback while async children resolve; `loading.tsx` provides a route-segment-level fallback automatically. Senior: notes streaming improves TTFB/perceived perf and that placing Suspense boundaries controls what blocks what.
Drill: what renders first when getAccounts is slow?

## Q10: How is global client state managed without Redux?
Tests: state architecture. Anchor: jotai ([jotai](../../../package.json#L91) dep), [context/](../../../context/), local `useState`.
Junior: "useState/useContext." Mid: jotai atoms for small shared state, context for cross-tree concerns, but most "state" is server state managed by re-rendering RSC after `revalidatePath` — so there's less client state than a SPA. Senior: argues server-state-as-source-of-truth reduces the sync bugs Redux exists to manage; jotai for genuinely-client UI state only.
Drill: classify three pieces of state in the app as server vs client state.

## Q11: Accessibility — what does the component layer give you for free, and what doesn't it?
Tests: a11y awareness. Anchor: Radix primitives under [components/ui/](../../../components/ui/); `aria-label` on the add-account button ([AccountsView.tsx](<../../../app/%5Blocale%5D/(routes)/crm/components/AccountsView.tsx>)).
Junior: "add alt text." Mid: Radix (under shadcn) provides focus management, roles, keyboard nav for dialogs/menus for free; but content-level a11y (labels, contrast, table semantics) is still yours — cites the explicit `aria-label`. Senior: notes vendored components can drift from upstream a11y fixes (you own them now), and that data tables need careful header/scope semantics.
Drill: audit the Sheet form for label associations.

## Q12: Since shadcn components are vendored, how do you fix a bug in a Button?
Tests: build-vs-vendor understanding. Anchor: [components/ui/](../../../components/ui/) + [components.json](../../../components.json).
Junior: "override the styles." Mid: the component *source* lives in the repo (copied by the shadcn generator), so you edit it directly like any file — no fork, no patch-package. Trade: upstream fixes don't auto-arrive. Senior: weighs vendoring (velocity, control) vs a dependency (auto-updates, less ownership) and when each wins.
Drill: find `cva` usage in a ui component and explain the variant system.
