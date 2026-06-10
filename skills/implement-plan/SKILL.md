---
name: implement-plan
description: Execute an approved implementation plan phase-by-phase, keeping progress docs and overview in sync. Takes a feature name or plan path as argument.
disable-model-invocation: false
argument-hint: "[feature name or path to plan, e.g. 'claiming' or 'context/Claiming/implementation-plan.md']"
---

# Implement Plan

You are implementing an approved plan for: $ARGUMENTS

## Rules

These rules are non-negotiable:

1. **Plan must be approved and reviewed first.** Check the plan's status field. If it says "Draft" or has no Review Log, stop and tell the user the plan needs review first.
2. **Never edit the plan or PRD.** They are frozen. All deviations, decisions, and trade-offs go in `progress.md`.
3. **Implement phase by phase.** Complete one phase fully before starting the next. After each phase, update the progress doc.
4. **Surface trade-offs.** If you encounter a decision not covered by the plan, surface it to the user. Document their decision in `progress.md`.
5. **No commits unless asked.** The user manages git.
6. **No dev server restarts.** The user manages dev servers.

## Step 1: Gather context

Before writing any code, read:

1. The implementation plan for this feature
2. The PRD for this feature
3. `context/overview.md`: current project state, shared utilities, key decisions
4. `.claude/CLAUDE.md`: project rules
5. `~/.claude/CLAUDE.md`: global rules
6. `prisma/schema.prisma`: current schema
7. The feature's `progress.md` if it exists (may have work from a previous session)

Check what has already been implemented. If a progress doc exists with phases marked "Done", skip those phases.

## Step 1b: Pre-implementation prerequisites check

Before writing any code, scan the plan for **prerequisites the user must handle manually**. These are actions that cannot or should not be performed by the assistant. Present them to the user as a checklist before proceeding.

Check for:

1. **Schema migrations that assume empty tables or no data conflicts.** If the plan adds NOT NULL constraints to existing nullable columns, adds unique constraints, or changes column types, and the database may have existing data: flag it. The user should reset/seed their database first. Never add data manipulation (UPDATE, DELETE, INSERT) to migration files as a workaround.
2. **Database resets or seeds.** If the plan's testability depends on specific data state (empty DB, seed data, specific test records), flag it.
3. **Environment changes.** If the plan requires new env vars, new Docker services, or infrastructure changes the user must configure.
4. **Branch or git state.** If the plan specifies a branch name, check if it exists or needs to be created.
5. **External dependencies.** If the plan requires API keys, third-party services, or tools the user must set up.

Present the checklist to the user and wait for confirmation before proceeding. Format:

```
## Pre-implementation checklist

Before I start coding, please handle these prerequisites:

- [ ] [Prerequisite 1: what to do and why]
- [ ] [Prerequisite 2: what to do and why]

Let me know when these are done, or if any don't apply.
```

If there are no prerequisites, skip this step silently.

## Step 2: Create the progress doc (once, before Phase 1)

Before starting any implementation, create a single `context/[Feature]/progress.md` that covers the entire plan. All phases go in one table. This doc is created once and updated throughout. Never create separate docs per phase.

If `context/[Feature]/progress.md` already exists (resumed work), skip to Step 3.

Structure:

```markdown
# [Feature]: Progress

**Feature:** [TOY-XX] · **Branch:** `[branch-name]`
**Plan:** `context/[Feature]/implementation-plan.md`
**PRD:** `context/[Feature]/[Feature]PRD.md`

---

## Implementation Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | [Phase title from plan] | Not started |
| 2 | [Phase title from plan] | Not started |
| ... | ... | ... |

---

## Deviations from Plan

[None yet]

---

## Trade-off Decisions

[None yet]

---

## Changelog

| Date | Change |
|------|--------|
| YYYY-MM-DD | Created progress doc, implementation starting |
```

**Status vocabulary** (keep consistent across all progress docs):

