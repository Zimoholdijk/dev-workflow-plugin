# Plan-Review Convergence: Design Spec

> Status: **Signed off and encoded** (plan-review, the grader and assessor agents,
> implement-plan, and overnight-delivery). This is the reference for how `plan-review`
> decides when a plan is done, and how `implement-plan` receives what review deferred.

## 1. The problem this solves

Iterative cold review (junior, senior, red-team, re-run each round) is excellent at
*finding* issues but had no principled *stop*. On a real plan (artvintory CaptureCore)
it ran 11 rounds: the architecture survived from round 7 on, but a single tightly
coupled subsystem (the upload/reconcile concurrency state machine) kept producing a
new correctness bug every round, often one introduced by the previous round's own fix.
A cold reader will always find "the next seam" in non-deterministic code, so prose
review of that area never converges. Those bugs are exactly the kind that deterministic
tests catch and prose review cannot.

Two failures to fix:
1. **No stop rule.** "Critical/High" is a vibe; reviewers always find something, so the
   loop never ends on its own.
2. **The grey middle.** The extremes were always easy (irreversible decisions must be
   settled; nits get ignored). The hard part is the reversible-but-real bug, and the
   pattern of one area churning round after round. The old design had no home for it.

## 2. Core principle

Grade a finding by **reversibility and blast radius/magnitude, not by "is it a bug."**

- Irreversible or expensive-and-broad to undo (a **one-way door**) must be settled in the
  plan before building.
- Reversible things are closed by **code plus a test obligation**, not by more review rounds.
- **Convergence is the signal that the plan is stable, never the target.** Optimizing for
  the stop corrupts it.

## 3. Four roles (separation of powers)

Each role is cold where it needs to be, so no one grades or stops their own work.

| Role | Job | Cold to |
|------|-----|---------|
| **Reviewers** (junior, senior, red-team) | Find issues. Run sequentially, each on the current plan. | The review history. They never see the `review-log.md` sidecar, so a late round scrutinizes as hard as the first. |
| **Grader** | After each reviewer, grade every one of that reviewer's findings into a tier, and tag each with an area/topic label. | The cost of fixing. It rates by reversibility/significance, not by how annoying the fix is. It does **not** decide fix-vs-defer. |
| **Orchestrator** | Fix everything the graders surfaced, regardless of tier. Run the inline self-consistency pass. Write the round to the sidecar. | Severity and stopping. It cannot grade, and it cannot decide convergence. |
| **Assessor** | Runs **every round**. The only agent holding the full log. Makes the converge/another-round call (One-way and Significant gate; Medium and Minor become test obligations), raises the design-unstable flag when One-way/Significant findings recur in one area, and compiles the test-obligation list. | Bias toward finishing. It did not make the fixes, so it has no stake in declaring done. It is history-aware by design (that is its purpose), unlike the reviewers. |

The orchestrator can **escalate** a grade (treat a lower tier as higher) or record a
disagreement for the user, but can **never silently downgrade** a finding to make it stop
mattering. Unblocking only ever happens by fixing or by the grader's grade, never by the
orchestrator's wish.

## 4. The severity rubric (what the grader applies)

**The reversibility test (the spine):** *"Can I change this and every dependant in one
atomic deploy?"* If yes, it is reversible (two-way door). If no (there are consumers you
cannot update in lockstep), it is a one-way door.

### 4.1 The four tiers

- **One-way door** (irreversible): touches a trigger-list category (4.2). Must be settled
  in the plan.
- **Significant** (reversible but large): big blast radius, OR large magnitude (e.g. a
  600-line file split), OR touches important parts.
- **Medium** (reversible, modest size, but a real correctness/behavior defect): data loss,
  wrong state, a race, an infinite loop, a broken flow. Not big, not cosmetic.
- **Minor** (cosmetic): clarity, naming, small-local-low-stakes.

Every grade carries a **one-line reason** (which trigger category, or which significance
driver) so it is auditable, not vibes.

### 4.2 One-way-door trigger list (stack-agnostic reference)

A finding touching any of these is a one-way door **by default**:

