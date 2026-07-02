---
name: full-code-review
description: Spawn 7 parallel code review agents (security, backend, frontend, architecture, documentation, regressions, testing) to review code, then filter their findings through a mechanical quote check and a per-finding validation pass before anything reaches the user. The testing reviewer checks that new code has tests and actually runs the suite. Two scopes; branch diff review (default, use after completing an implementation before committing) or whole-codebase health check (pass 'full', use as a periodic project health check).
disable-model-invocation: false
argument-hint: "[base branch or commit, e.g. 'main' or 'HEAD~5', or 'full' for a whole-codebase health check]"
---

# Full Code Review

You are orchestrating a 7-reviewer code review. The argument: $ARGUMENTS

If no argument was provided, do not silently assume `main`. Determine the base branch in this order:

1. Check `.claude/CLAUDE.md` and `context/overview.md` for a documented integration branch (gitflow projects diff against `develop`, others may use a release branch).
2. If none is documented, ask the user which branch to compare against, suggesting `main` as the likely answer.

Every reviewer brief below uses `<base>`; substitute the resolved base branch or commit.

The pipeline is: spawn reviewers → quote-check and dedup their findings mechanically → validate Critical/High findings with fresh second-opinion agents → consolidate and present → walk trade-offs. Findings the pipeline discards are still shown, in a Discarded section with the reason; nothing is silently dropped.

## Two scopes

**Branch scope (default).** The argument is a base branch or commit (e.g. `main`, `develop`, `HEAD~5`). Reviewers focus on what changed on the current branch vs that base and grade it against the feature's planning docs and project rules. They do not lecture about pre-existing codebase concerns unrelated to the diff; anything pre-existing they do flag is routed to its own bucket.

**Full scope.** If the argument is `full`, there is no diff. Reviewers scan the entire codebase: top-level patterns, conventions, shared infrastructure, and documentation. Use it as a periodic project health check. The reviewer briefs below are written for branch scope; apply the "Full scope adjustments" section when running full. The two modes are materially different and feeding the wrong context produces a noisy review.

## Step 1: Gather context (lightweight)

Run these commands to understand what changed at a high level:

1. `git diff <base>...HEAD --stat`: list of changed files
2. `git log <base>..HEAD --oneline`: commits being reviewed

Also read:
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)

For **full scope**, skip the diff commands. Instead, read `context/overview.md` and `.claude/CLAUDE.md`, then list the top-level directories the review should cover (e.g. `src/`, `server/`, `prisma/`). Tell the user which directories are in scope and ask to confirm before spawning: the review is expensive (seven parallel reviewers plus validators) and easy to misframe.

**Important:** Do NOT pre-read the full diff, source files, or overview yourself. Each reviewer agent will gather its own context by reading the codebase directly. Your job is to orchestrate and to run the *mechanical* checks in Step 3; the judgment calls belong to the reviewers and validators.

## Step 2: Spawn the 7 reviewer agents in parallel

Launch all seven named reviewer agents in a **single message** so they run in parallel. Each carries its own brief, the shared severity rubric, and gathers its own context (it runs `git diff` or explores the codebase itself); do **not** paste diffs, file contents, or project rules into their prompts. Tell each only the **scope**: for a branch review, the `<base>` to diff against; for a `full` review, the directories in scope.

| Agent | Lens |
|-------|------|
| `security-reviewer` | OWASP Top 10, auth, secrets, injection, SSRF, upload safety |
| `backend-reviewer` | API/query/error-handling correctness, performance, framework-idiom check |
| `frontend-reviewer` | component architecture, accessibility, responsive/UX, type safety |
| `architecture-reviewer` | design quality, plan conformance, DRY/repetition smell, scope, layer placement |
| `documentation-reviewer` | whether the docs reflect the change (or current code, in full scope) |
| `regression-reviewer` | the `-` lines: behavior, guards, or conventions deleted with no replacement |
| `testing-reviewer` | test coverage of the change, and actually runs the suite |

