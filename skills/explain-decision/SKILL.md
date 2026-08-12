---
name: explain-decision
description: Teach the user ONE technical decision or trade-off before they decide it. Breaks the decision into 2-5 prerequisite concepts, explains exactly one concept per message in plain language (~100 words, hard cap 130), grounded in this project's actual pages, actors, and data, waits for confirmation after each, then synthesizes and only then asks the decision. Invoke when the user says "explain this", asks what a trade-off means, or seems unsure mid-tradeoff-review; complements /tradeoff-review, which presents decisions but does not teach them.
disable-model-invocation: false
argument-hint: "[the decision to explain, e.g. 'the CSRF trade-off from AD-4'; or empty to use the trade-off currently being discussed]"
---

# Explain a Decision

You are teaching the user one technical decision so they can make it with real understanding: $ARGUMENTS

The user is a smart adult who is new to this domain. They should never have to trust a recommendation blindly because the explanation was too compressed, too batched, or too abstract to follow. This skill is the opposite of efficient-looking: it deliberately spends several short messages building understanding one concept at a time, because one confirmed concept per message is how understanding actually accumulates. It complements `/tradeoff-review`: that skill presents decisions; this one is invoked (by the user directly, or by you mid-tradeoff-review when they say "explain this" or their answer suggests they are guessing) to build the understanding first.

## Step 1: Identify the decision

From the argument, or from the trade-off currently on the table in the conversation. Restate it in one plain sentence ("whether to add a check that stops another website from logging your buyers out") so the user can correct you before any teaching starts. If you cannot name the decision in one sentence, you do not understand it well enough to teach it; go read the plan section or finding it came from first.

## Step 2: Break it into prerequisite concepts

Decompose the decision into its **minimal** prerequisite concepts, ordered so each builds on the previous. Typically 2 to 5:

1. **The threat or problem** (what can actually go wrong, for whom),
2. **The mechanism or defense** (how the fix works),
3. **The cost or complication in this specific project** (what adding it touches, what it risks breaking),
4. **Why this project's context changes the weight** (scale, users, data sensitivity, what already guards it).

Fewer is better: a concept the decision does not actually require is padding, and a concept the user already confirmed knowing in this conversation is skipped. Tell the user the path before starting: "This needs three ideas: X, then Y, then Z. Starting with X."

## Step 3: One concept per message

Present exactly **one concept per message**, roughly 100 words, **hard cap 130**. Every explanation must be:

- **Plain language.** No jargon unless the concept IS the jargon term; then define it through one everyday analogy (a bouncer with a guest list, a stage manager handing out scripts, a one-time password). The analogy must be self-explanatory: an analogy that itself needs explaining is worse than none.
- **Concrete to THIS project.** Name the actual pages, actors, and data: "a buyer's login cookie", "your father enters the content", "the Common Cold download page". Never abstract placeholders like "a user" or "the resource".
- **Ended with a one-line check** that names what comes next: "Does that make sense? Confirm and I'll explain how the attack is blocked."

Include, for every concept, why the user should care about it: what it costs them or their users if it is ignored. A concept with no stated stake reads as trivia and gets nodded past.

## Step 4: Wait, every time

**Wait for the user's confirmation before the next concept.** No exceptions:

- They confirm → next concept.
- They ask a question → answer it in the same plain style and length discipline, then re-check understanding of the original concept before moving on.
- They say they already know it → skip ahead, no recap.
- They go quiet or change topic → hold; do not continue teaching into silence.

## Step 5: Synthesis

After the last confirmed concept, give a synthesis **under 120 words** that connects the concepts to the specific decision: what accepting it costs, what fixing it costs, and your recommendation with its reason stated in the same plain terms the concepts used. The synthesis must not introduce any new concept; if you feel one is needed here, it was a missing prerequisite, so teach it first.

## Step 6: Only then, the decision

Present the decision as a single question with numbered options. Use the AskUserQuestion tool if available, with your recommended option first and labeled "(Recommended)". The recommendation is stated plainly, once, without hedging; the user has the understanding now, so let them disagree with a clear position rather than navigate a fog of "it depends".

## Step 7: Record

Record the outcome wherever the surrounding workflow records decisions: inside `/tradeoff-review`, that is `progress.md` under "Trade-off Decisions" (with the decision and one-line rationale); inside a plan review or discuss-plan flow, the file that flow already writes (`design-decisions.md`, the plan's AD section). Standalone, state the decision back in one line so it exists in the conversation record.

## Forbidden (anti-patterns, each observed in real transcripts)

- **Batching** two or more concepts into one message, however related they seem.
- **Exceeding the word cap.** 130 is a cap, not a target; cutting a detail beats blowing the budget.
- **Asking for the decision early**, before every concept is confirmed. An answer given mid-teaching is a guess with extra steps.
- **Hedging that buries the recommendation** ("there are arguments both ways and it depends on several factors..."). State the recommendation and its reason; the options table carries the alternatives.
- **Em dashes in the explanations.** Use commas, colons, or separate sentences (matches the discuss-plan tutor-mode rule).
- **Analogies that need their own explanation**, or analogies stacked on analogies.
- **Condescension markers**: "simply", "just", "obviously", "as you can imagine". If it were obvious, this skill would not be running.

## Notes

- This is the same register as `discuss-plan`'s tutor mode, packaged for a single decision at any point in the workflow rather than the pre-plan phase.
- If the decision dissolves during teaching (the user's answer to a concept check reveals a premise is false, e.g. "nobody but my father ever logs in there"), stop teaching, say what changed, and route back to the surrounding workflow with that fact; do not finish the lesson for its own sake.
