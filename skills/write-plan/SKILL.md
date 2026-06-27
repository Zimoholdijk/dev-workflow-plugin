---
name: write-plan
description: Draft an implementation plan for an approved PRD following the project's established format and workflow rules. Takes a feature name or PRD path as argument. Produces a phased implementation plan for review.
disable-model-invocation: false
argument-hint: "[feature name or path to PRD, e.g. 'claiming' or 'context/Claiming/ClaimingPRD.md']"
---

# Write Implementation Plan

You are drafting an implementation plan for: $ARGUMENTS

## Rules

These rules are non-negotiable:

1. **PRD must be approved first.** Do not write an implementation plan for a feature without an approved PRD. If the PRD status is "Draft", stop and tell the user.
2. **The plan describes *how*, not *what*.** The PRD covers what and why. The plan covers architecture decisions, schema changes, file changes, and phased implementation.
3. **No code in the plan.** Describe what to build in prose. No Prisma syntax, no TypeScript, no JSX, no code expressions. Implementation details like "use try/catch" or "return 500" are fine: code blocks are not. If you catch yourself writing backtick-wrapped code expressions (like `{ where: { status: "active" } }`), rewrite as prose.
4. **Every phase must be testable, by a human and by the test suite.** Each phase ends with a "Testable" section describing (a) what a human can verify in the browser or via curl, and (b) the automated tests that ship *with that phase*, unit tests for the logic it introduces, plus integration or end-to-end tests for the flow it exposes. A phase that adds logic without adding tests for that logic is not done. If a phase isn't testable, merge it with the next one.
9. **Testing is part of the plan, not an afterthought.** The plan must include a "Testing Strategy" section (see structure below) and every phase must name the tests it adds. Do not defer all testing to a final phase. New code is covered as it lands. If the project has no test infrastructure yet, the first phase establishes it (test runner, e2e harness, a first passing test) before feature work proceeds, and the plan says so explicitly.
5. **Pages before components.** Create the page shell first, then add interactive islands. This matches the Astro SSR-first architecture.
6. **Document freeze.** Once approved, the plan is frozen. Deviations during implementation go in `progress.md`.
7. **Surface all trade-offs.** Never accept trade-offs silently. Note known limitations, performance compromises, and security implications. Let the user decide.
8. **Structure smell-check before finalizing.** Before presenting the plan for review, re-read every phase looking for this shape: "for type A do X with separator Y1; for type B do X with separator Y2." When N branches differ only by a literal or small piece of metadata, surface "branched plan vs. single path with data difference" as an explicit Architecture Decision trade-off. A branched plan produces branched code; catching it in prose is cheaper than refactoring merged code.

## Step 1: Gather context

Before writing anything, read:

1. The approved PRD for this feature
2. `context/overview.md`: project overview, tech stack, existing features, shared utilities
3. `.claude/CLAUDE.md`: project rules
4. `~/.claude/CLAUDE.md`: global rules
5. Any existing feature docs that this feature depends on or interacts with
6. The current schema (`prisma/schema.prisma`)
7. Relevant existing code files that will be modified (middleware, routes, pages, components)

Understand the current state of the codebase before proposing changes. Check what already exists in shared utilities before creating new ones.

## Step 2: Draft the plan

Use this exact structure. Every section is required.