The `testing-reviewer` runs commands (it executes the suite); the rest are read-and-reason. All reviewers grade on the same anchored severity rubric (it lives in each agent's definition), and every finding must carry a verbatim **Evidence** quote — findings without one do not survive Step 3.

In **full** scope, spawn only six: skip `regression-reviewer` (it needs a diff) and note the skip in the final report rather than spawning an agent whose only output is "out of scope".

## Step 3: Mechanical quote gate and dedup (no judgment)

When the reviewers return, run two purely mechanical filters yourself. These require no code comprehension, so they don't violate the no-pre-digestion rule:

1. **Quote check.** Write `git diff <base>...HEAD` to a temp file once. For each finding, grep its Evidence quote with fixed-string matching (`grep -F`) first in that diff, then (if not found) in the cited file. Whitespace-insensitive matching is fine; paraphrase is not. A finding whose quote matches nothing goes to **Discarded (citation not found)**. Do not repair or reinterpret a failed quote on the reviewer's behalf.
2. **Dedup.** Two findings are duplicates when they cite the same file, lines within ±3 of each other, and describe substantially the same issue. Merge them into one entry listing every reviewer that flagged it. **Cross-reviewer agreement is the strongest confidence signal**: mark merged findings as corroborated and keep the highest severity assigned.

Also split out findings marked **Pre-existing: yes** into their own bucket now (branch scope); they skip validation and are presented separately as non-blocking.

## Step 4: Validation pass (Critical and High findings)

For every surviving Critical or High finding, spawn one `finding-validator` agent — all in a **single message** so they run in parallel. Give each validator one finding verbatim (severity, file, line(s), Evidence quote, the claim, the recommendation) plus the `<base>`. Do not editorialize or hint at an expected verdict.

Route on the results:

- **validated: yes** → the finding proceeds to consolidation, marked ✓ validated.
- **validated: no** → move it to **Discarded (failed validation)** with the validator's reason.
- **pre_existing: yes** → re-route to the Pre-existing bucket regardless of verdict.
- **Validator errored or returned garbage** → the finding proceeds anyway, marked **validation degraded**. A Critical or High finding is never silently dropped because infrastructure failed.

Medium/Low/Info findings skip validation (the quote gate already filtered fabrications, and validating everything would double the cost of the review for the least important tier).

## Step 5: Consolidate and present

Present the results to the user in this format:

```
## Full Code Review Results

### Security Review: [verdict]
[Findings by severity, highest first; each with its Evidence quote and validation mark]

### Backend Review: [verdict]
[Findings by severity, highest first]

### Frontend Review: [verdict]
[Findings by severity, highest first]

### Architecture & Quality Review: [verdict]
[Findings by severity, highest first]

### Documentation Review: [verdict]
[Findings by severity, highest first]

### Regression Review: [verdict]
[Deletions by classification: Likely regression first, then Unclear, then Intentional]

### Testing Review: [verdict]
[Coverage findings by severity, highest first. Then the test-run result: command,
pass/fail counts, any failing tests, and whether each failure is a branch regression.]

---

### Corroborated findings
[Findings multiple reviewers flagged independently — highest confidence, listed once
with all flagging reviewers named]

### Pre-existing (non-blocking)
[Findings not introduced by this branch; candidates for tickets or Deferred Items,
not pre-commit fixes]

### Discarded
[Every finding removed by the pipeline, one line each:
"[reviewer] [severity] [file]: [title] — citation not found | failed validation: [reason]"]

### My assessment
[Your critical judgment on the surviving findings:
- Which findings should definitely be fixed before committing?
- Which are valid but acceptable for now (document why)?
- Which contradict an explicit decision in the plan, CLAUDE.md, or a design-decisions
  record (the reviewer missed existing context)?]

### Recommended actions
[Numbered list of concrete changes to make, ordered by priority]
```

**Important:**
- Exercise critical judgment on what remains, but the evidence-level filtering is already done: do not re-argue validated findings, and do not resurrect discarded ones without new evidence.
- Treat the regression reviewer's "Likely regression" entries as Critical findings. Deletions of load-bearing behavior are not stylistic choices.
- Treat a red test suite caused by this branch, or new logic shipping with no test, as a Critical finding, not a nitpick. "It works when I click around" is not a substitute for an automated test. If the Testing reviewer could not run the suite, surface that as an open item rather than assuming green.
- Surface trade-offs to the user: do NOT silently accept or dismiss findings. Let the user decide on anything ambiguous.

## Full scope adjustments

The agents handle their own full-scope adaptation: each explores the codebase instead of a diff, and `regression-reviewer` is not spawned (note the skip in the report). The quote gate greps files directly (there is no diff), validators receive "full" as scope, and the Pre-existing bucket does not apply. The other orchestrator-level change is **consolidation**: severity framing shifts from "must fix before committing" to "must fix before the next sprint" and "before scaling", and accepted findings become tickets or `context/overview.md` Deferred Items entries rather than pre-commit fixes.

## Step 6: Walk through trade-offs one by one

After presenting the consolidated review, do NOT silently make trade-off decisions or batch all decisions into a single message. Instead:

1. Split findings into two categories:
   - **Clear fixes**: findings where the fix is obvious, non-controversial, and doesn't involve a trade-off (e.g., fixing a typo, using a constant instead of a string literal, removing dead code). List these as "I'll fix these unless you object."
   - **Trade-offs**: findings where there are multiple valid options, the fix involves a judgment call, or the change is debatable. These MUST be walked through with the user.

2. For each trade-off, present it as a single question with numbered options. Wait for the user's answer before moving to the next trade-off. Do not bundle multiple trade-offs into one message.

3. Only after ALL trade-offs are resolved, apply the agreed-upon fixes (clear fixes + user-chosen trade-off resolutions) in a single implementation pass.

## Where `/simplify` fits

Quality cleanup is deliberately **not** part of this skill. Running a pass that edits the working tree while seven reviewers are reading it is a race: reviewers see files mid-mutation and their citations go stale. If you want a simplification pass (reuse, dead code, over-abstraction), run the built-in `/simplify` **before** invoking this review, as its own step on a clean tree — the reviewers then review the simplified code. If `/simplify` is unavailable in the current Claude Code version, skip it; it is optional.

## Re-invocation

This skill can be invoked as many times as the user wants. Each invocation is a fresh review against the current state of the branch; prior runs don't count. If the user asks for another round, run it. Do not argue that previous reviews should be sufficient or ask whether it's worth running again.
