---
name: plan-review
description: Multi-stage plan review that loops to convergence, bounded at 5 rounds. Establishes the plan's load-bearing premises first; cold reviewers (clarifying-questions, deep-critique, adversarial red-team) find issues, each mechanically quote-checked; a cold grader rates each by reversibility and blast radius into One-way / Significant / Medium / Minor or discards it as Not-an-issue with evidence (only an irreversible decision is One-way, not every bug near auth); the orchestrator fixes everything surgically; a cold assessor runs every round, tracks the dry signal, defers reversible items to test obligations, banks areas that held through a cold pass, and decides converge / another-round / escalate (a repeated One-way, a third Significant in one area, or the round cap goes to the user instead of looping). Fixes that would add machinery to the plan are trade-offs for the user regardless of tier, and the premise gate covers how production is actually operated so reviewers don't invent guards for risks a manual workflow already handles. In interactive runs, a loop still open after round 3 pauses for a plain-language design checkpoint (recurring churn usually means an over-complicated design; one simplification conversation beats two more point-fix rounds). Use when an implementation or refactoring plan needs rigorous review before execution.
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
| **Assessor** | Runs **every round**. The only agent holding the full log. Defers reversible items to tests, banks areas that held through a cold pass, tracks new-unique findings per round, makes the converge / another-round / escalate call (only One-way/Significant gate; a recurring One-way area or the round cap escalates to the user). | A bias toward finishing (it made no fixes). History-aware by design. |

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

**First, decision or defect?** Before reversibility, classify the finding. A **decision** is a choice that could legitimately go more than one way and that a person should own ("*which* trust model / data shape / contract?" — and equally "*which* methodology or mechanism measures/verifies X?": when defensible approaches trade precision against complexity, a person owns that choice; treating it as a defect-with-one-fix licenses the loop to redesign it autonomously round after round). A **defect** is the plan being wrong, unsafe, or incomplete, with a correct fix ("is this right?", no). **Only a decision can be One-way.** A defect is graded by reversibility and blast radius, *even in security, auth, or data code*, because severity is not irreversibility: a serious security bug whose fix ships in one atomic deploy (redeploy a function, tighten a policy pre-prod-data, add a gate) is a reversible defect graded Significant, not One-way. One-way is reserved for the irreversible *decision* underneath (e.g. "role in a user-writable table?"), not for every bug near auth.

**The reversibility test (the spine):** *"Can I change this and every dependant in one atomic deploy?"* If yes, it is reversible (two-way door). If no (there are consumers you cannot update in lockstep), it is a one-way door.

**The four tiers:**

- **One-way** (irreversible *decision*): a design choice touching a trigger-list category below that cannot be changed with every dependant in one atomic deploy. Must be settled in the plan before building. Reason names the category and why it is a decision, not a defect.
- **Significant** (reversible but consequential): big blast radius, OR large magnitude (e.g. a 600-line file split), OR a serious *defect* in important code (cross-tenant leak, auth gap) whose fix is reversible. Reason names which driver.
- **Medium** (reversible, modest size, but a real correctness/behavior defect): data loss, wrong state, a race, an infinite loop, a broken flow. Not big, not cosmetic.
- **Minor** (cosmetic): clarity, naming, small-local-low-stakes.

**Size the defect, not the area.** Grading Significant because a finding sits in money/auth/safety-adjacent code, when its own consequence is small, is the same category shortcut the One-way rule forbids. A stale sentence, a self-contradiction, or a missing checklist line whose surrounding enforcement is intact defaults to **Medium**, even in a payment flow — a Significant grade costs a full extra round, so it is earned by the defect's observable consequence, never by the area's stakes.

**Plus one discard disposition — Not-an-issue:** the finding is factually wrong, already handled by the plan, or moot given the established premises. The grader must cite the evidence (the plan line or file that refutes it). A Not-an-issue finding **exits the pipeline**: it is not fixed, does not become a test obligation, and does not gate; it is recorded in the sidecar with the refutation so the round's record is complete. This is the false-positive filter, every production review system that beat noise added a subtractive stage, and reviewers under pressure to find something do produce refutable findings (real runs logged "CSP concern refuted", "email-change attack refuted via GoTrue source"). Use it only with evidence; when in doubt between Not-an-issue and Minor, grade Minor.

**One-way-door trigger list** (a *decision* touching any of these is One-way; a defect near them is still graded by reversibility per Gate 1):

