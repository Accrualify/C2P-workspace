# Migration Execution Plan

**Created:** 2026-09-19
**Companion to:** [cypress-to-playwright-migration-plan.md](cypress-to-playwright-migration-plan.md)

That document is the **strategy** — why the migration is shaped the way it is,
what was found in each codebase, and what the design constraints are. Read it
once, then work from this one.

This document is the **execution plan**: what to do next, in what order, and
what has to be true before each stage is considered finished. Stage 0 first
reconciles the Phase 0 completion claims in the strategy doc with the intended
checkout, then verifies the foundations before porting begins.

---

## How the work runs

Each port is one agent session driven by
`.github/prompts/migrate-cypress-feature.prompt.md` in `corpay-playwright`. That
prompt pins the contract so every run behaves the same way: the `.feature` file
is the specification, the Cypress step/page-object code is intent reference
only, and selectors are grounded by grepping the React and component-library
repos rather than copied across.

One feature area per pull request. Human review on every one, at least through
Stage 3.

### Migration scope decision (2026-09-21)

**Next action:** verify the Stage 0 foundations against the intended checkout,
then migrate and validate the single read-only vendor-search pilot in Stage 1.
The current goal is agreed coverage, correct behaviour, and repeatable
Playwright tests. **No parallel-execution framework needs to be designed now.**
Parallel execution and shared-resource scheduling are deferred to Stage 4
execution hardening, not prerequisites for porting scenarios.

During migration, validate focused scenarios or feature slices sequentially
with `--workers=1`, including repeat runs and CI. Do not overlap validation
runs that use the same mutable accounts, settings, or records. This is a simple
validation rule, not a requirement to build a scheduling framework.

Each port must still use test-owned data where possible and restore shared
state after each scenario, including failures. If cleanup fails or a run is
interrupted, restore a known-good state before retrying or starting the next
conflicting scenario. Fresh browser contexts do not reset server-side data.

Acceptance is based on the agreed behaviours and the application's expected
results, not matching Cypress pass/fail results. There is no mandatory
dual-suite overlap window. Consult Cypress selectively where scenario intent
is unclear. This scope decision takes precedence over the companion strategy
on execution parity and parallel-execution timing.

### Authentication decision (2026-09-21)

BDD scenarios use **full UI login for every scenario requiring authentication**.
Each starts in a fresh browser context and uses the shared admin or named-user
login step, calling `LoginPage.loginAsAdmin()` or `LoginPage.loginAs(...)`.
The shared login step must not become a no-op. Login scenarios exercise the
form directly, including invalid-credential cases.

There is no anonymous-login mode or separate anonymous BDD lane. Starting
logged out is only the initial state before exercising login. The
`chromium-bdd` project does not require saved `storageState` reuse or a `setup`
dependency. This decision supersedes the BDD storage-state design in the
companion strategy; the separate Playwright UI-spec projects are unchanged.

### What's genuinely different from cypress-cucumber

Short list of things that will bite otherwise — everything else transfers.

| Cypress | Here |
|---|---|
| `Given/When/Then` from `@cucumber/cucumber` | from `tests/support/fixtures.ts` (`createBdd`) |
| `this.page`, `CustomWorld` | destructured fixtures: `async ({ page }) => {}` |
| `cy.get(sel)` chains | `page.getByRole/getByLabel/getByTestId` |
| `cy.visit(url)` | `BasePage.goto()` — resolves the Angular vs React origin |
| `cy.session()` | Full UI login through the shared login step for each scenario requiring authentication; no session reuse |
| `cy.intercept()` | `page.route()` / `page.waitForResponse()` |
| `cy.wait(ms)` | nothing — web-first assertions auto-wait |
| `Cypress.Commands.add()` | fixtures in `tests/support/fixtures.ts`, or an `api/clients/*` method |
| `cypress/e2e/<area>/x.feature` + sibling `x.ts` | `tests/features/<module>/x.feature` + `tests/steps/<module>/x.steps.ts` |
| `cucumber-js --tags @vendor` | `BDD_TAGS=@vendor npm run bdd -- --workers=1` (Playwright has no `--tags`) |
| — | `bddgen` must run before `playwright test`; `npm run bdd` does both |

---

## Stage 0 — Prove the foundation

**Goal:** verify the intended Phase 0 foundations are present and prove them
against Stage. Nothing else starts until this is done, because every later
stage assumes it.

1. **Reconcile Phase 0 with the intended checkout.** Record the intended
   branch/commit and confirm the required API clients, shared `AgGrid`, and
   migration prompt are present. Locate or complete missing work before the
   pilot; do not assume the strategy doc's completion claims match this checkout.
2. **Create `.env` in `corpay-playwright`** from `.env.example`, using the same
   QA/Stage credentials the Cypress suite already uses. Map them to
   `ANGULAR_BASE_URL`, `REACT_BASE_URL`, `API_BASE_URL`, `ADMIN_EMAIL`,
   `ADMIN_PASSWORD`.
   > ⚠️ For a per-environment file the name is **`.env.stage`**, not
   > `.env.staging`. `.env.example` and `Folder_Structure.md` both say
   > "staging" and are wrong; every npm script and `playwright.config.ts` use
   > `stage`. A `.env.staging` file loads silently as nothing.
