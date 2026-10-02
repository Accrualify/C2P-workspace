# Migration Plan: Accrualify Cypress suite → Corpay Playwright

**Status:** Historical design snapshot; current implementation status is in the execution plan
**Date:** 2026-09-19 (rev. 3 — progress recorded, blockers listed)
**Source repo:** `accrualify-test-automation` (Cypress + cypress-cucumber-preprocessor)
**Target repo:** `corpay-playwright` (Playwright + playwright-bdd)
**Grounding repos:** `accrualify-reactjs`, `corpay-react-components-library`

> **This document is the _strategy_ — why the migration is shaped this way.**
> For the sequenced work from here to the finish line, see
> [migration-execution-plan.md](migration-execution-plan.md).

> **Current decisions (2026-09-25):** Stage 0 foundations are checkpointed with
> local verification; live acceptance remains separate. Stage 1 repairs the
> existing base E2E using React for every application page, including login,
> with full UI credential entry in a fresh context. Stage 2 completes the
> source-to-target coverage inventory before broader migration. CI provider
> choice and QA/Stage usage remain undecided. These decisions override the
> historical phase numbering, saved-session recipes, and Angular-login scope
> below. Environment snapshots and operational flag profiles remain private.

> **Update 2026-09-22:** The completion claims and blockers below describe the
> 2026-09-19 snapshot, not the current checkout. The execution plan records
> today's baseline and successful local Stage foundation checks. Its
> 2026-09-21 decisions supersede the saved-session BDD design, parallel
> migration execution, and mandatory Cypress overlap described here. Use
> full UI login per authenticated scenario and sequential validation; do not
> restore the historical auth design. Stage 0 is not yet signed off.

> **Review update 2026-09-23:** The execution plan now brings existing-port
> reconciliation, active instructions, missing foundations, and deployed-version
> evidence into Stage 0. Preliminary inventory can run alongside it as read-only
> analysis. Establish isolation and prove UI login/API authentication in the same
> scenario before a deterministic, coverage-preserving pilot. Use review-sized
> batches within each domain, with mapped assertions and retry-free acceptance
> before cutover. A smaller search-only pilot requires explicit approval and is
> not equivalent to the source create-and-approve scenario. Both local library
> declarations are `1.2.4`; the old 1.2.2/1.2.3 mismatch is obsolete. This review
> is static: no new live Stage results, deployed revisions, or successful CI
> execution were verified. The dated prior foundation evidence is not broader
> acceptance. Historical recipes below do not override the execution plan.

---

## 0. Historical status (2026-09-19)

> ⚠️ **Nothing in `corpay-playwright` has ever been executed against a live
> environment.** There is no `.env` in the repo, so no scenario has ever run.
> Every gate passed so far is **static**: `bddgen`, `tsc --noEmit`,
> `prettier --check`, and `playwright test --list`. Read Phase 0 as
> *"compiles and resolves"*, not *"works"*. Proving it works is Stage 0 of the
> [execution plan](migration-execution-plan.md).

| Phase | Item | Status |
|---|---|---|
| 0.1 | Agent instructions corrected for playwright-bdd | ✅ Done |
| 0.2 | Storage-state auth wired into the BDD lane | ✅ Done |
| 0.3 | API seeding layer | ◐ Foundation only |
| 0.4 | Shared ag-Grid component object | ✅ Done |
| 0.5 | Migration prompt | ✅ Done |
| 0.6 | Component-library / app test-hook audit | ◐ Findings recorded, no PRs raised |
| 1 | Triage inventory | ⬜ Not started |
| 2 | Vertical slices (porting) | ⬜ Not started |
| 3 | Hardening and cutover | ⬜ Not started |

### What was built

**0.1 — Agent instructions.** Seven files in `corpay-playwright` described
**cucumber-js** (`CustomWorld`, `cucumber.js`, `tsconfig.cucumber.json`,
`test:cucumber:*`), none of which exist. The repo runs **playwright-bdd**.
Corrected: `AGENTS.md`, `.github/copilot-instructions.md`,
`.github/agents/playwright-agent.agent.md`,
`.github/instructions/cucumber-bdd.instructions.md`, and three prompt files.
Left alone (already accurate): `page-objects.instructions.md`,
`playwright-specs.instructions.md`, `new-api-client.prompt.md`.

**0.2 — Auth.** The BDD suite now generates into two projects:
`chromium-bdd` (tags `not @anonymous`, loads `playwright/.auth/admin.json`,
`dependencies: ["setup"]`) and `chromium-bdd-anon` (tags `@anonymous`, blank
storage state). `LoginPage.ensureLoggedIn()` makes the shared
`Given I am logged in…` idempotent. Design note: this is **opt-out via
`@anonymous`**, not the opt-in `@authenticated` originally sketched — default
"authenticated" matches how every non-login feature already behaved, so it
needed one tag on one feature instead of tagging everything.

**0.3 — API seeding (foundation only).** `api/clients/apiClient.ts` base plus
`invoiceClient`, `purchaseOrderClient`, `vendorClient`; `api/endpoints.ts`
extended with paths read directly out of the React service layer; fixtures wired
into both `tests/support/fixtures.ts` (BDD) and `fixtures/testBase.ts` (spec).
**Payload shapes are unverified** — `create()` on invoices and POs needs a real
request body captured from the app before it can be trusted. `vendorClient` has
no `create()` on purpose (see §4.3).