1. **A published contract** any party you cannot atomically update depends on (versioned/public API, wire or RPC protocol, webhook payload, a library's public surface, a CLI's flags/output).
2. **Persisted data, once production data exists** (schema, stored formats, enum raw values, the meaning/format of IDs and keys). Data outlives code.
3. **Event/message schemas on an append-only or async channel** (once published you cannot recall it; consumers and replays depend on it).
4. **Security and trust posture**, meaning the trust-boundary *model* itself: who is allowed to reach what, the identity/tenancy model, where trust sits, secret/credential handling. This is the decision about the boundary, not every implementation detail near auth code, a missing ownership check in a redeployable function is a defect graded by reversibility, not a posture decision.
5. **Publicly observable behavior that becomes a de-facto contract** (URL/permalink structure, externally visible IDs, error shapes, anything third parties script against).
6. **Foundational platform commitments with broad lock-in** (primary datastore, language/runtime, cloud-proprietary primitives).
7. **Distributed-correctness commitments** (consistency model, ordering/idempotency/delivery semantics, the partition/sharding key).

Module/service boundaries are **not** a category: the irreversible kind is already item 1 (a published contract); the rest is a reversible refactor, graded by blast radius/magnitude under Significant.

**Nuances:**

- **Production-data input:** item 2 is One-way **only if production data already exists**. Pre-launch / empty store, schema changes are reversible (grade lower). Establish this from project context; if you cannot tell, **assume data exists** (the safe direction).
- **Asymmetric default (decisions only):** when a genuine *decision's* reversibility is uncertain, grade it **One-way**, mislabeling a one-way door as reversible costs a migration project; the reverse costs a little deliberation. This tie-breaks decisions; it is not a licence to inflate a reversible *defect* to One-way because it sits in a sensitive category. Size a defect by its actual blast radius.
- **Downgrade guard:** a One-way drops to reversible **only if** the plan names a real, in-use blast-radius bound (API versioning + deprecation window, expand/contract migration, consumer-driven contract tests) **and** states the migration path. Data or events already written stay irreversible even then.
- **Area tag:** label each finding with an area/topic (e.g. "upload/reconcile state machine", "share-token contract") so the test-obligation list, the assessor's banking, and its escalation test can name a consistent area. Use stable, consistent labels across rounds.

## Before starting

Read and hold this context to pass to every sub-agent (they have no prior knowledge of the project):
- The plan file itself
- `~/.claude/CLAUDE.md` (global rules)
- `.claude/CLAUDE.md` (project rules, if it exists)
- `context/overview.md` (project overview, if it exists)
- Any PRD or feature docs referenced in the plan

### Establish the plan's load-bearing premises first

Before round 1, verify the facts the plan's correctness rests on, against the code and, where you cannot tell, against the user. The failure this prevents: whole rounds spent reviewing a false premise (one real case reviewed four rounds against the wrong object store, inferred from an MCP connector that happened to be in the session; another spent rounds on installed-base and versioning concerns for a feature that was not even live). Cold reviewers inherit a bad premise every round and cannot fix it, only you can, up front. Check at least:

- **Is this live?** Are there installed clients, published surfaces, or real usage whose compatibility must be preserved, or is it pre-launch (which moots a whole class of migration / versioning / regression concerns)?
- **Does production data already exist?** This is the exact input the grader needs for trigger-category 2; establish it once, here.
- **What is the real infrastructure?** The actual datastore, object store, runtime, and deploy target, read from the repo (`package.json` / lockfiles, `docker-compose`, config, env), never assumed from what an MCP connector happens to expose in the session.
- **Installed versions** of anything the plan leans on.
- **How is it actually operated?** How code reaches production (a `git pull`? a managed deploy? who runs it?), how many instances run, and what the user already does by hand (checks, backups, deploy steps, dashboards they watch). Reviewers who don't know this invent machinery to guard risks a one-line human workflow already covers — one audited run built live-vs-git diff-verification tooling for a deploy that was a `git pull`. Whenever a finding's fix would guard an *operational* risk, check these premises first, and if they don't answer it, **ask the user how that operation works today** before any machinery enters the plan.

Record these as a short **Premises** note and pass them to every reviewer and the grader as project facts. Facts are project context, not review history, so this does not prime cold-start. If a load-bearing premise cannot be verified from the code, ask the user before spending a round on it.