3. **Run the login scenarios with one worker:**
   `TEST_ENV=stage BDD_TAGS=@login npm run bdd -- --workers=1`.
   This exercises the real login UI and proves credentials, origins, and
   `bddgen` wiring. Use the direct `bdd` script so Playwright receives the flag.
4. **Run exactly one read-only scenario requiring authentication.** Select
   "Vendors list page shows the expected components" by title using `--grep`
   and `--workers=1`. Do not use the whole `@vendor` tag as an authentication
   smoke: it also creates vendors and changes user roles.
   Confirm the shared Given performs the full email, Continue, password, and
   Login flow from a fresh context, then reaches the protected page. It must
   not skip credential entry or depend on a saved session.
5. **Settle blocker #3 — API authentication and the company nonce.** Once the
   API clients are available, wire them to the current scenario's token and
   company nonce after full UI login. Check `ngStorage-currentCompany` in the
   active browser context; decide whether to wait for hydration or supply
   `COMPANY_ID_NONCE` for the same company. Prove a company-scoped read call
   succeeds. A saved browser-session file is not a BDD prerequisite.
6. **Verify blocker #2: CI.** Inspect the current
   `.github/workflows/playwright.yml` and npm scripts. Confirm CI runs `bddgen`
   before Playwright, selects the intended migrated scenarios, uses
   `--workers=1`, and reports BDD results. Fix any missing wiring; do not assume
   the older claim that BDD has never run in CI matches this checkout.
7. **Decide blocker #5 — formatting.** Either land one formatting-only commit so
   the repo is Prettier-clean, or agree to scope Prettier to changed files.
   Leaving it unresolved means every PR carries ~12 files of unrelated churn.

**Exit gate**
- [ ] Intended branch/commit recorded; required API clients, shared `AgGrid`, and migration prompt present.
- [ ] Stage login scenarios green with `--workers=1` (step 3).
- [ ] One read-only protected scenario green with `--workers=1` after full UI login from a fresh context, without saved-session reuse.
- [ ] API helpers receive the current scenario's usable token **and** matching company nonce.
- [ ] One `api/clients/*` read call succeeds against Stage.
- [ ] CI runs `bddgen`, validates the selected scenarios with `--workers=1`, and reports BDD results.

---

## Stage 1 — Pilot port

**Goal:** prove the whole porting loop on the smallest possible piece of work,
before committing to volume.

Port **one** scenario — suggested:
`cypress/e2e/vendors/vendors.feature` → *"Verify the vendors table loads an
existing vendor when searching by vendor name"*. It is small, it exercises the
ag-Grid component object, and every selector it needs is already confirmed in
§3.1 of the strategy doc.

Run it through `migrate-cypress-feature.prompt.md` exactly as written. The point
is as much to test the prompt as the scenario. Keep the pilot read-only and
run it by itself with `--workers=1`.

**Exit gate**
- [ ] Scenario passes `--repeat-each=3 --workers=1`.
- [ ] Every `getByTestId` in the diff greps to a real occurrence in
      `accrualify-reactjs` or `corpay-react-components-library`.
- [ ] No `cy.wait` equivalents, retry counters, or conditional branching survived the port.
- [ ] `AgGrid` was used rather than hand-rolled row locators.
- [ ] Prompt corrections folded back into the prompt file — not just fixed in place.

> If the prompt needed significant hand-holding, fix the prompt before Stage 3.
> Every later port inherits its quality.

---

## Stage 2 — Triage inventory

**Goal:** decide what is actually being ported. "48 features" is not the target
and never was.

Build a checklist joining three sources:

1. Every scenario in `cypress/e2e/**/*.feature`, with Scenario Outline examples expanded.
2. `docs/tests_status.md` — the Passing / Failing ledger.
3. `docs/self-heal-flaky-history.json` — flake counts and failure signatures.

Columns: feature file · scenario · tags · current status · flake count ·
dominant failure signature · seeding required · `@setupEnvironment` dependency ·
target path · **decision**.

Decisions: **Port** · **Port + fix** · **Defer** (app bug — file a ticket) ·
**Drop** (obsolete or low-value) · **Redesign** (depends on `@setupEnvironment`).

> This stage is the one that most needs the original author's judgement rather
> than tooling. Mechanical analysis can propose the Drop and Defer candidates —
> particularly the combinatorial Scenario Outline expansion in
> `payment_run.feature` — but only someone who knows why each scenario was
> written can confirm them. Expect to overrule the mechanical suggestions.

**Exit gate**
- [ ] Every scenario carries a decision.
- [ ] Defer decisions have tickets raised.
- [ ] Agreed scope, with a real scenario count, signed off with Dan.

---

## Stage 3 — Port in vertical slices

One area per PR, green scenarios first within each area.

