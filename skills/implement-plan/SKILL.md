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

1. **Plan must be approved and reviewed first.** Check the plan's status field and the review-log sidecar. If the status is "Draft", or `context/[Feature]/review-log.md` doesn't exist (no review round has run), stop and tell the user the plan needs review first. (Review history lives in that sidecar, not in the plan.)
2. **Never edit the plan or PRD.** They are frozen. All deviations, decisions, and trade-offs go in `progress.md`.
3. **Implement phase by phase.** Complete one phase fully before starting the next. After each phase, update the progress doc.
4. **Research in-flight trade-offs, then surface only the genuine ones.** If you hit a decision not covered by the plan, research it first with `/research` (grounded in the project's stack) when it has a technical or best-practice dimension. If documented best practice or the project's own conventions clearly resolve it, apply that, record the decision and citation in `progress.md`, and don't stop the user for a settled question. Surface to the user only the genuinely contested choices, and bring the researched points with them. Document the user's decision in `progress.md`.
5. **No commits unless asked.** The user manages git.
6. **No dev server restarts.** The user manages dev servers.
7. **Tests ship with the code, in the same phase.** Each phase writes the tests named in the plan's Testing Strategy, that phase's "Testable" section, and that phase's referenced **Test Obligations** (see rule 7a), unit tests for the logic it adds, and an e2e test (via `/write-e2e-tests`) for any user-facing flow it exposes. A phase is not "Done" until its tests exist and pass. Do not batch all testing into a final phase, and do not mark a phase Done with the suite red. If the plan named no test for logic you just wrote, that's a gap, write the test and note it in `progress.md` rather than skipping it.
7a. **Test Obligations are mandatory, not optional.** If the plan carries a `## Test Obligations` section (plan-review writes one at convergence for everything review deferred to code+tests), every obligation must be fulfilled by a real test in the phase that references it. An obligation that came with a **simplification note** flags an area that churned in review: before re-implementing it as-is, consider the simpler structure the note points at, then cover it with the obligated test either way. A phase that references an obligation is not "Done" until that obligation's test exists and passes.
8. **You are the only writer; delegate only reads.** Do every file edit yourself, in the main session. Never spawn sub-agents to write code in parallel: parallel writers make conflicting implicit decisions the orchestrator can't reconcile. You may delegate read-only work (locating code, researching an idiom, reviewing a diff) to a sub-agent so its file-reading stays out of your context, but only when the reads are large enough to be worth the token cost. Don't spin up an agent for a trivial single-file lookup.
9. **Don't thrash.** After about two failed attempts at the same issue, stop. Re-anchor from `progress.md` and the plan phase with fresh framing, and `/research` the actual error instead of guessing again. If still stuck, surface it to the user with what you tried. Never loop on the same broken approach.

## Step 1: Gather context

Before writing any code, read:

1. The implementation plan for this feature
2. The PRD for this feature
3. `context/overview.md`: current project state, shared utilities, key decisions
4. `.claude/CLAUDE.md`: project rules
5. `~/.claude/CLAUDE.md`: global rules
6. `prisma/schema.prisma`: current schema
7. The feature's `progress.md` if it exists (may have work from a previous session)
8. The plan's `## Test Obligations` section if present (what plan-review deferred to code+tests), and note which phase each obligation is referenced from

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
4. **Pattern-first, then docs.** If the phase touches an unfamiliar surface (a hook lifecycle, RLS, a migration shape, a library API not used here yet), find how the repo already does it and follow that in-repo example. If there's no precedent, run `/research` docs-first *before* writing, not after it breaks. Researching the idiom up front is cheaper than having the code review reject a non-idiomatic pattern later.

### During the phase:
1. Follow the plan's sub-steps in order
2. Use existing shared code: check the Shared Utilities table in `overview.md`
3. If you create a new shared utility, note it for the overview update
4. Write the phase's tests alongside its code, not after. Unit/integration tests for the logic the plan named for this phase, **plus every Test Obligation this phase references** (per rule 7a), plus an e2e test for any user-facing flow (use `/write-e2e-tests`). Cover the error states, auth boundaries, and edge cases the plan calls out, not just the happy path. If you wrote non-trivial logic the plan didn't name a test for, write one anyway and note the gap in `progress.md`.
5. If you deviate from the plan (different approach, extra file, skipped step), document why in `progress.md` under "Deviations from Plan"
6. If you encounter a trade-off not covered by the plan, stop and research it first (`/research`). If the evidence settles it, apply it and record the decision + citation; otherwise surface it to the user with the researched points. Record the decision in `progress.md` under "Trade-off Decisions"

### After completing the phase:
1. **Verify with evidence.** Run the test suite (the new tests plus the full existing suite) and paste the actual command and its output, never a bare "tests pass" claim. If the phase exposed a user-facing flow, also observe it working (`/verify` or `/run`): a green suite is not proof the UX works. For a phase with no user-facing surface that can't yet be checked end-to-end, mark it `Test` rather than `Done`. If a test fails, fix it before marking Done; never weaken an assertion just to pass, and never mark Done with the suite red.
2. **Self-consistency re-check.** If this phase changed shared or existing code (a helper, a state machine, a schema field, an ordering rule, an interface), re-check its non-local effects before closing: trace what else depends on the thing you changed and confirm each dependant still holds. A fix that is locally correct but breaks a caller or a sibling change is the most common regression; catch it now, not in the final review.
3. **Optional self-review.** For a non-trivial phase, run `/code-review` on just this phase's diff before moving on. Tell it to flag only correctness and requirement gaps, not style or speculative hardening (a reviewer told to "find problems" invents them). Catching a bug now is cheaper than at the final full review.
4. Update the progress doc: set the phase status to "Done" (or "Test" per above)
5. Add a changelog entry describing what the phase produced, the tests added, and any surprises
6. Tell the user what's testable (per the plan's "Testable" section for that phase) and the test result
7. Wait for the user to confirm before moving to the next phase, unless they've told you to proceed through all phases

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

1. **Verify before extending.** Before writing any new code, read the progress doc and the recent git log, then run the test suite (and boot the app if the feature has a UI) to confirm the existing work is actually green. Undocumented breakage from a prior session is common; catch it before stacking new code on top.
2. Identify the first phase that is NOT "Done"
3. Resume from that phase
4. Do not re-implement completed phases

If all phases are already "Done" but the overview hasn't been updated (no feature summary matches current state), run Step 4 to update documentation.
