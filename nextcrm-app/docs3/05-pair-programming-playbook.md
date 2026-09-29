# Pair-Programming Playbook

Use this playbook while working through `docs3`. The point is to build real features while making your engineering instincts sharper.

## The Loop

1. Read the story and say the user outcome in one sentence.
2. Name the likely boundary: UI-only, server action, API route, Prisma model, background job, or cross-cutting quality.
3. Predict the first three files before searching.
4. Inspect the files and correct your mental model.
5. Write the smallest implementation plan that could pass the acceptance criteria.
6. Implement one boundary at a time.
7. Test the risky behavior, not every possible line.
8. Summarize what changed, what could break, and what follow-up is intentionally deferred.

## How I Should Pair With You

When you want me to build while teaching, ask in this shape:

```text
Let's implement docs3 Story N.
Drive the code, but pause at each boundary and ask me what I think happens next.
After each test failure, explain the failure mode before fixing it.
```

When you want more challenge:

```text
Let's implement docs3 Story N in senior mode.
Before editing, ask me to predict the data flow, auth boundary, and test plan.
Do not give me the answer until I attempt it.
```

When you are stuck:

```text
I'm on docs3 Story N and stuck at <specific file or behavior>.
Ask me debugging questions first.
Then give me the next concrete step if I miss it.
```

## Junior To Senior Signals

**Junior signal:** You can make the visible change after being shown where the code is.

**Mid-level signal:** You can trace the data flow, find the right owner file, and add focused tests without being told every step.

**Senior signal:** You can name the invariant, choose the right boundary, reduce blast radius, handle old data, and explain how the feature fails in production.

## Review Checklist

- Did we authenticate and authorize before any write?
- Did we avoid leaking data in errors, logs, audit entries, and MCP responses?
- Did Decimal values crossing client boundaries use the serialization helper?
- Did we keep preview/validation separate from commit/mutation?
- Did we handle empty, loading, forbidden, stale, and failure states where they matter?
- Did we add the smallest test that proves the risky behavior?
- Did we leave the trunk-based flow intact with small, reviewable changes on `dev`?

## Strong Commit Shape

For a feature story:

```text
feat(<area>): <user-visible outcome>
```

For a safety or test story:

```text
test(<area>): cover <policy or invariant>
fix(<area>): enforce <invariant>
```

Do not manually edit release-please version files or changelog entries for these stories.

