# 08 — Interview Prep

Target: **mid-level fullstack JS interviews**. This module is first-class, not an afterthought. ≥60% of the question cards are anchored to real NextCRM code so you practice answering with concrete evidence — the thing that separates a hire from a "knows the definitions."

## Files

1. [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) — 15 cards: event loop, async, closures, TS, Node
2. [02-frontend-framework-questions.md](02-frontend-framework-questions.md) — 12 cards: RSC, state, effects, data fetching, perf, a11y
3. [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) — 13 cards: REST/RPC, validation, pagination, authz, schema, tx, caching
4. [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — full whiteboard walkthrough + variations
5. [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) — timed debugging + review simulations
6. [06-behavioral-star-stories.md](06-behavioral-star-stories.md) — 10 STAR worksheets sourced from this repo
7. [07-two-week-cram-plan.md](07-two-week-cram-plan.md) — day-by-day

That's ~40 cards here + the design walkthrough + review/debug rounds → well past 40 total, with 8+ counting as full system-design/behavioral exercises.

## How mid-level fullstack loops are usually structured

| Round | What they test | Your NextCRM ammo |
| --- | --- | --- |
| Recruiter/phone screen | can you communicate, basic fit | the elevator description of what you studied |
| Technical deep-dive | JS/TS/framework depth | files 01–02 |
| Practical coding | build/modify something live | the tickets in [06-contribution-practice](../06-contribution-practice/) |
| System design | can you reason about tradeoffs | file 04 + [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) |
| Behavioral | judgment, collaboration | file 06 |

## Use this repo as your portfolio of talking points

You don't need to have *authored* NextCRM to talk about it — "I did a deep study of a production-grade open-source CRM" is a legitimate, strong frame. You can walk through a real IDOR remediation, a Serializable-transaction numbering system, an AI enrichment pipeline, and an MCP server, all with file-level specificity. That specificity is rare in candidates and disproportionately convincing.

## The golden rule

Never answer with a definition. Answer with **a concrete example, a tradeoff, and a failure mode.** "Idempotency means an operation you can run twice safely — like the campaign webhook here, which guards status updates with `if (send.status === 'sent')` so redelivery is harmless, but it lacks event-id dedup so a future counter-increment handler would double-count." That's the shape of every strong answer. Definitions are junior; examples-with-tradeoffs are mid; knowing-when-the-standard-answer-is-wrong is senior.