**0.4 — ag-Grid.** `pages/components/AgGrid.ts`, grounded on real hooks:
`quick-filters-button|clear|reset`, `datagrid-floating-filter-<colId>-select`,
and `col-id` from the column definitions.

**0.5 — Migration prompt.** `.github/prompts/migrate-cypress-feature.prompt.md`
encodes the porting contract, the §3.1 replacement table, the drop-list, and
three stop conditions.

> All of the above is **uncommitted** at time of writing — 17 modified files and
> 3 new paths in `corpay-playwright`. Commit and push before anyone clones.

### Open blockers

| # | Blocker | Impact |
|---|---|---|
| 1 | No `.env` in `corpay-playwright` | Nothing has ever run. Everything below is unproven. |
| 2 | CI runs `npm run test` with **no `bddgen` step** | `.features-gen/` never exists in CI, so **BDD scenarios have never run there**. Pre-existing; not caused by 0.2. |
| 3 | `ngStorage-currentCompany` hydration wait is commented out at the end of `LoginPage.loginAsAdmin()` | `readStoredAuth()` reads that key for the company nonce. If it's genuinely required, `admin.json` lacks it and **every API seeding call 401s**. `ApiClient.assertCompanyScoped()` surfaces this as a clear error rather than a mystery. |
| 4 | Does `POST /vendors` exist server-side? | The React app is PATCH-only and Cypress creates vendors through the UI. Blocks API seeding for the Vendors and Users slices. Needs a backend engineer. |
| 5 | `npm run format` rewrites ~12 unrelated files | `prettier --write "**/*.{ts,json,md}"` reformats the whole repo (quote normalisation, not just whitespace). The repo was never Prettier-clean. Scope Prettier to changed files, or land one formatting-only commit first. |
| 6 | **`.env.staging` vs `.env.stage`** | `.env.example` (lines 1, 6) and `Folder_Structure.md:110` say `staging`; every npm script and `playwright.config.ts` use `stage`. A `.env.staging` file **silently never loads**. |
| 7 | `@vendorAddEdit` / `@invoiceAddEdit` | Legacy camelCase tags violating the repo's own kebab-case rule. Documented as "don't copy"; not renamed, because renaming changes tag filtering. |

---

## 1. TL;DR

Port the suite agentically, but drive each port from the **`.feature` file as the
specification** — not by transpiling the Cypress step definitions and page objects.

A file-by-file translation would carry the instability across, because it comes
from the Cypress session layer and from selectors written against an app that
exposed very few test hooks at the time — not from the Gherkin. The Gherkin is
the asset worth keeping; almost everything below it should be rewritten against
Playwright idioms.

**The rewrite is tractable because the stable hooks already exist.** With the
React app and the shared component library open locally, every major Cypress
selector hack has been verified to have a real `data-testid` or role-based
replacement in source today (see §3.1). The component library forwards
`data-testid` on 28 of 29 components. This is a targeted rewrite against known
hooks, not an exploratory one.

Four foundations must land before bulk porting: correct agent instructions,
storage-state auth on the BDD project, an API seeding layer, and a shared
ag-Grid component object. Without them the migration will stall or produce a
second flaky suite.

---

## 2. Current state

### Source: `accrualify-test-automation`

| Item | Value |
|---|---|
| Runner | Cypress + `@badeball/cypress-cucumber-preprocessor` |
| Feature files | 48, under `cypress/e2e/**` |
| `Scenario:` lines | ~550 (plus Scenario Outline × Examples → materially more cases) |
| Step definition files | co-located `.ts` beside each `.feature` |
| Page objects | 14 domain folders under `cypress/support/page_objects/` |
| Selectors | centralised in `cypress/support/element_selectors/` (21 folders) |
| Custom commands | ~400 across 19 domain folders; ~130 are pure REST API seeding |
| Shared step defs | `cypress/e2e/step_definitions/` — `common_steps.ts`, `grid_steps.ts`, `ag_grid_steps.ts` |

### Target: `corpay-playwright`

| Item | Value |
|---|---|
| Runner | Playwright 1.60 + `playwright-bdd` 9.2 |
| Feature files | 5 (`base/login`, `budgets/budget`, `invoices/add_invoice`, `purchase_orders/request_purchase_order`, `vendors/vendor`) |
| Page objects | 6, in `pages/web/admin/` |
| Base class | `core/base/BasePage.ts` — per-app origin resolution via `app: "angular" \| "react"` + `path` |
| BDD glue | `tests/support/fixtures.ts` — `createBdd(test)` |
| Auth | `tests/setup/global.setup.ts` writes `playwright/.auth/admin.json`; `utils/storageStateAuth.ts` reads token/nonce/pod out of it |
| API layer | `api/endpoints.ts` only (categories, user, documents, documentRowResults, paymentRuns). **No `api/clients/` folder exists.** |

### Grounding sources (verified locally)

| Repo | Hosts | Verified facts |
|---|---|---|
| `accrualify-reactjs` | `/ap/*` React pages | **251 `data-testid` occurrences across 59 files**, consistent convention, no `testId`/`test-id`/`qaId` variants. API layer is `src/providers/restApi.ts`. |
| `corpay-react-components-library` | shared components | Package `@Accrualify/corpay-react-components-library`. **28 of 29 components forward `data-testid`.** Wraps react-select, ag-grid-react, react-bootstrap. Storybook in `src/stories/`. |
| `accrualify-angularjs` | `/login`, legacy admin | **Not currently in the workspace** — Angular grounding still needs the `githubRepo` tool. Low urgency: `LoginPage.ts` already exists and login is the only Angular surface in scope. |

