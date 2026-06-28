---
name: plan-review
description: Multi-stage plan review workflow. Use when an implementation or refactoring plan has been drafted and needs rigorous review before execution. Runs three lenses, a clarifying-questions pass, a deep-critique pass (graded on factoring, framework-idiom, implicit-deletion, test-coverage, and operational failure modes, with cited findings), and an adversarial red-team pass that tries to break the plan, then a quality/conformance check.
disable-model-invocation: false
argument-hint: "[path to plan file]"
---

# Plan Review Workflow

You have drafted or been given a plan at: $ARGUMENTS

Execute this multi-stage review process. You are the main agent: you orchestrate, critically assess all feedback, and make final decisions.

## Cold-start every reviewer (non-negotiable)

Every reviewer, in every stage and every round, reviews the plan **cold**. Pass each reviewer sub-agent only the current plan text and the project context (overview, CLAUDE.md, PRD, and the relevant code). **Never** pass:

- the prior reviewers' questions, or your responses to them,
- the trade-off decisions already made,
- any summary of "what changed", "what was already addressed", or "what a previous round approved",
- the review history in any form.

**The Review Log lives in a sidecar file, not in the plan.** All review history is written to `context/[Feature]/review-log.md`, never appended to `implementation-plan.md`. This is structural, not just procedural: reviewers have Read/Glob/Grep and are told to orient in the codebase, so if the log lived in the plan they could simply open the plan and read last round's conclusions even when you don't hand them the log. Keeping it in a separate file removes that temptation. Belt and suspenders: the reviewer agents are also instructed never to open a `review-log` or prior-review file, and you must not point them at one.

