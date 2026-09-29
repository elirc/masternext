# Maintainer Communication

The skill: get unblocked without outsourcing your thinking, and disagree without friction. NextCRM is an open-source project ([CONTRIBUTING.md](../../../CONTRIBUTING.md), Discord in the [README](../../../README.md)) — these are the interactions that build your reputation there and in any team.

## Asking for help (without making them do your job)

Bad: "How does auth work in this repo?"
Good: "I'm adding a scoped write for opportunities. I see `tryScopedUpdateContact` (lib/authz/scopes/crm.ts:33) does contacts via `updateMany` + ownership OR. For opportunities the ownership is `assigned_to`/`createdBy`/linked-account (opportunityReadScopeWhere, crm.ts:305). Is a `tryScopedUpdateOpportunity` mirroring that the intended pattern, or is there a reason opportunities weren't done yet?"

The template: **what I'm doing → what I already found (anchors) → the specific decision I'm stuck on → my current guess.** You've done the reading; you're asking to confirm a decision, not to be taught. Maintainers answer these fast because they're cheap to answer.

## Reporting a bug (minimal, reproducible)

Structure: environment → steps → expected → actual → evidence (anchor or log) → scope. Example for the numbering race:

> **Bug**: two concurrent `issueInvoice` calls on the same series can produce duplicate numbers.
> **Repro**: fire two `issueInvoice` promises for invoices sharing a series (script below).
> **Expected**: two distinct numbers. **Actual**: occasionally identical.
> **Root cause (hypothesis)**: `consumeNextNumber` (numbering.ts:13) is read-modify-write; safe only under the Serializable wrapper at issue-invoice.ts:128. Any other caller, or a swallowed serialization abort, reopens it.
> **Scope**: legal artifact — flagging before proposing a fix (relocate the lock into consumeNextNumber).

Note it *labels the hypothesis* and doesn't over-claim. That earns trust.

## Proposing a feature/fix

Lead with the problem and evidence, not your solution. "createInvoice silently maps unknown taxRateId to 0% VAT (create-invoice.ts:49) — is that intended, or should it 400? Happy to send a small PR with a test." You're inviting a decision, offering to do the work, and keeping it small.

## Respectful disagreement

When a maintainer pushes back: restate their point first (proves you understood), then your concern with evidence, then defer to their call if it's theirs to make.

> "Agreed that FOR UPDATE is simpler than Serializable+retry, and for our load it's plenty. My only worry is the series row becoming a contention point during month-end bulk issuance — but that's speculative without numbers. Happy to go FOR UPDATE now and add a load test we can revisit. Your call on the tradeoff."

You disagreed, brought evidence, and didn't dig in on something you can't measure yet. That's how mid-level engineers earn senior trust.

## Responding to review on your own PR

- Thank specific catches; don't defend reflexively.
- If you disagree, ask a question rather than assert: "Would a scoped `updateMany` be preferable to assert-then-write here for the atomicity, or is the TOCTOU window fine for ownership that doesn't change mid-request?"
- Push back on scope creep kindly: "Good idea — that's the manager/admin split, which I think deserves its own PR so this one stays reviewable. Filed as a follow-up."

## The anti-pattern: outsourcing your thinking

"It doesn't work, what do I do?" wastes everyone's time and teaches you nothing. Before you ask, you should be able to say what you tried, what you observed, and where you think the problem is. If you can't, you haven't debugged yet — go back to [systematic debugging](../05-quality-engineering/03-systematic-debugging.md) and narrow it first. The question that comes *out* of that process is the good one.

## Interview angle

- "Tell me about a time you disagreed with a senior/decision" → the FOR-UPDATE-vs-Serializable exchange as a template (STAR it in [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).
- "How do you ask for help?" → the what-I-found-first template; interviewers read it as autonomy.