Consequence: selector grounding becomes an exact local `grep_search` for
`data-testid=` rather than a remote semantic lookup that can silently return
nothing. This removes the main failure mode in the per-scenario recipe (§6).

### Structural compatibility

Both sides are Gherkin. That is the lever — `.feature` files port at close to 1:1.
Only step definitions, page objects, and the seeding layer need real work.

| Cypress layer | Playwright equivalent | Effort |
|---|---|---|
| `.feature` (Gherkin) | `tests/features/<module>/<name>.feature` | Trivial — copy, retag, reword |
| step defs `e2e/**/*.ts` | `tests/steps/<module>/<name>.steps.ts` | Mechanical — `(…) => {}` becomes `async ({ page }) => {}` |
| `support/element_selectors/**` | `readonly` locators inside page objects | **Rewrite from source, do not port** — replacements verified, see §3.1 |
| `support/page_objects/**` | `pages/web/admin/*Page.ts` extending `BasePage` | Medium — restructure + reground selectors |
| `support/commands/**` (API seeding) | `api/clients/*Client.ts` + a Playwright fixture | **Largest gap — nothing exists yet** |

### What the Cypress suite got right (and what carries forward)

This migration is a **platform change, not a rewrite of bad work**. A large part
of the Cypress suite is being carried over rather than replaced:

| Asset | How it's used here |
|---|---|
| **The Gherkin itself** — 48 features, ~550 scenarios | The specification. Ports at close to 1:1 and is the single reason this migration is tractable rather than a from-scratch rebuild. |
| Breadth of domain coverage | Payments, invoices, POs, credit memos, expenses, cards, approvals — years of encoded product knowledge that no amount of tooling replaces. |
| `docs/tests_status.md` | A per-scenario pass/fail ledger. Becomes the primary Phase 1 triage input (§5). Most suites have nothing like it. |
| `docs/self-heal-flaky-history.json` | Per-scenario flake signatures with timestamps — real diagnostic data, and how the auth root cause in §3.3 was identified at all. |
| `docs/self-heal-known-issues.md` | A root-cause knowledge base keyed by failure signature. |
| `scripts/self-heal/` | A working self-healing pipeline. `corpay-playwright/docs/self-healing-tests-plan.md` is explicitly modelled on it. |
| API-first seeding instinct | ~130 of ~400 custom commands seed over REST rather than through the UI — exactly the right call, and the model for §4.3. |
| Centralised `element_selectors/` | The correct instinct: one place to change when the app moves. The selectors inside are brittle because the app offered no stable hooks then, not because centralising was wrong. |
| `@setupEnvironment` | Correctly identified environment drift as a top test-killer. The *mechanism* needs rework (§3.4); the insight was right and is preserved as a separate monitoring concern. |

**The thing that changed is the application, not the standard of the testing.**
The React app now carries 251 `data-testid` attributes across 59 files, and the
shared component library forwards `data-testid` on 28 of 29 components. Almost
none of that existed when the Cypress selectors were written. That delta — not
test-writing quality — is what makes §3.1 possible today and impossible then.

---

## 3. Why a mechanical translation fails

> The four causes below are **structural** — properties of Cypress, of the
> application at the time, and of the environment the suite runs against. None
> of them are fixed by writing the same tests more carefully; all of them are
> fixed by the platform change. That is the case for migrating.

### 3.1 The selectors target hooks that didn't exist yet — and now do

From `cypress/support/element_selectors/vendors/vendors.ts`. Every one of these
is on the forbidden list in `AGENTS.md` (§ Anti-patterns) — but each was a
reasonable choice against an app that exposed no better hook at the time. The
right-hand column is what exists in the React source **today**:

| Cypress selector | Why it breaks | Verified replacement | Source file |
|---|---|---|---|
| `.grid-filter-dropdown-default-icon` | library auto-class | `data-testid="quick-filters-button"` | `gridFilterDropdown.jsx` |
| `.dropdown-menu > :nth-child(2)` (reset grid) | positional | `data-testid="quick-filters-reset"` | `gridFilterDropdown.jsx` |
| `.dropdown-menu > :nth-child(1)` (clear filters) | positional | `data-testid="quick-filters-clear"` | `gridFilterDropdown.jsx` |
| `.bulk-action-toggle-default-icon` | library auto-class | `data-testid="vendors-list-bulk-action"` | `listVendor.tsx` |
| `a > .btn` (Add) | structural | `data-testid="vendors-list-add-btn"` | `listVendor.tsx` |
| `input[name="name"]` (add vendor) | ambiguous | `data-testid="vendors-add-details-vendor-name"` | `formSections/addVendorDetails.tsx` |
| `#react-select-5-input` | index renumbers per render | `data-testid="<field>-select"` (from `inputId`) | `bootstrapFields.jsx` → `RenderReactSelect` |
| `.css-b62m3t-container` | emotion hash | `data-testid="<field>"` / `"<field>-group"` | `bootstrapFields.jsx` |
| `.rnc__notification-container--top-right` | library internal | `data-testid="notification-toast-{success\|error\|info\|warning}"` | `notifications.jsx` |
| `button:contains('Create New Vendor')` | jQuery-only pseudo-selector | `getByRole("button", { name: /create new vendor/i })` | — |

