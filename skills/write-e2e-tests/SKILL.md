---
name: write-e2e-tests
description: Write and run end-to-end browser tests for a feature using the bundled Playwright MCP. Drives a real browser to verify user-facing flows, then writes durable Playwright spec files so the behavior stays covered. Takes a feature name or flow description as argument.
disable-model-invocation: false
argument-hint: "[feature name or flow, e.g. 'claiming' or 'the listing creation flow']"
---

# Write End-to-End Tests

You are writing and running end-to-end browser tests for: $ARGUMENTS

This skill uses the **Playwright MCP** (bundled with this plugin, exposed as `mcp__playwright__*` browser tools) to drive a real browser, verify the flow actually works, and then capture that behavior as durable Playwright spec files. Live MCP exploration is how you confirm the test is correct before committing it; the spec file is the deliverable that keeps the behavior covered.

## Rules

These rules are non-negotiable:

1. **Every aspect of new code gets a test.** E2E tests cover the user-facing flow; they do not replace unit tests for logic. If the feature has units worth testing in isolation (helpers, reducers, validators, server utilities), say so and either write them or flag them as a gap. End-to-end coverage of a happy path is not "fully tested."
2. **Test against a running app, never a mock of it.** The user manages the dev server. Confirm the app is running and reachable before driving the browser. Do not start or restart dev servers yourself.
3. **Verify live before you commit a spec.** Use the Playwright MCP to walk the flow in a real browser first. Only write a spec assertion once you have observed the real behavior. Never write assertions against behavior you have not seen.
4. **Cover more than the happy path.** A flow is not tested until you have exercised the happy path, the obvious error states, empty/loading states, and the relevant auth boundaries (signed-out, wrong-owner, expired session). List which of these apply and cover each, or state explicitly why one does not apply.
5. **Tests must be deterministic.** No reliance on real wall-clock timing, network flakiness, or test-order coupling. Wait on application state (visible elements, URLs, network idle), not fixed sleeps. Each spec sets up and tears down its own data; tests must pass run in any order and in isolation.
6. **Surface gaps, don't paper over them.** If a flow cannot be tested deterministically (e.g. depends on a third-party redirect, a real payment, or unseeded data), do not write a flaky test that "mostly" passes. Surface the limitation to the user and propose how to make it testable (seed data, a test mode, a stubbed boundary).

## Step 1: Gather context

Before writing anything, read:

1. `context/overview.md`: project overview, tech stack, architecture
2. `.claude/CLAUDE.md`: project rules, and especially the **Testing Reality** section, what test infrastructure exists today
3. `~/.claude/CLAUDE.md`: global rules
4. The feature's PRD, implementation plan, and `progress.md` if they exist, the plan's **Testing Strategy** and per-phase **Testable** sections tell you what the acceptance flows are
5. Any existing tests (glob for `*.spec.ts`, `*.test.ts`, `e2e/`, `tests/`, `playwright.config.*`) to match the project's conventions, helpers, fixtures, and file placement

If the project has **no e2e setup at all** (no `playwright.config.*`, no test directory), do not silently invent one. Tell the user what you propose to add (a `playwright.config` with a `webServer` block so the suite starts and waits for the app, plus a `baseURL`; a `tests/e2e/` directory; a test script in `package.json`; and how the dev server / base URL is resolved) and confirm before scaffolding it. Reuse the project's existing browser, base URL, and auth-bootstrap conventions wherever they exist. For repeated sign-in, prefer Playwright's `storageState` (authenticate once, reuse the saved session) over logging in at the top of every spec.

## Step 2: Confirm the app is reachable

Ask the user for the base URL the dev server is running on if it is not documented in `overview.md` or `CLAUDE.md`. Then use the Playwright MCP to navigate to it and confirm the app loads. If it does not load, stop and tell the user, do not write tests against an app you cannot reach.

## Step 3: Walk the flow live with the Playwright MCP

For each flow under test, drive it in a real browser using the `mcp__playwright__*` tools before writing any spec:

1. Navigate to the entry point.
2. Perform the user actions in order (fill, click, select), taking an accessibility snapshot between steps to confirm the page is in the state you expect.
3. Observe the real outcome: the resulting URL, the visible text, the rendered state. This observed behavior is what your assertions will encode.
4. Repeat for each error state, empty state, and auth boundary that applies.

