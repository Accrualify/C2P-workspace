# Migration Plan: Accrualify Cypress suite → Corpay Playwright

**Status:** Draft for review
**Date:** 2026-09-16 (rev. 2 — grounding sources verified locally)
**Source repo:** `accrualify-test-automation` (Cypress + cypress-cucumber-preprocessor)
**Target repo:** `corpay-playwright` (Playwright + playwright-bdd)
**Grounding repos:** `accrualify-reactjs`, `corpay-react-components-library`

---

## 1. TL;DR

Port the suite agentically, but drive each port from the **`.feature` file as the
specification** — not by transpiling the Cypress step definitions and page objects.

A file-by-file translation would faithfully reproduce the existing flakiness,
because the flakiness lives in the Cypress selector strategy and session
handling, not in the Gherkin. The Gherkin is the asset worth keeping; almost
everything below it should be rewritten against Playwright idioms.

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

---

## 3. Why a mechanical translation fails

### 3.1 The selectors are exactly what the target repo forbids — and real replacements exist

From `cypress/support/element_selectors/vendors/vendors.ts`. Every one of these
is on the forbidden list in `AGENTS.md` (§ Anti-patterns). The right-hand column
is a hook **confirmed present in the React source today**:

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

### 3.2 The page objects compensate for instability with retry scaffolding

`cypress/support/page_objects/vendors/vendors.ts` contains `cy.wait(1000)`,
conditional `cy.get("body").then($body => $body.find(...))` branching, a literal
"retrying toggle click" fallback, and an explicit retry counter
(`search(vendor, retries = 12)`). These are flake-suppression wrappers. Porting
them carries the instability across rather than fixing it.

### 3.3 The dominant flake cause is auth, not selectors

`docs/self-heal-flaky-history.json` in the source repo — the recurring signature is:

> `Timed out retrying after 30000ms: login form or authenticated app loaded: expected false to equal true` — *This error occurred while creating the session.*

That is `cy.session` failing, and it takes down entire unrelated scenarios
(change orders, purchase orders, etc.). No amount of selector translation helps.

### 3.4 The `@setupEnvironment` orchestrator is a second systemic flake source

`cypress/support/commands/environment_setup/api_commands.ts` (~800–900 lines)
asserts ~48 hardcoded feature-flag values and a hardcoded approval-workflow
allowlist against the live Stage API. When Stage flips a flag, **every** scenario
tagged `@setupEnvironment` fails in the background step regardless of what it
tests. The source repo's own `docs/self-heal-known-issues.md` documents this as
its top two known failure signatures.

**Do not port this pattern.** Environment config drift should be a separate
monitoring concern, not a precondition wired into scenario execution.

### 3.5 Known-bad tests are already labelled as such

- `docs/tests_status.md` is a pass/fail ledger; roughly half the payment-run rows read "Failing".
- `cypress/e2e/expenses/expenses.feature` contains scenarios literally titled
  `FLAKY Deleting a receipt` and `FLAKY Upload a new receipt`.

Use this as the triage input. Do not port red tests without first deciding
whether each is an app bug or a bad test.

### 3.6 Grid state persists in localStorage — a hidden cross-test flake source

`serverSideDataGrid.tsx` persists column state (visibility, order, width,
filters) to localStorage under a per-grid key — `GRID_STORAGE_NAME = "listVendor"`
for the vendors grid.

This explains an otherwise baffling Cypress pattern: `Vendors.search()` opens the
quick-filters dropdown, clicks **reset grid**, reopens it, clicks **clear
filters**, and only then types — on *every single search*. That is not test
logic, it is compensation for grid state leaking between tests.

**This gets worse in Playwright, not better.** Once storage state is shared via
`playwright/.auth/admin.json` (§4.2), persisted grid state is shared across every
scenario in the run by default.

Fix it once at the fixture level — clear the grid localStorage keys in a
`Before` hook — rather than re-clicking reset inside every page object method.
This belongs with the ag-Grid work in §4.4.

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
- **Decide what to do about persisted grid state (§3.6)** — shared storage state means shared grid config unless explicitly cleared.
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
- **Owns the localStorage grid-state reset from §3.6**, exposed as a fixture-level
  `Before` hook rather than a per-method click sequence.

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
  - No stable hook exists for an element → **append it to the 0.6 audit list and
    continue with the best available role/label selector.** Do not halt, and do
    not fall back to a positional or hashed-class selector.
- Forbid porting `cy.wait(N)`, retry counters, and conditional `$body.find()`
  branching. Replace with Playwright auto-waiting and web-first assertions.

### 4.6 Component-library and app test-hook audit *(parallel with 0.1–0.5)*

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
4. **Version skew:** the library repo is at 1.2.2; `accrualify-reactjs` consumes
   1.2.3. Confirm the local clone matches the deployed version before trusting it
   as ground truth.

Deliverable: a running list of missing hooks produced as a by-product of Phase 2
(fed by the 0.5 stop condition), plus one batched upstream PR per repo rather
than one PR per scenario.

---

## 5. Phase 1 — Triage inventory

Produce a machine-generated porting checklist before writing any test code.
Runs in parallel with Phase 0.

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

One feature area per pull request. Within an area, port scenarios in inventory
order (green first).

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

1. Add the ported tag to the CI workflow (`.github/workflows/playwright.yml`) as
   it lands, so coverage grows incrementally rather than in one cutover.
2. Run both suites in parallel for an agreed overlap window. Compare results per
   scenario; investigate any case where Playwright passes and Cypress fails, or
   vice versa.
3. Retire the corresponding Cypress feature only after its Playwright equivalent
   has been green in CI for the agreed window.
4. Land the batched test-hook PRs from §4.6 upstream, so the next wave of porting
   has fewer gaps.
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
7. Scenario passes with a cold grid state *and* with a polluted one (§3.6).

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

**Out of scope**
- The Rails API source. The React app covers endpoint paths, auth, and the
  invoice payload; it does **not** cover server-side validation, the
  purchase-order payload schema, or vendor/user creation. Those need a backend
  engineer or a captured request body (§4.3).
- Porting `setupEnvironment` / feature-flag assertions (§3.4). Needs a separate design.
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

4. **Multi-user / roles.** Cypress scenarios switch users mid-scenario
   (`I am logged in with the "<User>" user without creating a session`) and mutate
   roles via API. Storage-state auth is single-user by default. Needs a decision on
   multi-role storage states before the Approvals and Users slices.

5. **Execution model.** One agent session per feature file with human review per PR,
   or a longer autonomous loop? *Recommendation: per-feature with review, at least
   through slices 1–3, until the prompt contract is proven.*

6. **Companies under test.** Cypress hardcodes "Automation Client 1" and
   "Automation Client NVP" with different feature-flag and workflow expectations.
   Does the Playwright suite need the same multi-company matrix?

7. **Component library version skew.** The library repo is at 1.2.2;
   `accrualify-reactjs` consumes 1.2.3. Should the local clone be updated before
   it is used as selector ground truth?

8. **Angular repo.** `accrualify-angularjs` is not in the workspace. Add it, or
   accept that Angular-side grounding stays on the remote `githubRepo` tool?
