---
name: doc-audit
description: Audit project documentation against the actual codebase. Checks that overview.md, CLAUDE.md, and feature docs accurately reflect the current code, schema, routes, and utilities. Run standalone or after implementing a plan.
disable-model-invocation: false
argument-hint: "[optional: 'local' to audit only uncommitted changes, or base branch e.g. 'main']"
---

# Documentation Audit

You are auditing project documentation against the actual state of the codebase.

If `$ARGUMENTS` is `local`, audit only uncommitted/unstaged changes. If a branch name is given (e.g. `main`), audit all changes since that base. If no argument is provided, run a full audit of all documentation vs the codebase.

## Step 1: Determine scope

Based on the argument, set the audit scope:

- **`local`**: Run `git diff --stat` and `git diff --staged --stat` to find changed files. The audit checks only that documentation reflects these local changes.
- **`<base branch>`**: Run `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline` to find all changes on this branch. The audit checks that documentation reflects all branch changes.
- **No argument (full audit)**: No diff scoping: audit ALL documentation against the full codebase.

## Step 2: Spawn two agents sequentially

### Agent 1: Change-scoped audit (skip if full audit with no argument)

Only spawn this agent if a scope was provided (`local` or a base branch). Launch a `general-purpose` agent:

> Your task is to review whether recent code changes are reflected in project documentation.
>
> **How to gather context:**
> 1. Run the appropriate git diff command to see what changed (the orchestrator will tell you whether to use `git diff` for local changes or `git diff <base>...HEAD` for branch changes)
> 2. Read `context/overview.md`: the project's source of truth for architecture, features, utilities, decisions, and tech debt
> 3. Read `.claude/CLAUDE.md`: project rules
> 4. Read any feature `progress.md` files in `context/` subdirectories that relate to the changed files
>
> **Check each changed file against documentation:**
>
> For each meaningful code change (skip whitespace-only, comment-only, or formatting-only changes):
>
> 1. **New shared utilities**: If a new file was created in `src/lib/`, `src/hooks/`, `server/lib/`, `server/middleware/`, or `server/routes/`, check whether it appears in the Shared Utilities table in `overview.md`. If not, flag it.
> 2. **New routes**: If a new file was added to `server/routes/`, check whether `.claude/CLAUDE.md` File Organization section lists it. If not, flag it.
> 3. **Schema changes**: If `prisma/schema.prisma` was modified, check whether relevant claims in `overview.md` (soft-delete models, model fields, indexes) are still accurate.
> 4. **New pages**: If a new page was added to `src/pages/`, check whether the feature description in `overview.md` mentions the route.
> 5. **Removed or renamed files**: If files were deleted or renamed, check whether documentation still references the old paths.
> 6. **Architecture changes**: If middleware, layout, or core infrastructure files changed, check whether the Architecture section of `overview.md` is still accurate.
> 7. **New decisions**: If the changes represent a significant architectural or product decision, check whether the Key Decisions table has an entry for it.
> 8. **Tech debt resolved**: If a change fixes something listed in the Deferred Items tables, flag it for removal.
> 9. **New tech debt introduced**: If the change introduces a known limitation, TODO, or workaround, check whether it's tracked in Deferred Items.
>
> **Output format:**
> For each finding, state:
> - **Category:** Missing utility / Missing route / Stale claim / Missing decision / Stale tech debt / Missing page / Other
> - **File:** which documentation file needs updating
> - **Finding:** what's wrong or missing
> - **Suggested fix:** specific text to add, update, or remove
>
> End with a summary: total findings by category, and a verdict (Up to date / Needs updates / Significantly outdated).

### Agent 2: Full documentation vs codebase audit

Always spawn this agent (for all scopes, including change-scoped, since it catches drift that predates the current changes). Launch a `general-purpose` agent:

> Your task is to perform a comprehensive check of project documentation against the actual codebase.
>
> **How to gather context:**
> 1. Read `context/overview.md` thoroughly: this is the primary document being audited
> 2. Read `.claude/CLAUDE.md`: project rules
> 3. Explore the codebase to verify claims (use Glob, Grep, Read as needed)
>
> **Audit each section of `overview.md`:**
>
> ### Tech Stack
> - Verify the tech stack table is accurate (check `package.json`, `deno.json`, `prisma/schema.prisma`)
>
> ### Architecture
> - Verify middleware structure claim: read `src/middleware/index.ts`, check the sequence and imports
> - Verify credit system claim: check both `src/lib/credits.ts` and `server/lib/credits.ts`
> - Verify upload system claim: check `server/lib/storage.ts` exists with described exports
> - Verify security headers claim: read `src/middleware/security.ts`
> - Verify CSRF claim: read `server/middleware/csrf.ts`
> - Verify any theming/layout claims against `src/styles/global.css` and `src/layouts/Layout.astro`
>
> ### Shared Utilities table
> - For EVERY file listed: verify it exists and exports match the description (read each file's first ~30 lines)
> - Check for unlisted shared utilities: glob `src/lib/*.ts`, `src/hooks/*.ts`, `server/lib/*.ts`, `server/middleware/*.ts`, `server/routes/*.ts` and flag any missing from the table
>
> ### Features
> - For each feature marked "Complete": spot-check that the described pages/components exist
> - Check that feature docs directories exist at the paths mentioned
>
> ### Key Decisions
> - Spot-check 5-8 recent decisions against the code (focus on decisions that name specific files, constants, or patterns)
> - Flag any decisions that contradict current code
>
> ### Deferred Items
> - For each "Should fix" and "Must fix" item: check whether it's been resolved (grep for the described pattern)
> - Flag any resolved items that should be removed
>
> ### CLAUDE.md
> - Verify File Organization section lists all actual route files in `server/routes/`
> - Verify tech stack claims are accurate
> - Check that project-specific rules match current patterns
>
> **Output format:**
> For each finding, state:
> - **Section:** which documentation section has the issue
> - **Category:** Missing / Stale / Incorrect / Resolved-tech-debt
> - **Finding:** what's wrong
> - **Suggested fix:** specific text to add, update, or remove
>
> End with a summary: total findings by category, and an overall verdict (Up to date / Needs updates / Significantly outdated).

## Step 3: Consolidate and present

Once both agents complete (or just Agent 2 for full audits), present consolidated results:

```
## Documentation Audit Results

### Scope
[What was audited: local changes / branch changes since X / full codebase]

### Findings

#### Must fix (documentation is wrong or misleading)
[Findings where docs contradict the code: these will mislead future development]

#### Should fix (documentation is incomplete)
[Findings where docs are missing information about existing code]

#### Resolved items (can be removed)
[Tech debt or deferred items that have been addressed in code]

#### Cosmetic (optional cleanup)
[Minor wording improvements, not functionally wrong]

---

### Verdict: [Up to date / Needs updates / Significantly outdated]
```

## Step 4: Apply fixes

After presenting findings, split into:

1. **Clear fixes**: straightforward additions/removals where there's no judgment call (e.g., adding a missing utility to the table, removing resolved tech debt). List these as "I'll apply these unless you object."

2. **Judgment calls**: findings where the fix involves a decision (e.g., how to describe a new architectural pattern, whether a limitation is worth documenting). Present each one for user input.

After the user confirms, apply all fixes in a single pass. Update:
- `context/overview.md`
- `.claude/CLAUDE.md`
- Any feature `progress.md` files as needed

Do NOT edit frozen documents (PRDs, implementation plans).
