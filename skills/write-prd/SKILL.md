---
name: write-prd
description: Draft a PRD for a new feature following the project's established format and workflow rules. Takes a feature name or description as argument. Produces a structured PRD for user review and approval.
disable-model-invocation: false
argument-hint: "[feature name or short description, e.g. 'claiming flow']"
---

# Write PRD

You are drafting a PRD (Product Requirements Document) for the feature: $ARGUMENTS

## Rules

These rules are non-negotiable:

1. **PRD first, implementation plan second, code last.** This PRD must be approved before any implementation planning begins.
2. **Document freeze.** Once the user approves, the PRD is frozen. All deviations during implementation go in `progress.md`, not by editing the PRD.
3. **Never accept trade-offs silently.** Surface every trade-off, limitation, and open question. Let the user decide. Document their decision, not your judgment.
4. **No placeholder text.** All user-facing copy must be real. No lorem ipsum, no "TBD" in decided items.
5. **Mobile first.** All UI descriptions assume mobile as the primary viewport.
6. **The PRD describes *what* and *why*, never *how*.** No implementation details, no code, no technology choices (unless the technology IS the decision, e.g. "magic link only, no OAuth"). Implementation details belong in the implementation plan. Exception: infrastructure/migration PRDs may reference file paths in "Affected Areas" since the files ARE the scope.

## Step 1: Gather context

Before writing anything, read:

1. `context/overview.md`: project overview, tech stack, existing features, deferred items
2. `context/toyswapprd.md`: the main product PRD (if it exists)
3. `.claude/CLAUDE.md`: project rules
4. `~/.claude/CLAUDE.md`: global rules
5. Any existing feature docs in `context/` that the new feature depends on or interacts with (check the "Depends on" chain)
6. The Linear ticket for this feature (if a ticket ID is provided in the argument, use the Linear MCP tools to fetch it)

Also check if a PRD already exists for this feature in `context/`. If it does, inform the user and ask whether they want to replace it or revise the existing one.

## Step 2: Draft the PRD

Use this exact structure. Every section is required for user-facing features. For infrastructure/migration features (no direct user interaction), you may omit "User Stories", "Screens / Flow", "Success Criteria", and "Open Questions". Replace User Stories and Screens/Flow with "Affected Areas" (grouped by Server/Frontend/Pages) and an "Environment Variables" table if the feature introduces new env vars. Use "Decided" as the section name (not "Decisions") for consistency.

```markdown
# [Feature Name] PRD

> **Feature:** [Feature Name] · **Ticket:** [TOY-XX] · **Status:** Draft
> **Depends on:** [TOY-XX (Feature, done/in progress), ...] or "None"

---

## Overview

[1-3 paragraphs. What problem does this solve? Why now? How does it fit into the product?
Be specific about the current gap: what can't users do today that they need to do?]

---

## Goals

[5-7 bullet points. Each goal is a concrete outcome, not a task.
Good: "Let users cancel a pending request before the owner responds"
Bad: "Build a cancel button"]

---

## Out of Scope

[Explicit list of what this feature does NOT include. Reference ticket IDs for future work.
Each item should be something a reasonable person might assume IS in scope.
Format: "- [Thing] ([reason or future ticket])"  ]

---

## User Stories

### [Group 1: e.g. "Requesting"]

- As a [role], I want [action] so I can [benefit]
- ...

### [Group 2: e.g. "Responding to Requests"]

- ...

[Group by user journey or feature area. 3-5 groups typical. 3-5 stories per group.
Stories should be testable: if you can't demo it in a browser, it's too vague.]

---

## Screens / Flow

### [Screen 1 Name]

[Prose description of what the user sees and can do. Not wireframes.
Include: what's visible, what actions are available, what happens on each action.
Call out mobile-specific considerations.
Reference related screens by name.]

### [Screen 2 Name]

[...]

[One subsection per distinct screen or state. Include empty states, error states,
and edge cases (e.g. "what if the user has 0 credits?")]

---

## Success Criteria

[Bullet list of testable statements. Each one should be verifiable by a human
clicking through the app. No technical criteria: those go in the implementation plan.
Format: "- A user can [do X] and [Y happens]"]

---

## Decided

| Question | Decision |
|----------|----------|
| [Design question that came up] | [The decision made, stated clearly] |
| ... | ... |

[3-10 rows. These are decisions that could reasonably go either way.
Don't include obvious things. Include anything the user explicitly decided.
This table is frozen after approval: it's the historical record.]

---

## Open Questions

[Items that need user input before the implementation plan can be written.
If there are no open questions, write "None at this time."
Format: numbered list with enough context to make a decision.]
```

## Step 3: File placement

Save the PRD to: `context/[FeatureName]/[FeatureName]PRD.md`

- Create the `context/[FeatureName]/` directory if it doesn't exist
- Use PascalCase for the feature directory name (e.g., `Claiming`, `Messaging`, `Notifications`)
- If a sensible short name exists, use it (e.g., `Claiming` not `RequestAndClaimFlow`)

## Step 4: Present for review

After writing the PRD, present it to the user with:

1. A brief summary of what the PRD covers
2. Call out any decisions you made assumptions about (these should be in the "Decided" table with your reasoning, the user may override)
3. List the open questions that need answers before proceeding to the implementation plan
4. Ask explicitly: "Should I adjust anything, or is this ready to approve?"

Do NOT proceed to implementation planning until the user explicitly approves. "Looks good", "approve", or similar explicit language counts as approval. Silence, "ok", "thanks", or a thumbs-up does not; if the signal is ambiguous, ask whether they're ready to move on to the implementation plan.

## Patterns from existing PRDs

Three PRDs exist in the project. Follow their established conventions:

### Header evolution
- Discovery PRD (earliest): `**Feature:** ... · **Version:** 0.1 · **Status:** Ready for Review`: no Ticket, no Depends on
- Listing PRD (later): `**Feature:** ... · **Ticket:** TOY-8 · **Status:** Approved` + `**Depends on:** ...`
- ImageStorage PRD (latest): `**Feature:** ... · **Status:** Approved` + `**Depends on:** ...`: no Ticket (infra, no dedicated ticket)

Use the **Listing format** (with Ticket and Depends on) for new PRDs. Omit Ticket only if no Linear ticket exists.

### Content patterns
- **Overview tone:** Direct, specific, problem-focused. Opens with the gap, not the solution. Example from Listing PRD: "After onboarding, users have no way to add more toys or manage their existing listings."
- **Goals:** Verb-led bullets. "Let users...", "Give users...", "Keep the experience consistent with..."
- **Out of Scope:** Each item includes a reason or ticket reference. "Notifications when someone requests a toy (depends on claiming flow, TOY-7)"
- **User Stories:** Standard "As a [role]..." format. Grouped by journey, not by technical component. 3-5 groups, 2-5 stories per group.
- **Screens / Flow:** Prose paragraphs per screen. Describe what the user sees, not how it's built. Include CTA copy verbatim. Reference ticket IDs for flows that depend on unbuilt features.
- **Decided table:** Two columns only: Question | Decision. No rationale column: the decision should be self-explanatory or the Overview/context makes it clear.
- **Open Questions:** Brief. "None at this time." if everything is decided. Don't manufacture questions.

### Infrastructure PRDs (like Image Storage)
Simpler structure. Skip User Stories, Screens/Flow, Success Criteria, and Open Questions. Add instead:
- **Affected Areas**: grouped by Server / Frontend / Pages, listing specific files and what changes
- **Environment Variables** table: Variable | Example (dev) | Description
