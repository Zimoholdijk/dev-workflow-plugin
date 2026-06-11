# dev-workflow

A Claude Code plugin packaging a PRD-first development workflow: write PRDs, draft phased implementation plans, run multi-stage reviews with junior and senior reviewer sub-agents, implement plans phase-by-phase with progress tracking, run parallel code reviews, collect tradeoff decisions, and audit documentation against the codebase.

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
| `implement-plan` | Execute an approved plan phase-by-phase, keeping progress docs in sync |
| `full-code-review` | 6 parallel review agents (security, backend, frontend, architecture, docs, regressions) in branch-diff or full codebase-health scope |
| `tradeoff-review` | Walk through accumulated tradeoffs one by one, collecting decisions |
| `doc-audit` | Audit project docs against the actual codebase |
| `overnight-delivery` | End-to-end pipeline: plan, 4 review rounds, implement, 2 code review rounds |
| `supabase-security` | Research and apply Supabase security fixes (RLS, SECURITY DEFINER, storage) |

### Agents

| Agent | Purpose |
|-------|---------|
| `junior-reviewer` | Anxious junior engineer who surfaces ambiguity and gaps in plans |
| `senior-reviewer` | Senior architect who reviews for fit, scope creep, and risk |

## Installation

From a local checkout:

```
/plugin marketplace add /path/to/dev-workflow-plugin
/plugin install dev-workflow@dev-workflow-marketplace
```

Or, once pushed to GitHub:

```
/plugin marketplace add <your-github-user>/dev-workflow-plugin
/plugin install dev-workflow@dev-workflow-marketplace
```

## Design principles baked in (merged from the work version, v1.1.0)

- Hard separation of what (PRD) from how (plan). No code in PRDs; no code blocks in plans.
- Independent testability: every phase must be verifiable without later phases (page shell before content, hook + display together, foundation before polish).
- Branch-vs-data smell: N near-identical branches differing only by a literal should usually be one path with the difference as data.
- Framework-idiom checks: reviewers verify SQL/ORM/hook patterns against official framework docs, not blogs. A pattern with no documented analog is a red flag.
- Behavior-deletion checks: a dedicated regression reviewer grades what a diff removes, and the senior plan reviewer grades what a rewrite implicitly drops.
- Strict approval gates: silence, "ok", or "thanks" is not approval.
- Reviews are re-runnable on request, without pushback.

## Notes

- When installed as a plugin, skills are namespaced: `/dev-workflow:write-prd` instead of `/write-prd`. Skill bodies reference each other by their short names (e.g. "Run /plan-review"); Claude resolves these to the namespaced versions.
- Some skills read `~/.claude/CLAUDE.md` for global rules. If that file doesn't exist in the environment (e.g. a fresh Cowork session), the skills proceed without it.
- The typical flow: `project-setup` once per repo, then per feature: `discuss-feature` → `write-prd` → `write-plan` → `plan-review` → `tradeoff-review` → `implement-plan` → `full-code-review` → `doc-audit`. `overnight-delivery` chains the per-feature steps end-to-end.
