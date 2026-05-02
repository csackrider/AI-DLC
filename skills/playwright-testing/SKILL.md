---
name: playwright-testing
description: Author Playwright E2E tests in TypeScript with Page Objects, fixtures, and optional Gherkin (Cucumber/BDD) so issues written as acceptance criteria map cleanly to automation. Use when creating, extending, or refactoring Playwright suites, wiring CI, or translating Gherkin into executable tests.
type: skill
aidlc_phases: [build, test, review]
tags: [playwright, e2e, cucumber, gherkin, bdd, page-objects, pom, fixtures, smoke-tests]
requires: [testing]
created_at: 2026-05-02
updated_at: 2026-05-02
---

# Playwright testing (TypeScript + optional Gherkin)

Official tool docs: [playwright-bdd documentation](https://vitalets.github.io/playwright-bdd/#/) and [playwright-bdd-example](https://github.com/vitalets/playwright-bdd-example) (minimal runnable layout). Pin `playwright-bdd` and `@playwright/test` to compatible versions per those sources when scaffolding.

## When to use

- Scaffolding or extending a Playwright suite for a web app (especially Next.js / App Router).
- Translating **acceptance criteria in Gherkin** (GitHub issues, Product Specs, or `.feature` files) into runnable tests.
- Aligning **smoke vs full** E2E runs in CI, auth fixtures, and Page Object Model (POM) layout.

For the general test pyramid, unit vs integration tradeoffs, and flakiness patterns, rely on the composed **`testing`** skill (`requires` above).

## Recommended directory layout

Use a layout like the following (paths are conventional; rename the root if the repo prefers `e2e/` or similar):

- **`tests/specs/`** — Playwright spec files (group by feature; tag smoke flows).
- **`tests/pages/`** — Page Object classes (locators, actions, assertions).
- **`tests/fixtures/`** — Extended `test` fixtures (e.g. authenticated `page`).
- **`tests/helpers/`** — Test data builders and shared helpers.
- **`tests/.env.test.example`** — Document required env vars; never commit secrets.
- **`playwright.config.ts`** — `baseURL`, reporters, traces/screenshots, **multiple `projects`** (desktop + mobile emulation, separate auth project if useful).
- **CI** — Unit job first, then Playwright; upload HTML report and failure artifacts (screenshots, traces).

Adapt paths if the consumer repo uses a different root (e.g. `e2e/` instead of `tests/`); keep the **separation of specs / page objects / fixtures** consistent.

## Agent workflow

1. **Discover** — Routes, public vs authenticated pages, forms, critical journeys, data setup (`db:seed`, API helpers, etc.).
2. **Source of truth** — If the user supplied Gherkin (issue body or `.feature`), treat each **Scenario** as one or more tests; keep step text stable so re-runs do not churn.
3. **Page objects first** — For each primary route or flow surface, add or extend a Page class: locators, user actions, **assertion helpers** (see conventions below).
4. **Fixtures** — Add an auth fixture (or per-role fixtures) before writing logged-in scenarios.
5. **Implement** — Plain `*.spec.ts` and/or Gherkin-backed tests (see Cucumber section); map **@tags** to CI `grep` / projects.
6. **Verify** — Run locally with the same `BASE_URL` and env contract as CI; fix flakiness with Playwright auto-waits, not fixed sleeps.

## Conventions (POM and runners)

- **Assertions** — Prefer encapsulating UI expectations in Page Object methods (e.g. `assertLoggedIn()`) so specs (or step definitions) stay short and UI changes touch one class.
- **Locators** — Prefer `getByRole`, `getByLabel`, then `getByTestId`; avoid brittle CSS and `nth-child`.
- **Independence** — Each scenario creates or uses isolated data; no cross-test reliance on order.
- **Stability** — Use `expect(locator).toBeVisible()` and built-in retries; avoid `waitForTimeout()`.
- **Secrets** — Only via env files ignored by git; document names in `*.example` files.

## Gherkin in GitHub issues (handoff to a testing agent)

Writing acceptance work as **Gherkin** makes it trivial for an agent to implement or update tests:

- Use a single fenced block per issue (or attach a `.feature` file) with a clear **Feature** title and **Scenarios** with **Given / When / Then** (and **And** where helpful).
- Include **@tags** for automation scope, e.g. `@smoke`, `@auth`, `@mobile` — mirror these in Playwright (`grep` in CI or dedicated `projects`).
- Add **Examples** tables for outline-style cases when the flow repeats with different data.
- Link **test data** expectations (seed emails, roles, feature flags) in plain text under the scenario.

Example issue fragment:

```gherkin
Feature: Account settings
  As a signed-in user
  I want to update my display name
  So that it appears on my profile

  @smoke @auth
  Scenario: Save a valid display name
    Given I am logged in as a seeded test user
    When I open the account settings page
    And I submit a valid display name
    Then I see a success confirmation
    And the new name is shown on the profile
```

The agent should: copy or save scenarios into **`features/`** (or keep them issue-only and generate `.feature` files from them), implement **step definitions** that delegate to Page Objects, and ensure tags map to CI jobs.

## Cucumber / BDD with Playwright

Many repos start **without** Cucumber; this section describes a **recommended** way to add Gherkin while keeping the same POM and fixtures.

**Goals**

- `.feature` files (or issue Gherkin) stay **human-readable** acceptance documentation.
- **Step definitions** are thin: resolve `page` (or fixtures), call Page Object methods, assert via PO helpers.
- **One automation stack** — still use `@playwright/test` reporters, traces, and CI patterns.

**Recommended approach: `playwright-bdd`**

Use **[playwright-bdd](https://github.com/vitalets/playwright-bdd)** to generate Playwright tests from Gherkin and run them with the normal Playwright runner (fixtures, projects, traces). Prefer **`defineBddConfig`** so `testDir` points at generated specs; run **`bddgen`** before **`playwright test`** whenever features or steps change.

### Greenfield BDD bootstrap (`playwright-bdd`)

Use this sequence when **introducing** BDD from scratch in a TypeScript repo (adapt paths if the team standardizes on `e2e/` instead of repo root).

**1. Dependencies (dev)**

```bash
npm i -D @playwright/test playwright-bdd
npx playwright install
```

**2. `package.json` scripts**

Run generation then the Playwright runner (same pattern as upstream example):

```json
{
  "scripts": {
    "test:bdd": "bddgen && playwright test",
    "test:bdd:ui": "bddgen && playwright test --ui"
  }
}
```

**3. Directory layout (co-located features + steps, v8+ `featuresRoot`)**

Keeping **`featuresRoot: './features'`** lets `playwright-bdd` default to `features/**/*.feature` and `features/**/*.{ts,...}` for steps—fewer moving parts than hand-rolling globs.

```text
features/
  account.feature
  steps/
    fixtures.ts       # createBdd(test) — export Given, When, Then
    account.steps.ts  # step defs → call Page Objects
tests/
  pages/              # Page Objects (optional but recommended)
  helpers/
.gitignore            # add .features-gen/ (generated Playwright files)
playwright.config.ts
```

**4. `playwright.config.ts` (minimal)**

`testDir` must be the return value of `defineBddConfig` so Playwright executes **generated** tests. Optional: `cucumberReporter()` from `playwright-bdd` for a Cucumber-style HTML report alongside Playwright’s HTML reporter.

```typescript
import { defineConfig, devices } from '@playwright/test';
import { defineBddConfig, cucumberReporter } from 'playwright-bdd';

const testDir = defineBddConfig({
  featuresRoot: './features',
  // Optional: filter generated/runnable scenarios — see upstream "tags" option
  // tags: '@smoke',
});

export default defineConfig({
  testDir,
  fullyParallel: true,
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  reporter: [
    cucumberReporter('html', { outputFile: 'cucumber-report/index.html', externalAttachments: true }),
    ['html', { open: 'never' }],
  ],
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
});
```

**5. `features/steps/fixtures.ts` — custom `test` + `createBdd`**

Extend this file when you need **auth or other fixtures**; step files import `Given` / `When` / `Then` from here so generated tests use the same `test` instance.

```typescript
import { test as base, createBdd } from 'playwright-bdd';

export const test = base.extend({
  // Example: authenticatedPage: async ({ page }, use) => { ... await use(page); },
});

export const { Given, When, Then } = createBdd(test);
```

**6. `features/steps/*.ts` — step definitions**

Match Gherkin steps **exactly** (wording). Use `{string}` and other parameter types per [playwright-bdd step docs](https://vitalets.github.io/playwright-bdd/#/). Keep bodies thin: construct Page Objects from `{ page }` (and any custom fixtures), then call action + assertion methods.

```typescript
import { expect } from '@playwright/test';
import { Given, When, Then } from './fixtures';

Given('I am on the home page', async ({ page }) => {
  await page.goto('/');
});

When('I click {string}', async ({ page }, name: string) => {
  await page.getByRole('link', { name }).click();
});

Then('I see the heading {string}', async ({ page }, text: string) => {
  await expect(page.getByRole('heading', { name: text })).toBeVisible();
});
```

**7. First `.feature` file**

Place files under `features/` (e.g. `features/account.feature`). Run `npm run test:bdd`; fix **missing steps** using the snippets `bddgen` prints (`missingSteps` defaults to failing at generation time—see upstream options).

**8. Git ignore**

Add **`.features-gen/`** (generated `*.spec.ts`); do not hand-edit files inside it.

**9. CI**

Before `playwright test`, run **`bddgen`** (same as local `test:bdd`). Install browsers, start the app (or point `BASE_URL` at a preview), then run tests and upload **Playwright HTML** + **`cucumber-report/`** if used.

**10. Mixing plain Playwright specs with BDD**

If the repo already has non-BDD `*.spec.ts` files, prefer **separate `projects`** with different `testDir` values, or a second config file—avoid two generators writing into the same `outputDir`. Consult upstream docs for the current recommended merge pattern.

Typical repo additions (alternative layout — steps under `tests/steps`):

```text
features/
  auth/
    login.feature
tests/
  steps/
    auth.steps.ts      # Given/When/Then → Page Objects
  specs/               # optional: non-BDD specs alongside
  pages/
  fixtures/
```

If `features/` and `tests/steps/` are split, set explicit `features` and `steps` globs in `defineBddConfig` instead of relying on `featuresRoot` alone.

**Step definition rule** — Each step body should:

1. Obtain `page` (or role-specific context) from a **fixture** or BDD world compatible with your chosen library.
2. Instantiate the right Page Object.
3. Call **actions** then **assertion** methods — no duplicated selector strings in steps.

**Alternative: `@cucumber/cucumber` + Playwright**

Some teams run **cucumber-js** with a custom `World` that owns `browser`, `context`, and `page`. That works but duplicates more runner configuration; prefer **playwright-bdd** unless the org already standardizes on pure Cucumber execution.

**Tag mapping**

- Map `@smoke` to a CI job that runs only smoke scenarios (`grep` / tag filter supported by your BDD integration).
- Keep **auth-heavy** scenarios in a separate project or job if they need different **storageState** or longer timeouts.

## `playwright.config.ts` baseline

- Load test env early (`dotenv` for `tests/.env.test` or project convention).
- Set `baseURL` from env with a sensible local default.
- Enable **`trace: 'on-first-retry'`** (or stricter in CI), **`screenshot: 'only-on-failure'`**.
- Define **`projects`** for at least Chromium desktop; add WebKit / mobile emulation for flows that need cross-browser or viewport coverage.
- Use **retries in CI**; keep local runs strict for faster feedback.

## CI

- Install browsers in the workflow (`npx playwright install --with-deps` as appropriate for the OS).
- Run DB migrations / seeds if the app requires them before E2E.
- Fail fast on **smoke** for PRs; schedule or gate **full suite** on merge or nightly.
- Always upload **HTML report** and on failure **test-results** (screenshots, traces).

## Deliverable checklist

- [ ] **`bddgen` runs in CI** (and locally) before `playwright test`; **`.features-gen/`** is gitignored.
- [ ] **`defineBddConfig`** `features` / `steps` / `featuresRoot` paths match where files actually live.
- [ ] Page Objects cover routes touched by new scenarios; no copy-pasted locators across steps.
- [ ] Auth and other shared setup live in **`features/steps/fixtures.ts`** (or equivalent) via `test.extend`, not duplicated across every step file.
- [ ] Gherkin (issue or `features/`) and step text match; tags explain CI scope.
- [ ] `.env.test.example` documents all required variables.
- [ ] Smoke paths stay under a few minutes total runtime where possible.
