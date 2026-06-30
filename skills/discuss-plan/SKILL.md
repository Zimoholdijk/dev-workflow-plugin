---
name: discuss-plan
description: Run a pre-plan architecture and design discussion in plain language, grounded in the approved PRD and the actual codebase. Walks the big design decisions one short topic at a time, researches technical trade-offs docs-first before surfacing them, asks you to run SQL or check production where a decision needs facts it can't get, collects the agreed decisions, and hands off to /write-plan. Sits between /write-prd and /write-plan to settle architecture before phasing, heading off later plan-review rounds. Takes a feature name or PRD path as argument.
disable-model-invocation: false
argument-hint: "[feature name or path to PRD, e.g. 'Claiming' or 'context/Claiming/ClaimingPRD.md']"
---

# Discuss Plan

You are facilitating a pre-plan design discussion for: $ARGUMENTS

The goal is to settle the architecture and design decisions, in plain language, before anyone writes a phased plan. This sits BETWEEN `/write-prd` and `/write-plan`: the PRD has fixed *what* and *why*; here you agree on the *shape of the solution*; then `/write-plan` turns that shape into phases and file changes. The decisions collected here become the plan's Architecture Decisions, already grounded and researched, so the cold plan-review rounds have far less to raise.

## Prerequisite

The PRD must be **approved** first. Read it before the first message. If its status is "Draft", stop and tell the user the PRD needs approval before a design discussion. This skill does not invent requirements; it decides how to build the ones the PRD already fixed.

## Response Rules

These are non-negotiable and apply to every message in the discussion:

1. **150 words maximum per message.** Hard cap. If a topic needs more, it is two topics.
2. **One topic per message.** Never bundle. Wait for the user's answer before moving to the next topic.
3. **Plain English.** Conversational prose, no jargon walls, no code blocks, no Prisma/TypeScript/SQL syntax, no headers, no bullet-heavy structure. (One exception: the decision summary table at the end.) Describe the approach in words, not code.
4. **Every message ends with exactly one question.** Either yes/no on your recommendation ("Agree we key off the existing tenant id?") or a single open lean ("Which way do you lean on storing the snapshot?"). Never numbered option menus, never stacked asks.
5. **Trade-offs in prose with a recommendation, researched first.** Describe 2-3 realistic options in flowing sentences, say which you lean toward and why, then let the user decide. But research the technical ones before you surface them (see Step 3): documented best practice often settles a trade-off outright, and the user should only adjudicate genuinely open choices. Never decide silently; never hand over a bare "A or B?" you could have researched.
6. **Record and move on.** Once the user decides, do not relitigate. Acknowledge in a few words and open the next topic.
7. **No em dashes in customer-facing copy** you draft. The discussion and the design doc are internal, em dashes there are fine.
8. **Architecture-decision altitude.** Discuss the shape of the solution: data model, contracts and API surface, auth and trust boundaries, how it integrates with existing code, error and edge-case strategy, performance at real data volumes, migration and rollback. Stay above the plan: no phase breakdown, no file-by-file list, no code (those are `/write-plan`'s job). Stay above the PRD too: do not reopen what or why; if a requirement seems wrong, flag it as a PRD question, do not redesign around it silently.

## Step 1: Ground yourself (silently, before the first message)

Read, do not dump at the user:

- The approved PRD for this feature.
- `context/overview.md`, `.claude/CLAUDE.md`, `~/.claude/CLAUDE.md`.
- The **actual code** the feature touches: the relevant routes, components, middleware, server utilities, and the current schema (`prisma/schema.prisma` or the baseline migration). Read the real function bodies, policies, and existing patterns, not just file names.
- The **installed versions** (`package.json` / `deno.json` / lockfiles / `prisma/schema.prisma`) so every option you raise is one this stack actually supports.

Use this to make each topic concrete and grounded in current behavior ("today claims live on the listing row, so adding status there is cheaper than a new table") instead of generic.

## Step 2: Build a private topic list, one-way doors first

Sketch the design topics this feature needs, then **order the expensive-to-reverse ones first**. The decisions that are costly to undo are exactly the ones `/plan-review` grades One-way or Significant and that trigger extra rounds; settling them here, grounded and agreed, is how this skill heads off that churn. Prioritize:

- **Published contracts** anything you can't update in lockstep depends on (a public/versioned API, wire or webhook payload, a library's public surface, CLI flags).
- **Persisted data shapes**, once production data exists (schema, stored formats, enum raw values, the meaning of IDs and keys).
- **Event or message schemas** on an append-only or async channel.
- **Auth, tenancy, and trust boundaries** (the security posture, where trust sits, secret handling).
- **Publicly observable behavior** that becomes a de-facto contract (URL structure, external IDs, error shapes).
- **Foundational platform commitments** (primary datastore, runtime, a cloud-proprietary primitive).
- **Distributed-correctness commitments** (consistency, ordering, idempotency, the partition key).

Then the reversible-but-consequential design choices: component architecture and state management, error/empty/loading strategy, performance approach at real row counts, build-vs-reuse, and the **branched-vs-unified** shape (N near-identical paths differing only by a literal usually want to be one path with the difference as data). End with the scope boundary: what this design deliberately defers to a later feature.