**Remaining gap — ag-Grid rows and cells carry no `data-testid`.** The grid
wrapper is `serverSideDataGrid.tsx` (AgGridReact v36). Column `col-id` values
mirror the `field` property in the column definitions (e.g. `getVendorsHeaders()`
in `vendorsGridHeader.tsx`), so `[col-id="vendor_id"]` **is** stable and portable.
What must go is the `:eq(0)` / `.ag-row-first` / `:nth-child` wrapping. Rows
should be located by cell text scoped inside `getByRole("grid")`, never by index.
See §4.4.

### 3.2 Cypress has no auto-waiting parity, so the page objects carry retry scaffolding

`cypress/support/page_objects/vendors/vendors.ts` contains `cy.wait(1000)`,
conditional `cy.get("body").then($body => $body.find(...))` branching, a
"retrying toggle click" fallback, and an explicit retry counter
(`search(vendor, retries = 12)`).

These were rational workarounds — Cypress retries assertions but not arbitrary
interaction sequences, so defensive scaffolding was the available tool.
Playwright's web-first assertions and auto-waiting make them unnecessary, which
is precisely why they must **not** be ported: carried across, they would
re-introduce the timing dependence the platform change removes.

### 3.3 The dominant flake cause is auth, not selectors

`docs/self-heal-flaky-history.json` in the source repo — the recurring signature is:

> `Timed out retrying after 30000ms: login form or authenticated app loaded: expected false to equal true` — *This error occurred while creating the session.*

That is `cy.session` failing at the framework level, and it takes down entire
unrelated scenarios (change orders, purchase orders, etc.). It is not something
a test author can fix from inside a scenario, and no amount of selector
translation helps — which is why §4.2 (storage-state auth) is a Phase 0 blocker
rather than a nice-to-have.

### 3.4 The `@setupEnvironment` orchestrator is a second systemic flake source

`cypress/support/commands/environment_setup/api_commands.ts` (~800–900 lines)
asserts ~48 hardcoded feature-flag values and a hardcoded approval-workflow
allowlist against the live Stage API. When Stage flips a flag, **every** scenario
tagged `@setupEnvironment` fails in the background step regardless of what it
tests. The source repo's own `docs/self-heal-known-issues.md` documents this as
its top two known failure signatures.

**Do not port this pattern** — but keep the insight behind it. Environment
config drift genuinely does break test runs; the problem is that asserting it
inside a scenario precondition converts an infrastructure event into hundreds of
unrelated test failures. It belongs in monitoring, owned separately, where a
flipped flag raises one alert instead of a red suite.

### 3.5 The suite already tells you which tests not to trust

- `docs/tests_status.md` is a pass/fail ledger; roughly half the payment-run rows read "Failing".
- `cypress/e2e/expenses/expenses.feature` contains scenarios explicitly titled
  `FLAKY Deleting a receipt` and `FLAKY Upload a new receipt`.

This is unusually good hygiene and it is doing real work here: it is the triage
input for Phase 1. Do not port a red test without first deciding whether it
reflects an application bug or a test that needs rethinking — a judgement call
that needs the original author's context, not a tool.

### 3.6 Grid state persists in localStorage — a Cypress-only flake source

`serverSideDataGrid.tsx` persists column state (visibility, order, width,
filters) to localStorage under a per-grid key — `GRID_STORAGE_NAME = "listVendor"`
for the vendors grid.

This explains an otherwise baffling Cypress pattern: `Vendors.search()` opens the
quick-filters dropdown, clicks **reset grid**, reopens it, clicks **clear
filters**, and only then types — on *every single search*. That is not test
logic, it is compensation for grid state leaking between tests. `cy.session`
caches and restores localStorage, so writes from one test genuinely do reach the
next.

**Playwright does not inherit this problem.** Each test gets a fresh browser
context seeded from `playwright/.auth/admin.json`; localStorage writes during a
test are held in that context and are **not** written back to the file. Grid
state therefore cannot leak from one scenario to the next.

So the reset-grid/clear-filters dance must **not** be ported. Drop it, and let
each scenario start from whatever the storage-state file contains.

**Residual risk worth guarding.** If `tests/setup/global.setup.ts` ever navigates
to a grid page before snapshotting, grid keys get baked into `admin.json` and
then apply to *every* scenario in *every* run — a silent, global, and very
confusing failure. Today it only logs in, so this is safe; keep it that way, and
have the ag-Grid component object (§4.4) assert a known column state rather than
assuming a clean grid.

---

## 4. Phase 0 — Blockers (must land before bulk porting)

These are sequential and must be complete before Phase 2 begins, except 0.6
which runs in parallel.

### 4.1 Fix the stale agent instructions

`AGENTS.md`, `.github/copilot-instructions.md`, `.github/instructions/*`, and all
four files in `.github/prompts/` describe **cucumber-js**:
`CustomWorld`, `tests/support/world.ts`, `tests/support/hooks.ts`, `cucumber.js`,
`tsconfig.cucumber.json`, `test:cucumber:*` npm scripts, and callbacks typed
`function (this: CustomWorld)`.

