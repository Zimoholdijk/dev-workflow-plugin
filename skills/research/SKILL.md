---
name: research
description: Spin up a research sub-agent to answer a technical question or find best practices, grounded in the repo, the plans, and the current discussion. For any named technology (Supabase, Prisma, a framework, a library), it reads the official documentation first, then corroborates with reputable engineering sources. Returns a cited, recommendation-first answer checked against the project's actual stack and versions. Takes the question as argument.
disable-model-invocation: false
argument-hint: "[the question, e.g. 'best way to paginate a large feed in Supabase' or 'debounce vs throttle for this input']"
---

# Research a Question

You are researching, on behalf of the current work, the question: $ARGUMENTS

This skill spins up a research sub-agent to answer a technical question or find the best-practice approach to something, then critically synthesizes its findings for the user. The answer must be grounded in three things: the **official documentation** of whatever technology is involved, **reputable secondary sources** for real-world tradeoffs, and the **project's own context** (its stack, its plans, the question as it actually came up). It is for answering questions that need an authoritative, current answer, not a guess from memory.

## When to use this

- A genuine "what's the right way to do this?" question surfaced during `/discuss-feature`, `/write-plan`, `/plan-review`, or implementation.
- You need to verify whether a pattern is idiomatic for a specific tool (this is the framework-idiom check those skills call for, done properly with sources).
- You're weighing two approaches and want the tradeoffs documented from credible sources, not asserted.

For a broad, multi-source, fact-checked report on an open-ended topic, use the heavier `deep-research` skill instead. This skill is the focused, repo-grounded version: one or a few specific questions, answered against the docs and the project.

## Rules

These are non-negotiable:

1. **Documentation first, always.** When the question involves any specific technology, the first pass is that technology's *official* documentation, not a search engine, not a blog, not memory. Only after the docs are exhausted do secondary sources come in. A best-practice claim that contradicts the official docs is wrong until proven otherwise.
2. **Do not answer from memory.** Model training is stale and these questions are version-sensitive. Every substantive claim must trace to a source read during this research. If a sub-agent returns an answer with no sources, send it back.
3. **Ground the answer in the actual project.** The recommendation must fit the project's real stack and *installed versions* (check `package.json`, `deno.json`, lockfiles, `prisma/schema.prisma`, etc.). A pattern the project's version doesn't support is not an answer. Tie findings back to the specific plan or code under discussion.
4. **No persona posturing in the sub-agent prompt.** Do not frame the research agent with an assumed identity ("you are a senior engineer", "act as an expert"). Persona framing does not improve research quality and can degrade it. State the task, the sources to use, and the quality bar directly. Judge the work by its sources, not by a role it was told to play.
5. **Weight sources by credibility, and say so.** Official docs and maintainer writing outrank well-regarded engineering blogs, which outrank random tutorials and unattributed forum answers. Cite every source and note its tier. A lone Stack Overflow answer is a lead to verify, not a conclusion.
6. **Surface conflicts, don't paper over them.** Where reputable sources disagree, or where best practice conflicts with the current plan, present the disagreement and let the user decide. Do not silently pick a side.

## Step 1: Frame the question and gather context

Before spawning anything, pin down what's actually being asked and assemble what the sub-agent needs (it has no prior knowledge of this project):

1. **State the question precisely.** If the user's phrasing is broad, narrow it to the decision actually in front of you. Split a compound question into separate questions if they'd be researched differently.
2. **Identify the technologies and their versions.** Read `package.json` / `deno.json` / lockfiles / `Cargo.toml` / `prisma/schema.prisma` to learn exactly which tools and which versions are in play. Name them explicitly; "Supabase" alone is not enough, note the client/library versions and whether it's RLS, Auth, Storage, Edge Functions, etc.
3. **Read the project context** the answer must fit: `context/overview.md`, `.claude/CLAUDE.md`, `~/.claude/CLAUDE.md`, and the specific PRD / implementation plan / `progress.md` the question arose from.
4. **Identify the relevant code.** Point the sub-agent at the specific files, schema, or plan section the question is about, so its answer is concrete, not generic.
5. **Note any available documentation tools.** If an MCP server for the technology is connected (e.g. a Supabase MCP with a `search_docs` tool, or a docs-search MCP), the sub-agent should use it for the docs-first pass. Tell it which tools exist.

## Step 2: Spawn the research sub-agent

Spawn a `general-purpose` agent (it has web search/fetch and can reach connected MCP doc tools). For multiple distinct questions, spawn one agent per question in a single message so they run in parallel.

Pass it the question, the technologies and versions, the project context from Step 1, and the prompt below. Substitute the bracketed parts; do not add persona framing.

