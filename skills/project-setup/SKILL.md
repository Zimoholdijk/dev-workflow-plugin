---
name: project-setup
description: Bootstrap a new project with context docs, CLAUDE.md, overview, and all accumulated best practices. Run in any repo to set up the full development workflow.
disable-model-invocation: false
argument-hint: "[project name, e.g. 'ToySwap' or 'my-saas-app']"
---

# Project Setup

You are setting up a new project: $ARGUMENTS

This skill creates the full project scaffolding: context directory, CLAUDE.md rules, overview doc, and working agreements, based on battle-tested conventions. It produces files that the other skills (`/write-prd`, `/write-plan`, `/plan-review`, `/implement-plan`, `/full-code-review`, `/doc-audit`, `/overnight-delivery`) depend on.

---

## Step 1: Detect project state

Before creating anything, check what already exists:

1. Run `git log --oneline -5` to see if the repo has commits
2. Check if `.claude/CLAUDE.md` exists
3. Check if `context/` directory exists
4. Check if `context/overview.md` exists
5. Read `package.json`, `deno.json`, `Cargo.toml`, `go.mod`, `requirements.txt`, or equivalent to detect tech stack
6. Read `prisma/schema.prisma`, `drizzle.config.ts`, `knexfile.js`, or equivalent to detect ORM/DB
7. Check for existing framework config (`astro.config.*`, `next.config.*`, `vite.config.*`, etc.)

If any of `.claude/CLAUDE.md`, `context/overview.md` already exist, ask the user whether to replace or skip them. Never silently overwrite.

---

## Step 2: Gather project info from the user

Ask the user these questions (skip any that were auto-detected in Step 1):

