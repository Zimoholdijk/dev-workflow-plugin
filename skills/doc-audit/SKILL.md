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

## Step 2: Spawn the doc-auditor (two passes)

Both passes use the `doc-auditor` sub-agent; the task message tells it which mode to run. The agent carries the audit method and output format, so you only pass it the mode and the scope.

### Pass 1: Change-scoped (skip if full audit with no argument)

Only run this pass if a scope was provided (`local` or a base branch). Spawn `doc-auditor` in **Mode A (change-scoped)**, telling it:
- which diff command to use (`git diff` for local, `git diff <base>...HEAD` for a branch),
- the project's primary docs to check against (`context/overview.md`, `.claude/CLAUDE.md`, relevant feature `progress.md` files).

### Pass 2: Full audit

Always run this pass (even for change-scoped, it catches drift that predates the current changes). Spawn `doc-auditor` in **Mode B (full)**, telling it the project's primary docs (`context/overview.md`, `.claude/CLAUDE.md`) and to verify every claim against the actual code.

Run the two passes in one message when both apply so they run in parallel. Each returns categorized findings (Section/File, Category, Finding, Suggested fix) and a verdict.

## Step 3: Consolidate and present

Once both passes complete (or just the full pass for full audits), present consolidated results:

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