| # | Area | Notes |
|---|---|---|
| 1 | Vendors | Best-grounded area. Proves out `AgGrid`. Seeding is UI-only — no vendor create API (blocker #4). |
| 2 | Purchase Orders | `POST /purchase_orders` and `/po_requests` both confirmed. Capture a real request body first. |
| 3 | Invoices | Best seeding story — `POST /invoices` with a fully typed payload. Large (~60 scenarios). |
| 4 | Credit Memos | Depends on invoice seeding from slice 3. |
| 5 | Users / Subsidiaries | **Gated on blocker #4** — no user-create endpoint is visible client-side. |
| 6 | Payments | Largest and most red. 16 feature files. Last, once everything else is proven. |
| — | Expenses, Cards, Approvals, Dashboard, Reports, Profile, Administration | Sequence by business priority. |

Per PR:
1. Port via the prompt.
2. Add `bdd:<tag>:qa` / `bdd:<tag>:stage` scripts for any new feature tag.
3. `npx bddgen && npx tsc --noEmit`, Prettier on changed files.
4. `--repeat-each=3 --workers=1` green, with no overlapping conflicting runs.
5. Add the tag to CI using the same sequential validation rule.
6. Append any selector gaps found to the Phase 0.6 audit list.

**Exit gate per slice**
- [ ] All in-scope scenarios ported or explicitly deferred.
- [ ] `--repeat-each=3 --workers=1` green for the tag.
- [ ] Test-owned data cleanup and shared-state restoration verified, including failures.
- [ ] Tag running in CI with `--workers=1`.
- [ ] Banned-pattern scan clean: `waitForTimeout`, `setViewportSize`, `xpath=`,
      `nth-child`, `:eq(`, `.css-`, `console.log`, `process.env.` outside `utils/env.ts`.

---

## Stage 4 — Hardening and cutover

1. Verify the agreed Playwright coverage against expected application behaviour,
   with repeated sequential passes and green CI. A mandatory Cypress overlap
   window or matching Cypress execution results is not a cutover gate.
2. Review execution isolation before enabling any parallel runs. Classify by
   shared accounts, company settings, policies, and records, including tests
   that depend on those resources without changing them. Known candidates
   include vendor/user roles, profile updates, expense/payment defaults,
   shared expense policies, user metadata, and bulk-import result tracking.
   Sequential execution can remain the default; design parallel execution
   only if needed as a separate hardening task.
3. Retire each Cypress feature after its agreed replacement coverage passes
   the Playwright review, repeat-run, and CI gates. Record dropped or deferred
   scenarios explicitly; no side-by-side execution comparison is required.
4. Land the batched test-hook PRs from Phase 0.6 upstream: `DropdownToggle` in
   the component library, and `role="alert"` on the app's toast usage.
5. Resolve the component-library version skew (repo at 1.2.2, app consumes 1.2.3).
6. Rename `@vendorAddEdit` / `@invoiceAddEdit` to kebab-case once their features
   are ported, updating any scripts that reference them.
7. Decide whether to port the self-heal pipeline. A stable suite needs far less
   of it; `corpay-playwright/docs/self-healing-tests-plan.md` already scopes the
   Playwright equivalent if it's wanted.

**Exit gate**
- [ ] Agreed scope ported and green in CI.
- [ ] Default execution mode documented; any enabled parallel runs have verified shared-resource isolation.
- [ ] Cypress suite retired or reduced to an explicitly-owned remainder.
- [ ] No scenario depends on `@setupEnvironment`-style live config assertions.

---

## Decision log

Decisions and remaining open questions, to be updated as the work proceeds.

| # | Question | Needed by | Status |
|---|---|---|---|
| 1 | Does `POST /vendors` exist server-side? Any user-create endpoint? | Stage 3 slices 1 and 5 | Open — needs a backend engineer |
| 2 | How should API helpers obtain the live session's token and company nonce? | Stage 0 | Open; verify hydration or a matching `COMPANY_ID_NONCE` after full UI login |
| 3 | Coverage scope: everything, or a prioritised subset? | Stage 2 | Open (recommend subset); matching Cypress execution results is not required |
| 4 | Keep Jira tags (`@PAY-961`, `@EXP-393`)? | Stage 3 | Recommend keep |
| 5 | Role-specific credentials, logout, and shared-state cleanup | Before validating the first role-switching port | Open; full UI login and restoration required during sequential validation; parallel account allocation deferred to Stage 4 |
| 6 | Two-company matrix ("Automation Client 1" / "Automation Client NVP")? | Stage 2 | Open |
| 7 | Formatting-only commit, or scope Prettier to changed files? | Stage 0 | Open |
| 8 | Port the self-heal pipeline at all? | Stage 4 | Open |
| 9 | BDD authentication model | Stage 0 | Decided 2026-09-21: full UI login per scenario requiring authentication; no saved-session reuse or separate anonymous lane |
| 10 | Migration validation versus parallel-execution design | Stages 0-3 / Stage 4 | Decided 2026-09-21: sequential migration validation with cleanup and no conflicting overlap; parallel-execution design deferred to hardening; no mandatory Cypress overlap window |
