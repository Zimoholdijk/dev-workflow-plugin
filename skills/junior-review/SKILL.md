---
name: junior-review
description: Spawn the junior-reviewer sub-agent to ask clarifying questions about a plan, PR, or piece of work. Use when you want to surface ambiguity, missing steps, and edge cases before implementation.
disable-model-invocation: false
argument-hint: "[path to plan or file to review]"
---

# Junior Engineer Review

You have been asked to get a junior engineer review of: $ARGUMENTS

## Before starting

Gather context that the reviewer will need. Read:
- The file to review
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the file

## Run the review

Spawn the `junior-reviewer` sub-agent. In your prompt, include:
1. The full text of the file to review
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. A summary of the current state of the codebase relevant to the review (e.g., current schema, file structure, related files)

## After the review

Present the junior reviewer's questions to the user. For each question, briefly note whether you think it's a valid concern or not, and suggest how to address it if applicable. Let the user decide what to act on.
