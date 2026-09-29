# Behavioral STAR Stories

10 worksheets sourced from studying this repo and doing its tickets. Even without having authored NextCRM, "I did a deep study / made a contribution to a production-grade open-source CRM" is a legitimate, strong framing. Fill each in your own words; the evidence anchors make them concrete.

Format: Situation / Task / Action / Result + evidence + senior-signal detail + one-line resume bullet. Rehearsal check: under 2 minutes, concrete, ends with impact.

---

## Story 1: Learning a large unfamiliar codebase fast
Prompts: "walk me through a complex codebase you learned"; "how do you onboard."
S: Handed a ~1400-file Next.js 16 CRM with no docs and three legacy naming strata. T: become productive enough to contribute safely. A: mapped it by ownership layer, traced 7 end-to-end flows, and read the migration history as the project's changelog before touching anything. R: could locate any feature and made a first tested contribution.
Evidence: the system map + key flows I built ([01-codebase-cartography](../01-codebase-cartography/)). Senior signal: I read `git`/migration history and the repo's own security audit before forming opinions. Resume: *Reverse-engineered a 1400-file Next.js/Prisma CRM into a documented architecture map and shipped a tested first contribution.*

## Story 2: Finding a security bug
Prompts: "tell me about a bug you found"; "a time you improved security."
S: while studying the CRM's authorization I noticed reads were scoped but some server-action writes weren't. T: determine if it was exploitable and fixable. A: confirmed against the repo's own advisory (GHSA-mg5f-m89f-4gmc), traced the fixed API route's scoped-`updateMany` pattern, and scoped the vulnerable write the same way — one entity, with a denial test. R: closed an IDOR path following the established remediation pattern.
Evidence: [update-account.ts](../../../actions/crm/accounts/update-account.ts) vs [contacts route](../../../app/api/crm/contacts/%5Bid%5D/route.ts). Senior signal: matched the existing fix pattern instead of inventing one, kept the PR to one entity. Resume: *Closed an object-level authorization gap in a CRM by extending its scoped-query remediation to an unprotected write path.*

## Story 3: A technical tradeoff I weighed
Prompts: "a difficult technical decision."
S: hardening invoice numbering against duplicate numbers. T: choose a concurrency-safe approach. A: compared Serializable-plus-retry against `SELECT FOR UPDATE`, reasoned about month-end contention on the series row, and recommended FOR-UPDATE in the helper with a load test to revisit. R: a fix where the invariant is owned by the function, not every caller.
Evidence: [numbering.ts](../../../lib/invoices/numbering.ts) + [refactor kata A](../06-contribution-practice/04-refactor-and-design-katas.md). Senior signal: I named the caller-owned-invariant anti-pattern and deferred the throughput question to measurement. Resume: *Eliminated a duplicate-invoice-number race by relocating a lock into the owning function.*

## Story 4: Disagreeing with a proposed change
Prompts: "a time you disagreed with a teammate/senior."
S: a proposal to drop Serializable from invoice issuance "for speed." T: prevent a correctness regression without shutting the person down. A: acknowledged the latency concern, showed the concrete failure (duplicate legal numbers), and offered a faster-and-safe alternative plus a load test to settle it with data. R: the change became the right change instead of a regression.
Evidence: [review kata 2](../04-code-reading-gym/04-review-katas.md). Senior signal: disagreed with evidence and a path forward, not just "no." Resume: *Turned a well-intentioned but unsafe optimization into a correct one through evidence-based review.*

## Story 5: Improving test coverage / quality
Prompts: "a time you improved quality"; "worked on something without immediate payoff."
S: the authorization scope functions — the app's security core — had zero tests. T: make accidental widening of access detectable. A: added snapshot tests pinning each role's scope-where, following the repo's pure-function test style. R: any future broadening of `user`-role access now fails fast.
Evidence: [recipe 4 / ticket 3](../06-contribution-practice/01-good-first-tickets.md). Senior signal: I tested the highest-risk untested thing, not the easiest. Resume: *Added regression tests around authorization scopes that previously had no coverage.*

## Story 6: Dealing with ambiguity
Prompts: "a time requirements were unclear."
S: a PDF might render live account data despite a billing snapshot existing — unclear if bug or intent. T: decide without guessing. A: traced both the snapshot write and the PDF render path, framed it as an investigation in the risk register with a confidence label rather than asserting a bug, and proposed confirming before changing legal output. R: a documented, evidence-based question instead of a risky fix.
Evidence: [risk #10](../09-reference/risk-register.md). Senior signal: labeled confidence and refused to over-claim on legal artifacts. Resume: *Investigated and documented a potential invoice-rendering discrepancy with appropriate uncertainty before proposing changes.*

## Story 7: A mistake / something you'd do differently
Prompts: "tell me about a mistake."
S/T: (use a real one from doing a ticket — e.g., a first attempt at a scoped write that returned the row via a second query, missing that `updateMany` doesn't return rows). A: caught it in self-review, switched to updateMany+count then a scoped read. R: correct and atomic. Senior signal: found it before review by writing the test first. Resume: *(personalize)*.

## Story 8: Handling a lot of work / prioritizing
Prompts: "how do you prioritize."
S: a backlog of improvements across authz, tests, and observability. T: sequence them. A: prioritized by blast radius and dependency — CI first (safety net), then authz enforcement, then consistency cleanup, because refactors without CI are dangerous. R: a defensible 90-day plan.
Evidence: [architecture critique 90-day plan](../03-architecture-and-patterns/06-architecture-critique.md). Senior signal: sequencing logic (why CI before refactors). Resume: *Prioritized a technical-debt backlog by risk and dependency into a staged remediation plan.*

## Story 9: Teaching / helping someone
Prompts: "a time you helped a teammate grow."
S: (from writing/using this curriculum) explaining the difference between authentication and object-level authorization using the CRM's real IDOR. T: make an abstract concept concrete. A: walked through the vulnerable vs fixed handler side by side. R: the concept stuck because it was anchored to real, consequential code. Senior signal: taught with a real failure, not a definition. Resume: *Mentored on secure authorization patterns using a real remediated vulnerability as the teaching example.*

## Story 10: Taking initiative / ownership
Prompts: "went beyond your assignment."
S: while doing a small ticket I noticed the audit trail was inconsistent between the API route and the server-action path. T: decide whether to expand scope. A: kept the original PR small, filed the audit inconsistency as a separate follow-up with evidence rather than sneaking it in. R: both got fixed, cleanly, without an unreviewable PR. Senior signal: resisted scope creep and used the issue tracker. Resume: *Identified and separately tracked a cross-path audit-logging inconsistency while keeping PRs focused.*

---

## Rehearsal protocol

For your top 4 (pick from 1, 2, 3, 4/8), record yourself. Cut to under 2 minutes. Every story must end on **impact**, cite **one concrete anchor**, and contain **one senior-signal detail** (a tradeoff weighed, a risk reduced, a person helped). Map each to the prompt families interviewers use: conflict (4), ambiguity (6), mistake (7), tradeoff (3), leadership/initiative (10), learning (1), quality (5). If you can deliver 1–5 cleanly you can handle 90% of a behavioral round.