```markdown
# [Feature Name]: Implementation Plan

**Feature:** [TOY-XX] · **Status:** Draft · **Depends on:** [TOY-XX (done), ...]
**Branch:** `[branch-name]`
**PRD:** `context/[Feature]/[Feature]PRD.md`

---

## Context

[1-2 paragraphs. Only needed when the plan replaces a previous plan, or when
there's important context not in the PRD that affects implementation.
Omit this section if the PRD provides sufficient context.]

---

## Architecture Decisions

### AD-1: [Decision title]

[Prose explanation. What's the current state? What changes? Why this approach?
Include "Known limitation:" or "Known trade-off:" where relevant.
Each AD should be a decision that could reasonably go a different way.]

### AD-2: [Decision title]

[...]

[3-15 ADs typical. Number them sequentially. Each has a short descriptive title.
Cover: data model choices, API design, auth/middleware changes, component architecture,
state management, error handling strategy, performance considerations.]

---

## Schema Changes

**File:** `prisma/schema.prisma`

[Describe new models, new fields, new indexes, new enums in prose.
State whether the migration is breaking or non-breaking.
If no schema changes: "None. The existing schema supports everything."]

---

## File Changes

| File | Action | Phase |
|------|--------|-------|
| `path/to/file` | Create / Modify (brief description) | N |
| ... | ... | ... |

[Every file that will be created or modified, with the phase it happens in.
End with a "No changes needed to:" line listing files that might seem related but aren't touched.
Include test files explicitly, a feature that changes `foo.ts` and adds no `foo.test.ts`
(or e2e spec) is incomplete and the table should show why.]

---

## Testing Strategy

[Describe how this feature will be tested. Required, never "we'll test manually."
Cover, in prose:

- **Unit tests:** which units (helpers, validators, reducers, server utilities, pure
  logic) get isolated tests, and the key cases each must cover (happy path, boundaries,
  error inputs). Name the units, not just "add unit tests."
- **Integration / API tests:** which endpoints or module boundaries get tested against a
  real DB / real adjacent module, and which paths (auth boundaries, validation failures,
  error responses) must be exercised.
- **End-to-end tests:** which user-facing flows get a Playwright browser test (run
  `/write-e2e-tests` to author them). List the flows and the states each must cover:
  happy path, error states, empty/loading states, signed-out and wrong-owner boundaries.
- **Existing infrastructure:** what test runner, harness, fixtures, and seed/data-reset
  approach already exist (from `.claude/CLAUDE.md` "Testing Reality"). If none exist,
  state that Phase 1 establishes them before feature work.
- **What is deliberately not tested**, and why (e.g. third-party redirect, real payment),
  with the trade-off surfaced per rule 7.

Every row of the File Changes table that adds or changes logic should map to a test
named here.]

---

## Phased Implementation

### Phase 1: [Phase title]

**Goal:** [One sentence: what the user can do after this phase.]

[Describe changes. For larger phases, use sub-steps (1a, 1b, 1c).
Each sub-step names the file and describes what changes in prose.
Reference Architecture Decisions by number (AD-N) when relevant.]

**Testable:** [Two parts. (a) Manual: what a human can verify in the browser or via curl
after this phase. (b) Automated: the tests this phase adds, the unit tests for any logic
introduced and the integration/e2e test for the flow exposed, named specifically and
expected to pass at the end of the phase.]

---

### Phase 2: [Phase title]

[...]

---

[3-7 phases typical. Each phase builds on the previous.
Phase 1 is usually schema + API foundation.
Last phase is usually cleanup + redirects.]

## Verification (end-to-end)

[Numbered list of end-to-end scenarios that should work after all phases.
These are the acceptance tests. 8-15 items typical.
Cover: happy paths, edge cases, auth boundaries, error states.
Mark which scenarios are covered by an automated test (the Playwright e2e specs from the
Testing Strategy) versus verified manually. Aim for the critical flows to be automated, a
scenario that only ever gets a manual check will regress silently.]

---

## Review Log

[Left empty, populated by /plan-review runs.]
```

## Step 2.4: Phasing patterns (independent testability)

Every phase must produce output that can be verified without any later phase. If Phase 1 builds a hook whose only observable effect is a page built in Phase 3, the ordering is wrong.

Good phasing patterns for UI work:

- **Page shell before content.** Mount the route/page first (even if empty), then fill it in. Navigation is testable immediately.
- **Hook + display together.** A hook ships in the same phase as the smallest possible consumer that renders its output.
- **Foundation before polish.** Happy path first; empty, loading, and error states as a later polish phase.
- **Refactor before feature.** Extract the shared component in its own phase, verify no regression, then add the new consumer.

Bad phasing (do not do this):

- Phase 1 creates a hook, Phase 2 creates a component that uses it, Phase 3 creates the page that renders it. Nothing is testable until Phase 3.
- Phase 1 migrates the DB, Phase 2 updates types, Phase 3 updates queries. The migration alone has no user-visible test.

