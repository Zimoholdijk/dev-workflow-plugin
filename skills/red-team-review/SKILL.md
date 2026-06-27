---
name: red-team-review
description: Spawn the red-team-reviewer sub-agent to adversarially stress-test a plan, PR, or design, it tries to break the plan and returns ranked, cited failure scenarios.
disable-model-invocation: false
argument-hint: "[path to plan or file to review]"
---

# Red-Team (Adversarial) Review

You have been asked to adversarially stress-test: $ARGUMENTS

This is the "try to break it" lens. Where a normal review grades whether a plan is good, the red-team's job is to argue it will fail and show how, the highest-signal distinct role in multi-agent review.

## Before starting

Gather context the reviewer will need. Read:
- The file to review
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the file
- The actual code the plan touches (so attacks are concrete, not hypothetical)

## Run the review

Spawn the `red-team-reviewer` sub-agent. In your prompt, include:
1. The full text of the file to review
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. A pointer to the actual code the plan touches, with the project's real scale (row counts, request rates) where known

Tell it: try to break this plan, ground every attack in a `file:line` or quoted line, be concrete, don't fabricate weaknesses, and rank findings by likelihood × blast radius.

## After the review

Present the red-team's findings to the user, structured as:
1. **Unaddressed critical failure mode?** (Yes / No), and the single most likely way the plan fails in practice
2. **Failure scenarios** ranked worst-first, each with its evidence and severity, and your take on whether it's realistic at this project's scale and stage
3. Recommended fixes, separating clear fixes from genuine trade-offs

Apply critical judgment: the red-team is instructed to attack, so weigh each scenario's realism before acting. Surface genuine trade-offs to the user one at a time; don't batch.
