---
name: overnight-delivery
description: End-to-end feature delivery pipeline. Writes implementation plan, runs 4 review rounds, gates on tradeoffs, implements all phases, runs 2 code review rounds with fixes. Takes a feature name with an approved PRD.
disable-model-invocation: false
argument-hint: "[feature name or path to PRD, e.g. 'RelatedToys' or 'context/RelatedToys/RelatedToysPRD.md']"
---

# Overnight Delivery

End-to-end delivery pipeline for: $ARGUMENTS

This skill orchestrates the full journey from approved PRD to reviewed, implemented code. It composes existing skills in sequence with iterative review loops and a tradeoff gate.

## Prerequisites

- An **approved PRD** must exist for this feature. If no PRD is found or its status is not "Approved", stop and tell the user.
- A **git branch** should already be checked out for this feature.

---

## Stage 1: Write Implementation Plan

Run `/write-plan $ARGUMENTS`.

This drafts a phased implementation plan based on the approved PRD. The plan is saved to `context/[Feature]/implementation-plan.md` with status "Draft".

**Gate:** Plan file must exist before proceeding.

---

## Stage 2: Plan Review Loop (4 rounds)

Run `/plan-review context/[Feature]/implementation-plan.md` **four times in sequence**. The plan is updated between rounds, so each round reviews a progressively refined artifact. But each round is a **cold start**: plan-review strips the Review Log before spawning reviewers, so a reviewer in round 4 scrutinizes the plan as hard as round 1 instead of deferring to prior "Reviewed" stamps. The plan improves between rounds; the reviewers are never told what changed or what a prior round approved, that anchoring is exactly what we keep out.

### Round 1
- Fresh clarifying-questions, deep-critique, and red-team passes against the initial draft.
- Apply all critical issues and clear improvements. Update the plan.

### Round 2
- Second pass catches issues introduced by Round 1 fixes, or concerns the first reviewers missed.
- Different reviewer agents may catch different things: diversity of perspective is the point.
- Apply fixes. Update the plan.

### Round 3
- By now the plan should be stable. This round focuses on edge cases, subtle correctness issues, and conformance.
- Apply any remaining fixes. Update the plan.

### Round 4
- Final polish. This round should produce mostly Info/Low findings. If Critical or High issues are still emerging, that's a signal the plan needs more fundamental rework. Flag this to the user.
- Apply fixes. Update the plan. Set plan status to "Reviewed".

After each round, append to the plan's Review Log section with the round number and summary of changes.

---

## Stage 3: Tradeoff Gate

Run `/tradeoff-review [Feature]`.

This walks the user through every tradeoff from the plan and 4 review rounds **one at a time**, collecting a decision on each (accept / fix now / skip). The tradeoff-review skill handles gathering, deduplication, and presentation.

**If the tradeoff-review verdict includes "fix now" items:** Apply those fixes to the plan before proceeding to implementation.

**If any tradeoff is assessed as a blocker** (e.g., data loss risk, security vulnerability, architectural dead end): The tradeoff-review skill will flag this. Do not proceed. Inform the user and wait for resolution.

**Gate:** All tradeoffs must be reviewed. User must have decided on each one. No unresolved blockers.

---

## Stage 4: Implementation

Run `/implement-plan $ARGUMENTS`.

This implements the plan phase by phase, creating the progress doc, writing code, and updating documentation. The implement-plan skill handles:
- Phase-by-phase execution with progress tracking
- Deviation documentation in `progress.md`
- Surfacing implementation-time tradeoffs to the user
- Updating `overview.md` after all phases complete
- Running `/doc-audit local` for documentation consistency

**Important:** Tell the implement-plan skill to proceed through all phases without waiting for confirmation between phases. The plan has been reviewed 4 times: phase-level confirmation adds friction without value at this point.

**Gate:** All phases must be marked "Done" in `progress.md` before proceeding.

---

## Stage 5: Code Review Loop (2 rounds)

### Round 1

Run `/full-code-review` with the feature branch's base (the branch this feature will merge into; check the plan's Branch field or the project's documented integration branch).

This spawns 7 parallel reviewers (security, backend, frontend, architecture, documentation, regressions, and testing, the testing reviewer checks that new code has tests and runs the suite). After consolidation:

1. **Clear fixes**: Apply immediately (typos, missing constants, accessibility, dead code).
2. **Tradeoffs**: Note them. They'll be reviewed after Round 2.
3. Apply all clear fixes in a single implementation pass.

### Round 2

Run `/full-code-review` again on the same branch.

This round verifies that Round 1 fixes didn't introduce regressions, and catches anything the first round missed. The second round should produce mostly clean results. If Critical/High issues persist, flag to the user: something systemic may need attention.

After Round 2:
1. Apply any remaining clear fixes.
2. Update `progress.md` with a "Code Review Rounds" section summarizing both rounds.

### Final Tradeoff Review

Run `/tradeoff-review [Feature]`.

This walks the user through all remaining tradeoffs from implementation and both code review rounds, one at a time. This is the last decision point before the feature is ready for commit. Apply any "fix now" items before proceeding to the final summary.

---

## Stage 6: Final Summary

Present the delivery summary:

```
## Overnight Delivery Complete

### Feature: [name]
### Branch: [branch]

### Plan
- [N] architecture decisions
- [N] phases implemented
- [N] plan review rounds (4)
- [N] total plan revisions applied

### Implementation
- [N] files created/modified
- [N] deviations from plan (see progress.md)
- [N] implementation-time tradeoffs decided

### Code Review
- [N] code review rounds (2)
- [N] findings fixed
- [N] tradeoffs accepted
- Final verdicts: Security [verdict], Backend [verdict], Frontend [verdict], Architecture [verdict], Documentation [verdict], Regressions [verdict], Testing [verdict]

### Documentation updated
- overview.md: [what changed]
- progress.md: [complete]
- doc-audit: [results]

### Remaining items
[Any deferred tradeoffs, tech debt, or follow-up work identified during the process]
```

Suggest the user:
1. Review the diff (`git diff main...HEAD --stat`)
2. Test in the browser
3. Commit when satisfied
4. Create a PR when ready

---

## Error handling

- **If a skill fails or produces unexpected output:** Do not retry blindly. Diagnose the issue, inform the user, and ask how to proceed.
- **If the user interrupts at any gate:** Resume from the last completed stage. Check progress.md and the plan's Review Log to determine current state.
- **If context runs low:** The conversation may be compacted between stages. Each stage re-reads its inputs (plan, PRD, progress doc) so no context is assumed to persist across stages.
