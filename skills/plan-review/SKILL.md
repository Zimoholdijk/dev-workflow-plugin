---
name: plan-review
description: Multi-stage plan review workflow. Use when an implementation or refactoring plan has been drafted and needs rigorous review before execution. Runs three lenses, a clarifying-questions pass, a deep-critique pass (graded on factoring, framework-idiom, implicit-deletion, test-coverage, and operational failure modes, with cited findings), and an adversarial red-team pass that tries to break the plan, then a quality/conformance check.
disable-model-invocation: false
argument-hint: "[path to plan file]"
---

# Plan Review Workflow

You have drafted or been given a plan at: $ARGUMENTS

Execute this multi-stage review process. You are the main agent: you orchestrate, critically assess all feedback, and make final decisions.

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
1. The full text of the plan
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
- **Trade-offs require user decision: never resolve silently.** If a question surfaces a genuine trade-off (two or more defensible options, where the "right" answer depends on user preference, product priorities, or risk tolerance), STOP and ask the user. One trade-off per message. Wait for the user's answer before moving to the next. Do NOT batch trade-offs into a single "pick A or B for each of these five" message, the user has repeatedly flagged this as the wrong pattern.
- What counts as a trade-off: UX behaviour choices (e.g., sticky vs scroll-away banner), strictness vs usability (e.g., case-sensitive vs case-insensitive email match), defer vs implement-now, scope boundaries, any question the junior reviewer phrased as "is this intentional?" where either answer is plausible. What does NOT count: obvious corrections, missing clarifications, gaps the plan author can resolve unambiguously.
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
1. The UPDATED plan (after Stage 1 revisions)
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. The list of junior questions and your responses (so the senior can see what was already addressed)
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
- **Same trade-off rule as Stage 1 applies.** If a critical issue or suggestion involves a trade-off the user should own (UX, strictness, deferral, scope), STOP and ask the user. One trade-off per message. Do NOT dump a list of "here are three open trade-offs, pick for each" at the end of the workflow. Resolve them interactively before finalising the plan. The Review Log should record the user's decision, not a list of open questions for them to answer later.
- If the senior raises multiple trade-offs, walk them one-by-one in sequence. After each user decision, apply the revision to the plan before asking the next question.

## Stage 3: Red-Team (Adversarial) Review

The junior and senior passes are cooperative and evaluative: one asks what's unclear, the other grades fit. Neither one's job is to *break* the plan. This stage adds the missing adversarial lens, the single highest-signal distinct role in the multi-agent review literature.

### Spawn

Spawn the `red-team-reviewer` sub-agent against the plan **as revised after Stages 1–2** (you want to stress-test the plan you'll actually ship, not the first draft). In your prompt, include:
1. The UPDATED plan
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

Update the plan file with all revisions. At the bottom of the plan, add a section:

```markdown
## Review Log

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

This log is part of the plan, it documents the reasoning behind the final version.

## Re-invocation

This workflow can be invoked as many times as the user wants on the same plan. There is no cap and no diagnostic gate before running again. Each invocation runs all stages fresh against the current state of the plan; rounds from prior invocations don't count. If the user asks for another round, run it. Do not argue that prior reviews should be sufficient, and do not propose skipping ahead as an alternative. Each round appends a new numbered entry to the Review Log.
