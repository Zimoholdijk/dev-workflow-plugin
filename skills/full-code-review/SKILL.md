---
name: full-code-review
description: Spawn 7 parallel code review agents (security, backend, frontend, architecture, documentation, regressions, testing) to review code. The testing reviewer checks that new code has tests and actually runs the suite. Two scopes; branch diff review (default, use after completing an implementation before committing) or whole-codebase health check (pass 'full', use as a periodic project health check).
disable-model-invocation: false
argument-hint: "[base branch or commit, e.g. 'main' or 'HEAD~5', or 'full' for a whole-codebase health check]"
---

# Full Code Review

You are orchestrating a 7-reviewer code review. The argument: $ARGUMENTS

If no argument was provided, do not silently assume `main`. Determine the base branch in this order:

1. Check `.claude/CLAUDE.md` and `context/overview.md` for a documented integration branch (gitflow projects diff against `develop`, others may use a release branch).
2. If none is documented, ask the user which branch to compare against, suggesting `main` as the likely answer.

Every reviewer brief below uses `<base>`; substitute the resolved base branch or commit.

## Two scopes

**Branch scope (default).** The argument is a base branch or commit (e.g. `main`, `develop`, `HEAD~5`). Reviewers focus on what changed on the current branch vs that base and grade it against the feature's planning docs and project rules. They do not lecture about pre-existing codebase concerns unrelated to the diff.

**Full scope.** If the argument is `full`, there is no diff. Reviewers scan the entire codebase: top-level patterns, conventions, shared infrastructure, and documentation. Use it as a periodic project health check. The reviewer briefs below are written for branch scope; apply the "Full scope adjustments" section when running full. The two modes are materially different and feeding the wrong context produces a noisy review.

## Step 1: Gather context (lightweight)

Run these commands to understand what changed at a high level:

1. `git diff <base>...HEAD --stat`: list of changed files
2. `git log <base>..HEAD --oneline`: commits being reviewed

Also read:
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)

For **full scope**, skip the diff commands. Instead, read `context/overview.md` and `.claude/CLAUDE.md`, then list the top-level directories the review should cover (e.g. `src/`, `server/`, `prisma/`). Tell the user which directories are in scope and ask to confirm before spawning: the review is expensive (six parallel agents) and easy to misframe.

**Important:** Do NOT pre-read the full diff, source files, or overview yourself. Each reviewer agent will gather its own context by reading the codebase directly. Your job is to orchestrate, not to pre-digest context.

## Step 2: Spawn the 7 reviewer agents in parallel

Launch all seven named reviewer agents in a **single message** so they run in parallel. Each carries its own brief and gathers its own context (it runs `git diff` or explores the codebase itself); do **not** paste diffs, file contents, or project rules into their prompts. Tell each only the **scope**: for a branch review, the `<base>` to diff against; for a `full` review, the directories in scope.

| Agent | Lens |
|-------|------|
| `security-reviewer` | OWASP Top 10, auth, secrets, injection, SSRF, upload safety |
| `backend-reviewer` | API/query/error-handling correctness, performance, framework-idiom check |
| `frontend-reviewer` | component architecture, accessibility, responsive/UX, type safety |
| `architecture-reviewer` | design quality, plan conformance, DRY/repetition smell, scope, layer placement |
| `documentation-reviewer` | whether the docs reflect the change (or current code, in full scope) |
| `regression-reviewer` | the `-` lines: behavior, guards, or conventions deleted with no replacement |
| `testing-reviewer` | test coverage of the change, and actually runs the suite |

The `testing-reviewer` runs commands (it executes the suite); the rest are read-and-reason. All seven run in the same parallel batch.

In **full** scope each agent adapts itself (it explores the codebase instead of a diff), except `regression-reviewer`, which returns "out of scope" because it needs a diff. It is still spawned, for the consistency of always running seven.

## Step 2b: Quality cleanup lens (`/simplify`)

Alongside the seven bug-focused reviewers, run `/simplify` on the same diff (the Step 1 base) as the **quality-cleanup lens**. The reviewers hunt for what is wrong; `/simplify` finds what is needlessly complex: reuse opportunities, hand-rolled stdlib, single-caller abstractions, dead code, and altitude/verbosity cleanups. It complements the bug lenses (it does not hunt for correctness bugs, that is `/code-review`'s and the reviewers' job).