## The loop (per-round flow)

Each round is: three cold lenses, each immediately graded, with you fixing after each; then the assessor. Concretely:

1. **Junior** reviews (cold) -> **grader** grades + tags the junior's findings -> **you** fix them -> **self-consistency pass**.
2. **Senior** reviews (cold) -> grader grades + tags -> you fix -> self-consistency.
3. **Red-team** reviews (cold) -> grader grades + tags -> you fix -> self-consistency.
4. **Quality and conformance pass** (you): a self-check for repetition smell, test coverage, and CLAUDE.md conformance; route anything substantive through the grader.
5. **Assessor** runs: reads the full log, applies the tier -> behavior rules, decides converge, another round, or escalate, and (on convergence) produces the test-obligation list for you to write into the plan.

You do not gate whether the assessor runs. **It runs every round**: it owns the convergence decision and is the only holder of the full log, so it is also where recurring One-way findings in one area get noticed and turned into an escalation instead of another round.

## Self-consistency: consolidate fixes before the next lens

**Edit surgically, never rewrite wholesale.** Apply each fix as a targeted edit to the specific section it concerns; do not regenerate whole sections or the whole plan to "clean it up". Wholesale rewrites silently drop constraints that live mid-document and undo earlier rounds' fixes (measured: whole-document regeneration loses roughly 3x more previously-settled content than scoped edits, and models progressively violate earlier constraints as turns accumulate). The churn where a fix re-breaks a prior fix is this failure mode.

**The plan must read as a plan, never as a review artifact.** Write every fix as if the plan had always said it. No "Round 2 addition", no "(added per red-team finding)", no reviewer names, severity tiers, finding numbers, or changelog markers anywhere in the plan text — a reader of the finished plan must not be able to tell which parts came from review. All provenance (which round, which finding, which reviewer, why) belongs exclusively in the sidecar; that is what it is for. This is not just cleanliness: reviewers read the plan cold each round, so inline round markers leak the review history straight past the sidecar quarantine and prime the next pass. The only review-derived content the plan may ever carry is the consolidated `## Test Obligations` section and the one-word `Status` stamp written at convergence.

After **every** stage's fixes and before the next lens, re-check what you just changed (you, inline, not a sub-agent: the next cold reviewer is the real un-biased catch; this just gets the obvious self-contradiction one step earlier and cheaper). Re-read each edited section against:

- the other phases,
- the File Changes table,
- the Architecture Decisions,
- any invariant stated elsewhere in the plan.

Fix the contradictions you find, then re-check those follow-on fixes. The dominant source of churn is a fix that introduces a regression the next cold lens then spends a whole round catching; catching your own contradiction here is cheaper. After the red-team stage (no lens follows it that round), next round's cold junior is the backstop.

## Grading a reviewer's findings (the grader)

After each reviewer returns, first run the **quote check**, then spawn the `grader` sub-agent.

**Quote check (mechanical, before grading).** Every finding must cite its evidence: a quoted plan line, a `file:line`, or a named rule. Verify each citation actually exists, grep the quoted text in the plan or the cited file. A finding whose citation cannot be found is unsubstantiated: do not silently fix it and do not grade it; note it in the sidecar as "citation not found" and drop it (if it *sounds* important, re-ask that reviewer to substantiate it once). This is the cheapest hallucination filter available, fabricated citations are a documented reviewer failure mode, and a quote's existence is mechanically checkable.

Give the grader: the reviewer's findings (quote-checked), the current plan, the project context, the **severity rubric above** (verbatim), and whether **production data exists** for this project. It returns each finding tagged with `{tier, area, one-line reason}`, where tier may also be **Not-an-issue** (with the refuting evidence).

**Spawning the grader is non-negotiable, and you never grade in its place.** After *every* reviewer that returned at least one finding, in *every* round, the grader runs. You are **cold to severity** (see the roles table): assigning a tier yourself, eyeballing a finding as "probably Minor" and skipping the grader, or grading in your head to move faster, all collapse the separation of powers that stops you from quietly downgrading findings to finish sooner. The only case where you skip the grader is a reviewer that returned no findings (nothing to grade). **Self-check before you fix anything: every finding must carry a tier the *grader* assigned. If any finding has a tier that came from you, you skipped or overrode the grader, stop and spawn it on that reviewer's findings.**