1. **A published contract** any party you cannot atomically update depends on (a
   versioned/public API, wire or RPC protocol, webhook payload, a library's public
   surface, a CLI's flags/output).
2. **Persisted data, once real production data exists** (schema, stored formats, enum raw
   values, the meaning/format of IDs and keys). Data outlives code.
3. **Event/message schemas on an append-only or async channel** (once published to a
   log/queue/stream you cannot recall it; consumers and replays depend on it).
4. **Security and trust posture** (the authorization model, identity/tenancy model, where
   trust boundaries sit, secret/credential handling).
5. **Publicly observable behavior that becomes a de-facto contract** (URL/permalink
   structure, externally visible IDs, error shapes, anything third parties script against).
6. **Foundational platform commitments with broad lock-in** (primary datastore,
   language/runtime, cloud-proprietary primitives).
7. **Distributed-correctness commitments** (consistency model, ordering/idempotency/
   delivery semantics, the partition/sharding key).

**Not a category:** module/service boundaries. The irreversible kind is already item 1
(a published contract); the rest is a reversible refactor, handled by blast radius and
magnitude under Significant.

### 4.3 Nuances

- **Production-data input:** item 2 is a one-way door **only if production data already
  exists**. Pre-launch / empty store, schema changes are reversible (grade lower). The
  grader establishes this from project context; if it cannot tell, it **assumes data
  exists** (the safe direction).
- **Asymmetric default:** when reversibility is genuinely uncertain, grade it **one-way
  door**. Mislabeling a one-way door as reversible costs a migration project; the reverse
  costs a little extra deliberation.
- **Downgrade guard:** a one-way door drops to two-way **only if** the plan names a real,
  in-use mechanism that bounds the blast radius (API versioning + deprecation window,
  expand/contract migration, consumer-driven contract tests) **and** states the migration
  path. Data or events already written stay irreversible even then.
- **Area tag:** the grader labels each finding with an area/topic (e.g. "upload/reconcile
  state machine", "share-token contract") so the test-obligation list and the
  design-unstable flag can name a consistent area. Use stable labels across rounds.

## 5. The per-round flow

1. **Junior** reviews (cold) -> **grader** grades + tags the junior's findings ->
   **orchestrator** fixes them, then runs the **self-consistency pass**.
2. **Senior** reviews (cold) -> grader grades + tags -> orchestrator fixes + self-consistency.
3. **Red-team** reviews (cold) -> grader grades + tags -> orchestrator fixes + self-consistency.
4. **Quality and conformance pass** (orchestrator): a self-check (plan prose, not a code
   tool) for repetition smell, test coverage, and CLAUDE.md conformance; anything
   substantive is routed through the grader like a reviewer finding. (`/simplify` belongs
   on code, not the plan, so it lives in `full-code-review` and `implement-plan`.)
5. **Assessor** runs (every round): reads the full log, applies the tier -> behavior rules
   (section 6), decides converge or another round, and on convergence compiles the
   test-obligation list.

The assessor runs every round because it owns the convergence decision and is the only
holder of the full log, so it is also the place that notices whether One-way/Significant
findings keep recurring in one area (the "design unstable, needs rework" signal).

## 6. Tier -> behavior (the gate logic, owned by the assessor)

Only **One-way and Significant** findings gate convergence. Medium and Minor do not.

- **One-way door:** **always another round** until settled. Never deferred (you cannot
  push an irreversible decision to "fix in code later"). If it will not stay settled across
  rounds, the assessor raises an advisory **"design unstable, needs rework"** flag, but
  still never auto-defers it.
- **Significant:** **always another round** (the change is big enough to verify in the
  plan). Never deferred. Self-limiting in practice; persistent recurrence in one area
  raises the same advisory flag.
- **Medium:** **does not gate convergence.** Fix it in the plan if the fix is cheap and
  local, and add it to the **test obligations** regardless. A reversible correctness bug is
  closed by a deterministic test at implementation, not by looping the plan. A round whose
  only findings are Mediums converges.
- **Minor:** does not gate. Fix if trivial, otherwise note it; it also goes on the
  obligation/backlog list.

**No recurrence count and no K threshold.** An earlier design counted Medium recurrence
per area and deferred at K=3, but that was an **endless-loop vector**: convergence required
"no live Medium below K=3", and a stream of *new* Mediums in fresh areas never reaches the
threshold, so a thorough cold reviewer that finds one more reversible bug each round keeps
the loop alive forever. Reversible findings belong in tests, not in another prose round, so
they simply do not gate.

## 6b. Who decides: trade-offs by tier

The tier governs not only convergence but **who owns the decision**:

- **Medium / Minor**: the orchestrator fixes autonomously. They are reversible and low-stakes; asking about them is noise.
- **One-way / Significant**: the user's call when they involve a genuine choice. Before deciding one, the orchestrator runs `/research` if it has a technical dimension, then routes: if the evidence settles it, apply the documented answer and tell the user (an irreversible call is surfaced even when clear); if it is a genuine choice (defensible either way, depends on product/UX/risk), STOP and ask the user, one at a time, **at the moment it is reached**, not batched at the end.

This is the long-standing "ask me about the things that need my input" rule, now tied explicitly to the rubric: the things that need input are the irreversible and consequential ones (One-way and Significant), which is exactly what the grader already identifies.

**Unattended exception:** inside an unattended pipeline (e.g. overnight-delivery), the loop cannot stop per answer; it applies evidence-resolved calls in-loop and accumulates the genuine One-way/Significant choices for that pipeline's trade-off gate.

## 7. Convergence definition

A plan is **converged** on a round that has **no open one-way door and no Significant
change**. Mediums and Minors may remain; they do not gate (they become test obligations).
The assessor then emits the **test-obligation list**, the orchestrator writes it into the
plan (section 9), and the plan is marked Reviewed. A round that made any one-way or
significant change is never the last round: that change must be verified by a clean cold
pass first.

## 8. Self-consistency pass (inline, orchestrator)

Runs **after each stage's fixes**, before the next cold lens (so up to 3 times per round).
Not a separate agent: the very next thing each pass precedes is a fresh cold reviewer,
which is the real un-biased catch; the inline pass just catches the obvious self-
contradiction one step earlier and cheaper. After the red-team stage (no lens follows it
that round) the next round's cold junior is the backstop.

The orchestrator re-reads each section it just edited against:
- the other phases,
- the File Changes table,
- the Architecture Decisions,
- any invariant stated elsewhere in the plan,

fixes the contradictions it finds, and re-checks those follow-on fixes.

## 9. Handoff to `implement-plan`: write obligations into the plan

The test-obligation list is not a loose handoff. It is **written into the implementation
plan itself at convergence**, before `implement-plan` is ever invoked. The assessor
produces the list (every Medium and Minor from the run); the
orchestrator (the only writer) then does two things to `implementation-plan.md`:

1. **Appends a consolidated `## Test Obligations` section** listing every Medium and
   Minor from the run: the behavior that must be pinned by a test and the area it belongs to.
2. **Adds a reference in each relevant phase** pointing to the obligations that phase must
   fulfil, so the test lands in the same phase as the code, matching `implement-plan`'s
   "tests ship with the code, in the same phase" rule.

This is the **last edit before the plan is marked Reviewed and frozen**, so it is part of
finalising the plan, not editing a frozen one (the freeze happens immediately after).

The obligations live in the **plan**, not the `review-log.md` sidecar: they are a
forward-looking spec for the implementer, not review history. A cold reviewer who sees the
Test Obligations section on a later re-invocation is not primed by it (it is spec, not an
approval stamp), so it does not violate cold-start.

`implement-plan` then reads each phase's referenced obligations and writes those tests as
it builds that phase (its existing "tests in the same phase" rule enforces it). It also
gets its own analogue of the self-consistency pass: before closing a phase, re-verify that
an applied fix's non-local effects did not break a caller or a sibling change.

## 10. Terminology

Use **"converged" / "exit condition"** throughout. Do **not** call it a "goal", `/goal` is
a reserved Claude Code session-level primitive and this is a skill-internal loop with a
deterministic exit condition.

## 11. What changes in the skills (encoding plan)

- **plan-review:** replace the current convergence section with sections 4-7 above. Add the
  grader (per-lens, tier + area tag) and the every-round assessor as defined roles; add the
  rubric and trigger list (section 4) as reference the grader is given. Keep the cold-start
  rule and the sidecar. Keep the self-consistency section (already drafted). On convergence,
  the orchestrator writes the test-obligation list into the plan, a consolidated section
  plus per-phase references (section 9), as the final edit before the plan is marked Reviewed.
- **agents:** add a `grader` and an `assessor` sub-agent definition (cold, with the rubric
  / log access as described).
- **implement-plan:** add test-obligation fulfilment (section 9) and the self-consistency
  re-check (already drafted).
- **overnight-delivery:** Stage 2 already says "run plan-review to convergence"; no change
  beyond it now meaning this convergence.

## 12. Open calibration knobs

- **"Production data exists?"** input for item 2, read from project context, default assume yes.
- **Area-tag granularity**, how finely the grader labels areas (too coarse hides churn, too
  fine never accumulates a count).

## 13. Provenance (why these choices)

- Reversibility as the severity spine: Bezos one-way/two-way doors; Fowler "architecture =
  the hard-to-change decisions"; Nygard ADRs ("architecturally significant").
- "Published contract" framing: Fowler public-vs-published interfaces; Hyrum's Law (all
  observable behavior becomes a depended-on contract); SemVer (a contract once consumers exist).
- Tiers via a decision table, never a multiplied score: FMEA RPN multiplication is
  mathematically invalid; AIAG-VDA replaced it with a severity-first Action Priority table.
- Deferral to implementation + tests: Last Responsible Moment, Real Options, set-based
  design; YAGNI and self-testing code as the safety net; SEI on concurrency defects evading
  review (they close in tests, not rounds).
- Stopping on saturation rather than a fixed count: defect-detection saturation and
  review-rate ceilings (Fagan, Wiegers, SmartBear/Cisco).
