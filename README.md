# dev-workflow

A Claude Code plugin packaging a PRD-first development workflow: write PRDs, settle architecture in a plain-language design discussion before any plan is phased, draft phased implementation plans, run multi-lens plan reviews (clarifying-questions, deep-critique, and adversarial red-team passes), implement plans phase-by-phase with progress tracking, run parallel code reviews, collect tradeoff decisions, and audit documentation against the codebase. Testing is treated as a first-class part of the workflow: plans must carry a testing strategy, every phase ships its own tests, a dedicated review agent runs the suite, and a bundled Playwright MCP drives real-browser end-to-end tests.

## What's included

### Skills

| Skill | Purpose |
|-------|---------|
| `project-setup` | Bootstrap a new project with context docs, CLAUDE.md, and workflow conventions |
| `discuss-feature` | Pre-PRD discussion in plain language: one topic per message, collects decisions, creates a backlog ticket, hands off to `write-prd` |
| `write-prd` | Draft a PRD for a new feature |
| `discuss-plan` | Pre-plan architecture discussion in plain language: settles the design decisions (one-way doors first) grounded in the PRD and the actual code, researches technical trade-offs before surfacing them, asks you to run SQL or check production where a decision needs facts, writes `design-decisions.md`, and hands off to `write-plan` |
| `write-plan` | Draft a phased implementation plan from an approved PRD (consuming `design-decisions.md` if present) |
| `plan-review` | Multi-lens plan review that loops to convergence: it first establishes the plan's load-bearing premises (is it live, prod data, real infra) so rounds aren't spent on false premises, then cold reviewers (clarifying-questions, deep-critique, red-team) find issues, a cold grader rates each by reversibility/blast-radius (One-way / Significant / Medium / Minor, where only an irreversible *decision* is One-way, a serious bug with a reversible fix is graded by blast radius not by touching auth), the orchestrator fixes everything, and a cold assessor (every round) defers reversible items to test obligations, banks areas that held through a cold pass, and decides converge / another-round / **escalate** (a structurally unstable area stops the loop and goes to the user as an architecture decision instead of looping forever) |
| `junior-review` | Spawn the junior-reviewer sub-agent for the clarifying-questions pass |
| `senior-review` | Spawn the senior-reviewer sub-agent for the deep-critique pass |
| `red-team-review` | Spawn the red-team-reviewer sub-agent to adversarially stress-test a plan or design |
| `research` | Spin up a research sub-agent to answer a technical question or find best practices, docs-first then reputable sources, grounded in the repo, plans, and current discussion |
| `implement-plan` | Execute an approved plan phase-by-phase, writing tests as each phase lands and keeping progress docs in sync |
| `write-e2e-tests` | Drive a real browser via the bundled Playwright MCP to verify flows, then write durable Playwright spec files and run them |
| `full-code-review` | 7 parallel review agents (security, backend, frontend, architecture, docs, regressions, testing) in branch-diff or full codebase-health scope. The testing reviewer checks new code has tests and runs the suite |
| `tradeoff-review` | Walk through accumulated tradeoffs one by one, collecting decisions |
| `doc-audit` | Audit project docs against the actual codebase |
| `overnight-delivery` | End-to-end pipeline: plan, plan-review to convergence, tradeoff gate, implement, 2 code review rounds |
| `supabase-security` | Research and apply Supabase security fixes (RLS, SECURITY DEFINER, storage) |

### Agents

All defined roles live in `agents/`, so skills spawn named sub-agents (consistent prompt, explicit model, restricted tools) rather than ad-hoc inline ones.

**Plan-review** (red-team-reviewer on Opus; junior-reviewer, senior-reviewer, grader, and assessor on Sonnet 5):

