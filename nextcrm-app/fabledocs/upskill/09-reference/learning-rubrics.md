# Learning Rubrics

Observable behaviors, not vague traits. Use these to self-assess and to know what "the next level" concretely looks like. The **Interview-ready** column is what an interviewer would score as a strong mid-level answer.

## Skill: Reading unfamiliar code

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Locating | Finds a feature with grep after a few tries | Predicts the layer from the ownership map, then confirms | Knows what a senior inspects first (authz, schema naming, money) | Can narrate the location strategy, not just the result |
| Understanding | Reads a function top to bottom | Identifies inputs/outputs/invariants/side effects | Spots the invariant's *owner* and where it can be violated | Explains a flow end to end citing file:line |
| Evidence | "I think it does X" | "It does X — see file:line" | Labels confidence (confirmed/investigate) | Never over-claims; distinguishes read-in-code from inferred |

## Skill: Authorization

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Concept | Checks the user is logged in | Distinguishes authn vs authz vs object-level | Reasons about atomicity (scope-in-the-where) and enforcement | Explains IDOR with the real CVE + scoped-updateMany fix + residual gap |
| Practice | Adds an `if (session)` | Uses/writes a scope-where or `assertCan*` | Designs enforcement so it can't regress (lint/wrapper/tests) | Cites the 404-vs-403 anti-enumeration choice |

## Skill: Data & transactions

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Money | Uses a decimal column | Per-line rounding + string serialization + why | The per-line-vs-total cent discrepancy | Walks the VAT-cent story with the anchor |
| Concurrency | Unaware of races | Names read-modify-write, uses a transaction | Relocates the invariant into its owner; picks isolation deliberately | Reproduces a race by forcing concurrency, fixes root cause + test |
| Schema change | Edits schema, migrates | Additive-first, checks scope builders | Expand/contract, roll-forward-only reality, compatibility window | States migration + rollback in the PR |

## Skill: Async & reliability

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Idempotency | Unaware | Identifies memoized/conditional/windowed idempotency | Distinguishes at-least-once + effectively-once; names the outbox gap | "Exactly-once email is impossible; here's the effectively-once design" |
| Retries | "It retries" | Knows step memoization saves repeated side effects | Cost-aware retry design (enrichment re-spend) | Rewrites a job as steps, marks what stops costing money |
| Failure policy | Everything fails loudly | Recognizes tolerate/swallow/void as decisions | Adds visibility without changing behavior | Names a specific swallowed-failure blind spot |

## Skill: Testing

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Placement | Tests everything the same way | Unit vs integration vs E2E by cost/value | Extracts logic to make it testable (functional core) | Explains the boundary with `shouldSkipBulkEnrichment` example |
| Coverage | Happy path | Adds the negative/denial case | Tests the highest-risk untested thing first | "I'd add the authz-scope snapshot the repo lacks" |

## Skill: Code review

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Depth | "Does it work?" | Correctness, edge cases, concurrency | "Will it stay correct? Does it fit? Kind to maintainers?" | Names all five layers + "make the codebase converge" |
| Language | Terse/blunt | Specific, cites the issue | Cites in-repo precedent + offers a path + kind | Delivers a real precedent-citing comment |
| Triage | Flags everything equally | Blocking/Important/Optional | Marks optionals as optional; blocks only real risk | Correct B/I/O on a live kata |

## Skill: Communication & judgment

| | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Asking | "It doesn't work" | Shows what tried + specific stuck point | Asks to confirm a decision, having done the reading | Uses the what-I-found-first template |
| Disagreeing | Concedes or digs in | Restates their point, brings evidence | Offers a measurable path, defers where appropriate | Has the FOR-UPDATE-vs-Serializable exchange ready |
| Scope | Sneaks in extra changes | Keeps PRs focused, files follow-ups | Sequences work by risk/dependency | Explains why CI before refactors |

## Self-assessment checklist (are you mid-level on this repo?)

- [ ] I can trace any of the 7 key flows aloud with file anchors.
- [ ] I can explain the IDOR remediation and the residual gap without notes.
- [ ] I can reproduce and fix the invoice-numbering race at the root.
- [ ] I can write a scope-where for a new entity and a denial test for it.
- [ ] I can review a PR with correct B/I/O triage and precedent-citing language.
- [ ] I can deliver the system-design walkthrough + one variation in time.
- [ ] I have 4 STAR stories, each under 2 minutes, ending on impact.
- [ ] I label my confidence and never present inferred behavior as confirmed.

Seven of eight = interview-ready for mid-level fullstack. The eighth (confidence labeling) is the senior habit that makes the other seven trustworthy.