None of that exists. The repo runs **playwright-bdd**:
`tests/support/fixtures.ts` exports `createBdd(test)`, steps destructure
`async ({ page }) => {}` (see `tests/steps/vendors/vendor.steps.ts`), scripts are
`bdd:*`, and `@cucumber/cucumber` is not a dependency.

Also stale:
- Documented file layout is flat (`tests/features/<name>.feature`); the repo is
  module-grouped (`tests/features/<module>/<name>.feature`).
- The shared auth step is documented at `tests/steps/commonAuth.steps.ts`; it
  actually lives at `tests/steps/common/auth.steps.ts`.
- **§ Source repositories names only `accrualify-reactjs` and
  `accrualify-angularjs`.** Add `corpay-react-components-library` — agents will
  not consult it otherwise, and it is where the shared component hooks live.
- The grounding workflow assumes the remote `githubRepo` tool. Add the local
  multi-root workspace path as the preferred method now that the repos are open.

**Why this is blocker #1:** running a 48-feature agentic migration against these
instructions produces hundreds of files written for a framework the repo does not
use. Every one would fail to compile.

Deliverable: corrected `AGENTS.md`, `.github/copilot-instructions.md`,
`.github/instructions/cucumber-bdd.instructions.md`, and the prompt files.

### 4.2 Wire storage-state auth into the BDD project

`playwright.config.ts` — the `chromium-bdd` project has **no `storageState`** and
**no `dependencies: ["setup"]`**, unlike `chromium-ui` and `firefox-ui`. Every BDD
scenario therefore performs a full UI login through the
`Given I am logged in with valid admin credentials` background step.

At 5 features this is tolerable. At ~550 scenarios it is both the runtime bottleneck
and a direct re-creation of the source repo's #1 flake signature.

Deliverable:
- Add `dependencies: ["setup"]` and `storageState: "playwright/.auth/admin.json"` to `chromium-bdd`.
- Implement the `@authenticated` tag described in `AGENTS.md` § Authentication ("PR 2") so the login Given becomes a no-op when storage state is present.
- Verify the state survives the Angular → React origin hop. `utils/storageStateAuth.ts` already decodes the `pod` claim out of the JWT, so the plumbing largely exists.
- **Decide what to do about persisted grid state (§3.6)** — Playwright reseeds each context from the file, so no reset fixture is needed; just keep `global.setup.ts` away from grid pages.
- Keep `tests/features/base/login.feature` running unauthenticated — it tests login itself.

**Note:** `playwright.config.ts` is on the "requires explicit human confirmation"
list in `AGENTS.md`. This edit needs sign-off, not an agent decision.

### 4.3 Build the API seeding layer

This is the true critical path, and the largest single piece of work.
The React app's API layer has now been surveyed and resolves most of the unknowns.

**Confirmed from `src/providers/restApi.ts`:**
- Base URL from `config.apiURL`.
- `Authorization: Bearer <token>` where token is `localStorage.getItem("Token")`.
- `Company-Id-Nonce: <companyId>` header.
- Arrays serialise Rails-style (`key[]=value`); nulls skipped; objects JSON-stringified.

This matches exactly what `utils/storageStateAuth.ts` already extracts (token,
company nonce, pod URL), so the shared request helper is a thin wrapper — not a
research project.

**Endpoint coverage available from the React source:**

| Domain | Create | Notes |
|---|---|---|
| Invoices | `POST invoices` | Payload `{ invoice: InvoiceDetailType }`, **fully typed** in `invoiceType.ts`. Directly portable. |
| Purchase orders | `POST purchase_orders`, `POST po_requests` | Endpoints confirmed; payload typed only as `Record<string, unknown>`. Nested `po_items_attributes[]` and `invoice_debit_accounts_attributes[]` visible from usage. Needs a captured request body to pin down. |
| Vendors | **none** | Only `PATCH vendors/{id}` and `POST vendors/{id}/contacts`. No create endpoint in the React layer. |
| Users | **none** | Only `forgot_password`, `reset_okta_mfa`, `reset_okta_password`. |

**Important corroboration:** the Cypress suite has no create-vendor API command
either — `cypress/support/commands/vendors/` exposes only `getVendors`,
`getVendorByApi`, `deleteVendorByApi`. Vendors are created through the UI in the
existing suite too. So this is a **confirmed product constraint, not a research
gap**: vendor and user creation have no client-visible API path and must either
go through the UI or be answered by a backend engineer.

Deliverable:
- A shared authenticated request helper on Playwright's `APIRequestContext`,
  sourcing token / nonce / pod from `readStoredAuth()` in `utils/storageStateAuth.ts`.
- `api/clients/invoiceClient.ts` and `purchaseOrderClient.ts` first — these have
  real create endpoints and unblock the highest-volume feature areas.
- Extend `api/endpoints.ts` with the routes each client needs.
- Expose the clients as a playwright-bdd fixture so step definitions can seed
  without touching the UI.
- **Open with the backend team:** does `POST /vendors` exist server-side? Does a
  user-creation endpoint exist? Both would materially reduce UI-driven seeding.

Scope control: only ~130 of the ~400 Cypress commands are API seeding, and only a
fraction of those are needed for the first two phases. Build per-phase.

### 4.4 Build a shared ag-Grid component object

A large share of the suite is grid assertions. `col-id` values derive from the
column definitions (`field` in `getVendorsHeaders()` etc.) and **are** stable —
port them as-is. Rows and cells carry no `data-testid`, so row targeting must be
by content, never by index.

