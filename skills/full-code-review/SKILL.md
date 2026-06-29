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

## Step 2: Spawn 7 reviewers in parallel

Launch ALL SEVEN agents in a **single message** so they run in parallel. Each is a `general-purpose` agent (not a sub-agent type like `junior-reviewer`). Each gathers its own context and reviews through a different lens.

The Testing reviewer (Reviewer 7) is the one that actually executes the test suite. It needs to run commands; the others are read-and-reason. All seven are still `general-purpose` agents and all run in the same parallel batch.

**Each reviewer is self-serve:** Tell them the base branch/commit to diff against and which documentation files to read (`.claude/CLAUDE.md`, `context/overview.md`, `.codereviewr`, implementation plans in `context/`). They will run `git diff`, read source files, and explore the codebase themselves. Do NOT paste diffs, file contents, or project rules into their prompts.

### Reviewer 1: Security

Prompt the agent:

> Your task is to review all code changes on the current branch vs `<base>` for security vulnerabilities and concerns.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` to see all committed changes (use `--stat` first for overview, then read individual changed files)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `.claude/CLAUDE.md` for project rules
> 4. Read `context/overview.md` for project architecture and decisions
> 5. Read `.codereviewr` for code review context (if it exists)
> 6. Read any source files you need to understand the full picture. Don't just review diffs in isolation
>
> **Focus areas (OWASP Top 10, 2021 checklist):**
>
> - **A01 Broken Access Control:** Missing authorization checks on endpoints, IDOR (user-supplied IDs used without ownership validation), client-side-only access control without server-side enforcement
> - **A02 Cryptographic Failures:** Sensitive data in plaintext (passwords, tokens, PII in logs), weak/deprecated algorithms, hardcoded secrets or keys in source
> - **A03 Injection:** Raw SQL string concatenation with user input, user input passed to eval/exec/shell commands, missing input validation and allowlisting for queries/paths/redirects
> - **A04 Insecure Design:** Missing rate limiting on sensitive operations, business logic flaws (unvalidated prices/quantities), no abuse-case limits on resource creation
> - **A05 Security Misconfiguration:** Stack traces or verbose errors exposed to users, missing security headers (CSP, HSTS, X-Content-Type-Options), overly permissive CORS, debug mode in production
> - **A06 Vulnerable and Outdated Components:** Dependencies with known CVEs, significantly outdated security-critical packages, unused dependencies expanding attack surface
> - **A07 Identification and Authentication Failures:** No brute-force protection on login/auth endpoints, session management flaws (no expiry, missing cookie flags), weak token generation or magic links that don't expire
> - **A08 Software and Data Integrity Failures:** Unsafe deserialization of untrusted data, CI/CD without integrity checks, auto-update without signature verification
> - **A09 Security Logging and Monitoring Failures:** Security-sensitive events not logged (failed logins, access control failures), sensitive data in logs, no alerting mechanism
> - **A10 Server-Side Request Forgery (SSRF):** User-supplied URLs passed to fetch without allowlist validation, no blocking of internal network ranges, URL validation bypassable via redirects
>
> **Additional focus areas:**
> - File upload / storage security (type validation, path traversal, size limits)
> - CSRF, CORS, CSP configuration
> - Secrets and credentials committed to git or git history
>
> **Output format:**
> For each finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **OWASP category:** (if applicable, e.g. A01, A03)
> - **File and line(s):** where the issue is
> - **Finding:** what the problem is
> - **Recommendation:** how to fix it
>
> End with a summary: total findings by severity, and an overall security verdict (Pass / Pass with concerns / Fail).

### Reviewer 2: Backend

Prompt the agent:

> Your task is to review all code changes on the current branch vs `<base>` for backend quality, correctness, and robustness.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` to see all committed changes (use `--stat` first for overview, then read individual changed files)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `.claude/CLAUDE.md` for project rules
> 4. Read `context/overview.md` for project architecture and decisions
> 5. Read `.codereviewr` for code review context (if it exists)
> 6. Read any source files you need to understand the full picture. Don't just review diffs in isolation
>
> **Focus areas:**
> - API design (RESTful conventions, response shapes, status codes, error handling)
> - Database queries (N+1 problems, missing indexes, transaction correctness, connection handling)
> - Error handling (catch blocks must log/rethrow per project rules, no leaked stack traces, proper HTTP status codes)
> - Performance (unnecessary queries, missing pagination, large payloads, blocking operations)
> - Data integrity (race conditions, constraint enforcement, soft-delete consistency)
> - Middleware correctness (auth checks, request validation, header handling)
> - Environment configuration (hardcoded values, missing env var validation)
> - Conformance with project rules (check against CLAUDE.md rules)
> - **Framework-idiom check:** when the diff contains SQL, ORM queries, migrations, RLS policies, or framework-managed patterns, verify the shape appears in the framework's *official* documentation (restrict lookups to the framework's own site, e.g. site:prisma.io/docs, site:supabase.com/docs; not blogs or Stack Overflow). A homegrown pattern with no documented analog is a finding even if it works. Pattern absence in the docs is a red flag, not a feature.
>
> **Output format:**
> For each finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **File and line(s):** where the issue is
> - **Finding:** what the problem is
> - **Recommendation:** how to fix it
>
> End with a summary: total findings by severity, and an overall backend verdict (Pass / Pass with concerns / Fail).