| Agent | Purpose |
|-------|---------|
| `junior-reviewer` | Clarifying-questions pass: surfaces ambiguity, gaps, and missing tests |
| `senior-reviewer` | Deep-critique pass: grades fit, scope, operational failure modes, and test coverage, with cited findings |
| `red-team-reviewer` | Adversarial pass: tries to break the plan and returns ranked, cited failure scenarios |
| `grader` | Rates each finding by reversibility and blast radius into One-way / Significant / Medium / Minor and tags it with an area. Decision-vs-defect gate: only an irreversible *decision* is One-way; a serious bug with a reversible fix (redeploy, tighten a policy) is Significant, not One-way, even in auth code. Cold to the cost of fixing |
| `assessor` | Runs every round, holds the full review log, defers reversible items to test obligations, banks areas fixed-and-held through a cold pass, and makes the converge / another-round / **escalate** call (only One-way/Significant gate; a recurring or unsettleable One-way decision in one area escalates to the user as an architecture decision rather than looping) |

**Code-review** (Opus), the seven lenses `full-code-review` runs in parallel:

| Agent | Purpose |
|-------|---------|
| `security-reviewer` | OWASP Top 10, auth, secrets, injection, SSRF, upload safety |
| `backend-reviewer` | API/query/error-handling correctness, performance, framework-idiom check |
| `frontend-reviewer` | component architecture, accessibility, responsive/UX, type safety |
| `architecture-reviewer` | design quality, plan conformance, DRY/repetition smell, scope, layer placement |
| `documentation-reviewer` | whether the docs reflect the change (or current code, in full scope) |
| `regression-reviewer` | the `-` lines: behavior, guards, or conventions deleted with no replacement |
| `testing-reviewer` | test coverage of the change, and actually runs the suite |

**Other:**

| Agent | Purpose |
|-------|---------|
| `researcher` (Opus) | Docs-first, cited, recommendation-first answer to a technical question, grounded in the project's stack and versions. Used by `research` |
| `doc-auditor` (Opus) | Audits docs against the code, change-scoped or full. Used by `doc-audit` |

### Bundled MCP servers

| Server | Purpose |
|--------|---------|
| `playwright` | Microsoft's Playwright MCP (`npx @playwright/mcp@latest`), exposes `mcp__playwright__*` browser tools. `write-e2e-tests` uses it to drive a real browser and verify flows before writing spec files. Starts automatically when the plugin is enabled. |

The server runs a real browser via `npx`, so Node.js must be available. It defaults to **headed** mode with **system Chrome**, ideal on a desktop where you want to watch tests run. For a **headless remote/CI session** (no display, no system Chrome), add `--headless --browser chromium` to the server's `args` in `.mcp.json` (and `--no-sandbox` in some containers); `--browser chromium` makes it use the bundled Chromium under `PLAYWRIGHT_BROWSERS_PATH` instead of hunting for system Chrome. No `playwright install` is needed where the bundled Chromium is already present. This applies to the MCP only; durable specs run under the project's own `@playwright/test`.

## Installation

From a local checkout:

```
/plugin marketplace add /path/to/dev-workflow-plugin
/plugin install dev-workflow@dev-workflow-marketplace
```

Or from GitHub:

```
/plugin marketplace add Zimoholdijk/dev-workflow-plugin
/plugin install dev-workflow@dev-workflow-marketplace
```

To pull updates later: `/plugin marketplace update` then `/reload-plugins`.

## Personalize after installing

This plugin ships generic on purpose: no personal workspace IDs, board IDs, or tracker choices are baked in, so it stays portable and safe to share or use at work. Personalization lives in two config layers the skills read at runtime, not in the plugin itself. After installing, set them up:

1. **Pick a default issue tracker (once, global).** Some skills (`discuss-feature`, `write-prd`) create tickets. Add a line to your `~/.claude/CLAUDE.md` naming your default tracker, for example: "Default issue tracker is Notion; use it for backlog tickets and feature boards unless a project overrides it." Project config always overrides this. Do not put a specific board or data source ID here; that is per-project.

2. **Capture each project's board (per repo).** Run `/dev-workflow:project-setup` in a repo. It asks which tracker and the exact target (Notion data source ID, Linear team/project) and writes it to an "Issue Tracker" section of that project's `.claude/CLAUDE.md`. `discuss-feature` and `write-prd` read that section and create tickets without re-asking. If a project has no board, leave it out and the skills just deliver the decision summary instead of creating a ticket.

