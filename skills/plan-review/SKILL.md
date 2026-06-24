---
name: plan-review
description: Multi-stage plan review workflow. Use when an implementation or refactoring plan has been drafted and needs rigorous review before execution. Runs junior engineer Q&A, senior architect review, and quality conformance checks.
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

## Clean-context rule (applies to every reviewer spawn, every stage, every invocation)

Every reviewer must assess the ENTIRE plan from clean context. Never nudge a reviewer toward "what's still open", "what was raised last round", "what the prior reviewer missed", or any other narrowed focus. The reviewer's job is to evaluate the whole plan as if seeing it for the first time, against the full set of criteria for their role.

Concretely:
- **Strip the `## Review Log` section (and any other prior-round artefacts) from the plan text before pasting it into a reviewer prompt.** The Review Log lives in the plan file for the human and for the orchestrator; reviewers must not see it. If you spawn a reviewer and forget to strip it, abort that spawn and redo.
- **Do not pass prior junior questions, prior senior verdicts, prior orchestrator responses, or prior-round diffs to any reviewer.** No "here's what was addressed last time", no "focus on the remaining concerns", no "the previous senior said X, do you agree". Each reviewer starts fresh.
- **Do not summarise the plan's history or narrate what changed.** Hand over the current plan and the project context. That's it.
- **Reviewer prompts must be invocation-invariant.** The prompt you send on round 3 should be structurally identical to round 1 (same context, same instructions, same criteria) — only the plan text differs because the plan itself has evolved.
- This rule overrides any instinct to be "efficient" by pointing reviewers at what's changed. Efficiency here produces blind spots; clean context produces coverage.

## Stage 1: Junior Engineer Review

Spawn the `junior-reviewer` sub-agent. **Apply the clean-context rule above: strip the `## Review Log` section from the plan text before sending it, and do not narrate what was raised or addressed in any prior round.** The junior reads the whole plan from scratch.

In your prompt, include:
1. The current plan, with the `## Review Log` section stripped
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. A factual snapshot of the current state of the codebase relevant to the plan (e.g., current schema, current file structure). State facts, not interpretations — do not flag "this looks risky" or "this changed since round 1"; just describe what's there.
6. **An orientation instruction.** The junior has Read, Glob, and Grep. Tell it: before asking anything, orient like a day-one engineer, open the files and directories the plan touches and the patterns it references. A question the orientation reading would answer (e.g. "what props does component X take?") is noise; a question that survives the orientation is signal (e.g. "the plan reuses pattern X from `routeZ.tsx:42`, but X is inline-defined and not exported, should I export, move, or duplicate it?"). Ask for project-grounded questions that cite file paths.

The junior reviewer will return a list of clarifying questions.

**Your job after Stage 1:**
- Work through EVERY question.
- For each question: either update the plan to address it, or write a brief note explaining why the concern doesn't apply.
- Do not skip questions. Do not dismiss without reasoning.
- **Trade-offs require user decision: never resolve silently.** If a question surfaces a genuine trade-off (two or more defensible options, where the "right" answer depends on user preference, product priorities, or risk tolerance), STOP and ask the user. One trade-off per message. Wait for the user's answer before moving to the next. Do NOT batch trade-offs into a single "pick A or B for each of these five" message, the user has repeatedly flagged this as the wrong pattern.
- What counts as a trade-off: UX behaviour choices (e.g., sticky vs scroll-away banner), strictness vs usability (e.g., case-sensitive vs case-insensitive email match), defer vs implement-now, scope boundaries, any question the junior reviewer phrased as "is this intentional?" where either answer is plausible. What does NOT count: obvious corrections, missing clarifications, gaps the plan author can resolve unambiguously.
- Resolve clear fixes and unambiguous clarifications inline. Surface trade-offs one-by-one before spawning the senior reviewer.
- If there are multiple accumulated trade-offs, invoke the `tradeoff-review` skill to walk them with the user rather than rolling your own list. That skill enforces the one-at-a-time cadence.

## Stage 2: Senior Architect Review

### Hard rules for this stage

- The senior must grade **internal factoring**, not just spec compliance. A plan that satisfies the PRD can still be structurally wrong.
- A plan that locks in N parallel branches differing only by a literal will produce N parallel code branches. Catch it here, not in code review.
- Repetition / factoring smell is a required evaluation axis. The senior cannot pass a plan without explicitly grading it.
- The senior must grade **against framework idioms**, not just against the spec. The most dangerous failure mode of this stage is endorsing a non-idiomatic pattern because "it implements the plan correctly" while the plan itself was wrong. A pattern with no analog in the framework's official docs is a red flag regardless of how internally consistent the plan looks.
- The senior must grade **what the plan implicitly deletes**, not just what it adds. Plans describe what the new code does; they rarely describe what the old code did that the new code doesn't. That gap is where regressions hide.

### Spawn

