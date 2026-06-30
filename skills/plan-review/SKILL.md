---
name: plan-review
description: Multi-stage plan review that loops to convergence. Cold reviewers (clarifying-questions, deep-critique, adversarial red-team) find issues; a cold grader rates each by reversibility and blast radius into One-way / Significant / Medium / Minor; the orchestrator fixes everything; a cold assessor runs every round, counts recurrence by area, defers churning reversible items to test obligations, and decides when the plan has converged. Use when an implementation or refactoring plan needs rigorous review before execution.
disable-model-invocation: false
argument-hint: "[path to plan file]"
---

# Plan Review Workflow

You have drafted or been given a plan at: $ARGUMENTS

You are the **orchestrator**. You run the loop and you fix the plan, but you do not grade severity and you do not decide convergence: those are held by other cold agents on purpose, so the agent with a stake in finishing is never the one who judges whether it is finished.

**Terseness and minimal-code modes do not override this skill.** If a mode like Honey is active, treat this as **lite** work: explanations and the trade-off questions you surface to the user are the deliverable, never compress them, and never compress the plan into something the implementer can't follow. A minimal-code / YAGNI default likewise never converts a One-way or Significant decision into an autonomous one, those still go to the user (see Trade-offs below). The cold reviewers, grader, and assessor run on their own prompts and are unaffected either way.

## The four roles (separation of powers)

| Role | Job | Cold to |
|------|-----|---------|
| **Reviewers** (junior, senior, red-team) | Find issues, one cold lens at a time. | The review history (never see the sidecar). |
| **Grader** | After each reviewer, rate every finding into a tier and tag it with an area, by the rubric below. | The cost of fixing. Does not decide fix-vs-defer; everything gets fixed. |
| **Orchestrator (you)** | Fix everything surfaced, run the self-consistency pass, write the sidecar, and on convergence write the test obligations into the plan. | Severity and stopping. You may *escalate* a grade or record a disagreement, but never silently *downgrade* a finding to make it stop mattering. |
| **Assessor** | Runs **every round**. The only agent holding the full log. Counts recurrence by area, defers churning reversible items to tests, makes the converge / another-round call. | A bias toward finishing (it made no fixes). History-aware by design. |

## Cold-start every reviewer (non-negotiable)

Every reviewer, in every stage and every round, reviews the plan **cold**. Pass each reviewer sub-agent only the current plan text and the project context (overview, CLAUDE.md, PRD, and the relevant code). **Never** pass:

- the prior reviewers' questions, or your responses to them,
- the trade-off decisions already made,
- any summary of "what changed", "what was already addressed", or "what a previous round approved",
- the review history in any form.

**The review log lives in a sidecar file, not in the plan.** All review history is written to `context/[Feature]/review-log.md`, never appended to `implementation-plan.md`. This is structural: reviewers have Read/Glob/Grep and are told to orient in the codebase, so if the log lived in the plan they could read last round's conclusions even when you don't hand them the log. The reviewer agents are also instructed never to open a `review-log` or prior-review file, and you must not point them at one.

