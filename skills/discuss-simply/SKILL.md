---
name: discuss-simply
description: Step back and break the current topic (a trade-off, a finding, a plan section, any concept) into small steps, one short message at a time. Invoke whenever the user says "explain this", "simplify", "break this down", or their reply suggests they are guessing. The first message must be SHORTER than the text that prompted it; teaching happens across several tiny confirmed messages, never one block.
disable-model-invocation: false
argument-hint: "[what to break down; empty = whatever is on the table right now]"
---

# Discuss Simply

The user asked you to step back and simplify: $ARGUMENTS (or whatever is currently being discussed).

**The prime rule: your next message must be shorter than the one that confused them.** A request to simplify answered with more text is a failure, whatever the text contains. Everything else in this skill serves that rule.

## The flow

1. **First message (hard cap 60 words).** One plain sentence naming the question ("Should re-running the seed file overwrite rows your team edited, or leave them alone?"), then the step list: "Three small ideas get us there: X, Y, Z. Starting with X." Nothing else. No background, no options, no recommendation yet.
2. **One idea per message, ~80 words, hard cap 100.** Plain language. One everyday analogy if the idea is a jargon term, and the analogy must not need its own explanation. Concrete to this project: real pages, real people, real data ("the rows Eric edited in the admin UI"), never "a user" or "the resource". End with a one-line check naming what's next.
3. **Wait every time.** Confirm → next idea. Question → answer it in the same tiny format, re-check, continue. "I know this" → skip. Silence → hold.
4. **Close (cap 80 words).** Connect the ideas to the question in one short paragraph: what each option really costs, and your recommendation with its reason, stated once, no hedging. Then ask the decision as one question with numbered options (AskUserQuestion if available, recommendation first, labeled "(Recommended)") — only if a decision was actually pending; if the user just wanted understanding, stop after the close.
5. **Record** any decision wherever the surrounding workflow records them (progress.md Trade-off Decisions, the plan's Architecture Decisions), then hand back to that workflow.

## Forbidden

- A first message longer than 60 words, or any message past its cap. Cut detail, never stretch the cap.
- Batching two ideas into one message.
- Re-presenting the original wall of text "with context". The step list replaces it.
- Asking the decision before the ideas are confirmed.
- Hedging that buries the recommendation.
- Em dashes in these messages (commas, colons, or separate sentences).
- "Simply", "just", "obviously", "as you can imagine".

## Notes

- Same register as `discuss-plan`'s tutor mode, available anywhere in the workflow.
- If a step reveals a false premise ("nobody but my father ever logs in there"), stop, say what changed, and hand that fact back to the surrounding workflow instead of finishing the lesson.
- If the topic genuinely needs only one small message, send that one message; the step machinery is for when one message can't stay small.
