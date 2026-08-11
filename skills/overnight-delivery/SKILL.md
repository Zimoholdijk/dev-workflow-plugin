---
name: overnight-delivery
description: End-to-end feature delivery pipeline. Writes implementation plan, runs plan review to convergence, gates on tradeoffs, implements all phases, runs 2 code review rounds with fixes. Takes a feature name with an approved PRD.
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

## Stage 2: Plan Review (run to convergence)

Run `/plan-review context/[Feature]/implementation-plan.md` and let it run to convergence. Do not pick a round count. plan-review loops internally: each round runs the cold lenses, a grader rates every finding by reversibility (One-way / Significant / Medium / Minor, or discards it as Not-an-issue with evidence), the orchestrator fixes everything, and a cold assessor decides each round: converged, another round, or escalate. The exit condition decides when to stop, and the loop is bounded (the assessor escalates at round 5 rather than running on).

Why convergence beats a fixed number: a one-way door or significant change is never the last round (it must be settled, then verified by a clean cold pass), while reversible defects (Mediums/Minors) never gate at all, they are handed to code+tests as test obligations instead of being looped on. The loop stops when a round has no open one-way door and no significant change.

What to rely on (plan-review owns the mechanics):
- **Cold start every round.** The `review-log.md` sidecar is withheld from reviewers, so a late round scrutinizes the plan as hard as the first.
- **Trade-offs carry to the Stage 3 gate.** This pipeline runs unattended, so genuine user-owned trade-offs surfaced during the loop are accumulated for `/tradeoff-review` at Stage 3, not blocked on mid-loop. Clear, best-practice-resolved decisions are applied in-loop with their citation.
- **Test obligations land in the plan.** At convergence, everything review deferred to code+tests is written into `implementation-plan.md` as a `## Test Obligations` section plus per-phase references. Stage 4 (`/implement-plan`) is required to fulfil them, so nothing deferred is silently lost.
- **Sidecar per round.** Each round appends to `context/[Feature]/review-log.md` with the round's graded findings and the assessor's verdict; the plan status is set to "Reviewed" once converged.

plan-review's interactive round-3 design checkpoint does not fire here (there is no user to discuss with overnight); a loop still open after round 3 runs straight on toward the cap, and any design question arrives via the escalation path below.

**If the assessor returns `Escalate`** (a recurring One-way area, an unsettleable One-way, or the round-5 cap), the plan has a problem more rounds cannot fix. **Stop the pipeline here.** Do not proceed to Stage 4 on an unsettled architecture, and do not keep looping. Record the escalation (trigger, root cause, the decision required) as the first and blocking item of the Stage 3 tradeoff gate, present Stage 3 to the user, and wait. The pipeline resumes only after the user decides (redesign the plan and re-run Stage 2, accept the residual explicitly, or knowingly authorize more review rounds).

---

## Stage 3: Tradeoff Gate

Run `/tradeoff-review [Feature]`.

This walks the user through every tradeoff from the plan and the review loop **one at a time**, collecting a decision on each (accept / fix now / skip). The tradeoff-review skill handles gathering, deduplication, and presentation.

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

**Important:** Tell the implement-plan skill to proceed through all phases without waiting for confirmation between phases. The plan has been reviewed to convergence: phase-level confirmation adds friction without value at this point.

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
- [N] plan review rounds (ran to convergence)
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
- **If the user interrupts at any gate:** Resume from the last completed stage. Check progress.md and the `review-log.md` sidecar to determine current state.
- **If context runs low:** The conversation may be compacted between stages. Each stage re-reads its inputs (plan, PRD, progress doc) so no context is assumed to persist across stages.
