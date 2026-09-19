# Migration Execution Plan

**Created:** 2026-09-19
**Companion to:** [cypress-to-playwright-migration-plan.md](cypress-to-playwright-migration-plan.md)

That document is the **strategy** — why the migration is shaped the way it is,
what was found in each codebase, and what the design constraints are. Read it
once, then work from this one.

This document is the **execution plan**: what to do next, in what order, and
what has to be true before each stage is considered finished. Phase 0 is
complete (see §0 of the strategy doc); everything here starts from that point.

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

### What's genuinely different from cypress-cucumber

Short list of things that will bite otherwise — everything else transfers.

| Cypress | Here |
|---|---|
| `Given/When/Then` from `@cucumber/cucumber` | from `tests/support/fixtures.ts` (`createBdd`) |
| `this.page`, `CustomWorld` | destructured fixtures: `async ({ page }) => {}` |
| `cy.get(sel)` chains | `page.getByRole/getByLabel/getByTestId` |
| `cy.visit(url)` | `BasePage.goto()` — resolves the Angular vs React origin |
| `cy.session()` | `storageState` + the `setup` project |
| `cy.intercept()` | `page.route()` / `page.waitForResponse()` |
| `cy.wait(ms)` | nothing — web-first assertions auto-wait |
| `Cypress.Commands.add()` | fixtures in `tests/support/fixtures.ts`, or an `api/clients/*` method |
| `cypress/e2e/<area>/x.feature` + sibling `x.ts` | `tests/features/<module>/x.feature` + `tests/steps/<module>/x.steps.ts` |
| `cucumber-js --tags @vendor` | `BDD_TAGS=@vendor npm run bdd` (Playwright has no `--tags`) |
| — | `bddgen` must run before `playwright test`; `npm run bdd` does both |

---

## Stage 0 — Prove the foundation

**Goal:** turn Phase 0 from "compiles" into "runs". Nothing else starts until
this is done, because every later stage assumes it.

1. **Get the Phase 0 work committed and pushed.** It is currently uncommitted
   (17 modified files, 3 new paths). Clone/pull before anything else.
2. **Create `.env` in `corpay-playwright`** from `.env.example`, using the same
   QA/Stage credentials the Cypress suite already uses. Map them to
   `ANGULAR_BASE_URL`, `REACT_BASE_URL`, `API_BASE_URL`, `ADMIN_EMAIL`,
   `ADMIN_PASSWORD`.
   > ⚠️ For a per-environment file the name is **`.env.stage`**, not
   > `.env.staging`. `.env.example` and `Folder_Structure.md` both say
   > "staging" and are wrong; every npm script and `playwright.config.ts` use
   > `stage`. A `.env.staging` file loads silently as nothing.
3. **Run the anonymous lane:** `npm run bdd:login:stage`. This exercises the
   real login UI and proves credentials, origins, and `bddgen` wiring.
4. **Run an authenticated scenario** (e.g. `BDD_TAGS=@vendor npm run bdd`). This
   proves the `setup` project writes `playwright/.auth/admin.json` and that
   `ensureLoggedIn()` correctly treats it as a live session.
5. **Settle blocker #3 — the company nonce.** After a successful run, inspect
   `playwright/.auth/admin.json` for `ngStorage-currentCompany`. If absent,
   either restore the commented-out hydration wait at the end of
   `LoginPage.loginAsAdmin()` or set `COMPANY_ID_NONCE` in `.env`. Confirm by
   calling any `api/clients/*` list method — `assertCompanyScoped()` will throw
   a clear error rather than return a bare 401.
6. **Fix blocker #2 — CI.** `.github/workflows/playwright.yml` runs
   `npm run test`, which never invokes `bddgen`, so the BDD suite has never run
   in CI. Do this now while the config is already open, not at cutover.
7. **Decide blocker #5 — formatting.** Either land one formatting-only commit so
   the repo is Prettier-clean, or agree to scope Prettier to changed files.
   Leaving it unresolved means every PR carries ~12 files of unrelated churn.

**Exit gate**
- [ ] `npm run bdd:login:stage` green.
- [ ] At least one authenticated scenario green.
- [ ] `playwright/.auth/admin.json` contains a usable token **and** company nonce.
- [ ] One `api/clients/*` read call succeeds against Stage.
- [ ] CI runs `bddgen` and reports BDD results.

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
is as much to test the prompt as the scenario.

**Exit gate**
- [ ] Scenario passes `--repeat-each=3`.
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
4. `--repeat-each=3` green.
5. Add the tag to CI.
6. Append any selector gaps found to the Phase 0.6 audit list.

**Exit gate per slice**
- [ ] All in-scope scenarios ported or explicitly deferred.
- [ ] `--repeat-each=3` green for the tag.
- [ ] Tag running in CI.
- [ ] Banned-pattern scan clean: `waitForTimeout`, `setViewportSize`, `xpath=`,
      `nth-child`, `:eq(`, `.css-`, `console.log`, `process.env.` outside `utils/env.ts`.

---

## Stage 4 — Hardening and cutover

1. Run both suites in parallel for an agreed overlap window. Investigate every
   disagreement — a scenario passing in one and failing in the other means one
   of them is lying.
2. Retire each Cypress feature only after its Playwright equivalent has been
   green in CI for the full window.
3. Land the batched test-hook PRs from Phase 0.6 upstream: `DropdownToggle` in
   the component library, and `role="alert"` on the app's toast usage.
4. Resolve the component-library version skew (repo at 1.2.2, app consumes 1.2.3).
5. Rename `@vendorAddEdit` / `@invoiceAddEdit` to kebab-case once their features
   are ported, updating any scripts that reference them.
6. Decide whether to port the self-heal pipeline. A stable suite needs far less
   of it; `corpay-playwright/docs/self-healing-tests-plan.md` already scopes the
   Playwright equivalent if it's wanted.

**Exit gate**
- [ ] Agreed scope ported and green in CI.
- [ ] Cypress suite retired or reduced to an explicitly-owned remainder.
- [ ] No scenario depends on `@setupEnvironment`-style live config assertions.

---

## Decision log

Open questions, to be answered and recorded here as the work proceeds.

| # | Question | Needed by | Status |
|---|---|---|---|
| 1 | Does `POST /vendors` exist server-side? Any user-create endpoint? | Stage 3 slices 1 and 5 | Open — needs a backend engineer |
| 2 | Restore the `ngStorage-currentCompany` wait, or set `COMPANY_ID_NONCE`? | Stage 0 | Open |
| 3 | Parity target — everything, or a prioritised subset? | Stage 2 | Open (recommend subset) |
| 4 | Keep Jira tags (`@PAY-961`, `@EXP-393`)? | Stage 3 | Recommend keep |
| 5 | Multi-role storage states for role-switching scenarios | Stage 3 slice 5 | Open |
| 6 | Two-company matrix ("Automation Client 1" / "Automation Client NVP")? | Stage 2 | Open |
| 7 | Formatting-only commit, or scope Prettier to changed files? | Stage 0 | Open |
| 8 | Port the self-heal pipeline at all? | Stage 4 | Open |