Deliverable: one component object (suggested `pages/components/AgGrid.ts`) that:
- Scopes to `getByRole("grid")`.
- Exposes `rowCount()`, `rowByCellText(colId, text)`, `cell(row, colId)`,
  `header(colId)`, `filterBy(colId, text)`.
- Wraps the quick-filter controls using the confirmed hooks:
  `quick-filters-button`, `quick-filters-reset`, `quick-filters-clear`.
- **Owns the quick-filter controls**, using the confirmed hooks:
  `quick-filters-button`, `quick-filters-reset`, `quick-filters-clear` — exposed
  as explicit methods for scenarios that genuinely test filtering, **not** called
  defensively before every search (see §3.6).

The source repo has the equivalent behaviour concentrated in
`cypress/e2e/step_definitions/ag_grid_steps.ts` and `grid_steps.ts`, so the
required surface is well defined.

This is the highest-leverage single item after auth — it unblocks a meaningful
fraction of the suite and removes a whole class of cross-test contamination.

### 4.5 Write the migration prompt

Add `.github/prompts/migrate-cypress-feature.prompt.md` pinning the contract so
every agent run behaves identically:

- **Input:** one Cypress `.feature` plus its sibling `.ts` step file.
- **The Cypress step and page-object code is intent reference only.** Never copy a
  selector from `element_selectors/`.
- **Grounding is a local grep, not a remote lookup.** Search `accrualify-reactjs`
  and `corpay-react-components-library` for `data-testid=`, `aria-label=`, and
  `getByRole`-able markup. Use the §3.1 replacement table as the starting point.
- **Output:** retagged `.feature` + playwright-bdd `.steps.ts` + page object
  extending `BasePage`.
- **Stop conditions:**
  - Scenario needs an API client that does not exist yet → report and halt. Do not
    improvise UI-driven seeding.
  - No stable hook exists for an element → **append it to the §4.6 audit list and
    continue with the best available role/label selector.** Do not halt, and do
    not fall back to a positional or hashed-class selector.
- Forbid porting `cy.wait(N)`, retry counters, and conditional `$body.find()`
  branching. Replace with Playwright auto-waiting and web-first assertions.

### 4.6 Component-library and app test-hook audit *(parallel with 4.1–4.5)*

The shared component library is in good shape and should be treated as the
primary source of truth for component-level hooks.

**Confirmed state:**
- 28 of 29 components accept `'data-testid'?: string` and forward it to a DOM node.
- The convention is the literal HTML attribute name as the prop key, destructured
  as `{ 'data-testid': testId }` — matching the React repo's documented pattern.
- A `warnMissingTestId` helper in `src/components/helpers/testUtils.ts` emits
  dev-time warnings when the prop is omitted.
- `Select.tsx` cascades ids to react-select options as
  `${testId}-option-${data.value}` — so option selection needs no positional index.
- `CorpayGrid` and its 8 supporting classes all forward `data-testid`.
- Strong `role=` / `aria-label` / `aria-labelledby` coverage supports
  `getByRole` and `getByLabel` throughout.
- Storybook (`src/stories/`, 30+ component stories) documents the rendered DOM —
  the fastest way to confirm what a component emits.

**Known gaps to close:**
1. `DropdownToggle` does not forward `data-testid` — it carries a literal
   `//TODO: Add testIds` comment. One upstream PR.
2. ag-Grid rows and cells have no `data-testid` (§3.1). Decide whether to add
   cell-level ids in `serverSideDataGrid.tsx` or accept content-based row
   targeting. *Recommendation: accept content-based targeting — it is more
   meaningful as an assertion anyway.*
3. Toast has a `data-testid` but **no `role="alert"` / `role="status"`** in the
   React app's `notifications.jsx` usage, even though the library's `Toast`
   supports those roles. Worth aligning.
4. **Deployed-version evidence:** the local library package and
   `accrualify-reactjs` dependency both read **1.2.4**, verified 2026-09-23.
   The earlier local mismatch is resolved. Separately verify the deployed app
   build and bundled library revision before accepting affected selectors;
   matching local declarations do not prove what Stage is running.

Deliverable: a running list of missing hooks produced as a by-product of Phase 2
(fed by the §4.5 stop condition), plus one batched upstream PR per repo rather
than one PR per scenario.

---

## 5. Phase 1 — Triage inventory

Produce a machine-generated porting checklist before writing any test code.
Preliminary read-only inventory runs alongside Stage 0 reconciliation; complete
the pilot's source-to-existing-target assertion map before implementation.
Broader scope decisions are signed off in Stage 2 of the execution plan, using
the same inventory. Its existing-target mapping and Unknown-evidence rules
supersede the historical three-source recipe below.

Join three sources:
1. Every scenario in `accrualify-test-automation/cypress/e2e/**/*.feature`
   (including expanded Scenario Outline examples).
2. `docs/tests_status.md` — the Passing / Failing ledger.
3. `docs/self-heal-flaky-history.json` — per-scenario flake occurrences and error signatures.

Output columns: feature file, scenario title, tags, current status, flake count,
dominant failure signature, required API seeding commands, `@setupEnvironment`
dependency (yes/no), proposed target path, port decision.

