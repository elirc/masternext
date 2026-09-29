# NextCRM Upskill Curriculum

A training lab built on the real NextCRM codebase for one learner: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, its failure modes, and how to articulate it in an interview.

## What this repo is

NextCRM is an open-source CRM built on **Next.js 16 (App Router) + React 19 + TypeScript**, with **PostgreSQL via Prisma 7** (recently migrated from MongoDB — the migration scripts are still in [scripts/](../../scripts/)), **better-auth** for authentication (Google OAuth + email OTP), **Inngest** for background jobs, **MinIO/S3** for file storage, **Resend** for email, and **next-intl** for i18n. It covers CRM entities (accounts, contacts, leads, opportunities, contracts), a full invoicing module with Serializable-transaction invoice numbering, cold-email campaigns, AI-powered contact/target enrichment (OpenAI agents + Firecrawl), pgvector semantic search, and an **MCP server** exposing CRM tools to AI agents. It ships Jest unit tests and Playwright E2E suites. Notably, the project had a real published security advisory (BOLA/IDOR, `GHSA-mg5f-m89f-4gmc`) and the [lib/authz/](../../lib/authz/) scope layer is the remediation — which makes it an unusually honest teaching ground for authorization.

## How to use it

| Time budget | Path |
| --- | --- |
| One weekend | [00-fast-track.md](00-fast-track.md) — run it, trace two flows, make one safe change |
| Two weeks | Fast track → [01-codebase-cartography/](01-codebase-cartography/) → [03-architecture-and-patterns/05-pattern-catalog.md](03-architecture-and-patterns/05-pattern-catalog.md) → one junior ticket from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md) |
| Eight weeks | All modules in order, one contribution ticket per week, weekly teach-back |
| Ongoing contributor | [06-contribution-practice/](06-contribution-practice/) + [07-career-and-collaboration/](07-career-and-collaboration/) as your working manual |
| **Interview in two weeks** | Go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) |

## Recommended paths by profile

- **Brand-new junior**: 00 → 01 (all) → 02 (all) → 04-code-reading-gym → one Easy ticket.
- **Junior who knows Next/React**: 00 → 01/05-key-flows → 03 (all) → 05-quality-engineering → Medium tickets.
- **Mid-level, new to this repo**: 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06/02-mid-level-feature-tickets.
- **Senior doing architecture review**: 03/06-architecture-critique → 09-reference/risk-register.md → 06/03-senior-build-projects.
- **Candidate, interview in 2 weeks**: [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md), which pulls from everything else.

## Conventions

- **Anchors**: `[file.ts](../../path/file.ts#L10-L20)` links point at the real code. Line numbers were verified against the working tree on 2026-07-11; they will drift as the repo evolves — trust the path, re-find the lines.
- **Fake code**: any snippet not from this repo starts with `// Illustrative fake code: not from this repo`.
- **Verification labels**: commands are marked __verified__ (actually run while writing these docs) or __inferred__ (read from config, not run). See [09-reference/verification-log.md](09-reference/verification-log.md).
- **Drills** end with self-grading criteria (Basic / Solid / Strong). Grade yourself honestly; the gap between Solid and Strong is exactly the junior→mid gap.
- **Suspicions are labeled**. "Confirmed" means the behavior was read directly in code. "Possible risk" / "investigate" means a hypothesis you should verify before repeating it in a PR or interview.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What breaks it?"
- **Senior asks:** "What does this commit us to, who pays the cost, and how do we reduce risk?"

Interviews for mid-level roles test exactly the second and third questions. When this curriculum makes you trace `issueInvoice`'s Serializable transaction or explain why `updateAccount` is missing a scope check, it is rehearsing the answers interviewers actually want: concrete example, tradeoff, failure mode.

## Module map

| Module | What it gives you |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | Weekend orientation: run, trace, change |
| [01-codebase-cartography/](01-codebase-cartography/) | System map, reading order, glossary, tooling, 7 key flows |
| [02-stack-and-language-mastery/](02-stack-and-language-mastery/) | JS/TS runtime, Next.js/React mental models, types, build tooling |
| [03-architecture-and-patterns/](03-architecture-and-patterns/) | Boundaries, data model, authz, async reliability, pattern catalog, critique |
| [04-code-reading-gym/](04-code-reading-gym/) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering/](05-quality-engineering/) | Testing, debugging, performance, security, observability |
| [06-contribution-practice/](06-contribution-practice/) | Junior tickets → mid features → senior projects → design katas |
| [07-career-and-collaboration/](07-career-and-collaboration/) | Review mindset, PR/RFC writing, maintainer communication |
| [08-interview-prep/](08-interview-prep/) | 50+ question cards, system design walkthrough, STAR stories, cram plan |
| [09-reference/](09-reference/) | Command cheatsheet, risk register, rubrics, verification log |
