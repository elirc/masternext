# Writing PRs and RFCs

## Commit messages (this repo uses conventional commits)

Release automation ([release-please](../../../.github/workflows/release-please.yml)) parses your commit prefixes to build the changelog and bump the version. So the prefix isn't style — it's function.

- `feat: add invoice overdue reminders` → minor bump, appears under Features.
- `fix: reject unknown taxRateId in createInvoice` → patch bump, under Bug Fixes.
- `chore:`/`docs:`/`test:`/`refactor:` → no release, categorized accordingly.
- Breaking: `feat!:` or a `BREAKING CHANGE:` footer.

Body: *why*, not *what* (the diff shows what). One logical change per commit.

## PR description template (tailored to NextCRM)

```md
## What
One sentence: the behavior change.

## Why
The problem/ticket. Link the risk-register item or issue if it's a known one.

## How
Key decisions. Which existing pattern you followed (e.g. "routes the write through
tryScopedUpdateContact-style scoped updateMany").

## Authz & validation impact
- Object-level check: <where / N/A>
- Input validation: <Zod schema / allowlist / N/A>
(Reviewers here look for this first — put it up top.)

## How tested
- Unit: <files/commands, e.g. `pnpm test -- numbering`>
- E2E: <spec or "manual: steps">
- Negative case: <the denial/failure test>

## Risks & rollback
- Blast radius: <files/users affected>
- Migration: <additive? backfill? down-path? — remember: roll-forward only>
- Rollback: <revert / disable flag>

## Follow-ups
What you deliberately left out (keep PRs small — say so).
```

Why these sections: NextCRM's reviewers care most about authorization and validation (its scar tissue), tests (there's no CI to catch you), and migration safety (build runs migrations). A PR that leads with those clears review faster.

## When to write an RFC instead of just a PR

Write an RFC when the change: alters a contract (token format, event names, API shape), touches authorization broadly (the manager/admin split), introduces a new cross-cutting pattern (outbox, enforcement wrapper), or has a migration that can't be trivially rolled back. Rule of thumb: if reviewers would argue about the *approach* rather than the *code*, RFC first.

## RFC template (NextCRM-flavored)

```md
# RFC: <title>

## Summary
2–3 sentences.

## Problem
Current behavior with anchors (file:line). Evidence it's a problem (advisory, incident,
risk-register item, perf number).

## Goals / Non-goals
Explicit non-goals prevent scope creep (e.g. "Non-goal: multi-tenancy").

## Proposal
The design. Data model changes. New patterns. Where authz/validation land.

## Alternatives considered
At least one, with why-not. (A senior RFC always shows the rejected road.)

## Migration & rollout
Expand/contract steps. Compatibility window. Per-entity/per-PR sequencing so the system
is never in a half-migrated insecure state. Roll-forward-only reality noted.

## Testing
How each new invariant is pinned. Any new test suite this creates.

## Risks
Blast radius, concurrency/partial-failure edge cases, what could go wrong and the mitigation.

## Open questions
The things you want maintainer input on before coding.
```

## Worked example: the RFC skeleton for Project 1 (authz enforcement)

- **Problem**: `rg`-count of session-only mutations; cite GHSA + [risk #1](../09-reference/risk-register.md).
- **Proposal**: `authorizedAction(schema, scope, handler)` wrapper + lint ban on `getSession` in actions/.
- **Alternative**: per-action manual `assertCan*` (rejected: nothing forces it — same failure mode as today).
- **Migration**: one entity/PR; each keeps behavior, adds denial test.
- **Non-goal**: multi-tenancy, manager/admin split (separate RFC).

## Interview angle

- "How do you write a good PR?" → the authz/validation-first structure and *why* this repo demands it. [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md).
- "When do you write a design doc?" → the contract/authz/cross-cutting/irreversible-migration test above.