Spawn the `senior-reviewer` sub-agent. **Apply the clean-context rule above: strip the `## Review Log` section from the plan text, do NOT pass junior questions or your Stage 1 responses, do NOT narrate what changed since the last round.** The senior reviews the whole plan from scratch.

In your prompt, include:
1. The current plan, with the `## Review Log` section stripped
2. The full text of the project overview
3. The full text of the project CLAUDE.md rules
4. Any relevant PRD content
5. **Explicit instruction (Axis 9): Repetition / factoring smell.** Tell the senior: "Grep the plan's prose for near-identical branch descriptions (verb + object that recur with only a literal, key, separator, or metadata differing). If found, flag it as a structural issue (not a spec violation) and sketch what a unified path would look like. This is grading internal factoring, not just spec compliance. You cannot return a Pass verdict without explicitly stating whether the plan has repetition smell."
6. **Explicit instruction (Axis 10): Framework-idiom check.** Tell the senior: "When the plan involves SQL, RLS policies, ORM queries, framework hooks, middleware, or any code shape governed by a third-party tool's conventions, verify the pattern appears in that tool's official documentation. Constrain doc lookups to the framework's own site (e.g. site:prisma.io/docs, site:supabase.com/docs, site:react.dev), not blogs, Stack Overflow, or community discussions, which routinely endorse non-idiomatic patterns. A pattern that implements the spec correctly but has no documented analog is still likely wrong; homegrown patterns are usually invented to satisfy a prose spec, not because the framework lacks a documented solution. Grade against the framework, not just against the plan."
7. **Explicit instruction (Axis 11): Behavior the plan implicitly removes.** Tell the senior: "When the plan rewrites an existing hook, module, route, or shared file, read the actual file before the rewrite and enumerate its current responsibilities: every switch case, guard clause, ref and its side effects, exported helper, and documented convention. Check the plan against that list: is each one preserved or intentionally dropped, or does the plan only describe additions and leave preservation implicit? Also scan the plan for 'remove', 'delete', and 'no longer needed': when the plan removes a documented convention, verify the code that convention protected is either preserved or genuinely obsolete. Do not grade 'is the new design coherent'; grade 'would shipping this design quietly break anything the old code did?'"

These axes are general criteria the senior must always apply. They are not "focus areas raised last round" — they apply identically on every invocation.

The senior reviewer will return a structured review with verdict, critical issues, and suggestions.

**Your job after Stage 2:**
- For each critical issue: update the plan to address it, or document why you disagree.
- For each suggestion: apply if it improves the plan, or note why you're not applying it.
- Be willing to push back on the senior reviewer if their feedback conflicts with the project's stated goals or working agreements.
- **Same trade-off rule as Stage 1 applies.** If a critical issue or suggestion involves a trade-off the user should own (UX, strictness, deferral, scope), STOP and ask the user. One trade-off per message. Do NOT dump a list of "here are three open trade-offs, pick for each" at the end of the workflow. Resolve them interactively before finalising the plan. The Review Log should record the user's decision, not a list of open questions for them to answer later.
- If the senior raises multiple trade-offs, walk them one-by-one in sequence. After each user decision, apply the revision to the plan before asking the next question.

## Stage 3: Quality & Conformance Check

Run the `/simplify` skill on the plan to check for unnecessary complexity.

Additionally, review the final plan yourself against the project's CLAUDE.md rules. Check:
- Does the plan introduce any patterns that violate the rules?
- Does the plan address existing violations it should fix?
- Are there DRY, error handling, or hardcoded value concerns?
- Is the scope minimal, does every change serve the stated goal?
- **Repetition smell:** Do any phases describe near-identical work differing only by a literal, key, separator, or metadata value? If yes, propose a unified path (single code path reading the differing value from a small lookup or parameter) before finalizing. Do not rely solely on the senior reviewer for this. Verify yourself.

## Stage 4: Final Plan

Update the plan file with all revisions. At the bottom of the plan, add a section:

```markdown
## Review Log

### Junior Engineer Questions
- [Summary of questions raised and how each was addressed]

### Senior Architect Review
- Verdict: [their verdict]
- [Summary of critical issues and how each was addressed]
- [Summary of suggestions and whether they were applied or why not]

### Quality Check
- [Any simplification or conformance changes made]
```

This log is part of the plan, it documents the reasoning behind the final version.

## Re-invocation

This workflow can be invoked as many times as the user wants on the same plan. There is no cap and no diagnostic gate before running again. If the user asks for another round, run it. Do not argue that prior reviews should be sufficient, and do not propose skipping ahead as an alternative. Each round appends a new numbered entry to the Review Log.

Each invocation runs all stages from clean context against the current state of the plan; rounds from prior invocations don't count and must not influence the reviewers. Re-read the "Clean-context rule" above before spawning anything: strip the Review Log, do not pass prior rounds' questions/verdicts/responses, do not narrow the reviewer's focus to "what's still open" or "what the last round missed". The senior on round 5 must receive the same shape of prompt (with the same clean plan, no prior-round artefacts) as the senior on round 1 — the only thing that should differ is the current plan text.