### Reviewer 3: Frontend

Prompt the agent:

> Your task is to review all code changes on the current branch vs `<base>` for frontend quality, UX, and correctness.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` to see all committed changes (use `--stat` first for overview, then read individual changed files)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `.claude/CLAUDE.md` for project rules
> 4. Read `context/overview.md` for project architecture and decisions
> 5. Read `.codereviewr` for code review context (if it exists)
> 6. Read any source files you need to understand the full picture. Don't just review diffs in isolation
>
> **Focus areas:**
> - Component architecture (single responsibility, appropriate island hydration, server vs client rendering)
> - Accessibility (ARIA labels, keyboard navigation, semantic HTML, focus management)
> - Mobile-first design (responsive layout, touch targets, viewport handling)
> - Performance (unnecessary hydration, bundle size, image optimization, lazy loading)
> - State management (unnecessary re-renders, stale state, proper cleanup)
> - UX consistency (loading states, error states, empty states, skeleton screens)
> - CSS/Tailwind usage (consistent spacing, dark mode support, responsive breakpoints)
> - Type safety (proper TypeScript types, no `any`, prop validation)
>
> **Output format:**
> For each finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **File and line(s):** where the issue is
> - **Finding:** what the problem is
> - **Recommendation:** how to fix it
>
> End with a summary: total findings by severity, and an overall frontend verdict (Pass / Pass with concerns / Fail).

### Reviewer 4: Architecture & Quality

Prompt the agent:

> Your task is to perform a quality and conformance review. Review all code changes on the current branch vs `<base>` for overall design quality, plan conformance, and adherence to project standards.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` to see all committed changes (use `--stat` first for overview, then read individual changed files)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `.claude/CLAUDE.md` for project rules
> 4. Read `context/overview.md` for project architecture and decisions
> 5. Read `.codereviewr` for code review context (if it exists)
> 6. Read implementation plans in `context/` subdirectories (Discovery, Listing, Claiming, Navigation, ImageStorage) to check conformance
> 7. Read any source files you need to understand the full picture. Don't just review diffs in isolation
>
> **Focus areas:**
> - Plan conformance (if an implementation plan exists, check every change against it; flag deviations)
> - Code organization (file placement, separation of concerns, import structure)
> - DRY violations (duplicated logic across files, missed extraction opportunities)
> - **Repetition smell:** Grep the diff for repeated lexical patterns, identical sequences that recur 3+ times with only a literal, key, separator, or metadata value differing. For each cluster, ask: is the difference *structural* (genuinely different behaviour) or *just data* (same shape, different value)? If "just data," flag as a factoring issue with a sketch of the unified form (single function reading the differing value from a small lookup or parameter). Specifically scan for files containing 3+ near-identical handlers, branches, or case arms that differ only by a literal.
> - Scope discipline (changes beyond what the task requires, unnecessary refactoring, feature creep)
> - Naming consistency (variables, functions, files follow existing conventions)
> - Test coverage gaps (are critical paths testable? are there obvious missing tests?)
> - Documentation (are complex decisions documented? are comments accurate?)
> - CLAUDE.md conformance (check every project rule against the diff)
> - Known limitations (are trade-offs documented? are TODOs tracked?)
> - **Layer placement vs framework conventions:** verify rules live at the layer the framework's official docs prescribe. Workflow-state validation inside an RLS policy, access control inside a DB trigger, or UX gating in the database are wrong layers even when they work. The official docs are the canonical source for which layer owns which concern.
>
> **Output format:**
> For each finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **File and line(s):** where the issue is
> - **Finding:** what the problem is
> - **Recommendation:** how to fix it
>
> End with a summary: total findings by severity, and an overall architecture verdict (Pass / Pass with concerns / Fail).