Why this matters: telling (or letting) a reviewer learn that part of the plan was "already addressed" or "already reviewed" anchors it to treat that part as settled, so it stops scrutinizing exactly where prior rounds drew their conclusions. A cold reviewer re-examines those conclusions and routinely finds issues a primed reviewer rubber-stamps. The plan artifact itself reflects the changes you've accepted, that is the thing under review, but nothing in the reviewer's context should signal that any part of it carries a stamp of approval. This holds **within** a single run (the senior and red-team do not see the junior's exchange) and **across** re-invocations (round 2+ sees the plan, not the history).

## Before starting

Gather context that the reviewers will need. Read:
- The plan file itself
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the plan

You will pass ALL of this context to the sub-agents, since they have no prior knowledge of the project.

## Stage 1: Clarifying-Questions Review (junior-reviewer)

Spawn the `junior-reviewer` sub-agent. In your prompt, include:
1. The full text of the plan. (Review history lives in the `review-log.md` sidecar, not the plan, so there's nothing to strip from the plan itself; just never point the junior at the sidecar.)
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. A summary of the current state of the codebase relevant to the plan (e.g., current schema, current file structure)
6. **An orientation instruction.** The junior has Read, Glob, and Grep. Tell it: before asking anything, orient like a day-one engineer, open the files and directories the plan touches and the patterns it references. A question the orientation reading would answer (e.g. "what props does component X take?") is noise; a question that survives the orientation is signal (e.g. "the plan reuses pattern X from `routeZ.tsx:42`, but X is inline-defined and not exported, should I export, move, or duplicate it?"). Ask for project-grounded questions that cite file paths.
7. **A testing instruction.** Tell the junior: as part of orientation, check how the project tests things today (look for `*.test.*`, `*.spec.*`, `playwright.config.*`, a `tests/` or `e2e/` directory, and test scripts in `package.json`). Then, for each phase that adds logic, ask "how will I know this works, and what test proves it?" Flag any phase that adds non-trivial logic but names no test, any critical flow with only a manual check, and any case where the plan assumes test infrastructure that does not appear to exist. Missing test coverage is a first-class clarifying question, not an afterthought.

The junior reviewer will return a list of clarifying questions.

**Your job after Stage 1:**
- Work through EVERY question.
- For each question: either update the plan to address it, or write a brief note explaining why the concern doesn't apply.
- Do not skip questions. Do not dismiss without reasoning.
- **Research a trade-off before you surface it.** Before STOPping to ask the user, run `/research` on any trade-off with a technical or best-practice dimension (framework behavior, idiom, security, performance, data modeling, API design), grounded in the project's stack and versions. Research several in parallel. Then route it:
  - **Resolved by evidence** → the docs/best practices clearly favor one option, or the downside is negligible at this project's scale. This is now a *clear fix, not a user trade-off*: apply the recommended option, record the decision and its citation in the plan, and tell the user what you did and why so they can object. Don't make the user adjudicate a question the docs already answer.
  - **Genuine trade-off** → still defensible either way and the choice depends on product/UX/risk preference. Surface it to the user **with the researched points**: what the docs say, the real cost of each side, and your recommendation. Never hand the user a bare "A or B?" they have to go research themselves.
- **Trade-offs require user decision: never resolve silently.** If a genuine trade-off survives research (two or more defensible options, where the "right" answer depends on user preference, product priorities, or risk tolerance), STOP and ask the user. One trade-off per message. Wait for the user's answer before moving to the next. Do NOT batch trade-offs into a single "pick A or B for each of these five" message, the user has repeatedly flagged this as the wrong pattern.
- What counts as a trade-off: UX behaviour choices (e.g., sticky vs scroll-away banner), strictness vs usability (e.g., case-sensitive vs case-insensitive email match), defer vs implement-now, scope boundaries, any question the junior reviewer phrased as "is this intentional?" where either answer is plausible. What does NOT count: obvious corrections, missing clarifications, gaps the plan author can resolve unambiguously, **and anything documented best practice resolves cleanly** (apply it and note the citation).
- Resolve clear fixes and unambiguous clarifications inline. Surface trade-offs one-by-one before spawning the senior reviewer.
- If there are multiple accumulated trade-offs, invoke the `tradeoff-review` skill to walk them with the user rather than rolling your own list. That skill enforces the one-at-a-time cadence.

## Stage 2: Deep-Critique Review (senior-reviewer)

### Hard rules for this stage

- The senior must grade **internal factoring**, not just spec compliance. A plan that satisfies the PRD can still be structurally wrong.
- A plan that locks in N parallel branches differing only by a literal will produce N parallel code branches. Catch it here, not in code review.
- Repetition / factoring smell is a required evaluation axis. The senior cannot pass a plan without explicitly grading it.
- The senior must grade **against framework idioms**, not just against the spec. The most dangerous failure mode of this stage is endorsing a non-idiomatic pattern because "it implements the plan correctly" while the plan itself was wrong. A pattern with no analog in the framework's official docs is a red flag regardless of how internally consistent the plan looks.
- The senior must grade **what the plan implicitly deletes**, not just what it adds. Plans describe what the new code does; they rarely describe what the old code did that the new code doesn't. That gap is where regressions hide.
- The senior must grade **test coverage** as a required axis. A plan can be architecturally sound and still ship untested. Every phase that adds logic must name the tests it adds for that logic, and the plan must carry a Testing Strategy section. A plan that defers all testing to the end, or omits unit tests for non-trivial logic, or has no automated coverage of its critical flows, cannot pass.
- The senior must grade **operational reality and failure modes** as a required axis: error/empty/auth-boundary handling, rollback for risky migrations, and whether a production failure would be observable. A plan that describes only the happy path is not done.
- **Every finding must be grounded in a citation** (`file:line`, a quoted plan line, or a named rule). An ungrounded criticism is noise; require the senior to verify against the actual files or drop it. Over-flagging correct work is as much a failure as missing a real problem.

### Spawn

Spawn the `senior-reviewer` sub-agent. In your prompt, include:
1. The current plan text. The plan already incorporates the Stage 1 fixes; that's fine, it's the artifact under review. Just don't narrate what changed, and don't point the senior at the `review-log.md` sidecar.
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. **Cold start, per the rule above.** Do NOT include the junior's questions, your responses, the trade-off decisions, or any "here's what we already addressed" summary. If the senior independently re-raises something the junior already surfaced, that is signal (an independent second hit), not wasted effort, do not suppress it by pre-loading the senior with answers.
6. **Explicit instruction (Axis 9): Repetition / factoring smell.** Tell the senior: "Grep the plan's prose for near-identical branch descriptions (verb + object that recur with only a literal, key, separator, or metadata differing). If found, flag it as a structural issue (not a spec violation) and sketch what a unified path would look like. This is grading internal factoring, not just spec compliance. You cannot return a Pass verdict without explicitly stating whether the plan has repetition smell."
7. **Explicit instruction (Axis 10): Framework-idiom check.** Tell the senior: "When the plan involves SQL, RLS policies, ORM queries, framework hooks, middleware, or any code shape governed by a third-party tool's conventions, verify the pattern appears in that tool's official documentation. Constrain doc lookups to the framework's own site (e.g. site:prisma.io/docs, site:supabase.com/docs, site:react.dev), not blogs, Stack Overflow, or community discussions, which routinely endorse non-idiomatic patterns. A pattern that implements the spec correctly but has no documented analog is still likely wrong; homegrown patterns are usually invented to satisfy a prose spec, not because the framework lacks a documented solution. Grade against the framework, not just against the plan." If the senior flags a pattern as possibly non-idiomatic but can't confirm against the docs, run `/research` to settle it with sources before deciding.
8. **Explicit instruction (Axis 11): Behavior the plan implicitly removes.** Tell the senior: "When the plan rewrites an existing hook, module, route, or shared file, read the actual file before the rewrite and enumerate its current responsibilities: every switch case, guard clause, ref and its side effects, exported helper, and documented convention. Check the plan against that list: is each one preserved or intentionally dropped, or does the plan only describe additions and leave preservation implicit? Also scan the plan for 'remove', 'delete', and 'no longer needed': when the plan removes a documented convention, verify the code that convention protected is either preserved or genuinely obsolete. Do not grade 'is the new design coherent'; grade 'would shipping this design quietly break anything the old code did?'"
9. **Explicit instruction (Axis 12): Test coverage.** Tell the senior: "Grade the plan's Testing Strategy and per-phase tests, not just its architecture. Check: (a) does a Testing Strategy section exist at all? (b) does every File Changes row that adds or changes logic map to a named test, unit tests for pure logic, integration tests for endpoints/module boundaries, e2e tests for user-facing flows? (c) are tests written *as each phase lands*, or deferred to a final phase (deferral is a finding)? (d) for each critical flow in the Verification section, is there an automated test, or only a manual check (manual-only critical flows regress silently, flag them)? (e) if the project has no test infrastructure, does Phase 1 establish it before feature work? (f) do the named tests cover error states, auth boundaries, and edge cases, or only the happy path? A plan with no test strategy, or with non-trivial logic and no unit tests, or with critical flows covered only manually, cannot receive a Pass verdict. State explicitly whether the plan's test coverage is adequate."
10. **Explicit instruction (Axis 13): Operational reality & failure modes.** Tell the senior: "Run a DFMEA-style pass over the plan, what could go wrong, ranked by likelihood × blast radius. For each new or rewritten surface: are error states, empty states, and loading states handled or only the happy path? Are auth/ownership boundaries (signed-out, wrong-owner, expired session) enforced or assumed? If a migration or deploy fails mid-way, does the plan name a rollback? Could you tell in production that this broke, is there enough logging/observability to debug it? The most consequential gaps are usually what the plan is *silent* about, not what it states wrongly. You cannot return a Pass while a high-likelihood, high-blast-radius failure mode is unaddressed."
11. **Output requirements.** Tell the senior: "Lead with the single strongest reason this plan should NOT ship. Cite every critical issue and suggestion to a `file:line`, a quoted plan line, or a named rule. Before emitting your critical issues, try to refute each against the actual files; downgrade any you can't substantiate. Do not return Approve while any required clearance (implicit deletions, test coverage, operational failure modes) is an un-justified gap."

The senior reviewer will return a structured review with verdict, critical issues, and suggestions.

**Your job after Stage 2:**
- For each critical issue: update the plan to address it, or document why you disagree.
- For each suggestion: apply if it improves the plan, or note why you're not applying it.
- Be willing to push back on the senior reviewer if their feedback conflicts with the project's stated goals or working agreements.
- **Same trade-off rule as Stage 1 applies.** If a critical issue or suggestion involves a trade-off the user should own (UX, strictness, deferral, scope), STOP and ask the user. One trade-off per message. Do NOT dump a list of "here are three open trade-offs, pick for each" at the end of the workflow. Resolve them interactively before finalising the plan. The review-log sidecar should record the user's decision, not a list of open questions for them to answer later.
- If the senior raises multiple trade-offs, walk them one-by-one in sequence. After each user decision, apply the revision to the plan before asking the next question.

## Stage 3: Red-Team (Adversarial) Review

The junior and senior passes are cooperative and evaluative: one asks what's unclear, the other grades fit. Neither one's job is to *break* the plan. This stage adds the missing adversarial lens, the single highest-signal distinct role in the multi-agent review literature.

### Spawn

Spawn the `red-team-reviewer` sub-agent against the plan **as revised after Stages 1-2** (you want to stress-test the plan you'll actually ship, not the first draft). Cold start still applies: the red-team sees the current artifact, not the history of how it got there. In your prompt, include:
1. The current plan text, with no summary of what Stages 1-2 changed (a "this part was already hardened" hint is exactly the anchor that makes it skip that part). Don't point the red-team at the `review-log.md` sidecar.
2. The full project overview and CLAUDE.md rules
3. Any relevant PRD content
4. A pointer to the actual code the plan touches (so its attacks are concrete, not hypothetical)
5. **The instruction:** "Try to break this plan. Find the wrong assumption, the unhandled failure or partial-failure path, the edge/boundary case, the scale reality at the project's real row counts, the race or data-integrity gap, and above all what the plan is *silent* about. Ground every attack in a `file:line` or quoted plan line, be concrete (a specific scenario, not 'what if it's slow'), and don't fabricate weaknesses to look thorough. If you genuinely can't break it, say so and name the one or two most fragile spots. Rank findings by likelihood × blast radius and end with whether the plan has an unaddressed critical failure mode."

**Your job after Stage 3:**
- Treat each red-team scenario the way you'd treat a senior critical issue: fix the plan, or document why the scenario doesn't apply (with reasoning, not dismissal).
- A genuine, grounded "unaddressed critical failure mode" is a blocker, resolve it before finalizing.
- Same trade-off rule applies: if a fix is a genuine user-owned trade-off (defer the edge case vs handle it now, scope), surface it one at a time. Don't batch.
- Apply your critical judgment: the red-team is instructed to attack, so weigh whether each scenario is realistic at this project's scale and stage before acting. Note dismissed scenarios and why.

## Stage 4: Quality & Conformance Check

Run the `/simplify` skill on the plan to check for unnecessary complexity.

Additionally, review the final plan yourself against the project's CLAUDE.md rules. Check:
- Does the plan introduce any patterns that violate the rules?
- Does the plan address existing violations it should fix?
- Are there DRY, error handling, or hardcoded value concerns?
- Is the scope minimal, does every change serve the stated goal?
- **Repetition smell:** Do any phases describe near-identical work differing only by a literal, key, separator, or metadata value? If yes, propose a unified path (single code path reading the differing value from a small lookup or parameter) before finalizing. Do not rely solely on the senior reviewer for this. Verify yourself.
- **Test coverage:** Does the plan have a Testing Strategy section? Does every phase that adds logic name the tests it adds? Are the critical Verification flows backed by automated tests, not just manual checks? Is test infrastructure established before feature work if none exists? Do not rely solely on the senior reviewer for this. Verify yourself, and if coverage is thin, add the missing tests to the plan before finalizing.

## Stage 5: Final Plan

Update the plan file with all revisions (the plan itself, never a Review Log section). Then write the review history to the **sidecar** at `context/[Feature]/review-log.md`, NOT to the plan. If the sidecar already exists (a prior invocation), append a new round; if not, create it. Use this structure:

```markdown
# [Feature]: Review Log

> Sidecar for `implementation-plan.md`. Review history lives here, never in the plan, so reviewers reading the plan can't be primed by prior rounds. For the author and user only.

## Round [N], [date]

### Clarifying Questions (junior-reviewer)
- [Summary of questions raised and how each was addressed]

### Deep-Critique Review (senior-reviewer)
- Verdict: [their verdict]
- [Summary of critical issues and how each was addressed]
- [Summary of suggestions and whether they were applied or why not]

### Red-Team Review (red-team-reviewer)
- Unaddressed critical failure mode? [Yes/No]
- [Summary of failure scenarios raised, and how each was fixed or why it was dismissed]

### Quality Check
- [Any simplification or conformance changes made]
```

The sidecar documents the reasoning behind the final version. Do NOT add a Review Log section to `implementation-plan.md`. The plan may carry a `Status` field (e.g. "Reviewed") in its header, that one-word stamp is acceptable; the detailed round-by-round history is what must stay in the sidecar.

## Re-invocation

A single invocation runs review rounds in an **internal loop until the plan converges**: you invoke it once and it keeps running rounds until one comes back clean. There is no cap and no diagnostic gate. If the user later asks for another pass on an already-converged plan, run it; never argue that prior reviews should be sufficient or propose skipping ahead. Every round appends a numbered entry to the `review-log.md` sidecar.

## Convergence (the exit condition)

Run rounds in a loop. After each round, evaluate the **exit condition**:

> **Converged** = the round produced no Critical/High findings AND applied no fixes.

- If converged, stop. That clean, no-op round is the signal it's safe to implement.
- If the round applied any fix (or surfaced any Critical/High), run another round.

The rule that drives the loop: **a round that changed anything is never the last round.** Its fixes are themselves unreviewed changes, and fixes routinely introduce new bugs, a fix made to satisfy one round becomes the next round's regression. Only the next cold round verifies them. So you stop on the first round that changes nothing, not on the round that "looks done".

Pausing for the user: the loop resolves clear fixes on its own, but it still obeys the trade-off rule. When a round surfaces a genuine user-owned trade-off (per Stage 1 and Stage 2), stop and ask, one at a time, then resume the loop after the decision. Converging never means deciding trade-offs on the user's behalf.

State the exit-condition result explicitly at the end of each round so the stop is unambiguous:
- **Converged**: no Critical/High, no fixes this round. Safe to implement.
- **Another round**: list what was fixed; those fixes are unverified until a clean round confirms them.

Do not call a plan implementation-ready off a round that made fixes, even if everything looks resolved; the fixes haven't faced a cold reviewer yet.

This is an internal loop with a deterministic exit condition, **not** the Claude Code `/loop` or `/goal` primitives, and it should not use them. `/loop` is for time-spaced recurring tasks; `/goal` is a session-level model evaluator. Here the skill already knows deterministically whether it applied a fix, so it owns the exit decision directly. Cold-start holds across the internal rounds for the same reason it holds across manual re-invocations: each round spawns fresh reviewer sub-agents fed only the plan, with the `review-log.md` sidecar withheld. The loop's accumulating orchestrator context never reaches the reviewers.

The sidecar is for **you and the user**, not for the reviewers. Because it's a separate file (and the reviewer agents are told never to open a review-log file), every round is a genuine cold start: the reviewer evaluates the plan as if for the first time rather than learning it already passed N rounds. A plan that has survived five rounds should still get a reviewer that scrutinizes it as hard as round one; the accumulating "Reviewed" stamps are precisely the anchor we keep out of the reviewer's context.
