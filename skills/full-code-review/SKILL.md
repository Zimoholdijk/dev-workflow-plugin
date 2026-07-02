---
name: full-code-review
description: Parallel multi-lens code review (security, backend, frontend, architecture, documentation, regressions, testing) with the roster selected mechanically from the diff's own signals — small single-surface diffs get a reduced roster, anything ambiguous or risky gets all seven. Findings then pass a mechanical quote check and a per-finding validation pass before anything reaches the user. The testing reviewer checks that new code has tests and actually runs the suite. Two scopes; branch diff review (default, use after completing an implementation before committing) or whole-codebase health check (pass 'full', use as a periodic project health check).
disable-model-invocation: false
argument-hint: "[base branch or commit, e.g. 'main' or 'HEAD~5'; or 'full' for a whole-codebase health check; add 'depth:full' to force all seven reviewers]"
---

# Full Code Review

You are orchestrating a multi-lens code review. The argument: $ARGUMENTS

If no base was provided, do not silently assume `main`. Determine the base branch in this order:

1. Check `.claude/CLAUDE.md` and `context/overview.md` for a documented integration branch (gitflow projects diff against `develop`, others may use a release branch).
2. If none is documented, ask the user which branch to compare against, suggesting `main` as the likely answer.

Every reviewer brief below uses `<base>`; substitute the resolved base branch or commit. If the argument contains the token `depth:full`, the full seven-reviewer roster is forced regardless of what Step 2 finds.

The pipeline is: read the diff's *signals* (not its content) to select the roster → spawn reviewers → quote-check and dedup their findings mechanically → validate Critical/High findings with fresh second-opinion agents → consolidate and present → walk trade-offs. Findings the pipeline discards are still shown, in a Discarded section with the reason; nothing is silently dropped.

## Two scopes

**Branch scope (default).** The argument is a base branch or commit (e.g. `main`, `develop`, `HEAD~5`). Reviewers focus on what changed on the current branch vs that base and grade it against the feature's planning docs and project rules. They do not lecture about pre-existing codebase concerns unrelated to the diff; anything pre-existing they do flag is routed to its own bucket.

**Full scope.** If the argument is `full`, there is no diff. Reviewers scan the entire codebase: top-level patterns, conventions, shared infrastructure, and documentation. Use it as a periodic project health check. It always runs the full roster (minus `regression-reviewer`, which needs a diff). The reviewer briefs below are written for branch scope; apply the "Full scope adjustments" section when running full. The two modes are materially different and feeding the wrong context produces a noisy review.

## Step 1: Gather context (lightweight)

Run these commands to understand what changed at a high level:

1. `git diff <base>...HEAD --stat`: list of changed files
2. `git log <base>..HEAD --oneline`: commits being reviewed

Also read:
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)

For **full scope**, skip the diff commands. Instead, read `context/overview.md` and `.claude/CLAUDE.md`, then list the top-level directories the review should cover (e.g. `src/`, `server/`, `prisma/`). Tell the user which directories are in scope and ask to confirm before spawning: the review is expensive (up to seven parallel reviewers plus validators) and easy to misframe.

**Important:** Do NOT pre-read the full diff, source files, or overview yourself. Each reviewer agent will gather its own context by reading the codebase directly. Your job is to orchestrate and to run the *mechanical* checks in Steps 2 and 4; the judgment calls belong to the reviewers and validators.

## Step 2: Select the roster from the diff's signals (mechanical)

Branch scope only; full scope always runs the full roster minus regression. Derive the roster from the diff itself — **never from how the change was described to you**. The author's characterization of a change ("just a small frontend tweak") is the least reliable input in a review: it can *add* reviewers ("also run security"), it can force everything (`depth:full`), but it can never remove one. The diff is ground truth, and checking it is nearly free.

Classify mechanically from `git diff <base>...HEAD --stat` and fixed-string/regex greps over `git diff <base>...HEAD` output. Do not read or interpret the code itself — file paths, extensions, line counts, and grep hits only:

| Reviewer | Include when |
|----------|--------------|
| `testing-reviewer` | **Always.** Every change gets its coverage checked and the suite run. Not skippable. |
| `regression-reviewer` | The diff contains deleted lines beyond pure whitespace/formatting/renames. |
| `frontend-reviewer` | Client-side files changed: components, pages, styles, client hooks (`.tsx`/`.jsx`/`.vue`/`.svelte`, `.css`/`.scss`, `app/`/`pages/`/`components/` client code). |
| `backend-reviewer` | Server-side files changed: API routes/handlers, services, db/schema/migrations, background jobs, middleware, server config. |
| `security-reviewer` | Risk greps hit anywhere in the diff: auth, session, token, password, secret, key, permission, role, policy, RLS, payment, price, upload, deserialize, exec, raw SQL, `fetch(`/HTTP calls with user input, redirect, CORS, cookie. Or any new/changed endpoint. |
| `architecture-reviewer` | The diff adds new files, touches 2+ modules/layers, or moves code between layers. |
| `documentation-reviewer` | The diff touches docs (`*.md`, `docs/`, `context/`) or adds a route, exported utility, env var, or other documented surface. |

**Escape hatches — run all seven whenever any of these hold:**

- The diff is roughly ≥ 150 changed lines or ≥ 10 files (past that size, single-surface claims stop being credible).
- Any changed file doesn't classify cleanly (unknown extension, generated code, vendored deps, mixed client/server file).
- The user asked for it (`depth:full` or words to that effect).
- You are uncertain for any reason. Ambiguity always resolves toward the full roster, never away from it.