If a step does not behave as the plan or PRD claims, stop, that is a bug in the code under test, not a test to be written around it. Surface it to the user.

## Step 4: Write durable spec files

Translate each verified flow into a Playwright spec, following the project's existing conventions (file location, naming, fixtures, auth helpers). For each spec:

- Name the test for the behavior it proves, not the mechanics ("claims an available toy and sees it in My Claims", not "click test 3").
- Assert on user-visible state and application state, not implementation details.
- Use role- and text-based locators (`getByRole`, `getByText`, `getByLabel`) over brittle CSS/XPath selectors.
- Wait on state via Playwright's web-first, auto-retrying assertions (`expect(locator).toBeVisible()`, `toHaveURL`, etc.), never fixed timeouts or `waitForTimeout`.
- Set up and tear down the spec's own data so it is isolated and order-independent.
- Wrap multi-step flows in `test.step()` so a failure (and its trace) reads as a named sequence, not one opaque block.
- Don't test third-party sites or APIs you don't control. Intercept them with `page.route` and assert your app's behavior at the boundary. This is also how you make an otherwise-flaky external dependency deterministic (rule 6).
- Group related flows in one spec file per feature or page, matching how the project already organizes tests.

## Step 5: Run the suite and confirm green

Run the project's test command (from `package.json` scripts or `CLAUDE.md`) and confirm the new specs pass. Then run the **full** suite, not just your new file, to confirm you did not break or flake any existing test.

- If a new spec fails, fix the spec if the test is wrong, or surface the failure if the app is wrong. Never weaken an assertion just to make it pass.
- If an existing test starts failing because of timing or shared state your spec introduced, fix the isolation, do not delete or skip the other test.
- When a failure is hard to diagnose (especially one that only reproduces in CI), enable the Playwright **trace viewer** (`trace: 'on-first-retry'` in config) and read the trace instead of guessing.
- Report the actual command you ran and its actual result. If tests fail, say so with the output. Do not claim green you have not seen.

## Step 6: Report

Tell the user:

1. Which flows are now covered, by file.
2. Which states you exercised for each (happy path, error states, empty/loading, auth boundaries).
3. Any **unit-level** coverage gaps you noticed but did not fill (logic that deserves an isolated test), so they can be picked up.
4. Any flow you could not make deterministic, and your proposed path to making it testable.
5. The exact test command and its result.

## Notes

- E2E tests are the top of the pyramid: slow, high-confidence, few. They complement unit and integration tests, they do not substitute for them. Below them, integration tests (units working together) usually deserve the most coverage, with unit tests for isolated logic, the closer a test resembles real usage, the more confidence it buys. When a bug could be caught by a fast unit test, prefer that and reserve e2e for whole-flow confidence.
- This skill pairs with `/full-code-review`, whose Testing reviewer checks that new code has tests and runs the suite. Use this skill to *add* the coverage; that reviewer verifies it exists and passes.
- Lint specs with `eslint-plugin-playwright`. The single most common e2e failure is a missing `await` on an action or assertion, which the linter catches before it becomes flakiness.
- **Provisioning the browser.** The Playwright MCP (`npx @playwright/mcp@latest`) launches **headed** and uses your **system Chrome** by default, good for watching tests on a desktop. In a **headless remote/CI session there is no display and no system Chrome**, so the default fails. Add flags to the `mcpServers.playwright.args` in your `.mcp.json`: `--headless` and `--browser chromium` (and `--no-sandbox` if Chromium won't launch in a container). `--browser chromium` is what makes the MCP use the bundled Chromium under `PLAYWRIGHT_BROWSERS_PATH`; without it the MCP looks for system Chrome and errors with "Browser 'chrome' is not installed". Don't run `playwright install` where the bundled Chromium is already present; if the MCP still can't find a browser, point it at the binary with `--executable-path`. (Note: this provisioning applies to the MCP only. The durable specs in Step 5 run under the project's `@playwright/test`, which defaults to bundled Chromium and honors `PLAYWRIGHT_BROWSERS_PATH` on its own.)