If a phase fails this check, fix it by one of: build the page shell first so the phase has somewhere to render, make the phase verifiable another way (a log, a dev-only surface, direct SQL or curl), or merge it into the phase that makes it observable. For DB/backend work, a phase is independently testable if it runs cleanly with only prior phases' migrations and the result can be verified via direct SQL or curl, no frontend required.

## Step 2.5: Structure smell-check

Before saving and presenting the plan, run this self-check:

1. Grep your own draft prose for near-identical bullets across phases or sub-steps. Look for descriptions that share verb + object structure and differ only by a literal, key, separator, or metadata value (e.g., "for textarea use \n\n; for free-text use ; for typed use ,").
2. If you find N parallel descriptions that only differ by data, you have two structural options:
   - **Branched:** N code paths, one per type.
   - **Unified:** One code path that reads the differing value from a small lookup or parameter.
3. Add this comparison as an explicit Architecture Decision with both options and a "Known trade-off:" line. Do not silently lock in the branched shape. Let the user pick during review.

A branched plan produces branched code. Catching the shape in prose is far cheaper than refactoring after merge.

## Step 3: File placement

Save the plan to: `context/[Feature]/implementation-plan.md`

The feature directory should already exist (created during PRD writing). If not, create it.

## Step 4: Present for review

After writing the plan, present it to the user with:

1. A count of architecture decisions and phases
2. Call out any trade-offs or known limitations that need user input
3. Note any deviations from the PRD (there shouldn't be any, but flag if the implementation reveals issues)
4. Summarize the Testing Strategy: what gets unit-tested, what gets an e2e test, and anything deliberately left untested (with the reason)
5. Suggest running `/plan-review` for the multi-stage review workflow

Do NOT proceed to implementation until the user explicitly approves the plan (after reviews). Approval means explicit language like "approved", "looks good, move on", or "start implementing". Silence, "ok", or "thanks" is not approval; if the signal is ambiguous, ask whether they're ready to move on.

## Patterns from existing plans

Two implementation plans exist in the project. Follow their established conventions:

### Header format
Both plans use the same header:
- Feature ticket + Status + Depends on (first line)
- Branch name (second line)
- PRD path (third line)

### Architecture Decisions
- Numbered AD-1 through AD-N with descriptive titles
- Prose format: current state → change → why
- "Known limitation:" or "Known trade-off:" inline when relevant
- Reference other ADs by number when they interact

### Schema Changes
- Name the file (`prisma/schema.prisma`)
- Describe changes in prose (new models, fields, indexes)
- State whether migration is breaking or non-breaking
- "None" with explanation if no schema changes needed

### File Changes table
- Three columns: File | Action | Phase
- Action column: "Create" or "Modify (brief description)"
- Sorted by phase number
- End with "No changes needed to:" line

### Phased Implementation
- Each phase has: title, Goal (one sentence), Changes (prose, optionally with sub-steps), Testable
- Sub-steps use format: "### 1a. [Title] (`file/path`)"
- Reference ADs by number: "per AD-3" or "(AD-7)"
- Testable section is imperative: "User clicks X → Y happens"
- Larger plans (Discovery) use sub-steps within phases; smaller plans (Listing) use paragraphs

### Verification
- Numbered list, 8-15 items
- End-to-end scenarios, not unit tests
- Cover: happy path, auth boundaries, error states, edge cases
- Format: "N. Action → expected result"

### Review Log
- Structured by review round
- Junior questions: count + key changes applied + questions noted but not applied
- Senior review: verdict + critical issues (applied) + suggestions (applied or noted)
- Quality check: conformance changes
- Each round clearly numbered

### Things to avoid
- No code blocks, Prisma syntax, or TypeScript in the plan
- No implementation details that belong in the code (exact prop shapes, CSS classes, etc.)
- No phases that aren't browser-testable
- No phase that adds logic without naming the tests it adds for that logic
- No plan without a Testing Strategy section
- No file changes without a phase assignment