### Reviewer 5: Documentation

Prompt the agent:

> Your task is to review all code changes on the current branch vs `<base>` and check whether project documentation accurately reflects the changes.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` to see all committed changes (use `--stat` first for overview, then read individual changed files)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `context/overview.md`: the project's source of truth for architecture, features, utilities, decisions, and tech debt
> 4. Read `.claude/CLAUDE.md`: project rules
> 5. Read any feature `progress.md` files in `context/` subdirectories that relate to the changed files
>
> **For each meaningful code change, check:**
>
> 1. **New shared utilities**: If a new file was created in `src/lib/`, `src/hooks/`, `server/lib/`, `server/middleware/`, or `server/routes/`, is it in the Shared Utilities table in `overview.md`?
> 2. **New routes**: If a new file was added to `server/routes/`, is it listed in `.claude/CLAUDE.md` File Organization?
> 3. **Schema changes**: If `prisma/schema.prisma` was modified, are related claims in `overview.md` still accurate?
> 4. **New pages**: If a new page was added to `src/pages/`, does the relevant feature description mention the route?
> 5. **Removed or renamed files**: Does documentation still reference old paths?
> 6. **Architecture changes**: If middleware, layout, or core infrastructure changed, is the Architecture section still accurate?
> 7. **New decisions**: If the changes represent a significant decision, is there a Key Decisions table entry?
> 8. **Resolved tech debt**: If a change fixes something in the Deferred Items tables, flag it for removal.
> 9. **New tech debt**: If the change introduces a known limitation or workaround, is it tracked?
>
> **Output format:**
> For each finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **Category:** Missing utility / Missing route / Stale claim / Missing decision / Stale tech debt / Missing page / Other
> - **File:** which documentation file needs updating
> - **Finding:** what's wrong or missing
> - **Recommendation:** specific text to add, update, or remove
>
> End with a summary: total findings by severity, and an overall documentation verdict (Up to date / Needs updates / Significantly outdated).

### Reviewer 6: Regressions & Behavior Deletion

Prompt the agent:

> Your task is to review what the changes on the current branch vs `<base>` REMOVE. The other reviewers grade the new code forward; your lens is the `-` lines of the diff, not the `+` lines. A rewrite that produces correct new code but silently drops a previously load-bearing guard, handler, or documented convention is a real failure mode that forward-focused review misses.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD` and extract every deletion (use `--stat` first for an overview)
> 2. Run `git diff` to see uncommitted changes
> 3. Read `.claude/CLAUDE.md` for project rules
> 4. Read any implementation plans and `progress.md` files in `context/` subdirectories related to the changed files
>
> Skip pure formatting, whitespace, and rename-only deletions. For each substantive deletion (deleted function, switch case, guard clause, log call, ref assignment, error handler, side-effect line, CLAUDE.md section, comment marked "do not remove" or "load-bearing"), run three checks:
>
> 1. **Reference check:** grep the rest of the codebase (including docs, plans, and comments) for the deleted symbol or string. If anything outside the diff still references it, the deletion has likely broken a caller or a documented convention.
> 2. **Convention check:** was the deleted block documented as important in CLAUDE.md, implementation plans, or inline warnings? A deletion that also removes its own warning comment is a strong signal of unintentional removal.
> 3. **Replacement check:** does the diff contain new code on the same surface that subsumes the deleted behavior, or is the behavior truly gone with no equivalent? Replaced is fine; removed with no replacement is the regression.
>
> **Output format:**
> For each substantive deletion, state:
> - **File and line(s):** where the deletion is
> - **Classification:** Intentional / Likely regression / Unclear (needs author input)
> - **Evidence:** for regressions, cite the caller, convention, or doc that depends on the deleted code
>
> End with a summary: counts per classification, and an overall regression verdict (Pass / Pass with concerns / Fail).