`/simplify` reviews and applies in one pass, so treat its changes like any other finding: review the cleanups it makes, present them in the consolidation (Step 3) under a Simplification heading, and fold them into the single sign-off-and-apply pass (Step 4). Do not ship a cleanup the user has not seen; if a cleanup is actually a behavior change or load-bearing, pull it back out and surface it as a trade-off instead.

For **full** scope, run `/simplify` over the directories in scope rather than a diff.

## Full scope adjustments

The agents handle their own full-scope adaptation: each explores the codebase instead of a diff, and `regression-reviewer` returns "out of scope" (it needs a diff). The one orchestrator-level change is **consolidation**: severity framing shifts from "must fix before committing" to "must fix before the next sprint" and "before scaling", and accepted findings become tickets or `context/overview.md` Deferred Items entries rather than pre-commit fixes.

## Step 3: Consolidate and present

Once all 7 reviewers and the `/simplify` lens complete, present the results to the user in this format:

```
## Full Code Review Results

### Security Review: [verdict]
[List findings by severity, highest first]

### Backend Review: [verdict]
[List findings by severity, highest first]

### Frontend Review: [verdict]
[List findings by severity, highest first]

### Architecture & Quality Review: [verdict]
[List findings by severity, highest first]

### Documentation Review: [verdict]
[List findings by severity, highest first]

### Regression Review: [verdict]
[List deletions by classification: Likely regression first, then Unclear, then Intentional]

### Testing Review: [verdict]
[List coverage findings by severity, highest first. Then the test-run result: command,
pass/fail counts, any failing tests, and whether each failure is a branch regression.]

### Simplification (`/simplify`)
[Reuse, over-engineering, dead-code, and verbosity cleanups it surfaced/applied, each with
the line saving. Flag any that are actually behavior changes for the trade-off walk.]

---

### Cross-reviewer themes
[Identify findings that multiple reviewers flagged: these are highest confidence]

### My assessment
[Your critical judgment on the findings:
- Which findings should definitely be fixed before committing?
- Which are valid but acceptable for now (document why)?
- Which do you disagree with (explain why)?]

### Recommended actions
[Numbered list of concrete changes to make, ordered by priority]
```

**Important:**
- Exercise critical judgment. Not every finding needs action. Some are nitpicks, some conflict with project decisions already made.
- Treat the regression reviewer's "Likely regression" entries as Critical findings. Deletions of load-bearing behavior are not stylistic choices.
- Treat a red test suite caused by this branch, or new logic shipping with no test, as a Critical finding, not a nitpick. "It works when I click around" is not a substitute for an automated test. If the Testing reviewer could not run the suite, surface that as an open item rather than assuming green.
- If a finding contradicts an explicit decision in the implementation plan or CLAUDE.md, note that the reviewer missed existing context.
- Group duplicate findings (same issue flagged by multiple reviewers), present once with a note that multiple reviewers caught it.
- Surface trade-offs to the user: do NOT silently accept or dismiss findings. Let the user decide on anything ambiguous.

## Step 4: Walk through trade-offs one by one

After presenting the consolidated review, do NOT silently make trade-off decisions or batch all decisions into a single message. Instead:

1. Split findings into two categories:
   - **Clear fixes**: findings where the fix is obvious, non-controversial, and doesn't involve a trade-off (e.g., fixing a typo, using a constant instead of a string literal, removing dead code). List these as "I'll fix these unless you object."
   - **Trade-offs**: findings where there are multiple valid options, the fix involves a judgment call, or the change is debatable. These MUST be walked through with the user.

2. For each trade-off, present it as a single question with numbered options. Wait for the user's answer before moving to the next trade-off. Do not bundle multiple trade-offs into one message.

3. Only after ALL trade-offs are resolved, apply the agreed-upon fixes (clear fixes + user-chosen trade-off resolutions) in a single implementation pass.

## Re-invocation

This skill can be invoked as many times as the user wants. Each invocation is a fresh review against the current state of the branch; prior runs don't count. If the user asks for another round, run it. Do not argue that previous reviews should be sufficient or ask whether it's worth running again.