3. **Tune the global rules the skills inherit.** Every skill reads `~/.claude/CLAUDE.md`. Conventions you keep there (approval semantics, no-hardcoded-config, mobile-first, writing style) are obeyed by the whole workflow. This is where your cross-project standards and voice live.

Creating tickets also requires the relevant MCP server connected in your environment (e.g. the Notion or Linear MCP). Without it, the skills still run the discussion and hand off; they just skip the ticket step.

Why this split: project-specific values differ per repo and should not leak into a shared plugin, so they live in each project's config; cross-project preferences live in your global `~/.claude/CLAUDE.md`. The plugin stays portable, and "you" lives in the layers it reads.

## Design principles baked in

- Hard separation of what (PRD) from how (plan). No code in PRDs; no code blocks in plans.
- Independent testability: every phase must be verifiable without later phases (page shell before content, hook + display together, foundation before polish).
- Branch-vs-data smell: N near-identical branches differing only by a literal should usually be one path with the difference as data.
- Framework-idiom checks: reviewers verify SQL/ORM/hook patterns against official framework docs, not blogs. A pattern with no documented analog is a red flag.
- Behavior-deletion checks: a dedicated regression reviewer grades what a diff removes, and the senior plan reviewer grades what a rewrite implicitly drops.
- Reviewers organized by lens, not seniority role-play: a clarifying-questions pass, a deep-critique pass (cited findings; a Pass is blocked until implicit-deletions, test coverage, and operational failure modes are explicitly cleared), and an adversarial red-team pass that tries to break the plan. Persona framing like "you are a senior engineer" is deliberately avoided, evidence shows it can lower accuracy on this kind of judgment.
- Testing is non-optional: plans carry a testing strategy and every phase names the tests it adds; tests are written as each phase lands, not batched to the end; a dedicated review agent runs the suite; critical flows get automated Playwright coverage rather than manual-only checks.
- Strict approval gates: silence, "ok", or "thanks" is not approval.
- Reviews are re-runnable on request, without pushback.
- Cold-start reviews: every reviewer, every round, sees only the current plan and project context, never the prior reviewers' questions, the trade-off decisions, or any review history. The Review Log lives in a **sidecar file** (`review-log.md`), never in the plan, and the reviewer agents are told never to open it, so a reviewer can't prime itself on prior rounds even though it has file access. Priming a reviewer with "this was already addressed" anchors it to rubber-stamp those parts; a cold reviewer re-scrutinizes them, so round 4 reviews as hard as round 1.
- Trade-offs are researched before they reach you: across write-plan, plan-review, implement-plan, and tradeoff-review, the workflow runs `/research` on any trade-off with a technical or best-practice dimension. The ones documented best practice settles are resolved with a citation (shown as FYI you can object to); only genuinely contested choices are surfaced for your decision, each pre-loaded with the evidence and a recommendation, never a bare "A or B?".

## Notes

- When installed as a plugin, skills are namespaced: `/dev-workflow:write-prd` instead of `/write-prd`. Skill bodies reference each other by their short names (e.g. "Run /plan-review"); Claude resolves these to the namespaced versions.
- Some skills read `~/.claude/CLAUDE.md` for global rules. If that file doesn't exist in the environment (e.g. a fresh Cowork session), the skills proceed without it.
- The typical flow: `project-setup` once per repo, then per feature: `discuss-feature` → `write-prd` → `discuss-plan` → `write-plan` → `plan-review` → `tradeoff-review` → `implement-plan` (tests ship with each phase, `write-e2e-tests` for browser flows) → `full-code-review` → `doc-audit`. `discuss-plan` is an interactive design gate that settles architecture before phasing, so `plan-review` has less to raise; it sits outside the unattended `overnight-delivery`, which chains the rest end-to-end from an approved PRD.