Adapt the list to the feature; drop what does not apply, add what does. Keep it internal. Optionally preview the next topic in one trailing sentence.

## Step 3: Research a trade-off before you surface it

For any topic with a **technical or best-practice dimension** (framework behavior or idiom, data modeling, API design, security, performance, a library choice), run `/research` before you present it, grounded in the project's stack and versions. Run several in parallel when topics are independent. Then route by what the evidence shows:

- **Settled by evidence** (docs/best practice clearly favor one option, or the downside is negligible at this scale): present the recommended choice as the lean, name the source in one phrase, and let the user object. It is a resolved decision with a citation, not an open question.
- **Genuine choice** (defensible either way, depends on product, UX, or risk preference): surface it as a real trade-off, carrying the researched points (what the docs say, the real cost of each side, your recommendation). One decision per message, never a bare "A or B?".

Pure product/UX-preference choices with no documented answer don't need research; raise them as plain leans. This is the same research-before-surfacing discipline `/write-plan` and `/plan-review` use; doing it now means the plan arrives with the technical trade-offs already settled.

## Step 4: When a decision needs facts, get them or ask for them

Some design calls turn on facts you cannot read off the code: whether production data already exists, how many rows a table really holds, what a query currently costs, whether an index or constraint is actually present, what a live policy or trigger does at runtime, current error rates. Do not guess these, they change the answer (and a wrong guess on a one-way door is the expensive mistake).

- **Investigate yourself** where you can: read the migration, the schema, the function body. If a documentation or database MCP is connected (for example a Supabase MCP with `search_docs`, `list_tables`, or `execute_sql`, or read-only logs/advisors), use it to check docs and inspect non-destructive state.
- **Ask the user to investigate** what you can't reach: a specific SQL query to run against production or staging, a row count, a metric to read, a check in the dashboard. Give them the exact thing to run and what you'll do with each outcome, then fold the result into the decision before moving on.
- When a fact governs an irreversible call and you genuinely cannot confirm it, **assume the safe direction** (e.g. assume production data exists) and say you did, so the design errs toward reversibility.

## Step 5: Walk the topics

Open with the shape topic: state in one or two plain sentences how you'd build this overall, grounded in what already exists, and check it matches the user's mental model. Then one topic per message, each following the Response Rules. Within a topic:

- If the user asks a clarifying question, answer it inside the word budget, then re-ask your question.
- If the user proposes something, evaluate it honestly: agree and build on it, or push back with the concrete cost.
- If the user seems lost in your phrasing, restate concretely (name the two or three real artifacts: the table, the endpoint, the boundary) rather than re-explaining abstractly.
- Make reversibility explicit when it matters ("this one is hard to undo once we ship data in that shape, so it's worth getting right now") so the user weights the decision correctly.

## Step 6: Wrap up

When the topic list is exhausted, ask whether there is anything else to settle before planning. Then:

1. **Decision summary.** One short table: Decision area | Choice | Basis (research citation, an established fact, or "product call"). This is the one place structure is allowed.
2. **Write the design doc.** Save the agreed decisions to `context/[Feature]/design-decisions.md` (structure below). Create the feature directory if needed; it usually exists from PRD writing.
3. **Handoff.** Suggest `/write-plan [Feature]` as the next step, noting that these decisions pre-fill the plan's Architecture Decisions already researched and grounded, so the plan-review rounds have less to raise.

Do not write the implementation plan inside this skill. The discussion ends at the design doc.

## Output: the design-decisions doc

Save to `context/[Feature]/design-decisions.md`:

```markdown
# [Feature]: Design Decisions

> Pre-plan architecture decisions agreed before `/write-plan`. Feeds the plan's
> Architecture Decisions. PRD: `context/[Feature]/[Feature]PRD.md`. Internal record.

## Decisions

| # | Decision area | Choice | Basis | Reversibility |
|---|---------------|--------|-------|---------------|
| 1 | [e.g. claim storage] | [the choice agreed] | [citation / fact / product call] | [One-way / Reversible] |
| ... | ... | ... | ... | ... |

## Resolved by research

- [decision]: [choice] per [official source]. [one line on why it's settled]

## Facts established (code / SQL / production)

- [fact the design relies on, and how it was confirmed]

## Open / deferred to the plan

- [anything intentionally left for `/write-plan` to detail, or out of scope for this feature]
```

Keep it prose and tables, no code. It is a decisions record, not a plan.

## Notes

- **This is an interactive gate, not part of `/overnight-delivery`.** Overnight delivery runs unattended from an approved PRD; a design discussion needs the user in the loop. Run `discuss-plan` before kicking off an autonomous pipeline if you want the architecture settled first.
- **It does not replace `/plan-review`.** The cold multi-lens review still runs on the written plan. This skill reduces what that review finds by settling the one-way doors up front; it does not pre-approve anything.
- **Persona-free.** This is a job description, not a character. Do not role-play a "senior architect"; just run the discussion well, grounded in the docs and the code.
