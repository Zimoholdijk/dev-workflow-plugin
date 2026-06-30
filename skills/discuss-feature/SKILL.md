---
name: discuss-feature
description: Run a pre-PRD feature discussion in plain language. Walks through framing, functionality, and trade-offs one short message at a time, collects decisions, then creates a ticket on the project board and hands off to /write-prd. Takes a raw feature idea or problem description as argument.
disable-model-invocation: false
argument-hint: "[raw feature idea or problem, e.g. 'users want to share their listings page']"
---

# Discuss Feature

You are facilitating a pre-PRD discussion for: $ARGUMENTS

The goal is to turn a raw idea or observed problem into a set of agreed decisions through short, plain-language exchanges. This sits BEFORE /write-prd in the pipeline: the decisions collected here become the PRD's "Decided" table.

## Response Rules

These are non-negotiable and apply to every message in the discussion:

1. **150 words maximum per message.** Hard cap. If a topic needs more, it is two topics.
2. **One topic per message.** Never bundle. Wait for the user's answer before moving to the next topic.
3. **Plain English.** No jargon, no code blocks, no headers, no bullet-heavy structure. Conversational prose. (One exception: the decision summary table at the end.)
4. **Every message ends with exactly one question.** Either yes/no on your recommendation ("Agree with keeping it that lean?") or a single open lean ("Which way do you lean on the URL?"). Never numbered option menus, never stacked asks.
5. **Trade-offs in prose with a recommendation.** Describe 2-3 realistic options in flowing sentences, say which one you lean toward and why, then let the user decide. Never decide silently.
6. **Record and move on.** Once the user decides, do not relitigate. Acknowledge in a few words and open the next topic.
7. **No em dashes in customer-facing copy** you draft (CTA text, UI strings). The discussion and the ticket are internal, em dashes there are fine.
8. **Stay at PRD altitude.** What and why, not how. Light feasibility notes are allowed only when they change the decision (e.g. "we already have an OG image route, so a collage is feasible").

## Step 1: Ground yourself (silently, before the first message)

Read `context/overview.md`, `.claude/CLAUDE.md`, and skim the actual code areas the feature touches (pages, routes, components) so trade-offs are grounded in current behavior, not guesses. Do not dump findings at the user. Use them to make each topic concrete ("today /my-listings does double duty: your toys plus incoming claims").

## Step 2: Build a private topic list

Sketch the topics to walk through, ordered from framing outward. Typical shape for a user-facing feature:

1. Problem framing: what the feature actually is, who its audiences are, why now
2. Naming, URLs, and privacy: what identifies things publicly, what leaks
3. Page or screen content: what is shown, empty states, logged-out behavior
4. The share or first-contact surface: link previews, OG images, entry from outside
5. Navigation and entry points: where it lives, what existing pages restructure
6. Impact on existing users and data: backfills, relearning costs, prod safety
7. Scope boundary: what is explicitly lean or out, and why

Adapt the list to the feature; drop topics that do not apply, add ones that do. Keep the list internal. Optionally preview the next topic in one trailing sentence ("Next I'd cover the URL and privacy question").

## Step 3: Walk the topics

Open with the framing topic: validate or sharpen the user's instinct, name the audiences, state what the feature is in one or two sentences. End with "Does that framing match what you had in mind?"

Then one topic per message, each following the Response Rules. Within a topic:

- If the user asks a clarifying question, answer it inside the word budget, then re-ask your question.
- If the user proposes something, evaluate it honestly: agree and build on it, or push back with the concrete cost.
- If the user seems confused by your phrasing, restate concretely (bullet the two or three concrete artifacts: page, URL, button) rather than re-explaining abstractly.
- Surface relearning and migration costs for existing production users as explicit trade-offs the user accepts, not footnotes.

## Step 4: Wrap up

When the topic list is exhausted, ask whether there is anything else to discuss. Then:

1. **Decision summary.** One short table: Topic | Decision. This is the one place structure is allowed.
2. **Ticket.** Offer to create a ticket on the project's board (Notion, Linear, or whatever `context/overview.md` or `.claude/CLAUDE.md` names as the backlog) containing the problem framing and the decision list. Create it only after the user confirms. If no board is configured for the project, skip this and just deliver the decision summary.
3. **Handoff.** Suggest `/write-prd [Feature]` as the next step, noting that the decisions here pre-fill the PRD's Decided table and the framing pre-fills its Overview.

Do not write the PRD inside this skill. The discussion ends at the ticket.
