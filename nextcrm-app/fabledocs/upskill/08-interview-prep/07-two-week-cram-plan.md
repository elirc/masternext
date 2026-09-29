# Two-Week Cram Plan

For a candidate with an interview in two weeks who has this repo as their study lab. ~2–3 focused hours/day. Adjust to your gaps; the checkpoints matter more than the schedule.

## Week 1 — Build the foundation and the talking points

**Day 1 — Orientation.** [00-fast-track.md](../00-fast-track.md): get it running (or at least trace it), open the first 10 files, trace Flows A and B aloud. Deliverable: you can describe what NextCRM is in 5 sentences.

**Day 2 — The map.** [01-codebase-cartography](../01-codebase-cartography/) 01, 03, 05 (flows 1–3). Deliverable: draw the system map from memory; explain the accounts edit flow including the authz asymmetry.

**Day 3 — JS/TS/Node cards.** [08/01](01-js-ts-node-deep-dive.md) Q1–Q8, out loud, 90s each, citing anchors. Deliverable: Q1, Q7, Q8 fluent.

**Day 4 — Frontend cards.** [08/02](02-frontend-framework-questions.md) Q1–Q6. Re-read [02/02-framework-mental-models](../02-stack-and-language-mastery/02-framework-mental-models.md). Deliverable: RSC-vs-client and server-action-security answers fluent.

**Day 5 — API/data cards.** [08/03](03-api-and-data-modeling-questions.md) Q1–Q8. These are your strongest cards (IDOR, money, numbering). Deliverable: the IDOR answer (Q1) is airtight with the real CVE.

**Day 6 — Timed debugging.** [08/05](05-debugging-and-code-review-rounds.md) Rounds 1–2 against a clock. Deliverable: narrate reproduce→narrow→fix→test without notes.

**Day 7 — CHECKPOINT + mock system design.** Do the full [08/04 system-design walkthrough](04-system-design-from-this-repo.md) end to end, 35 min, out loud/whiteboard. Self-assess against the checklist below. Re-drill any card from Days 3–5 you fumbled.

### Day 7 self-assessment (be honest)
- [ ] Describe NextCRM in 5 sentences without notes.
- [ ] Explain IDOR with the real CVE + the scoped-updateMany fix.
- [ ] Explain the invoice numbering race and two fixes.
- [ ] RSC vs client components + server-action security.
- [ ] Trace one flow end to end aloud.
- [ ] One clean STAR story delivered in <2 min.
If 4+ checked, proceed. If not, repeat the weak days before moving on.

## Week 2 — Depth, breadth, and rehearsal

**Day 8 — Remaining cards.** [08/01](01-js-ts-node-deep-dive.md) Q9–Q15, [08/02](02-frontend-framework-questions.md) Q7–Q12, [08/03](03-api-and-data-modeling-questions.md) Q9–Q13. Deliverable: no card is a blank.

**Day 9 — Architecture judgment.** [03/06 architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) + [03/05 pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md). Deliverable: deliver the 3-strengths / 3-risks / 90-day-plan in 4 min.

**Day 10 — Second mock system design + a variation.** [08/04](04-system-design-from-this-repo.md) main prompt again (should be faster now) + variation 1 (multi-tenancy). Deliverable: variation handled in 5 min.

**Day 11 — Code review round.** [08/05](05-debugging-and-code-review-rounds.md) Review Rounds 1–2 aloud; re-read [07/01 review mindset](../07-career-and-collaboration/01-code-review-mindset.md). Deliverable: kind, specific, precedent-citing review language.

**Day 12 — Behavioral.** [08/06](06-behavioral-star-stories.md): finalize and record your top 4 stories. Deliverable: 4 STARs under 2 min each, each ending on impact.

**Day 13 — Practical coding warmup.** Actually do one Easy ticket from [06/01](../06-contribution-practice/01-good-first-tickets.md) (3, 4, or 15 — pure, fast) on a branch. Deliverable: a green test you wrote. This rehearses the live-coding round's muscle.

**Day 14 — CHECKPOINT + light review.** Second timed debugging round (3 or 4), skim your weakest module, re-record any STAR that ran long. Rest the evening.

### Day 14 self-assessment (interview-ready)
- [ ] Every card in 01–03 answerable at mid-level (example + tradeoff + failure mode).
- [ ] System design + one variation delivered in time.
- [ ] Two debugging rounds narrated cleanly.
- [ ] One review round with kind/specific language.
- [ ] 4 STAR stories, <2 min, impact-first.
- [ ] Wrote and tested one real change on a branch.

## Rules for the two weeks

1. **Out loud, always.** Reading ≠ retrieval. If you can't say it in 90 seconds you don't know it yet.
2. **Anchor every answer.** The file:line specificity is your unfair advantage — use it.
3. **Example → tradeoff → failure mode.** Never a definition. If you catch yourself defining, add the concrete NextCRM example.
4. **The golden four**: IDOR/authz (03/Q1), invoice numbering race (03/Q8), RSC-vs-client (02/Q1), and your best STAR. If you nail these, you're mid-level-credible in any fullstack loop.
