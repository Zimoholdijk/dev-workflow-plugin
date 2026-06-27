# dev-workflow

A Claude Code plugin packaging a PRD-first development workflow: write PRDs, draft phased implementation plans, run multi-stage reviews with junior and senior reviewer sub-agents, implement plans phase-by-phase with progress tracking, run parallel code reviews, collect tradeoff decisions, and audit documentation against the codebase. Testing is treated as a first-class part of the workflow: plans must carry a testing strategy, every phase ships its own tests, a dedicated review agent runs the suite, and a bundled Playwright MCP drives real-browser end-to-end tests.

## What's included

### Skills

| Skill | Purpose |
|-------|---------|
| `project-setup` | Bootstrap a new project with context docs, CLAUDE.md, and workflow conventions |
| `discuss-feature` | Pre-PRD discussion in plain language: one topic per message, collects decisions, creates a backlog ticket, hands off to `write-prd` |
| `write-prd` | Draft a PRD for a new feature |
| `write-plan` | Draft a phased implementation plan from an approved PRD |
| `plan-review` | Multi-stage plan review: junior Q&A, senior architecture review, conformance check |
| `junior-review` | Spawn the junior-reviewer sub-agent for clarifying questions |
| `senior-review` | Spawn the senior-reviewer sub-agent for architectural review |
| `research` | Spin up a research sub-agent to answer a technical question or find best practices, docs-first then reputable sources, grounded in the repo, plans, and current discussion |
| `implement-plan` | Execute an approved plan phase-by-phase, writing tests as each phase lands and keeping progress docs in sync |
| `write-e2e-tests` | Drive a real browser via the bundled Playwright MCP to verify flows, then write durable Playwright spec files and run them |
| `full-code-review` | 7 parallel review agents (security, backend, frontend, architecture, docs, regressions, testing) in branch-diff or full codebase-health scope. The testing reviewer checks new code has tests and runs the suite |
| `tradeoff-review` | Walk through accumulated tradeoffs one by one, collecting decisions |
| `doc-audit` | Audit project docs against the actual codebase |
| `overnight-delivery` | End-to-end pipeline: plan, 4 review rounds, implement, 2 code review rounds |
| `supabase-security` | Research and apply Supabase security fixes (RLS, SECURITY DEFINER, storage) |

### Agents

| Agent | Purpose |
|-------|---------|
| `junior-reviewer` | Anxious junior engineer who surfaces ambiguity and gaps in plans |
| `senior-reviewer` | Senior architect who reviews for fit, scope creep, and risk |

### Bundled MCP servers

| Server | Purpose |
|--------|---------|
| `playwright` | Microsoft's Playwright MCP (`npx @playwright/mcp@latest`), exposes `mcp__playwright__*` browser tools. `write-e2e-tests` uses it to drive a real browser and verify flows before writing spec files. Starts automatically when the plugin is enabled. |

The server runs a real browser via `npx`, so Node.js must be available in the environment. In remote/CI sessions a browser is typically pre-installed, no `playwright install` needed.

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
- Testing is non-optional: plans carry a testing strategy and every phase names the tests it adds; tests are written as each phase lands, not batched to the end; a dedicated review agent runs the suite; critical flows get automated Playwright coverage rather than manual-only checks.
- Strict approval gates: silence, "ok", or "thanks" is not approval.
- Reviews are re-runnable on request, without pushback.

## Notes

- When installed as a plugin, skills are namespaced: `/dev-workflow:write-prd` instead of `/write-prd`. Skill bodies reference each other by their short names (e.g. "Run /plan-review"); Claude resolves these to the namespaced versions.
- Some skills read `~/.claude/CLAUDE.md` for global rules. If that file doesn't exist in the environment (e.g. a fresh Cowork session), the skills proceed without it.
- The typical flow: `project-setup` once per repo, then per feature: `discuss-feature` → `write-prd` → `write-plan` → `plan-review` → `tradeoff-review` → `implement-plan` (tests ship with each phase, `write-e2e-tests` for browser flows) → `full-code-review` → `doc-audit`. `overnight-delivery` chains the per-feature steps end-to-end.