**Announce the roster before spawning**, one line of evidence per decision, e.g.: "Reduced roster (3 of 7): frontend (only `components/*.tsx` and `.css` changed, 28 lines), testing (always), regression (14 deleted lines). No backend files, no risk-grep hits — skipping backend, security, architecture, docs. Re-run with `depth:full` for all seven." Then proceed; don't block on confirmation. If the roster is the full seven, one line ("full roster: [reason]") is enough.

## Step 3: Spawn the selected reviewers in parallel

Launch every selected reviewer agent in a **single message** so they run in parallel. Each carries its own brief, the shared severity rubric, and gathers its own context (it runs `git diff` or explores the codebase itself); do **not** paste diffs, file contents, or project rules into their prompts. Tell each only the **scope**: for a branch review, the `<base>` to diff against; for a `full` review, the directories in scope.

| Agent | Lens |
|-------|------|
| `security-reviewer` | OWASP Top 10, auth, secrets, injection, SSRF, upload safety |
| `backend-reviewer` | API/query/error-handling correctness, performance, framework-idiom check |
| `frontend-reviewer` | component architecture, accessibility, responsive/UX, type safety |
| `architecture-reviewer` | design quality, plan conformance, DRY/repetition smell, scope, layer placement |
| `documentation-reviewer` | whether the docs reflect the change (or current code, in full scope) |
| `regression-reviewer` | the `-` lines: behavior, guards, or conventions deleted with no replacement |
| `testing-reviewer` | test coverage of the change, and actually runs the suite |

The `testing-reviewer` runs commands (it executes the suite); the rest are read-and-reason. All reviewers grade on the same anchored severity rubric (it lives in each agent's definition), and every finding must carry a verbatim **Evidence** quote — findings without one do not survive Step 4.

In **full** scope, spawn six: skip `regression-reviewer` (it needs a diff) and note the skip in the final report rather than spawning an agent whose only output is "out of scope".

## Step 4: Mechanical quote gate and dedup (no judgment)

When the reviewers return, run two purely mechanical filters yourself. These require no code comprehension, so they don't violate the no-pre-digestion rule:

1. **Quote check.** Write `git diff <base>...HEAD` to a temp file once. For each finding, grep its Evidence quote with fixed-string matching (`grep -F`) first in that diff, then (if not found) in the cited file. Whitespace-insensitive matching is fine; paraphrase is not. A finding whose quote matches nothing goes to **Discarded (citation not found)**. Do not repair or reinterpret a failed quote on the reviewer's behalf.
2. **Dedup.** Two findings are duplicates when they cite the same file, lines within ±3 of each other, and describe substantially the same issue. Merge them into one entry listing every reviewer that flagged it. **Cross-reviewer agreement is the strongest confidence signal**: mark merged findings as corroborated and keep the highest severity assigned.

Also split out findings marked **Pre-existing: yes** into their own bucket now (branch scope); they skip validation and are presented separately as non-blocking.

## Step 5: Validation pass (Critical and High findings)

For every surviving Critical or High finding, spawn one `finding-validator` agent — all in a **single message** so they run in parallel. Give each validator one finding verbatim (severity, file, line(s), Evidence quote, the claim, the recommendation) plus the `<base>`. Do not editorialize or hint at an expected verdict.

Route on the results:

- **validated: yes** → the finding proceeds to consolidation, marked ✓ validated.
- **validated: no** → move it to **Discarded (failed validation)** with the validator's reason.
- **pre_existing: yes** → re-route to the Pre-existing bucket regardless of verdict.
- **Validator errored or returned garbage** → the finding proceeds anyway, marked **validation degraded**. A Critical or High finding is never silently dropped because infrastructure failed.

Medium/Low/Info findings skip validation (the quote gate already filtered fabrications, and validating everything would double the cost of the review for the least important tier).

## Step 6: Consolidate and present

Present the results to the user in this format, listing only the reviewers that ran (note skipped ones and why in one line under the heading):

```
## Full Code Review Results

Roster: [N of 7 — one line on what ran and what was skipped, with the Step 2 evidence]

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

## Step 7: Walk through trade-offs one by one

After presenting the consolidated review, do NOT silently make trade-off decisions or batch all decisions into a single message. Instead:

1. Split findings into two categories:
   - **Clear fixes**: findings where the fix is obvious, non-controversial, and doesn't involve a trade-off (e.g., fixing a typo, using a constant instead of a string literal, removing dead code). List these as "I'll fix these unless you object."
   - **Trade-offs**: findings where there are multiple valid options, the fix involves a judgment call, or the change is debatable. These MUST be walked through with the user.

2. For each trade-off, present it as a single question with numbered options. Wait for the user's answer before moving to the next trade-off. Do not bundle multiple trade-offs into one message.

3. Only after ALL trade-offs are resolved, apply the agreed-upon fixes (clear fixes + user-chosen trade-off resolutions) in a single implementation pass.

## Where `/simplify` fits

Quality cleanup is deliberately **not** part of this skill. Running a pass that edits the working tree while reviewers are reading it is a race: reviewers see files mid-mutation and their citations go stale. If you want a simplification pass (reuse, dead code, over-abstraction), run the built-in `/simplify` **before** invoking this review, as its own step on a clean tree — the reviewers then review the simplified code. If `/simplify` is unavailable in the current Claude Code version, skip it; it is optional.

## Re-invocation

This skill can be invoked as many times as the user wants. Each invocation is a fresh review against the current state of the branch; prior runs don't count (the roster is re-derived from the diff as it now stands). If the user asks for another round, run it. Do not argue that previous reviews should be sufficient or ask whether it's worth running again.