Why it matters: letting a reviewer learn that part of the plan was "already addressed" anchors it to treat that part as settled, so it stops scrutinizing exactly where prior rounds drew their conclusions. A cold reviewer re-examines those conclusions and routinely finds issues a primed reviewer rubber-stamps. This holds **within** a run (senior and red-team do not see the junior's exchange) and **across** re-invocations (round 2+ sees the plan, not the history). The **grader** and the **assessor** are different: the grader sees one reviewer's findings + the plan (no history); the assessor is the designated holder of the log and reads all of it, that is its job, not a leak.

## Severity rubric (the grader's reference)

The grader assigns exactly one **tier** to each finding, plus an **area tag**, plus a one-line reason.

**The reversibility test (the spine):** *"Can I change this and every dependant in one atomic deploy?"* If yes, it is reversible (two-way door). If no (there are consumers you cannot update in lockstep), it is a one-way door.

**The four tiers:**

- **One-way** (irreversible): touches a trigger-list category below. Must be settled in the plan before building. Reason names which category.
- **Significant** (reversible but large): big blast radius, OR large magnitude (e.g. a 600-line file split), OR touches important parts. Reason names which driver.
- **Medium** (reversible, modest size, but a real correctness/behavior defect): data loss, wrong state, a race, an infinite loop, a broken flow. Not big, not cosmetic.
- **Minor** (cosmetic): clarity, naming, small-local-low-stakes.

**One-way-door trigger list** (a finding touching any of these is One-way by default):

1. **A published contract** any party you cannot atomically update depends on (versioned/public API, wire or RPC protocol, webhook payload, a library's public surface, a CLI's flags/output).
2. **Persisted data, once production data exists** (schema, stored formats, enum raw values, the meaning/format of IDs and keys). Data outlives code.
3. **Event/message schemas on an append-only or async channel** (once published you cannot recall it; consumers and replays depend on it).
4. **Security and trust posture** (authorization model, identity/tenancy model, where trust boundaries sit, secret/credential handling).
5. **Publicly observable behavior that becomes a de-facto contract** (URL/permalink structure, externally visible IDs, error shapes, anything third parties script against).
6. **Foundational platform commitments with broad lock-in** (primary datastore, language/runtime, cloud-proprietary primitives).
7. **Distributed-correctness commitments** (consistency model, ordering/idempotency/delivery semantics, the partition/sharding key).

Module/service boundaries are **not** a category: the irreversible kind is already item 1 (a published contract); the rest is a reversible refactor, graded by blast radius/magnitude under Significant.

**Nuances:**

- **Production-data input:** item 2 is One-way **only if production data already exists**. Pre-launch / empty store, schema changes are reversible (grade lower). Establish this from project context; if you cannot tell, **assume data exists** (the safe direction).
- **Asymmetric default:** when reversibility is genuinely uncertain, grade it **One-way**. Mislabeling a one-way door as reversible costs a migration project; the reverse costs a little deliberation.
- **Downgrade guard:** a One-way drops to reversible **only if** the plan names a real, in-use blast-radius bound (API versioning + deprecation window, expand/contract migration, consumer-driven contract tests) **and** states the migration path. Data or events already written stay irreversible even then.
- **Area tag:** label each finding with an area/topic (e.g. "upload/reconcile state machine", "share-token contract") so the assessor's recurrence count is a mechanical tally over labels, not a re-clustering of prose each round. Use stable, consistent labels across rounds.

## Before starting

Read and hold this context to pass to every sub-agent (they have no prior knowledge of the project):
- The plan file itself
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the plan

## The loop (per-round flow)

Each round is: three cold lenses, each immediately graded, with you fixing after each; then the assessor. Concretely:

1. **Junior** reviews (cold) -> **grader** grades + tags the junior's findings -> **you** fix them -> **self-consistency pass**.
2. **Senior** reviews (cold) -> grader grades + tags -> you fix -> self-consistency.
3. **Red-team** reviews (cold) -> grader grades + tags -> you fix -> self-consistency.
4. **Quality and conformance pass** (you): a self-check for repetition smell, test coverage, and CLAUDE.md conformance; route anything substantive through the grader.
5. **Assessor** runs: reads the full log, updates per-area recurrence counts, applies the tier -> behavior rules, decides converge or another round, and (on convergence) produces the test-obligation list for you to write into the plan.

You do not gate whether the assessor runs. **It runs every round**, because the recurrence count must be maintained even on rounds with top-tier items: a churning area keeps top items present, so an assessor that only ran on quiet rounds would never count or defer it, which is the exact failure this design exists to fix.

## Self-consistency: consolidate fixes before the next lens

After **every** stage's fixes and before the next lens, re-check what you just changed (you, inline, not a sub-agent: the next cold reviewer is the real un-biased catch; this just gets the obvious self-contradiction one step earlier and cheaper). Re-read each edited section against:

- the other phases,
- the File Changes table,
- the Architecture Decisions,
- any invariant stated elsewhere in the plan.

Fix the contradictions you find, then re-check those follow-on fixes. The dominant source of churn is a fix that introduces a regression the next cold lens then spends a whole round catching; catching your own contradiction here is cheaper. After the red-team stage (no lens follows it that round), next round's cold junior is the backstop.

## Grading a reviewer's findings (the grader)

After each reviewer returns, spawn the `grader` sub-agent. Give it: the reviewer's findings, the current plan, the project context, the **severity rubric above** (verbatim), and whether **production data exists** for this project. It returns each finding tagged with `{tier, area, one-line reason}`.

You then **address every finding it surfaced**, regardless of tier. How you address it depends on the tier: **Medium and Minor you fix autonomously**; **One-way and Significant you treat as the user's call** per the trade-off rule below, they are the irreversible or consequential decisions, and the point of grading by reversibility is that a person, not the loop, owns them. The tier also governs convergence later. You may **escalate** the grader's tier (treat a Minor as Medium) or record a disagreement in the sidecar, but you may **never downgrade** a tier to avoid a round, a fix, or a question to the user: severity is the grader's call, not yours.

## Trade-offs: ask the user about One-way and Significant decisions

The tier decides who owns the call. **One-way and Significant findings are the user's to decide** whenever they involve a genuine choice: those are the irreversible or consequential decisions, and the whole point of grading by reversibility is that a person owns them, not the loop. **Medium and Minor you fix autonomously**, do not bother the user with reversible, low-stakes fixes.

So as you address each finding:

- **Medium / Minor** → just fix it. No question.
- **One-way / Significant** → before you decide it, run `/research` if it has a technical or best-practice dimension (grounded in the project's stack and versions; several in parallel is fine), then route by what the evidence shows:
  - **Resolved by evidence** (docs/best practice clearly favor one option, or the downside is negligible at this scale): apply the documented answer, record the decision + citation in the plan, and **tell the user what you did and why so they can object**. An irreversible or consequential call still gets surfaced even when the answer is clear.
  - **Genuine choice** (defensible either way, depends on product/UX/risk preference): **STOP and ask the user before deciding it.** One decision per message, with the researched points (what the docs say, the real cost of each side, your recommendation). Never a bare "A or B?". Wait for the answer before continuing.

**Ask at the moment you reach it, not batched at the end.** When the orchestrator hits a One-way or Significant finding that needs a call, surface it then, one at a time, and resume after the user answers. Do not silently decide a user-owned call to keep the loop moving, and do not pile them into one end-of-run "pick for each of these" message (the user has repeatedly flagged that as the wrong pattern). If several genuinely accumulate, use `/tradeoff-review` to walk them one at a time.

**Unattended exception.** When this loop runs inside an unattended pipeline (e.g. `/overnight-delivery`), you cannot stop for each answer. There, apply only the evidence-resolved One-way/Significant calls in-loop (with citations), and accumulate the genuine choices for that pipeline's trade-off gate instead of blocking mid-loop.

## Stage 1: Clarifying-Questions Review (junior-reviewer)

Spawn the `junior-reviewer` sub-agent. In your prompt include the plan, overview, project CLAUDE.md, any PRD, a summary of the relevant codebase state, and:

- **Orientation instruction.** The junior has Read, Glob, Grep. Tell it: before asking anything, orient like a day-one engineer, open the files and directories the plan touches and the patterns it references. A question the orientation would answer ("what props does X take?") is noise; a question that survives it is signal ("the plan reuses pattern X from `routeZ.tsx:42`, but X is inline-defined and not exported, export, move, or duplicate?"). Cite file paths.
- **Testing instruction.** As part of orientation, check how the project tests today (`*.test.*`, `*.spec.*`, `playwright.config.*`, a `tests/`/`e2e/` dir, test scripts in `package.json`). For each phase adding logic, ask "how will I know this works, and what test proves it?" Flag any phase adding non-trivial logic with no named test, any critical flow with only a manual check, and any assumed-but-absent test infrastructure.

Then grade its findings (the grader), fix them, self-consistency.

## Stage 2: Deep-Critique Review (senior-reviewer)

Spawn the `senior-reviewer` cold (do not include the junior's exchange; an independent re-hit is signal, not waste). Include the plan, overview, CLAUDE.md, PRD, and these required axes:

- **Axis 9, repetition/factoring smell.** Grep the plan's prose for near-identical branch descriptions (verb + object recurring with only a literal/key/separator/metadata differing); flag as a structural issue and sketch the unified path. Cannot pass without explicitly grading this.
- **Axis 10, framework-idiom.** For SQL/RLS/ORM/hooks/middleware or any third-party-governed shape, verify the pattern appears in that tool's **official** docs (the framework's own site, not blogs/SO). A pattern that implements the spec but has no documented analog is a red flag. If unsure, run `/research`.
- **Axis 11, behavior the plan implicitly removes.** When the plan rewrites a hook/module/route/shared file, read the actual file first and enumerate its current responsibilities (every case, guard, ref + side effect, exported helper, documented convention); check each is preserved or intentionally dropped. Scan for "remove/delete/no longer needed". Grade "would shipping this quietly break anything the old code did?"
- **Axis 12, test coverage.** Does a Testing Strategy section exist? Does every File-Changes row that adds logic map to a named test (unit for pure logic, integration for boundaries, e2e for user flows)? Are tests written per-phase, not deferred? Are critical Verification flows automated, not manual-only? If no infra exists, does Phase 1 establish it? Do tests cover error/auth-boundary/edge cases, not just the happy path?
- **Axis 13, operational reality & failure modes.** A DFMEA-style pass: error/empty/loading states, auth/ownership boundaries (signed-out, wrong-owner, expired), rollback for risky migrations, production observability. The worst gaps are what the plan is *silent* about.
- **Output:** lead with the single strongest reason this plan should not ship; cite every finding to `file:line` / a quoted plan line / a named rule; try to refute each against the actual files before emitting it and drop what you cannot substantiate (over-flagging correct work is as much a failure as missing a real problem).

Then grade its findings, fix them, self-consistency.

## Stage 3: Red-Team (Adversarial) Review

The junior and senior are cooperative and evaluative; neither's job is to *break* the plan. Spawn the `red-team-reviewer` cold against the current plan. Include the plan, overview, CLAUDE.md, PRD, a pointer to the real code it touches, and:

> Try to break this plan. Find the wrong assumption, the unhandled failure or partial-failure path, the edge/boundary case, the scale reality at real row counts, the race or data-integrity gap, and above all what the plan is *silent* about. Ground every attack in a `file:line` or quoted plan line, be concrete (a specific scenario, not "what if it's slow"), and don't fabricate weaknesses to look thorough. If you can't break it, say so and name the one or two most fragile spots. Rank by likelihood × blast radius and end with whether the plan has an unaddressed critical failure mode.

Then grade its findings, fix them, self-consistency.

## Quality and conformance (each round, before the assessor)

After the red-team stage and its fixes, self-check the plan before handing the round to the assessor (the plan is prose, so this is a read-through, not a code tool):

- Verify yourself, do not rely solely on the senior: does any phase describe near-identical work differing only by a literal/key/separator/metadata (repetition smell)? Does every phase that adds logic name its tests, and are critical Verification flows automated rather than manual-only? Does the plan violate any CLAUDE.md rule (DRY, error handling, hardcoded values, scope creep)?
- Anything substantive you surface here, route it through the **grader** like a reviewer finding so it gets a tier and an area and counts toward convergence. Trivial conformance tidy-ups can be applied directly; either way, run the self-consistency pass on what you changed.

## The assessor (runs every round)

After the three stages, spawn the `assessor` sub-agent. Give it the **full `review-log.md` sidecar plus this round's graded findings** (tiers + area tags) and the rubric. It does four things:

1. **Update per-area recurrence counts.** For each area tag, how many rounds has a Medium (or Minor) in that area now been fixed.
2. **Apply the tier -> behavior rules** (below) to decide: another round, or converged.
3. **Defer at K=3.** Any area whose Medium/Minor has now been fixed in **three** rounds: stop looping it. Mark the whole area a **test obligation** and raise a **simplification flag** (recurring self-introduced defects in one subsystem is a design smell, not a coverage gap).
4. **On convergence, produce the test-obligation list**: every deferred Medium/Minor and every simplification-flagged area, each with the behavior a test must pin and the area it belongs to.

The assessor returns: the verdict (`Another round` or `Converged`), the updated counts, and (if converged) the test-obligation list. It is cold (made no fixes) and is the only role permitted to read the whole log.

### Tier -> behavior

- **One-way:** always another round until settled. No count, never deferred (you cannot push an irreversible decision to "fix in code later"). If it will not stay settled, the assessor raises an advisory **"design unstable"** flag to the user, but still never auto-defers it.
- **Significant:** always another round (big enough to verify in the plan). No count, never deferred. Self-limiting in practice; persistent churn raises the same advisory flag.
- **Medium:** counted per area. 1st and 2nd appearance in an area force another round. **3rd appearance in the same area -> defer that area to a test obligation + simplification flag.**
- **Minor:** never forces a round on its own. Counted only so a recurring nit is deferred to backlog/test on its 3rd appearance instead of being re-fixed forever.

**Recurrence threshold K = 3** (fixed in prose up to twice; the third recurrence in the same area is a defer).

### Convergence

A plan is **converged** on a round with **no open One-way, no Significant, and no live (un-deferred) Medium**, only Minors and already-deferred items remain. A round that made any One-way or Significant change, or that still has a live Medium below K=3, is never the last round.

State the assessor's verdict explicitly each round: `Another round` (with what was fixed and what is still live) or `Converged`.

This is an internal loop with a deterministic exit condition, **not** the Claude Code `/loop` or `/goal` primitives, and it does not use them. `/loop` is for time-spaced recurring tasks; `/goal` is a session-level model evaluator. Here the assessor owns the exit decision directly, and cold-start holds across rounds because each round spawns fresh reviewer sub-agents fed only the plan, with the sidecar withheld.

## On convergence: write obligations into the plan, then the sidecar

When the assessor returns `Converged`:

1. **Write the test obligations into `implementation-plan.md`** (you are the only writer). Append a consolidated `## Test Obligations` section listing each deferred item (the behavior a test must pin, its area, and any simplification note), **and** add a reference in each relevant phase to the obligations that phase must fulfil, so the test lands in the same phase as the code. This is the last edit before the plan is marked `Reviewed`; it is finalising the plan, not editing a frozen one. The obligations live in the plan (forward-looking spec), not the sidecar; a cold reviewer seeing them on a later re-invocation is not primed (spec, not an approval stamp).
2. **Set the plan's `Status` to `Reviewed`** (a one-word header stamp is fine; detailed history stays in the sidecar).

Every round (converged or not), append a numbered entry to the **sidecar** at `context/[Feature]/review-log.md` (create it if absent). Never add review history to the plan. Structure:

```markdown
# [Feature]: Review Log

> Sidecar for `implementation-plan.md`. Review history lives here, never in the plan, so cold reviewers can't be primed by prior rounds. For the author and user only.

## Round [N], [date]

### Findings (graded)
- [Each finding: tier, area, one-line reason, and how it was fixed]

### Area recurrence counts
- [area: count] for any Medium/Minor area in play

### Assessor verdict
- Converged / Another round. [If converged, the test-obligation list. If another round, what is still live.]
- [Any "design unstable" or simplification flags]
```

## Re-invocation

This workflow runs review rounds in an internal loop until the assessor returns `Converged`. There is no cap and no diagnostic gate. If the user later asks for another pass on an already-converged plan, run it; never argue that prior reviews should be sufficient or propose skipping ahead. Each round appends a numbered entry to the sidecar, and the recurrence counts carry across re-invocations (the assessor reads them from the sidecar).