1. **What are you building?** (1-2 sentence description, this goes in the overview)
2. **Tech stack**: confirm or correct what was auto-detected (runtime, frontend, API, ORM, CSS, auth)
3. **Architecture**: how many servers/processes in dev? What serves pages vs. API?
4. **Auth approach**: magic link, OAuth, session-based, JWT, etc.?
5. **Database**: what DB, any soft-delete conventions, any existing models?
6. **Deployment target**: where will this run? (helps inform env var conventions)
7. **Local-dev gotchas**: any non-default ports, service quirks, pre-existing build/lint errors, or env vars that have bitten before? (these go in CLAUDE.md so a fresh session doesn't waste its first turns rediscovering them)
8. **Testing reality**: what test infrastructure exists today, if any? Can new features assume a suite?
9. **Issue tracker / board**: which tracker does this project use, and what's the specific target? (e.g. Notion: backlog data source ID; Linear: team and project keys). This is what `discuss-feature` and `write-prd` read when creating tickets. If the global `~/.claude/CLAUDE.md` names a default tracker, confirm it applies here or override it. Skip only if the project has no board.

Do NOT proceed until the user has confirmed or provided this info. The overview and CLAUDE.md must reflect the actual project, not assumptions.

---

## Step 3: Create directory structure

```
.claude/
  CLAUDE.md                  # Project rules (created in Step 4)
context/
  overview.md                # Project overview (created in Step 5)
```

Feature directories (`context/<Feature>/`) are NOT created here: they are created by `/write-prd` when the first feature PRD is written.

---

## Step 4: Create `.claude/CLAUDE.md`

This file contains project-specific rules that Claude must follow. Create it with the following structure, filling in project-specific details from Step 2:

```markdown
# [Project Name] Project Rules

## Tech Stack
- Runtime: [detected/confirmed]
- Frontend: [detected/confirmed]
- API: [detected/confirmed]
- ORM: [detected/confirmed]
- CSS: [detected/confirmed]
- Auth: [detected/confirmed]
[Add any project-specific tech notes, e.g. "Prisma v7 with driver adapter"]

## Architecture
[Fill in from Step 2: dev server setup, page vs API separation, middleware structure]

## Local-Dev Gotchas
[Things that have bitten before and would waste a fresh session's first turns. Examples:
- Non-default dev server port or config
- Docker or service naming quirks
- Pre-existing build/lint errors that are NOT yours to fix (the bar: add zero new ones)
- Env vars whose behavior differs from what their name suggests]
[Fill from Step 2 question 7. Omit this section until something has actually bitten.]

## File Organization
[Map the actual project structure. List key directories and what lives where. Example:]
- `server/routes/`: one file per API domain
- `server/db/`: database client singleton
- `server/lib/`: shared server constants and utilities
- `server/middleware/`: API middleware
- `src/components/`: UI components
- `src/lib/`: shared client constants and utilities
- `src/pages/`: pages/routes

## Database & Migrations
- Migration files are schema-only. Never add data manipulation (UPDATE, DELETE, INSERT) to migration files. If a migration would fail due to existing data, surface it as a prerequisite for the user to handle (reset DB, seed clean data, etc.).
- Before implementing schema changes, check whether the database has existing data that would conflict. Flag it to the user before writing any migration code.
[Add any project-specific DB conventions, soft delete patterns, ID format, etc.]

## Decision Making
- Never accept trade-offs silently. Always surface trade-offs to the user and let them decide. This includes architectural trade-offs, known limitations, performance compromises, and security acceptances. Document the user's decision, not your own judgement.

## Project-Specific Rules
[Fill in any that apply:]
- [API call convention, e.g. "All client-side API calls go through `src/lib/api.ts`"]
- [Error response format, e.g. "{ error: 'message' } for simple errors, { message: '...', errors: { field: 'reason' } } for 422"]
- [File storage convention, if applicable]
[Leave empty if none yet, these accumulate during development.]
[For non-obvious, load-bearing conventions (the ones where doing the "obvious" thing is wrong), capture three things: the rule, why it exists, and the path to the canonical example in the code to read before copying it. A rule without its why gets deleted by the next refactor.]

## Build & Verify
- Check existing packages before installing new ones; prefer reusing installed dependencies.
- Check for existing components and helpers to reuse before creating new ones.
- Run [build command] to surface specific build errors before attempting fixes. [Adapt to the toolchain.]
- Run [lint command] to surface specific lint errors before attempting fixes.
- At task completion, review changes for type safety, unused imports/variables, and proper error handling.

## TypeScript
[Include this section only when the detected stack uses TypeScript. Omit it entirely otherwise.]
- Never use `any`: use `unknown`, a specific type, or a generic instead.
- Use template literals (`${variable}`) instead of `+=` string concatenation.
- Prefer interfaces over type aliases for object shapes.
- Comply with strict mode.

## Testing Reality
[What test infrastructure actually exists today. Whether new features can assume a suite. What to look at before adding tests. Fill from Step 2 question 8; if the answer is "none yet", say that explicitly so sessions don't invent a suite.]

## Known Concerns (live, [date])
[Real concerns verified against current code, each with a tracking ticket if one exists. Date the heading so staleness is visible. Omit this section until the first concern exists.]

## Production Deploy Notes
[Where production runs, migration ordering relative to app boot, rollback playbook. Omit until there is a production environment.]

## Context
[Ordered by trust: read top to bottom.]
- Overview: `/context/overview.md` (primary source of truth, trust it before older docs)
- PRD: `/context/[product-prd-name].md` (created separately if needed)
- Feature docs: `/context/<Feature>/` with PRD, implementation plan, and progress files
[If the project keeps older hand-written docs, list them last and mark them as potentially stale: cross-check against live code where it matters.]

## Issue Tracker
[Fill from Step 2 question 9. State the tracker and the exact target so discuss-feature and write-prd can create tickets without asking. Examples:
- Notion: backlog data source ID `[id]` (workspace [name])
- Linear: team `[KEY]`, project `[name]`
Omit this section only if the project has no board; the skills then deliver the decision summary without creating a ticket.]

## Workflow Skills
The dev-workflow skills are available: `/discuss-feature`, `/write-prd`, `/write-plan`, `/plan-review`, `/implement-plan`, `/full-code-review`, `/doc-audit`, `/tradeoff-review`. For anything non-trivial, walk the user through the planning workflow (optionally `/discuss-feature` first, then PRD, then implementation plan, then implementation) before writing code.
```

**Important:** Only include sections that are relevant. If there's no database, omit the Database section. If the tech stack is simple, keep it brief. The CLAUDE.md should grow organically as conventions are established. Don't front-load rules that haven't been tested yet.

---

## Step 5: Create `context/overview.md`

This is the project's living source of truth. Create it with this structure:

```markdown
# [Project Name]: Project Overview

> One source of truth for project context, architecture decisions, working agreements, and key feedback.
> Details live in feature docs under `/context/<Feature>/`. This is the summary.

---

## What We're Building

[1-2 paragraphs from Step 2: what the product is, who it's for, core value prop]

---

## Tech Stack

| Layer | Choice |
|---|---|
| Runtime | [value] |
| Frontend | [value] |
| API | [value] |
| ORM | [value] |
| Database | [value] |
| Auth | [value] |
| CSS | [value] |

### Architecture

[Prose description of how the pieces fit together: servers, middleware, auth flow, etc.]

### Shared Utilities

Check these before creating new constants, types, or helpers.

| File | Purpose |
|------|---------|
[Leave empty initially, populated as utilities are created during implementation]

---

## Features

Each feature has its own `/context/<Feature>/` folder with PRD, implementation plan, and progress docs.

[None yet, features are added here as they are implemented]

---

## Working Agreements

- **PRD first, implementation plan second, code last.** No code until plan is approved.
- **Testable phases.** Each phase produces something visible in the browser. Pages before components.
- **Mobile first.** Desktop is progressive enhancement.
- **No auto-commits.** Claude does not commit unless explicitly asked.
- **Document freeze.** PRDs and plans are frozen once agreed. Deviations go in `progress.md`, not by editing the plan.
- **The overview is a summary.** Details live in feature docs. Updated only for stable project-wide decisions.
- **Surface all trade-offs.** Never accept trade-offs silently. Document the user's decision, not Claude's judgement.
- **No data in migrations.** Migration files are schema-only. Flag DB prerequisites to the user instead.

---

## Key Decisions

| Date | Decision |
|------|----------|
[Populated as decisions are made during development. Format: date + concise decision statement.]

---

## Deferred Items & Tech Debt

### Must fix before production

| Item | Notes |
|------|-------|
[Populated during implementation: security, compliance, infrastructure items]

### Should fix before scaling

| Item | Notes |
|------|-------|
[Populated during implementation: performance, consistency, correctness items]

### Post-v1

| Item | Notes |
|------|-------|
[Nice-to-haves, future features, polish items]

---

## Open Questions

[Numbered list of unresolved decisions. Struck through and annotated when resolved.]
```

---

## Step 6: Verify global rules exist

Check if `~/.claude/CLAUDE.md` exists. If it does, read it and confirm it contains the core workflow rules. If it doesn't exist, create it with these universal best practices:

```markdown
# Global Rules (all projects)

## Workflow
- PRD first, implementation plan second, code last. No code until the plan is approved.
- Each implementation phase must produce something testable in the browser. Pages before components.
- Do not commit unless explicitly asked.
- Do not start or restart dev servers. The user manages them.

## Decision Making
- Never accept trade-offs silently. Always surface trade-offs to the user and let them decide. This includes architectural trade-offs, known limitations, performance compromises, and security acceptances. Document the user's decision, not your own judgement.

## Code Quality
- Before creating a constant, type, or utility, search the codebase for existing definitions. Do not redeclare, import from the shared location.
- If the same logic appears 3+ times (across files or within one file), extract it into a helper or middleware.
- Never write `catch {}` without logging, showing user feedback, or re-throwing. Silent catches hide bugs.
- Never `throw new Error(...)` in API route handlers. Always return an explicit HTTP response with a status code and structured body.
- Wrap all database calls in try/catch. Return 500 with `{ error: "Internal server error" }` on failure, never leak stack traces.
- Keep functions under CC=15. If a function has more than ~15 branch points, refactor it.
- React components should do one thing. If a component manages more than 3 concerns, split it. Target <200 lines per file.
- All configuration (URLs, ports, reward amounts, feature flags) must come from environment variables, never hardcoded.
- No placeholder, lorem ipsum, or TODO strings as committed UI text. All user-facing strings must be real copy.
- Mobile first. All UI is mobile-first. Desktop is progressive enhancement.

## Planning Workflow
- When creating an implementation or refactoring plan, use `/plan-review` to run the multi-stage review workflow after drafting.
- The workflow spawns a junior-reviewer (clarifying questions) and senior-reviewer (architectural review) sub-agent in sequence, followed by a quality/conformance check.
- Sub-agents receive full project context (CLAUDE.md, overview, PRDs), treat them as team members with zero prior knowledge of the project.
- The main agent exercises critical judgement on all feedback: apply it or document why not. Sub-agents are advisory, not authoritative.
- Use judgement on when to invoke the full workflow. It's valuable for plans spanning 3+ files or involving schema/architecture changes. Skip for trivial changes.

## Database & Migrations
- Migration files are schema-only. Never add data manipulation (UPDATE, DELETE, INSERT) to migration files. If a migration would fail due to existing data (e.g., adding NOT NULL to a nullable column with NULL rows), surface it as a prerequisite for the user to handle (reset DB, seed clean data, etc.).
- Before implementing schema changes, check whether the database has existing data that would conflict. Flag it to the user before writing any migration code.

## Document Freeze
- PRDs and implementation plans are frozen once agreed. Never edit them unless explicitly instructed.
- Deviations from the plan go in the feature's `progress.md`, not by editing the plan.
- The progress doc is the living record of what actually happened during implementation.
```

If the file already exists, do NOT overwrite it. Only suggest additions if it's missing key rules.

---

## Step 7: Verify companion skills exist

Check that these companion skills are available (as personal skills in `~/.claude/skills/` or via the dev-workflow plugin, where they appear with a `dev-workflow:` prefix). List any that are missing, they are needed for the full workflow:

| Skill | Purpose | Required for |
|-------|---------|-------------|
| `discuss-feature` | Pre-PRD discussion, collects decisions | Before write-prd (optional) |
| `write-prd` | Draft feature PRDs | Starting any new feature |
| `write-plan` | Draft implementation plans | After PRD approval |
| `plan-review` | Multi-stage plan review (junior + senior) | After drafting a plan |
| `implement-plan` | Phase-by-phase implementation | After plan approval |
| `full-code-review` | 6-reviewer parallel code review (branch diff or `full` codebase health check) | After implementation; periodically with `full` |
| `doc-audit` | Documentation vs. codebase audit | After implementation |
| `overnight-delivery` | End-to-end PRD-to-code pipeline | Full feature delivery |
| `tradeoff-review` | Walk through trade-offs one by one | During review gates |
| `junior-review` | Sub-agent: clarifying questions | Used by plan-review |
| `senior-review` | Sub-agent: architectural review | Used by plan-review |

If any are missing, inform the user which skills are not installed and what workflow steps will be unavailable.

---

## Step 8: Present summary

Tell the user what was created:

```
## Project Setup Complete

### Created:
- `.claude/CLAUDE.md`: [N] sections of project rules
- `context/overview.md`: project overview with [tech stack, architecture, working agreements]
- `~/.claude/CLAUDE.md`: [created / already exists, verified]

### Workflow ready:
- [List available skills]
- [List missing skills, if any]

### Next steps:
1. Review `.claude/CLAUDE.md` and `context/overview.md`: adjust anything that doesn't match your project
2. Create your first feature PRD with `/write-prd [feature name]`
3. The overview and CLAUDE.md will grow organically as you build, don't front-load rules
```

---

## Design principles for this skill

1. **Detect, don't assume.** Read the repo before generating files. The scaffolding must match the actual project.
2. **Ask, don't guess.** If something can't be auto-detected, ask the user. Wrong scaffolding is worse than no scaffolding.
3. **Minimal viable structure.** Create only what's needed now. Empty tables and placeholder sections are fine. They get populated during development.
4. **Universal rules, project-specific details.** Working agreements and code quality rules are universal. Tech stack, architecture, and file organization are project-specific.
5. **Never overwrite.** If files exist, ask before replacing. The user may have customized them.