**Hand it inputs, not conclusions (anti-priming).** Pass the reviewer's findings as they came, the plan, the context, the rubric, and the production-data fact, then let the grader rate them. Do **not**:

- suggest, pre-state, or hint a tier ("this one's probably One-way, grade accordingly") — anchoring the grader to a tier defeats the reason grading is a separate cold role, exactly as it would for the assessor;
- pre-sort, rank, or group the findings by how serious they look to you (hand them in the reviewer's order);
- drop or withhold a finding you personally judge trivial — whether it's Minor is the grader's call, not yours; every finding the reviewer surfaced goes in.

The grader assigns the tiers and returns `{tier, area, reason}`; you consume its output, you do not walk it to your answer.

You then **address every finding it surfaced**. How depends on the disposition: **Not-an-issue is recorded in the sidecar with its refutation and nothing else happens** (do not fix a refuted finding "to be safe", that is churn). **Medium and Minor you fix autonomously** — unless the fix would add machinery, which routes through the complexity gate below regardless of tier. **One-way and Significant go through the trade-off rule below**: the ones that involve a *genuine choice* are the user's call; a Significant *defect* with one unambiguous correct fix (a completeness gap, a missing phase, a contradiction) has no choice in it, fix it autonomously and surface it FYI, do not manufacture a question. The tier also governs convergence later. You may **escalate** the grader's tier (treat a Minor as Medium) or record a disagreement in the sidecar, but you may **never downgrade** a tier (including to Not-an-issue) to avoid a round, a fix, or a question to the user: severity is the grader's call, not yours.

## Trade-offs: ask the user about One-way and Significant decisions

The tier decides who owns the call. **One-way and Significant findings are the user's to decide** whenever they involve a genuine choice: those are the irreversible or consequential decisions, and the whole point of grading by reversibility is that a person owns them, not the loop. **Medium and Minor you fix autonomously**, do not bother the user with reversible, low-stakes fixes.

So as you address each finding:

- **Medium / Minor** → just fix it. No question — unless the fix adds machinery (complexity gate below).
- **One-way / Significant** → before you decide it, run `/research` if it has a technical or best-practice dimension (grounded in the project's stack and versions; several in parallel is fine), then route by what the evidence shows:
  - **Resolved by evidence** (docs/best practice clearly favor one option, or the downside is negligible at this scale): apply the documented answer, record the decision + citation in the plan, and **tell the user what you did and why so they can object**. An irreversible or consequential call still gets surfaced even when the answer is clear.
  - **Genuine choice** (defensible either way, depends on product/UX/risk preference): **STOP and ask the user before deciding it.** One decision per message, with the researched points (what the docs say, the real cost of each side, your recommendation). Never a bare "A or B?". Wait for the answer before continuing.

**The complexity gate: a fix that adds machinery is a trade-off, whatever its tier.** The tier decides how *urgent* a finding is; it does not decide who owns a fix that *grows the plan*. Any fix that would add a mechanism the plan did not have — a new file, table, job, guard, verification step, config surface, or moving part — is never applied autonomously, even for a Medium or Minor. Present it as a genuine choice with the do-less options always stated: accept the risk and record it, cover it with a manual/runbook step, or delete the premise that created the need. This is the single biggest endless-loop fuel in audited runs: every area that churned past round 3 was machinery a review round itself added (a re-check classifier, an audit discriminator, a reconciler with heartbeats and attempt counters, a cron parser, a replay simulation, a deploy-parity guard), each generated Significants until it was deleted, and none survived as a fix. Edits to text and mechanisms the plan already has stay autonomous per the tier rules — the gate is on *additions*.

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
- Anything substantive you surface here, route it through the **grader** like a reviewer finding so it gets a tier and an area (One-way/Significant gate convergence; Medium/Minor become test obligations). Trivial conformance tidy-ups can be applied directly; either way, run the self-consistency pass on what you changed.

## The assessor (runs every round)

After the three stages, spawn the `assessor` sub-agent. Give it **a pointer to the `review-log.md` sidecar** (its path, for it to read), this round's **raw graded findings** (each as `{tier, area, one-line reason}`, exactly as the grader emitted them), and the rubric.

**Hand it inputs, not conclusions (anti-priming).** The assessor is history-aware on purpose, but it must reach the verdict *itself*. Do **not** pre-digest the round for it, and in particular do **not**:

- state, suggest, or "expect" a verdict ("the expected verdict is Another round — confirm" makes it a rubber stamp; let it decide and tell *you*),
- pre-judge the escalation question for it (don't assert "postMessage was a different area than logo-render, so no escalation" — hand it the area-tagged findings and let it compare across rounds),
- replace the sidecar with your own summary of "what happened in rounds 1-N" (point it at the file; reading the log is its job and yours might shade it),
- re-state which tiers gate as if instructing it (the rubric and its own agent prompt already carry that; repeating it as "the rules for this round are…" invites you to mis-state them).

Pass the round's findings and the path; ask for the verdict. Your job is to give it complete, unspun inputs, not to walk it to your answer. It does five things:

1. **Apply the tier -> behavior rules** (below) to decide the verdict: another round, converged, or escalate. Only **One-way and Significant** findings gate convergence; Medium, Minor, and Not-an-issue do not.
2. **Track the dry signal.** Deduplicate this round's gated findings against the full log and count the **new unique** ones. A round whose gated findings are all repeats of already-addressed items (or which has none) is a **dry round**, the strongest convergence evidence there is; per-round new-finding counts decaying to zero is how real audits end. It also counts the **self-inflicted** findings — those against plan text or machinery added in the previous two rounds; a rising share means the loop is reviewing its own fixes, and the exit is deletion, not another fix. Both counts are recorded in the sidecar every round.
3. **Escalate a structurally unstable area, or a loop out of budget (hard stop).** Three triggers, same verdict:
   - **Area instability:** the same area produces a **second** One-way *decision*, or a One-way in it will not stay settled across two rounds. The problem is that area's design; point-fix rounds cannot converge it.
   - **Significant recurrence:** the same area produces its **third Significant** across the run (banked or not). Three rounds of prose fixes in one area means the area's shape is wrong; the user chooses the simplification — usually deleting the machinery — instead of the loop re-fixing it a fourth time. A dry-signal reversal in an area with a live Significant is supporting evidence.
   - **Round cap:** the run reaches **round 5** without converging. The evidence on iterative review is unambiguous: gains concentrate in the first 2-3 passes, and past the plateau revision degrades more than it repairs, so a loop still finding gated issues at round 5 is not converging, something structural is wrong (an unstable design, a false premise, or reviewer noise), and burning rounds 6-10 point-fixing is the documented failure mode. The cap is a guardrail, not a target; most plans should converge in 2-3 rounds.
   The assessor returns **`Escalate`**, naming the trigger (area + root cause, or cap-hit + what is still churning) and the decision the user must make (redesign vs documented risk-acceptance; or for a cap-hit: rework the plan, accept the residual explicitly, or authorize more rounds knowingly). Repeated irreversible findings or an exhausted budget **stop the loop and go to the user**, they do not silently license another round.
4. **Bank cold-verified areas (so the loop can end).** An area whose One-way/Significant was fixed and then survived one **fully-cold** round with no new finding in it is **settled**, it stops gating. A genuinely *new, different* finding in that area later is graded fresh for *gating* purposes, **but the area's One-way count for the escalation test persists across banking**: banking stops an area from gating, it does not erase its history. (Without this, a hard subsystem oscillates forever: fix, clean round, bank, new One-way with the counter reset, repeat, a real run's identity cluster did exactly that, clearing twice and re-arming twice across nine rounds. The second One-way in an area escalates whether or not the area was banked in between.)
5. **On convergence, produce the test-obligation list**: every Medium and Minor from the run, each with the behavior a test must pin and the area it belongs to. Reversible findings are closed by code + tests, not by more rounds.

The assessor returns: the verdict (`Another round`, `Converged`, or `Escalate`), the round's new-unique-gated-findings count, any escalation (trigger + root cause + the decision for the user), the areas banked settled this round, and (if converged) the test-obligation list. It is cold (made no fixes) and is the only role permitted to read the whole log. On `Escalate`, **you stop the loop** and put that decision to the user per the trade-off rule, you do not run another point-fix round hoping the area settles itself.

### Tier -> behavior

- **One-way:** always another round until settled; never deferred (you cannot push an irreversible decision to "fix in code later"). But a **second** One-way in the same area, or one that will not stay settled across two rounds, is no longer "just another round", the assessor returns **`Escalate`** and the loop stops for a user architecture decision (assessor item 3). A One-way fixed once and then clean through a fully-cold round is settled (item 4) and stops gating, though its area keeps its escalation count.
- **Significant:** always another round (big enough to verify in the plan), then settled once a fully-cold round finds nothing new in that area (item 4). Never deferred. A **third** Significant in one area across the run is an `Escalate` (item 3): the still-open behaviors get pinned with test obligations and the user chooses the simplification; the loop does not re-fix the area in prose a fourth time. And **simplification means deletion**: the replacement must have strictly fewer moving parts than what it replaces — a "simplification" that introduces a new subsystem, job, guard, or abstraction is more machinery wearing the word (one audited run's pin-and-simplify produced a bucket-replay simulation, which broke on first cold contact). First question when an area recurs: did its machinery originate in a *review round* rather than the plan's original scope? Then the default fix is deleting it back to the pre-review shape.
- **Medium:** does **not** gate convergence. Fix it in the plan if the fix is cheap and local, and add it to the **test obligations** regardless, a reversible correctness bug is closed by a deterministic test at implementation, not by looping the plan. A round whose only findings are Mediums converges.
- **Minor:** does not gate. Fix if trivial, otherwise note it; it also goes on the test-obligation / backlog list. Never forces a round.
- **Not-an-issue:** does not gate, is not fixed, is not a test obligation. Sidecar record only.

**Mediums and Minors carry no recurrence count and no K threshold**, they never extend the loop, no matter how many accumulate or recur (a per-area Medium count was an endless-loop vector: a stream of new Mediums in fresh areas never reaches a threshold, so the loop never converges; reversible findings belong in tests, not in another prose round). The **only** counted things are One-way recurrence per area, Significant recurrence per area, and the round number, all of which trigger `Escalate`, a hard stop to the user, never another automatic round.

### Convergence

A plan is **converged** on a round with **no open One-way and no Significant**, only Mediums, Minors, and Not-an-issues remain, and those do not gate (Mediums/Minors become test obligations). A round that made any One-way or Significant change is never the last round: that change must be verified by a clean cold pass (which, if clean, also banks the area as settled per assessor item 4). A dry round (no new unique gated findings) that is also clean of open One-way/Significant is the ideal convergence: the loop ended because the finding stream ran dry, not because anyone got tired.

The whole loop is bounded: expect convergence in **2-3 rounds** (that is where iterative review's gains concentrate); the assessor escalates at **round 5** if the loop is still open (its item 3). There is no silent round 6. In interactive runs, a loop still open after round 3 first pauses for the design checkpoint below before spending rounds 4-5.

State the assessor's verdict explicitly each round: `Another round` (with what was fixed and what is still live), `Converged`, or `Escalate` (with the trigger, root cause, and the decision for the user). On `Escalate`, stop the loop and put that decision to the user per the trade-off rule, do not run another point-fix round in the hope the area settles itself.

This is an internal loop with a deterministic exit condition, **not** the Claude Code `/loop` or `/goal` primitives, and it does not use them. `/loop` is for time-spaced recurring tasks; `/goal` is a session-level model evaluator. Here the assessor owns the exit decision directly, and cold-start holds across rounds because each round spawns fresh reviewer sub-agents fed only the plan, with the sidecar withheld.

## The round-3 design checkpoint (interactive runs)

If the assessor returns `Another round` at the end of **round 3**, do not spawn round 4 yet. The first three rounds are where iterative review's gains live; a loop that still has gated findings after them is usually churning on a *design* problem — something over-complicated whose consequences the reviewers keep re-finding — and one plain-language conversation resolves that faster than two more point-fix rounds.

Pause and run a `/discuss-plan`-style design checkpoint with the user:

1. **Distill the pattern, not the findings list.** Read the sidecar (you are its writer; this breaks no reviewer cold-start) and work out which areas keep producing findings, what single design choice sits underneath them, and what a simpler shape would look like. Present it in plain language per the discuss-plan conventions: short, jargon glossed, one question at a time.
2. **Put the simplification question directly:** "rounds 1-3 keep circling [area]; the root looks like [design choice]. Options: [simpler alternative] / [cut the scope] / [keep it and continue reviewing]." Genuine choices only — if round 3 was nearly dry and round 4 is plain mop-up, say exactly that and continue the loop without a discussion; the checkpoint is a circuit-breaker, not a ritual.
3. **Record and apply.** Decisions go into the plan's own `## Architecture Decisions` section (update superseded ADs in place, surgically, written as if the plan always said so per the provenance rule — later rounds then treat them as settled spec), and the checkpoint itself is logged in the sidecar.
4. **Resume at round 4 on the same budget** — the round-5 escalation cap still stands. If the redesign was substantial enough that the plan is effectively new, say so; the user can order a fresh review pass on the reshaped plan (a fresh budget, per the re-invocation rule) instead of spending rounds 4-5 re-reviewing a rewrite.

**Self-check before spawning round 4** (same force as the grader self-check): the sidecar must contain a round-3 checkpoint entry — either the discussion's outcome, or the explicit "round 3 was nearly dry, checkpoint skipped" note. If neither exists, you skipped the checkpoint; stop and run it now.

**Any mid-loop design pivot goes through `/research` before the loop resumes** — whether it comes from this checkpoint or from a user-initiated `/discuss-plan` interlude. An architecture swap adopted mid-loop touches the same One-way categories as any planned decision, and adopting one on discussion alone is how one audited run replaced a vendor's documented webhook mechanism with a hand-built reconciler that then churned for three rounds before anyone read the vendor's docs. Research it at adoption (official docs first, per `/research`), record the decision and citation in the plan's Architecture Decisions section, then resume.

**Unattended runs skip the checkpoint** — there is no user to discuss with. Under `overnight-delivery`, the loop proceeds straight to rounds 4-5 and the existing `Escalate` path carries any design question into the blocking tradeoff gate instead.

## On convergence: write obligations into the plan, then the sidecar

When the assessor returns `Converged`:

1. **Write the test obligations into `implementation-plan.md`** (you are the only writer). Append a consolidated `## Test Obligations` section listing every Medium and Minor from the run (the behavior a test must pin and its area), **and** add a reference in each relevant phase to the obligations that phase must fulfil, so the test lands in the same phase as the code. This is the last edit before the plan is marked `Reviewed`; it is finalising the plan, not editing a frozen one. The obligations live in the plan (forward-looking spec), not the sidecar; a cold reviewer seeing them on a later re-invocation is not primed (spec, not an approval stamp).
2. **Set the plan's `Status` to `Reviewed`** (a one-word header stamp is fine; detailed history stays in the sidecar).

Every round (converged or not), append a numbered entry to the **sidecar** at `context/[Feature]/review-log.md` (create it if absent). **Append-only, in round order:** before writing, confirm the new entry's round number and its position in the file both come after every existing round — never insert an entry above one already written (one real sidecar carries Round 7 physically before Round 6, corrupting exactly the recurrence history a later assessor reads in file order). When the user answers an `Escalate` with more rounds, keep the absolute numbering counting upward (round 6, 7, …) and note the fresh budget in the heading; never restart from 1. Never add review history to the plan. Structure:

```markdown
# [Feature]: Review Log

> Sidecar for `implementation-plan.md`. Review history lives here, never in the plan, so cold reviewers can't be primed by prior rounds. For the author and user only.

## Round [N], [date]

### Findings (graded)
- [Each finding: tier, area, one-line reason, and how it was fixed, deferred to a test, or refuted (Not-an-issue, with the refuting evidence). Note any finding dropped at the quote check.]

### Assessor verdict
- Converged / Another round / Escalate. [If converged, the test-obligation list. If another round, which One-way/Significant items are still live. If escalate, the trigger (area recurrence or round cap), root cause, and the decision put to the user.]
- New unique gated findings this round: [N] (the dry signal; 0 across a clean round is the strongest convergence evidence) · Self-inflicted (against text added in the prior two rounds): [N]
- Areas banked settled this round: [list, or none] · Per-area One-way counts so far: [area: N, ...] · Per-area Significant counts so far: [area: N, ...]
```

The verdict block is the assessor's memory across rounds (each round's assessor is a fresh spawn reading this file), so the banked areas, per-area One-way counts, and dry-signal counts **must** be recorded every round, they are state, not commentary.

## Re-invocation

This workflow runs review rounds in an internal loop until the assessor returns `Converged` or `Escalate` (the round-5 cap is one of Escalate's triggers, so the loop cannot run silently past its budget). If the user later asks for another pass on an already-converged plan, run it; never argue that prior reviews should be sufficient or propose skipping ahead. A user-requested extra pass starts a fresh budget. Each round appends a numbered entry to the sidecar.