- `Not started`: no work yet
- `In progress`: actively being worked on
- `Done`: implemented and verified
- `Test`: implemented but end-to-end unverified (e.g. blocked on another feature or manual QA)
- `Blocked`: cannot proceed; name the blocker in the table row or a changelog entry

**Changelog discipline.** The changelog is the most useful part of a progress doc; keep it honest. Add an entry whenever a phase starts or completes, a decision diverges from the plan, a bug surfaces and is fixed, scope is added or removed, or a dependency on other work is discovered. Be specific about what the phase produced and any surprises, not just "Phase 1 done". Append new entries; never edit old ones.

## Step 3: Implement each phase

For each phase (in order):

### Before starting the phase:
1. Update the progress doc: set the phase status to "In progress"
2. Re-read the plan's phase description carefully
3. Check shared utilities in `context/overview.md` before creating new constants, types, or helpers

### During the phase:
1. Follow the plan's sub-steps in order
2. Use existing shared code: check the Shared Utilities table in `overview.md`
3. If you create a new shared utility, note it for the overview update
4. If you deviate from the plan (different approach, extra file, skipped step), document why in `progress.md` under "Deviations from Plan"
5. If you encounter a trade-off not covered by the plan, stop and surface it to the user. Record their decision in `progress.md` under "Trade-off Decisions"

### After completing the phase:
1. Update the progress doc: set the phase status to "Done" (or "Test" if it can't be end-to-end verified yet)
2. Add a changelog entry describing what the phase produced and any surprises
3. Tell the user what's testable (per the plan's "Testable" section for that phase)
4. Wait for the user to confirm before moving to the next phase, unless they've told you to proceed through all phases

## Step 4: After all phases are complete

Once every phase is done:

### 1. Update `context/overview.md`

Update these sections as needed:

- **Features section:** Update the feature's status to "Complete" and write a 2-3 sentence summary of what was built (follow the format of existing feature entries)
- **Shared Utilities table:** Add any new shared utilities created during implementation (constants, hooks, helpers, lib files)
- **Key Decisions table:** Add any significant architectural or product decisions made during implementation, with today's date
- **Deferred Items:** Add any tech debt, known limitations, or follow-up items identified during implementation to the appropriate table (must-fix, should-fix, or post-v0.1)

Do NOT rewrite existing entries: only add new ones or update the feature's status.

### 2. Update `context/[Feature]/progress.md`

Ensure all phases are marked "Done". Add a final section:

```markdown
## Code Review Rounds

[Summary of code reviews performed, if any. Key fixes applied.]
```

### 3. Run documentation audit

Run `/doc-audit local` to verify that all documentation accurately reflects the implementation. This catches missed utility entries, stale claims, resolved tech debt, and missing key decisions. Apply any clear fixes found. Surface judgment calls to the user.

### 4. Present summary to user

Tell the user:
- All phases complete
- Any deviations from the plan
- Any trade-offs that were decided during implementation
- What documentation was updated (including audit fixes)
- Suggest running `/full-code-review` if not already done

## Documentation update triggers

These are the moments when documentation MUST be updated:

| Trigger | What to update |
|---------|---------------|
| Starting a phase | `progress.md`: phase status to "In progress" |
| Completing a phase | `progress.md`: phase status to "Done" |
| Deviating from plan | `progress.md`: add to "Deviations from Plan" |
| User makes a trade-off decision | `progress.md`: add to "Trade-off Decisions" |
| Creating a new shared utility | Note for `overview.md` update (applied in Step 4) |
| All phases complete | `overview.md`: feature status, utilities, decisions, deferred items |
| Code review findings applied | `progress.md`: add to "Code Review Rounds" |
| All phases complete | `/doc-audit local`: catch any missed documentation updates |

## Resuming interrupted work

If the user invokes this skill and a `progress.md` already exists with some phases marked "Done":

1. Read the progress doc to understand current state
2. Identify the first phase that is NOT "Done"
3. Resume from that phase
4. Do not re-implement completed phases

If all phases are already "Done" but the overview hasn't been updated (no feature summary matches current state), run Step 4 to update documentation.
