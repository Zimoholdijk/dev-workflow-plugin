---
name: senior-review
description: Spawn the senior-reviewer sub-agent to review a plan, PR, or architecture decision for architectural fit, scope creep, risk, and alignment with project goals.
disable-model-invocation: false
argument-hint: "[path to plan or file to review]"
---

# Senior Architect Review

You have been asked to get a senior architect review of: $ARGUMENTS

## Before starting

Gather context that the reviewer will need. Read:
- The file to review
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the file

## Run the review

Spawn the `senior-reviewer` sub-agent. In your prompt, include:
1. The full text of the file to review
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. A summary of the current state of the codebase relevant to the review

## After the review

Present the senior reviewer's feedback to the user, structured as:
1. **Verdict** (Approve / Approve with changes / Request changes)
2. **Critical issues**: with your take on each
3. **Suggestions**: with your take on each
4. **What's good**

Let the user decide what to act on.