### Reviewer 7: Testing & Test Execution

Prompt the agent:

> Your task is to review the **test coverage** of the changes on the current branch vs `<base>`, and to **actually run the test suite** and report what happened. The other reviewers reason about the code; your job is to verify, empirically, that the new code is tested and that the tests pass. You have a full toolset including the ability to run commands, use it.
>
> **How to gather context:**
> 1. Run `git diff <base>...HEAD --stat` and `git diff <base>...HEAD` to see what changed (and `git diff` for uncommitted changes)
> 2. Read `.claude/CLAUDE.md`, especially the **Testing Reality** section, for what test infrastructure exists and how to run it
> 3. Read `context/overview.md` for architecture, and any implementation plan's **Testing Strategy** section in `context/` for what the plan said it would test
> 4. Find the test setup: glob for `*.test.*`, `*.spec.*`, `playwright.config.*`, `vitest.config.*`, `jest.config.*`, a `tests/` or `e2e/` directory, and the test scripts in `package.json` (or the equivalent for the stack)
>
> **Part A, coverage analysis (does the new code have tests?):**
> For every changed source file that contains logic (not pure config, types, or styling), check whether a corresponding or updated test exists and whether it covers the behavior that changed:
> - **New logic with no test at all** is a finding. Pure functions, validators, reducers, server utilities, and API handlers should have unit/integration tests; user-facing flows should have an e2e test.
> - **Tests that only cover the happy path** when the change introduces error states, auth boundaries, or edge cases is a finding, name the uncovered case.
> - **Assertions that don't actually assert** the changed behavior (snapshot-only, or asserting on a mock instead of real output) is a finding.
> - **Critical flows covered only manually** (the plan's Verification list has no matching automated test) is a finding.
>
> **Part B, run the suite (does it actually pass?):**
> 1. Determine the test command from `package.json` / `CLAUDE.md` and run the **full** test suite. Report the exact command and the exact result (pass/fail counts, failing test names).
> 2. If the project has an e2e suite (Playwright), run it too if the environment allows (a dev server / base URL is reachable). If it cannot be run here, say so explicitly rather than claiming it passed.
> 3. If tests fail, capture the failure output and determine whether the failure is caused by the branch's changes (a real regression) or a pre-existing/environmental issue. Do not fix anything, report.
> 4. If there is **no runnable test suite at all**, that itself is the top finding for this review: the change shipped with no way to verify it automatically.
>
> Report results faithfully. If tests fail, say so with the output. If you could not run them, say that, do not claim a green suite you did not see.
>
> **Output format:**
> For each coverage finding, state:
> - **Severity:** Critical / High / Medium / Low / Info
> - **File and line(s):** the untested or under-tested code
> - **Finding:** what behavior lacks a test, or what the existing test fails to assert
> - **Recommendation:** the specific test to add (which case, which boundary), and whether it's a unit, integration, or e2e test (suggest `/write-e2e-tests` for browser flows)
>
> Then a **Test run** section: the exact command(s) run, pass/fail counts, names of any failing tests, and whether each failure is a branch regression or pre-existing.
>
> End with a summary: total coverage findings by severity, the test-run result, and an overall testing verdict (Pass / Pass with concerns / Fail). Fail if new logic ships untested, if the suite is red because of this branch, or if there is no runnable suite.

## Step 2b: Quality cleanup lens (`/simplify`)

Alongside the seven bug-focused reviewers, run `/simplify` on the same diff (the Step 1 base) as the **quality-cleanup lens**. The reviewers hunt for what is wrong; `/simplify` finds what is needlessly complex: reuse opportunities, hand-rolled stdlib, single-caller abstractions, dead code, and altitude/verbosity cleanups. It complements the bug lenses (it does not hunt for correctness bugs, that is `/code-review`'s and the reviewers' job).

`/simplify` reviews and applies in one pass, so treat its changes like any other finding: review the cleanups it makes, present them in the consolidation (Step 3) under a Simplification heading, and fold them into the single sign-off-and-apply pass (Step 4). Do not ship a cleanup the user has not seen; if a cleanup is actually a behavior change or load-bearing, pull it back out and surface it as a trade-off instead.

For **full** scope, run `/simplify` over the directories in scope rather than a diff.

## Full scope adjustments

When the argument is `full`, keep the same seven reviewers and output formats but adapt each brief:

- **All reviewers:** replace the diff-gathering steps with "read `context/overview.md` and `.claude/CLAUDE.md`, then explore the codebase directly (Glob, Grep, Read)". They range over the whole project; nothing is out of bounds as "pre-existing". Skip plan-conformance checks against a single feature; grade against project-level docs instead.
- **Security:** scan auth gates, session handling, token generation, env-var usage, query patterns, error responses, headers and CORS, file uploads, and the dependency surface.
- **Backend:** scan service layers, query helpers, background jobs, third-party integrations, and the logging baseline. Spot-check the codebase's SQL, migration, and ORM shapes against the framework's documented patterns; hand-rolled retry or dedup logic where the framework offers one is a smell worth flagging.
- **Frontend:** scan top-level UI patterns, the shared component library, route conventions, design-system compliance, and the accessibility baseline.
- **Architecture & Quality:** scan directory structure, lib vs route vs component boundaries, where types and shared logic live, and dependency direction across the codebase. Scan for repetition smell at the codebase level: files or modules containing 3+ near-identical handlers, branches, or case arms differing only by a literal are refactor candidates. Cross-check layer placement against the framework's documented separation of concerns; where the codebase has accumulated rules at non-canonical layers, flag those as architectural debt.
- **Documentation:** audit all top-level docs against current code reality: stale literals, drifted conventions, missing onboarding info, inconsistent sibling planning docs.
- **Regressions:** still spawn it, but its brief is one line: return immediately with "out of scope for full mode". Regression review only makes sense with a diff to inspect; the consistency of always spawning seven is worth more than the saved tokens.
- **Testing:** scan the whole test suite, not a diff. Map the major modules/features against the tests that exist and report the coverage gaps, untested subsystems, critical flows with no e2e test, and whole areas (auth, payments, data integrity) with thin or no coverage. Run the full suite and report its health (green/red, flaky tests, slowest tests, total count). The verdict reflects the project's overall testing posture, not a single diff.
- **Consolidation:** severity framing shifts from "must fix before committing" to "must fix before the next sprint" and "must fix before scaling". Findings become tickets or Deferred Items entries rather than pre-commit fixes, so route them through `context/overview.md`'s Deferred Items tables when the user accepts them.

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
