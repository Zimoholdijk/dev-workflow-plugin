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
2. `context/[Feature]/implementation-plan.md` Review Log: look for reviewer suggestions that were noted but not applied, disagreements, and deferred items
3. `context/[Feature]/progress.md` (if it exists): look for "Trade-off Decisions" entries and "Deviations from Plan"
4. Any code review output in the current conversation: look for findings categorized as tradeoffs (not clear fixes)

Build a numbered list of every distinct tradeoff. Deduplicate: if the same concern was raised by multiple reviewers or in multiple rounds, merge into one entry.

## Step 2: Present overview

Show the user a summary before starting:

```
## Tradeoff Review: [Feature]

Found [N] tradeoffs to review. I'll walk through each one individually.

| # | Summary | Source |
|---|---------|--------|
| 1 | [one-line description] | [plan AD-3 / review round 2 / code review / etc.] |
| 2 | ... | ... |
| ... | ... | ... |

Starting with #1.
```

## Step 3: Walk through one by one

For **each** tradeoff, present it as a single message with this structure:

```
### Tradeoff [N] of [total]: [short title]

**What:** [1-2 sentence description of the tradeoff]

**Why it exists:** [context: why this came up, what constraint or design choice created it]

**Risk if we accept it:** [what could go wrong, who is affected, how likely]

**Alternatives:**
1. [Accept as-is]: [consequence]
2. [Fix now]: [what the fix involves, rough effort]
3. [Other option if applicable]: [consequence]

**My recommendation:** [which option and why, stated clearly]

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