> Your task is to research and answer this question so it can be acted on in a real codebase: **[question]**.
>
> **Context you must work within** (this is the project; you have no other knowledge of it):
> - Technologies and versions in play: [list, with versions]
> - Project stack and architecture: [from overview.md]
> - Project rules and conventions: [relevant CLAUDE.md excerpts]
> - The plan / code this question is about: [the specific plan section or files]
> - Documentation tools available to you: [e.g. "the Supabase MCP `search_docs` tool", or "none, use web search and fetch"]
>
> Answer from sources you read during this task, not from prior knowledge, training data on these tools is stale and these answers are version-sensitive. This prompt deliberately does not assign you a role or persona; answer from the evidence, not from an assumed identity.
>
> **Research in this order:**
>
> 1. **Official documentation first.** Go to the canonical docs for the specific technology (e.g. `supabase.com/docs`, `prisma.io/docs`, `react.dev`, the library's own docs site or GitHub README). If a documentation-search tool is available, use it here. Read the *actual current docs*, and prefer the page that matches the project's installed version. Establish what the tool officially supports and recommends before looking anywhere else.
> 2. **The tool's own source, changelog, and issue tracker** when the docs are ambiguous or silent. Release notes, maintainer answers in GitHub issues/discussions, and the source itself are high-signal for "is this supported and idiomatic in this version?"
> 3. **Reputable secondary sources** for real-world tradeoffs, pitfalls, and patterns the docs omit: well-regarded engineering blogs and newsletters (quality Substacks, company engineering blogs, maintainers' personal writing), and conference talks. Weight by author credibility and recency. Prefer recent, dated material from authors with a track record.
> 4. **Treat with skepticism** and do not rely on alone: SEO content farms, undated tutorials, AI-generated listicles, and unattributed forum answers. A single Stack Overflow answer is a lead to verify against the docs, not a conclusion.
>
> **Cross-check:** where a secondary source conflicts with the official docs on what is supported or idiomatic, the docs win, note the conflict. Secondary sources are for tradeoffs and gotchas, not for overriding the documented behavior. Where reputable sources genuinely disagree on a judgment call, present both positions rather than picking one silently.
>
> **Ground every recommendation in the project's actual versions and constraints given above.** Do not recommend a pattern the project's version does not support, or one that violates its stated conventions, flag it if the idiomatic approach conflicts with a project rule.
>
> **Output format:**
> 1. **Recommendation** (lead with it): the concrete answer for *this* project, in 2-4 sentences.
> 2. **Why:** the reasoning, citing the documentation that supports it.
> 3. **Tradeoffs / alternatives:** other viable approaches and when they'd be better; pitfalls to avoid.
> 4. **Fit check:** how this lands against the project's stack, versions, and rules (any conflicts).
> 5. **Sources:** every source as a URL, each tagged with its tier (official docs / maintainer / reputable blog / other) and one line on what it supports.
> 6. **Confidence and open questions:** how settled this is, and anything you could not verify.

## Step 3: Assess and synthesize

The sub-agent is advisory, not authoritative. When it returns:

- **Check its sources.** Did the docs-first pass actually happen, or did it lead with a blog? Are the cited URLs real and on-point? If the answer rests on weak sources or none, send it back or re-run with a sharper prompt.
- **Re-ground it yourself** against the project's versions and rules if the sub-agent's fit check is thin.
- **Reconcile conflicts** it surfaced. If best practice conflicts with the current plan or a project rule, that's a trade-off for the user, surface it (one at a time, per the workflow's trade-off rule); don't resolve it silently.

## Step 4: Present to the user

Give the user:

1. The **recommendation**, grounded in their project.
2. The **key tradeoffs** and any place where best practice conflicts with the current plan (as a decision for them, not a decision you made).
3. The **sources**, so they can verify, leading with the official documentation.
4. **What this means for the work in progress**: a concrete next step, an edit to the plan, a note for `progress.md`, or a flagged trade-off for `/tradeoff-review`.

If the research was prompted by a plan or PRD discussion, offer to fold the conclusion into that document (as prose, per the no-code-in-plans rule) rather than leaving it only in chat.

## Notes

- This skill pairs with the framework-idiom checks in `/plan-review` and `/full-code-review`: when a reviewer flags "is this pattern idiomatic?", `/research` is how you answer it with sources instead of asserting.
- Keep the question specific. "How does auth work" is a reading task; "does this Supabase version support `getClaims()` for JWT verification, and is it preferred over `getUser()` here?" is a research question this skill answers well.
- Persona-free prompting is deliberate (Rule 4). If you adapt this skill, do not add "act as an expert" framing back in.
