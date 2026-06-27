---
name: tradeoff-review
description: Walk the user through accumulated tradeoffs one by one, collecting decisions. Used at tradeoff gates and after code reviews. Takes a feature name or "pending" to review all unresolved tradeoffs.
disable-model-invocation: false
argument-hint: "[feature name, e.g. 'RelatedToys']"
---

# Tradeoff Review

You are walking the user through tradeoffs for: $ARGUMENTS

This skill presents tradeoffs **one at a time** and collects the user's decision on each before moving to the next. It is used at two points in the overnight-delivery pipeline:
1. **Stage 3 (Tradeoff Gate):** After 4 plan review rounds, before implementation begins.
2. **End of pipeline:** After code reviews, collecting any remaining tradeoffs from implementation and review findings.

It can also be invoked standalone to review accumulated tradeoffs for any feature.

## Step 1: Gather tradeoffs

Read the following files to collect all unresolved tradeoffs:

1. `context/[Feature]/implementation-plan.md`: look for "Known limitation:", "Known trade-off:", and items in Architecture Decisions marked as accepted risks
2. `context/[Feature]/review-log.md` (the review-log sidecar, if it exists): look for reviewer suggestions that were noted but not applied, disagreements, and deferred items across rounds
3. `context/[Feature]/progress.md` (if it exists): look for "Trade-off Decisions" entries and "Deviations from Plan"
4. Any code review output in the current conversation: look for findings categorized as tradeoffs (not clear fixes)

Build a numbered list of every distinct tradeoff. Deduplicate: if the same concern was raised by multiple reviewers or in multiple rounds, merge into one entry.

## Step 1.5: Research each tradeoff before presenting it

Many "tradeoffs" dissolve under a few minutes of docs-grounded research: one option is the documented best practice, or the downside is a non-issue at this project's scale. Don't make the user adjudicate a question the framework's docs already answer.

For each tradeoff that has a **technical or best-practice dimension** (framework behavior, idiom, security, performance, data modeling, API design), run `/research` on it, grounded in the project's stack and versions. Research several in parallel rather than serially. Then sort each tradeoff:

- **Resolved by evidence** — the docs/best practices clearly favor one option, or the concern is negligible here. Pull it OUT of the one-by-one walk: take the recommended option and list it in the overview under "Resolved by research" with a one-line rationale and citation, so the user can still object but doesn't spend a decision turn on a settled question.
- **Genuine tradeoff** — still defensible either way, or the choice depends on product/UX/risk preference the user owns. Keep it in the walk-through, and attach the researched evidence to its presentation (Step 3).

Pure product/UX-preference tradeoffs with no documented answer (e.g. sticky vs scroll-away banner) skip research, keep them in the walk-through as-is.

## Step 2: Present overview

Show the user a summary before starting:

```
## Tradeoff Review: [Feature]

Researched [M] tradeoffs; [K] were resolved by documented best practice, [N] genuinely need your call.

### Resolved by research (FYI — tell me if you disagree)
| # | Tradeoff | Resolution | Source |
|---|----------|-----------|--------|
| - | [description] | [option taken + one-line rationale] | [doc/best-practice citation] |

### Need your decision ([N])
| # | Summary | Source |
|---|---------|--------|
| 1 | [one-line description] | [plan AD-3 / review round 2 / code review / etc.] |
| 2 | ... | ... |
| ... | ... | ... |

Starting with #1.
```

Omit the "Resolved by research" table if nothing was resolved that way. If ALL tradeoffs were resolved by research, say so and skip the one-by-one walk entirely — just confirm the resolutions and let the user object to any.

## Step 3: Walk through one by one

For **each** tradeoff, present it as a single message with this structure:

```
### Tradeoff [N] of [total]: [short title]

**What:** [1-2 sentence description of the tradeoff]

**Why it exists:** [context: why this came up, what constraint or design choice created it]

**Researched evidence:** [what the docs / best practices say about this, with citation; or "no documented best practice — this is a judgment call." This is why it reached you instead of being auto-resolved: either the evidence is genuinely split, or the choice is a preference the docs can't settle.]

**Risk if we accept it:** [what could go wrong, who is affected, how likely]

**Alternatives:**
1. [Accept as-is]: [consequence]
2. [Fix now]: [what the fix involves, rough effort]
3. [Other option if applicable]: [consequence]

**My recommendation:** [which option and why, informed by the researched evidence above]

Which option?
```

**Rules:**
- Present ONE tradeoff per message. Do not batch.
- Wait for the user's answer before presenting the next one.
- If the user says "accept" or picks option 1, record it and move on.
- If the user says "fix" or picks option 2+, note it as an action item.
- If the user asks a clarifying question, answer it, then re-present the options.
- If the user says "skip" or "later", mark it as unresolved and move on.

## Step 4: Summary and action items

After all tradeoffs are reviewed, present:

```
## Tradeoff Review Complete

### Accepted ([N])
| # | Tradeoff | Decision |
|---|----------|----------|
| 1 | [description] | Accepted: [brief rationale from user] |

### Fix now ([N])
| # | Tradeoff | Action needed |
|---|----------|---------------|
| 3 | [description] | [what to fix] |

### Unresolved ([N])
| # | Tradeoff | Note |
|---|----------|------|
| 5 | [description] | Skipped: revisit later |

### Verdict
[Can implementation proceed? / Are fixes needed first? / Should any accepted items be tracked as tech debt?]
```

If there are "fix now" items:
- Apply the fixes before proceeding to the next pipeline stage.
- After fixing, briefly confirm what was changed.

If all items are accepted or skipped:
- Record accepted tradeoffs in `progress.md` under "Trade-off Decisions" (if progress doc exists).
- Proceed to the next pipeline stage.

## Standalone usage

When invoked outside the overnight-delivery pipeline (e.g., the user runs `/tradeoff-review RelatedToys` directly), gather tradeoffs from the plan and progress doc only (no conversation-level code review output). Present the same one-by-one flow. At the end, update `progress.md` with decisions.