Port decisions:
- **Port** — currently passing, no `@setupEnvironment` dependency.
- **Port + fix** — passing but flaky; port and address the root cause.
- **Defer** — failing due to a genuine app bug; file/link a ticket, do not port yet.
- **Drop** — obsolete, duplicated, or superseded coverage. Expect a meaningful
  number here; `payment_run.feature` in particular has heavy Scenario Outline
  combinatorial expansion of marginal value.
- **Redesign** — depends on `@setupEnvironment`; needs a different approach (§3.4).

This inventory is the shared artifact the team works from. It also gives an
honest denominator — "48 features" overstates the real target once drops and
defers are removed.

---

## 6. Phase 2 — Vertical slices

Use review-sized batches within one domain per pull request. A domain can span
multiple PRs; each batch must have a bounded source-to-target assertion map and
independently reviewable setup, cleanup, and retry-free acceptance evidence.
Within a domain, follow the agreed inventory order, verified passing cases first.

### Recommended order

| # | Area | Source | Rationale |
|---|---|---|---|
| 1 | Vendors | `cypress/e2e/vendors/` | Best-grounded area — list page, add form, quick filters and toasts all have confirmed testids (§3.1). Heavy ag-Grid usage proves out the component object. **Caveat: no vendor-create API, so seeding is UI-only (§4.3).** |
| 2 | Purchase Orders | `cypress/e2e/purchase_orders/` | `RequestPurchaseOrderPage.ts` exists; `POST purchase_orders` / `po_requests` confirmed; exercises the multi-step form pattern |
| 3 | Invoices | `cypress/e2e/invoices/` | **Best API seeding story** — `POST invoices` with a fully typed payload. Large (~60 scenarios) but unblocked |
| 4 | Credit Memos | `cypress/e2e/credit_memos/` | Depends on invoice seeding from slice 3 |
| 5 | Users / Subsidiaries | `cypress/e2e/users/`, `subsidiaries/` | Small, but **no user-create API** — blocked on the §4.3 backend question |
| 6 | Payments | `cypress/e2e/payments/` | Largest and most broken area; 16 feature files. Do last, once foundations are proven |
| — | Expenses, Cards, Approvals, Dashboard, Reports, Profile, Administration | remainder | Sequence after the above based on business priority |

> Changed from rev. 1: Invoices moved ahead of Users/Subsidiaries. Invoices has a
> real typed create endpoint; Users has none, so it is now gated on a backend
> answer rather than being "small and easy".

**Caveat on VendorPage.ts:** `AGENTS.md` names the existing
`pages/web/admin/VendorPage.ts` as its *anti-pattern reference* — it already
contains hashed CSS-module classes, `:nth-child` chains, and viewport hacks.
Slice 1 should rewrite it against `LoginPage.ts` conventions using the §3.1
replacement table, not extend it.

### Per-scenario porting recipe

1. Read the Cypress `.feature` and its step file to establish **intent**.
2. Rewrite the Gherkin into `tests/features/<module>/<name>.feature`:
   - First-person UI-narrative voice.
   - Retag to the target taxonomy: one suite tag (`@smoke` / `@regression`),
     one feature tag, one lane tag (`@ui` / `@api`). Lowercase, kebab-case.
   - Preserve Jira tags (`@PAY-961`, `@EXP-393`) — see §10.2.
3. Ground every selector by grepping `accrualify-reactjs` and
   `corpay-react-components-library` for `data-testid=` / `aria-label=`.
   Start from the §3.1 table. Log genuine gaps to the §4.6 audit list.
4. Write or extend the page object under `pages/web/admin/`, extending `BasePage`,
   with `readonly` locators initialised in the constructor. Use the shared
   `AgGrid` component object for any grid interaction.
5. Write the step definitions in `tests/steps/<module>/<name>.steps.ts` using
   `Given/When/Then` from `tests/support/fixtures.ts`. Shared step text goes in
   `tests/steps/common/`, defined exactly once across the tree.
6. Add `bdd:<tag>:qa` and `bdd:<tag>:stage` scripts to `package.json` for any new
   feature tag.
7. Run `npm run format`.

---

## 7. Phase 3 — Hardening and cutover

1. Add only the accepted batch selection to Stage CI as it lands, with exact
   counts and `--workers=1 --repeat-each=3 --retries=0`; a domain tag can include
   unaccepted mutations. Workflow changes still require approval.
2. Collect comparable scheduled Playwright evidence for the approved observation
   window. A mandatory Cypress overlap or matching pass/fail results is not
   required. Do not overlap suites that depend on shared mutable resources.
3. Retire corresponding Cypress coverage only after the source-to-target
   behaviour/assertion map, retry-free acceptance, scheduled evidence, and
   coverage-owner approval are complete. Keep any required uncovered behaviour
   in an explicitly owned remainder; a smaller pilot is not full replacement.
4. Finish only non-blocking hook improvements here. Required hooks must already
   have landed and been deployed before acceptance of the affected batch.
5. Port the self-heal pipeline last, if at all. The source repo's
   `scripts/self-heal/` tooling is substantial and `corpay-playwright/docs/self-healing-tests-plan.md`
   already scopes the Playwright equivalent. Treat it as a separate project —
   a well-built Playwright suite should need far less of it.

---

## 8. Verification

Per pull request:

1. `npx bddgen && npx playwright test --project=chromium-bdd --grep @<tag>` passes locally.
2. **`--repeat-each=3` passes** for the ported tag. A single green run is exactly
   how the Cypress suite reached its current state. This is the primary
   anti-flake gate.
3. `npx tsc --noEmit` clean.
4. `npm run format` produces no diff.
5. Automated grep of the diff for banned patterns: `waitForTimeout`, `setViewportSize`,
   `xpath=`, `nth-child`, `:eq(`, `.css-`, `.ag-row-first`, `console.log`,
   `process.env.` outside `utils/env.ts`. Worth wiring as a CI step early.
6. **Every `getByTestId` string in the diff resolves to a real occurrence in
   `accrualify-reactjs` or `corpay-react-components-library`.** This is now
   mechanically checkable with a grep — make it a review checklist item, or
   automate it.
7. Scenario passes on a cold run (no `playwright/.auth/admin.json`) *and* on a warm one, so the idempotent-login fallback stays exercised.

Per phase:

8. The full ported tag set passes three consecutive scheduled CI runs before the
   corresponding Cypress features are retired.
9. Wall-clock runtime tracked per phase. If it grows super-linearly, storage-state
   auth or seeding is regressing.

---

## 9. Scope boundaries

**In scope**
- Gherkin features, step definitions, page objects for the slices listed in §6.
- API seeding clients under `api/clients/` and endpoint constants in `api/endpoints.ts`.
- The ag-Grid component object and grid-state reset fixture.
- Instruction / prompt file corrections, including adding the component library
  to the documented source repos.
- The batched test-hook PRs from §4.6.
- CI workflow additions for newly ported tags.
- Company-scoped React/legacy page setup through Rails Console MCP on the
  existing test companies, following the safeguards in decision 6 below.

**Out of scope**
- General Rails API source exploration or changes. The React app covers endpoint paths, auth, and the
  invoice payload; it does **not** cover server-side validation, the
  purchase-order payload schema, or vendor/user creation. Those need a backend
  engineer or a captured request body (§4.3). Decision 6 permits only the relevant
  Rails feature-flag/module-setting inspection needed for MCP page setup, not
  broader backend development or API-contract discovery.
- Porting `setupEnvironment` / hardcoded feature-flag assertions (§3.4). This
  remains separate from the approved MCP-controlled page setup in decision 6.
- Porting the `scripts/self-heal/` pipeline (§7.5).
- Fixing app bugs surfaced by "Defer"-classified tests. File tickets instead.
- The external testmail.app email-verification commands — decide separately whether
  those scenarios move at all.
- Cypress's 4 UI file-upload helpers — Playwright's `setInputFiles` handles these,
  but they need per-scenario attention rather than a blanket port.
- Angular/legacy admin screens beyond login, until `accrualify-angularjs` is
  available locally or via `githubRepo`.

---

## 10. Open questions

1. **Parity target.** Full parity across all 48 feature files, or a prioritised
   subset (e.g. `@smoke` plus the top 3 business areas)? *Recommendation: prioritised
   subset. The Phase 1 inventory will show that full parity means porting a large
   number of already-failing and low-value combinatorial scenarios.*

2. **Jira tags.** The Cypress features carry issue tags (`@PAY-961`, `@EXP-393`).
   Preserve for traceability, or drop? *Recommendation: preserve — they are the only
   link back to the originating requirement.*

3. **Vendor and user creation (blocking §6 slices 1 and 5).** Does `POST /vendors`
   exist server-side? Is there any user-creation endpoint? Neither is reachable
   from the React app, and the Cypress suite creates vendors via UI. Needs a
   backend engineer's answer before slice 5 can start.

3. **Vendor and user creation (blocking §6 slices 1 and 5).** Does `POST /vendors`
   exist server-side? Is there any user-creation endpoint? Neither is reachable
   from the React app, and the Cypress suite creates vendors via UI. Needs a
   backend engineer's answer before slice 5 can start.

4. **Multi-user / roles.** Cypress scenarios switch users mid-scenario
   (`I am logged in with the "<User>" user without creating a session`) and mutate
   roles via API. Storage-state auth is single-user by default. Needs a decision on
   multi-role storage states before the Approvals and Users slices.

5. **Execution model.** One agent session per feature file with human review per PR,
   or a longer autonomous loop? *Recommendation: per-feature with review, at least
   through slices 1–3, until the prompt contract is proven.*

6. **Companies under test (decided 2026-09-24).** Keep scenarios on their existing
  approved test companies, with their existing users and credentials. Do not
  move scenarios or remap credentials to obtain
  React or legacy pages. Use Rails Console MCP to apply the reviewed,
  company-scoped page-flag selection before an exclusive test batch. Confirm
  environment and exact company IDs, capture prior override state, check
  user-level overrides and related module settings, and verify the effective
  page selection after fresh UI login. Never change global defaults or toggle
  flags during a conflicting Cypress, Playwright, CI, or manual run. Restore
  the captured state after the batch, including failures and originally absent
  overrides; interrupted cleanup blocks the next conflicting run. See the
  execution plan's company/page setup procedure. This decision does not itself
  change live flags or authorize unrelated company configuration changes.

7. **Deployed component-library version.** Both local declarations now read
   **1.2.4**; no local mismatch remains. Who can provide the deployed app/build
   and bundled library revision so Stage 0 can record verified release evidence?

8. **Angular repo.** `accrualify-angularjs` is not in the workspace. Add it, or
   accept that Angular-side grounding stays on the remote `githubRepo` tool?
