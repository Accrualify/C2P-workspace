# Migration Execution Plan

**Created:** 2026-09-19
**Last implementation update:** 2026-09-25
**Last plan review:** 2026-09-25 (source-to-target review; no new live or CI validation)
**Companion to:** [cypress-to-playwright-migration-plan.md](cypress-to-playwright-migration-plan.md)

That document is the **strategy** — why the migration is shaped the way it is,
what was found in each codebase, and what the design constraints are. Read it
once, then work from this one.

This document is the **execution plan**: what to do next, in what order, and
what has to be true before each stage is considered finished. Stage 0 first
reconciles existing ports, active authoring instructions, missing foundations,
and deployed versions against the intended checkout. Preliminary, read-only
inventory can run alongside that work; live pilot validation waits for the
isolation, current-scenario authentication, and foundation gates.

### Local implementation and CI deferral (2026-09-25)

Continue the remaining **local Phase 0 work**; the existing environment, session
authentication, grid, runner, and reporting foundations do not need rebuilding.
Start with the selected base E2E source-to-target map and its React routing
requirements. Preserve the existing port and all agreed lifecycle assertions.

The user deferred the outstanding live/CI sign-off exercise for now. CI provider
selection and integration remain open to **GitHub Actions or AWS CodePipeline
with CodeBuild**. Keep both existing definitions as options; do not deploy,
dispatch, provision cloud secrets, or require successful runs on both providers
to continue local implementation. The common runner remains provider-neutral.
The user also leaves **QA versus Stage usage on either provider undecided**;
the available template defaults are not an approved deployment strategy.
Automatic GitHub PR/push checks are offline-only. Live GitHub execution requires
manual dispatch with `run_live=true`, preventing this foundations PR from
starting an unapproved application run. AWS is not deployed by this change.
Deferred evidence is not a pass: the live foundation checks still precede live
pilot acceptance, and the selected provider must be verified before CI sign-off
or scheduled Cypress retirement. Local static and offline checks continue now.

The **other Rails MCP chat owns company page-routing flag changes and
restoration**. This implementation chat supplies the required React flow and
consumes its recorded handoff; it must not become a second flag writer. Before
an affected live run, require the selected environment/company/user, reviewed
flag delta, original snapshot, effective-state verification, exclusive window,
and restore owner/procedure. A baseline snapshot or this instruction alone is
not evidence that the requested flags have been applied. Use the handoff
contract in [Company and page setup](#company-and-page-setup-through-rails-console-mcp).

---

## How the work runs

Each port is one agent session driven by
`.github/prompts/migrate-cypress-feature.prompt.md` in `corpay-playwright`. That
prompt pins the contract so every run behaves the same way: agreed behaviours
and assertions are the specification. Read the source `.feature` and its steps
to uncover setup, approval flows, and assertions hidden behind short step names.
Map existing Playwright coverage before adding another port. Ground selectors
in the matching React or Angular source and shared component library, then
verify them in the deployed application; do not copy Cypress selectors.

Use review-sized batches within one domain per pull request, not a whole domain
in one large PR. Each batch has a bounded behaviour/assertion map and its own
acceptance evidence. Human review on every one, at least through Stage 3.

### Migration scope decision (2026-09-21)

**Implementation update (2026-09-23):** The user approved strict QA-default
environment selection, grid response-completion checks, and shared GitHub/AWS
CI commands with configurable isolated workers/shards, 75-minute run and
90-minute job limits, first-failure traces, and full pass/fail reports.
The configuration and scripts are now implemented locally. This supersedes
the earlier requirement to obtain permission for those specific CI/config
edits, not the account-allocation, pilot-scope, or deployment acceptance gates.
Default live validation stays sequential; parallel capability is not proof
that shared-role scenarios are safe to run concurrently.

The existing `.env.qa` already contained QA settings. The generic `.env`
contained Stage settings, so it was preserved without renaming or overwriting
credentials and is now ignored. Explicit environments never fall back.

The shared runner produces unique HTML, JSON, JUnit, Markdown summaries,
selection manifests, and failure artifacts; it checks exact selected versus
reported executions and rejects skips, retry-only passes, expected failures,
and incomplete runs. GitHub's default selection is the six-case foundation;
manual custom selections and the CodeBuild runner support isolated workers
and shards. The existing Okta/TOTP case remains explicitly excluded, not
accepted as passing. Cloud credentials/resources and actual cloud runs are
still pending. See `corpay-playwright/ci/README.md` for setup and evidence.

Vendor filtering now requires a newly issued matching request, successful
response, `meta.count === 1`, and the matching rendered identity. Offline
tests cover delayed/failed responses and virtualized extra matches. This
repairs the helper contract; it does not approve or complete the Stage 1
coverage-preserving pilot.

**Local verification for this implementation:**

| Check | Result |
|---|---|
| Pinned install/runtime | `npm ci` succeeds with Node `24.21.0` and npm `11.19.0` |
| Offline regression acceptance | 54 cases x 3 repeats = **162 passed**, 3 workers, zero retries/skips |
| First-failure trace evidence | Three controlled child failures retained nonempty traces; each was asserted by a passing parent regression |
| Reports | HTML, JSON, JUnit, selection manifest, full Markdown, compact CI summary, and trace attachments verified present |
| Native shard selection | 27 + 27 = 54 unique offline cases, no duplicate or missing identities |
| QA and Stage foundation discovery | Exactly six selected cases / 18 executions in each environment; listing only, no new live run |
| Static validation | TypeScript, BDD generation, GitHub/CodeBuild YAML, and runtime/timeout/artifact checks pass |

Offline report: `corpay-playwright/reports/local/offline-2026-09-23T19-50-44-137Z-7cac742f/`.
These are local results, not successful GitHub/AWS executions or new deployed
selector evidence. No cloud resources were deployed and no commits/pushes were
made. The lockfile's two transitive high-severity dependency advisories remain
a separate maintenance item; they were not silently fixed with broad upgrades.

**Phase 0 closeout fixes (2026-09-24):**

- Playwright's soft timeout now precedes the hard run deadline by 70 seconds:
   60 seconds for teardown/report flushing and up to 10 seconds for termination.
   Controlled child-process tests verify graceful report writing, forced shutdown,
   and rejection when the remaining budget cannot accommodate cleanup.
- Reports retain actual skipped, interrupted, timed-out, and not-run outcomes;
   `expectedStatus` is recorded separately. Expected failures and unexpected
   passes remain distinct and cannot mask failed acceptance.
- Invalid arguments, missing CI environment selection, and setup errors now
   produce failed Markdown/JSON summaries plus diagnostic HTML and JUnit. Errors
   before environment resolution use `reports/setup/`. Diagnostic JUnit cases
   represent runner failures, not executed application coverage.
- `npm run report:last-trace` now selects the newest nonempty trace from the
   current per-run artifact directories. It supports a report-directory argument
   and `--print`, and ignores the obsolete `test-results/` location.

Verification: **73 offline cases x 3 repeats = 219 passed**, three workers,
zero retries/skips; TypeScript and focused report/runner checks pass. Report:
`corpay-playwright/reports/local/offline-2026-09-24T13-31-11-845Z-63da6ab6/`.
The existing first-failure trace regression also passes. Timeout coverage uses
controlled child processes; a separate nested Playwright global-timeout probe
was inconclusive and is not counted as accepted evidence. No new live Stage
tests, cloud runs, or infrastructure changes were performed by these fixes.
All remaining account, pilot, deployed-build, and cloud-validation gates below
remain open until their evidence is recorded.

**Runner boundary fixes (2026-09-24):**

- Commands now own separate process groups. Timeout and SIGINT/SIGTERM cleanup
   discovers and terminates descendant groups, including detached workers, and
   waits for the observed processes to stop instead of returning when only the
   parent exits. The cleanup/report grace period remains intact. Offline tests
   exercise normal and detached descendants, both cancellation signals, graceful
   cleanup, and forced shutdown. A process-reaping race found during repeated
   validation was fixed; a subsequent full run passed without retries.
- Sharded discovery first validates the full foundation manifest. Playwright
   then lists all native partitions, whose combined identities and repetitions
   must match the full selection with no omissions or duplicates. A `1/1` shard
   retains all six-case checks. Full and partition manifests are retained, and
   reports distinguish one shard's results from whole-suite acceptance.
- Managed execution uses Linux, macOS, or WSL with `ps`; native Windows is
   rejected before spawning tests rather than promising partial tree cleanup.
   Both supplied cloud runner definitions use Linux. External SIGKILL/container
   eviction still requires the documented shared-state recovery procedure.

Verification: **78 offline cases x 3 repeats = 234 passed**, three workers,
zero retries/skips. Both offline shards also executed successfully (**39 + 39**),
and their result identities reconcile to all 78 cases without duplication.
QA `1/1` discovery verified 18 foundation executions; Stage `1/2` discovery
verified the full 18 plus both native 9-execution partitions. These QA/Stage
checks were listing only; no live login or application request was made.

Full report: `corpay-playwright/reports/local/offline-2026-09-24T14-11-17-429Z-c9af866e/`.
Shard reports: `corpay-playwright/reports/local/offline-2026-09-24T14-11-37-458Z-91b2257d/`
and `corpay-playwright/reports/local/offline-2026-09-24T14-11-41-824Z-ce4ebfef/`.
Cloud deployment, actual Stage/cloud acceptance, and pilot scope/data approval
are not certified by these offline results and remain pending below.

### Pre-PR fixes and verification (2026-09-25)

The CI provider and QA/Stage usage remain undecided. The private Playwright
foundations PR retains both provider options, but the live GitHub job now
requires explicit manual `run_live` opt-in. Opening a PR cannot start the live
job. This does not deploy AWS, choose a production CI strategy, or claim live
integration acceptance.

The repeated offline review found one synchronous `ps` timeout (305 passes,
one failure). Process inspection is now asynchronous and single-flight, with
independent hard-kill and shutdown-confirmation timers. A stalled scan is
cancelled at the hard deadline so it cannot consume the confirmation window.
Transient inspection failure may recover within that window; persistent failure
still reports unconfirmed shutdown and requires recovery. Regression tests
exercise both outcomes without extending the hard deadline or adding test retries.

Repeat labels now use project, file/location, and the full scenario title path,
while native execution IDs remain the exact-match keys. Selection-based labels
survive missing/reordered results, and shards use the full discovery manifest.
The original failed report was reprocessed in memory to confirm 102 executions
at each of repeats 1, 2, and 3 with its original 305/1 outcomes unchanged.

Final local gate: **107 cases x 3 repeats = 321 passed**, three workers, zero
retries or skips. Report:
`corpay-playwright/reports/local/offline-2026-09-25T15-13-24-414Z-b2288329/`.
A native shard of the mocked React-login case also passed; its third execution
retains repeat label 3 from the full selection. Report:
`corpay-playwright/reports/local/offline-2026-09-25T15-14-37-792Z-6ba75b69/`.
Earlier failed evidence remains intact. TypeScript and focused report/cleanup
checks pass. These results do not certify real staff login or the payment E2E.

**Publication boundary:** this public plan records scope, decisions, and
aggregate verification results only. Environment snapshots, account identifiers,
exact flag profiles, and operational restore records remain private. The Stage 0
implementation PR targets the existing migration branch so earlier migration
commits are not included in the foundations checkpoint's review.

### E2E-first, React-only decision (updated 2026-09-25)

The user selected the base **E2E payment lifecycle** as the first implementation
target, replacing the earlier vendor-search pilot. Reuse Automation Client 1
and Automation Client NVP with the corresponding Cypress named-user identities;
no new automation companies or silent staff-to-admin aliases are required.

Named-user configuration was completed by the parallel account-configuration
work and verified here without displaying or rewriting credentials: all 12
complete QA/Stage pairs (`staff`, `service`, `roles`, `okta`, `nvp_service`,
`vendor_portal`) resolve to the same email/password pairs as their selected
Cypress configuration. The `dummy` source entry intentionally has no password;
no dummy login is planned, and an attempted login fails closed instead of using
admin. The user excluded dedicated Okta/2FA coverage on 2026-09-25 due to setup
issues; no MFA provisioning is required for the supported selection. The
retained scenario is statically skipped. All 33 environment/named-user regressions
and `tsc --noEmit` pass. This resolves the local named-user mapping issue, not
live authentication, cloud secret provisioning, or effective flag evidence.

The 2026-09-25 correction requires **React for every application page in the
selected E2E, including login**, list, form, detail, approval, and payment
actions. Start at `/login` on the configured React origin, not the same path
on Angular. No Angular-login, post-login redirect, or business-page exception
is allowed. If a required React capability is unavailable, record a blocker
instead of substituting Angular, an API business action, or weaker assertions.
Keep full UI credential entry and a fresh browser context for every run.
Existing Angular code is legacy implementation, not an accepted destination.

Use Rails MCP for the reviewed company-scoped A2R page-routing profile. Inspect
user overrides, capture exact original values and row presence, verify effective
flags and routes after login, and restore the original state without overwriting
concurrent edits. Do not change global defaults, security or payment behaviour,
module settings, account membership, or roles as part of page selection. Keep
the profile stable and prevent conflicting Cypress/CI/manual runs during use.

**Next action:** map the source E2E lifecycle to existing React routes and target
coverage, publish the routing requirements for the Rails MCP chat, and resolve
its data, payment-sandbox, and recovery prerequisites. Prove full UI
login and API authentication with the selected company account before any live
E2E run. The company/first-scenario decisions are settled; they do not certify
flag changes, payment execution, or the remaining foundation gates.
The current goal is agreed coverage, correct behaviour, and repeatable
Playwright tests. **No parallel-execution framework needs to be designed now.**
Custom parallel scheduling is deferred to Stage 4; native worker/shard options
are available now for verified isolated selections. Account allocation, owned
data, and failure-safe restoration are prerequisites for mutating runs,
including the first vendor run, not work to postpone until Users or Stage 4.

During migration, validate focused scenarios or feature slices sequentially
with `--workers=1 --retries=0`, including repeat runs and CI. One worker only
serialises that one process: it does not stop Cypress, another Playwright run,
CI, or a person from changing the same account. Reserve non-conflicting accounts
and data, or agree an exclusive run window across those users and suites.
Include readers of shared settings in this check, not just writers. This is a
simple allocation rule, not a requirement to build a scheduling framework.

Each port must still use test-owned data where possible and restore shared
state after each scenario, including failures. If cleanup fails or a run is
interrupted, restore a known-good state before retrying or starting the next
conflicting scenario. Fresh browser contexts do not reset server-side data.

Acceptance is based on the agreed behaviours and the application's expected
results, not matching Cypress pass/fail results. There is no mandatory
dual-suite overlap window. Consult Cypress selectively where scenario intent
is unclear. This scope decision takes precedence over the companion strategy
on execution parity and parallel-execution timing.

**Coverage parity** means preserving the selected source scenario's behaviours,
preconditions, and business assertions in an explicit source-to-target map.
It does not mean matching Cypress's pass/fail results or copying its mechanics.
Any smaller scope needs explicit approval and must remain labelled partial;
approval of a small practice test does not establish full replacement coverage.

### Account and data safety

Before any live selection, identify its company, accounts, data dependencies,
and all setup/cleanup hooks. Title or tag filtering does not disable unscoped
hooks imported from other features. The whole `@vendor` tag is not an
authentication smoke: it includes vendor creation and shared-user role changes.

Before the first mutating slice, record and verify:

- An owner and allocated account for each required role, plus either dedicated
   resources or an agreed exclusive window with Cypress, CI, and manual users.
   Do not change a shared administrator's roles without explicit approval.
- Test-owned records with a unique run identity and company boundary. Cleanup
   must target those exact records, never everything matching a broad name.
- The original shared roles/settings and a confirmed way to restore them.
   Tag-scoped fixture teardown or `finally` cleanup must run on test failure;
   verify the restored values instead of assuming the cleanup request worked.
- An interruption recovery owner and procedure. Teardown cannot guarantee
   recovery after a killed process. Failed or interrupted restoration blocks
   every conflicting run until the original state is verified again.

Missing account allocation or a safe cleanup operation blocks the mutating
slice. Do not guess API contracts or remove assertions to get around the block.
Read-only foundation runs need the same hook and conflict check, but do not
need unused vendor-creation clients or a parallel scheduler.

### Acceptance evidence

For each accepted selection, record the environment, test/source revisions,
deployed app/library revisions, exact tags and titles, and expected case count.
Expand Scenario Outline examples when counting. Use the same selection and
repeat flags with `--list` before the live run; verify the actual selected
titles as well as the number. Tags alone are not an allowlist of safe tests.

Acceptance uses `--workers=1 --repeat-each=3 --retries=0`. For N agreed cases,
require exactly 3 x N executed passes, with no extra cases, skips, unexpected
failures, expected-failure masking, or retry-hidden failures. A zero-test run
is not green. Keep reports/traces and selected/executed counts with the result.
The earlier six-case foundation result below remains valid historical evidence,
but is not evidence that these new repeat-run or CI gates have passed.

Before retiring Cypress coverage, require comparable **scheduled Playwright
runs**: the agreed environment, coverage, data preconditions, count, and strict
flags must match. Agree the observation window before retirement; three
consecutive scheduled runs is the proposed minimum, not an approved decision.
No Cypress/Playwright pass-fail comparison or mandatory overlap is required.

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

**Goal:** reconcile existing ports, active instructions, and local foundations;
record deployed evidence and prove the agreed foundations before live acceptance.
The 2026-09-25 decision defers the live sign-off exercise and cloud integration,
not the implementation work or the safety requirements for a later live run.
Preliminary inventory and other read-only source analysis can proceed alongside
Stage 0. This does not authorise overlapping live suites: pilot validation and
mutating migration runs remain blocked by the relevant safety and auth gates.

### Review evidence (2026-09-23)

This plan review is **static**. It adds no new proof of live Stage behaviour,
deployed app/library revisions, or successful CI execution. The dated
2026-09-22 results below are retained as historical evidence for only the
selected foundation cases; they do not certify the pilot, other ports, today's
deployment, or the stricter repeat-run/CI gates.

The local component-library package and the React app's declared dependency
both read **`1.2.4`**. The earlier `1.2.2`/`1.2.3` mismatch is outdated, not an
open local upgrade task. Matching local manifests do not establish what Stage
is running. Record deployed build/release identifiers separately; unavailable
deployment evidence stays **Unknown** and blocks acceptance of affected hooks.

### Implementation record (2026-09-22)

Stage 0 is **partially implemented and locally validated**, not complete.
Stage 1 has not started. No branch changes, commits, or pushes were made.
The plan corrections below add requirements; they do not certify new test or
CI runs. Unchecked gates stay open until their evidence or approval is recorded.

| Checkout | Verified baseline |
|---|---|
| `corpay-playwright` | `cypress-to-playwright-migration` at `5e08055e3852a2f96c5c86b1e181ae9e6ab780c5`; clean before this implementation |
| `C2P-workspace` | `docs/migration-execution-plan-phase-0` at `b9bb36ddb56e52809e67677a4ca3e1cadf4d59b9`; clean before this implementation |
| Cypress intent source | `accrualify-test-automation` at `5e95ced1` |
| React grounding source | `accrualify-reactjs` at `df145c8a8` |
| Component-library grounding source | `corpay-react-components-library` at `415f3e8`; both its local version and the app dependency are `1.2.4` |

Implemented in the Playwright worktree:

- Canonical, scoped, agent, and prompt instructions now use `playwright-bdd`,
   module-relative fixture imports, full UI login, and sequential validation.
   The migration prompt is present and explicitly preserves the read-only
   pilot scope. Formatting is scoped to changed files.
- `api/clients/apiClient.ts` creates an authenticated request context from
   the current scenario after login. `userClient.ts` implements the existing
   read-only `GET /user` operation; the lazy BDD fixture disposes its contexts.
   It does not load saved sessions or fall back to another run's token.
- React authentication uses the active origin's `Token` and hydrated
   `userDetails.company.id`, matching the app's request-header implementation.
   Angular storage parsing is covered offline. The protected React-page smoke
   verified a successful API read and matching returned company identity.
- `pages/components/AgGrid.ts` provides shared semantic grid readiness,
   filter-menu actions, text filtering, and exact-cell row assertions.
   `VendorPage.openVendorList()` uses its readiness check. The existing search
   implementation has not yet been accepted as the Stage 1 pilot.
- Live validation exposed two unscoped profile cleanup hooks running during
   login tests. Their Before/After hooks now apply only to `@update-profile`
   and `@profile-email`; the same foundation selection then passed.

| Validation | Result |
|---|---|
| Session parsing and read-only API client, offline | 12 passed; includes cross-origin rejection and redirect/header behavior |
| Shared grid helper, offline DOM | 3 passed; no setup dependency or live application requests |
| Stage login and protected-page foundation | 6 passed, one worker, zero retries: five runnable login cases plus the vendor-page/API smoke |
| Shared grid readiness on Stage | Protected vendor-page/API smoke passed after integration |
| BDD step resolution and TypeScript | `bddgen` and `tsc --noEmit` passed |

The existing `@okta @incomplete` login case was excluded because dedicated
Okta credentials/TOTP are not provisioned. This is not a claim that every
login scenario is green. Existing `.env.stage` and `.env` files were reused;
no credentials were copied, displayed, or changed.

Reproduce the six-case Stage foundation selection:

```sh
TEST_ENV=stage BDD_TAGS='(@login or @vendor) and not @incomplete' npm run bdd -- --grep 'Successful login and logout|Attempt to login with (an invalid|a blank) (username|password)|Vendors list page shows the expected components' --workers=1 --retries=0
```

First add `--list` and verify six selected cases. BDD appends tags to titles
for grep matching, so an end-anchored scenario-name filter can select zero.

**Remaining before Stage 0 sign-off:**

- Reconcile existing target scenarios with source behaviours, starting with
   the proposed pilot. Record reuse/repair/gaps before commissioning a new port;
   the broader preliminary inventory can continue alongside Stage 0.
- Map the selected base E2E lifecycle and its required React routes, approvals,
   data, and recovery operations. E2E is the approved first target; exact
   provisioning, safe payment destinations, and reversible flag settings remain
   prerequisites for live execution. The read-only API client proves
   authentication, not readiness of write clients or the payment workflow.
- Complete the active-instruction audit and account/hook safety checks below.
  Reserve the foundation accounts now; record mutating vendor runs as blocked
  until their account allocation and restoration proof are ready.
- Cloud integration is deferred to the separate backlog below. GitHub Actions
   and AWS CodePipeline/CodeBuild remain alternative consumers of the same
   runner and report contract, not two deployments required for local Phase 0.
   Do not mark either provider accepted based on local checks.
- Keep the dedicated Okta/2FA case excluded per the 2026-09-25 user decision;
   it is not a remaining provisioning task. Verify deployed hooks/revisions for the
  selected foundation; unresolved Angular access blocks affected later slices,
  not unrelated React-only work.

### Base E2E source-to-target map (2026-09-25)

**Decision: repair existing.** Map source `cypress/e2e/base/e2e.feature`,
scenario `e2e`, to the existing `tests/features/base/e2e.feature`, scenario
`E2E` (`@base-e2e`). Do not create a duplicate flow or silently expand it into
a second-company matrix. The detailed source audit, test-data values, source
revisions, and operational evidence remain in the private implementation review.
This public overview is a coverage contract, not accepted runtime evidence.

| Map ID | Required behavior | Acceptance requirement |
|---|---|---|
| `E2E-01` | Full UI login through React with the approved named user. | Verify the React landing, current session, and matching company; no saved-session or Angular fallback. |
| `E2E-02` | Use the agreed existing subsidiary. | Verify its identity without silently provisioning or substituting another record. |
| `E2E-03` | Create and approve the linked vendor. | Use React UI, preserve contact and payment-method preconditions, and confirm sandbox destinations. |
| `E2E-04` | Create and approve the linked purchase order. | Preserve the agreed item values, terms, dates, and vendor/subsidiary relationships. |
| `E2E-05` | Create and submit the linked invoice. | Preserve the agreed values and PO relationship through the real React form and inbox. |
| `E2E-06` | Create, submit, and approve the credit memo. | Preserve its line values and terms; verify availability through the supported workflow. |
| `E2E-07` | Apply the agreed credit amount to the invoice. | Verify the exact credit memo, invoice, and amount applied. |
| `E2E-08` | Submit the selected invoice into a payment run. | Verify the resulting link and processing-date prerequisites; do not force statuses. |
| `E2E-09` | Approve the created payment run. | Select the exact run, not an arbitrary first row. |
| `E2E-10` | Complete required separate linked-payment approvals. | Use the appropriate React approval flow for each exact linked payment. |
| `E2E-11` | Verify every final record status. | Run `CLOSED`, payment `PROCESSING`, invoice `PAID`, PO `CLOSED`, and credit memo `AVAILABLE`, using exact linked identities and independent expectations. |
| `E2E-12` | Keep each lifecycle independently runnable. | Use test-scoped state, semantic waits, shared grid helpers, and verified cleanup or compensation after success/failure. |

**First local implementation slice:** the E2E API wrapper now uses the existing
current-session factory in `corpay-playwright/api/clients/apiClient.ts`.
It waits for the active origin's token and matching company rather than scanning
other origins or borrowing saved authentication. All **15 existing session-auth
regressions pass**. TypeScript, scoped Prettier, and QA-configured BDD generation
pass; an AST check confirms one generated E2E case, the shared factory call,
preserved context disposal, and removal of the legacy fallback. Generation did
not execute a browser or a live test. No selectors, business actions, feature
flags, or credentials were changed by this slice. The other mapped repairs and
the live acceptance gates remain open.

**React login correction (2026-09-25):** the E2E now selects the explicit shared
React login step, starting at `/login?redirect=%2Fap%2Fvendors` on the configured
React origin. The existing `LoginPage` supports this mode without changing
unrelated legacy callers. A guarded inspection of QA's deployed public React
login verified unique username, password, Continue, and Login controls. Only
the identifier-check response was mocked; no real credentials were entered,
the Login button was not submitted, and outbound writes were blocked. This
proves deployed controls, not real account authentication or post-login routing.

All **18 session-auth regressions pass**, including full mocked credential
entry, a React landing/current session, blocking a direct Angular navigation,
and rejecting a non-React destination before any request. TypeScript and BDD
step resolution pass. The E2E has no `@angular` tag and is intentionally retained
with `@skip @incomplete` while its business-page migration is unfinished.
`bddgen` omits an all-skipped feature; generation-only verification without the
skip tag resolved all E2E steps before that guard was restored. This is not an
E2E pass, and it does not certify all redirect mechanisms or live navigation.
No company flags were changed by this login work. Live access readiness is
tracked privately and is not certified by the mocked tests.

### Phase 1 started (2026-09-25)

To avoid stacked pull requests, the first Phase 1 slice is included in the
single Stage 0 commit of the private Playwright foundations PR. It moves base
E2E record state into a test-scoped fixture, aligns the subsidiary precondition
with the selected source scenario, removes PO/invoice status forcing and
invoice request rewriting, and makes invoice payment eligibility an
observation-only poll of the exact record. These changes do not complete the
React UI migration.

All **20 fixture/authentication regressions pass**, including two tests that
verify fresh fixture state across consecutive scenarios. The full offline suite
passed **109 cases x 3 repeats = 327** with three workers and zero retries or
skips. Report:
`corpay-playwright/reports/local/offline-2026-09-25T18-40-09-680Z-7c3f1a5d/`.
This is local verification, not a live E2E result or a cleanup proof.

Live login, deployed page checks, and writes remain paused until the private
handoff confirms a run window and the applied routing profile. The E2E retains
`@skip @incomplete`; its legacy page objects, credit-memo approval path, and
cleanup still need repair. No live login, flag change, or payment was made as
part of this slice.

### E2E routing request for the Rails MCP chat

The exact source-backed routing candidates and per-environment delta belong
in the private operator handoff, not this public plan. Every application page
in the selected flow, including login and its landing, must use React. A route
declaration alone does not establish deployed availability, permissions,
selector uniqueness, or a need to change a flag. Already-enabled routing flags
need no redundant override; a company flag alone may not control pre-login
navigation.

**Handoff status: live applied-profile evidence is still required.** Before any
write, the operator must refresh the selected environment's original-state
snapshot, verify company/user identities and higher-priority overrides, and
record the reviewed insert/update/no-op and restore actions privately. Do not
reuse another environment's company IDs or snapshot. Keep module settings,
approval workflows, payment/security behavior, global defaults, and unrelated
flags unchanged under this page-routing request. Any additional setup requires
separate approval. This chat will wait for the ready handoff and a stable run
window before browser validation; no per-test MCP or flag-toggling hook is added.

### Foundation tasks

1. **Reconcile Phase 0 with the intended checkout.** Record the intended
   branch/commit and confirm the API clients required by the agreed slice,
   shared `AgGrid`, and migration prompt are present. Locate or complete missing
   work before the pilot; do not assume the strategy's completion claims match
   this checkout. Start a preliminary source-to-existing-target inventory now,
   rather than waiting until Stage 2. Recount feature/scenario/example totals,
   identify duplicate or partial ports, and map the pilot's preconditions and
   assertions in detail. Carry this same inventory into Stage 2 for broader
   coverage decisions; completing the full inventory is not a Stage 0 gate.
   Do not require unused write clients for an approved read-only selection.
2. **Audit all active authoring instructions and templates.** Check `AGENTS.md`,
   `.github/copilot-instructions.md`, scoped instructions, agent files, prompts,
   and README/template examples against `tests/support/fixtures.ts`.
   They must consistently use `createBdd(test)` / `playwright-bdd`, fixture-based
   callbacks, and full UI login through the shared steps in a fresh context.
   No active example may prescribe `@cucumber/cucumber`, `CustomWorld`, saved
   BDD sessions, or a no-op login. Historical/comparison examples must be clearly
   labelled. Record the audited files; verify `bddgen` and `tsc --noEmit`.
   The current canonical rules have already been corrected; this explicit gate
   prevents another template from reintroducing the old model.
3. **Establish account and data safety before running anything live.** Apply
   the account/hook checks above to the exact foundation selection. Record the
   account/company owner and run window. Before any mutating vendor validation,
   prove owned-data cleanup and restoration after a controlled failure. Keep
   that slice blocked until the proof exists; do not wait for the Users stage.
4. **Reuse the existing environment files; create only missing ones.** Map
   approved allocated accounts to `ADMIN_EMAIL` / `ADMIN_PASSWORD` and named-user
   settings, with the intended `ANGULAR_BASE_URL`, `REACT_BASE_URL`, and
   `API_BASE_URL`. Do not assume sharing Cypress credentials is safe, overwrite
   existing files, or put secrets in this plan or chat.
   > ⚠️ For a per-environment file the name is **`.env.stage`**, not
   > `.env.staging`. The template and folder-structure example now use
   > `.env.stage`, matching the scripts and config. A `.env.staging` file is
   > not loaded by a `TEST_ENV=stage` run.
5. **Confirm source access and deployed selector evidence.** The selected E2E
   needs React source for login and every application page, plus the component
   library for shared controls. Missing React capability is a blocker, not
   permission to use Angular. Other legacy selections still need their own
   source grounding and do not establish exceptions for this E2E. Do not copy
   an Angular dependency just because it exists in an old page object. If no
   supported, source-grounded route is available, stop that action and record
   the gap. Record local source commits/package versions
   separately from the deployed app build and bundled library revision, using
   release/build evidence from the deployment owner if needed. Both local
   library declarations are `1.2.4`; deployed versions remain Unknown until
   verified. Verify rendered hooks, uniqueness, and visibility in that deployed
   build. Required upstream hooks must land and be deployed **before the
   affected slice**, not wait for Stage 4.
6. **Run the required Stage login selection.** Dedicated Okta/TOTP coverage is
   excluded by the 2026-09-25 decision and retained with `@skip @incomplete`;
   do not count it as passing. Generate/list the approved titles first, then
   run with `--workers=1 --repeat-each=3 --retries=0`. The five runnable login
   cases plus the protected smoke below mean six cases and 18 executions.
   Reintroducing Okta requires a new scope decision and count review.
7. **Include exactly one read-only protected scenario in that selection.**
   Select "Vendors list page shows the expected components" by title using
   `--grep`. Do not use the whole `@vendor` tag as an authentication smoke.
   Confirm the shared Given performs the full email, Continue, password, and
   Login flow from a fresh context, then reaches the protected page. It must
   not skip credential entry or depend on a saved session.
8. **Verify API authentication and the company nonce.** Use the current
   scenario's active-origin token and matching company after full UI login.
   The implemented React read client uses `Token` and hydrated
   `userDetails.company.id`; Angular has separate storage parsing. Prove a
   company-scoped read succeeds in that same scenario and returns that company.
   Record the login/API evidence before validating the pilot. Do not use another
   run's token, an unrelated fallback nonce, or a saved browser-session file.
9. **Keep CI integration deferred and provider-neutral.** Retain the shared
   runner, generation, exact-count checks, and private report contract for
   local use. GitHub Actions or AWS CodePipeline/CodeBuild can be selected
   later without redesigning the tests. Do not dispatch or deploy a provider
   as part of this local work. The selected provider's credentials, isolated
   run window, success/failure artifacts, and strict accepted selection belong
   to the deferred integration backlog, not an implementation prerequisite.
10. **Keep formatting scoped to changed files.** This was decided on
    2026-09-22; no repo-wide formatting commit is required.

**Exit gate**
- [ ] Intended branch/commit and agreed pilot scope recorded; required clients, shared `AgGrid`, and migration prompt present.
- [x] Initial base E2E source-to-target behaviour/assertion map and repair-existing decision recorded above; runtime and deployed coverage are still unaccepted.
- [ ] All active instructions/templates audited for `createBdd`, fixture callbacks, and full UI login; `bddgen` and `tsc --noEmit` pass.
- [ ] Foundation accounts/run window allocated and imported hooks checked; mutating slices remain blocked until their ownership and failure-restoration checks pass.
- [ ] Local source versions and deployed app/library revisions recorded separately; required foundation hooks verified and missing Angular access gates affected slices.
- [ ] Okta disposition recorded and required Stage selection passes with `--workers=1 --repeat-each=3 --retries=0`, exact counts, and no hidden skips.
- [ ] Under the current accepted run, isolation is established before login and a company-scoped API read is proven in the same fresh scenario, before pilot validation.
- [x] One read-only protected scenario green with `--workers=1` after full UI login from a fresh context, without saved-session reuse.
- [x] The new read-only BDD client receives the current scenario's usable token **and** matching company nonce; existing domain-specific helpers are not certified by this check.
- [x] One `api/clients/*` read call succeeds against Stage and returns the active company.

**Deferred CI integration backlog (not a local Phase 0 blocker)**
- [ ] Select GitHub Actions or AWS CodePipeline/CodeBuild; both remain options.
- [ ] Provision only the selected provider's protected credentials, permissions,
   artifacts, and approved run window.
- [ ] Run the approved selection with generation, exact counts, one worker,
   three repetitions, zero retries, and retained private reports.
- [ ] Verify success reporting and controlled failure/trace publication on the
   selected provider before claiming CI integration is accepted.

---

## Stage 1 — Pilot port

**Goal:** prove the migration on the user-selected E2E lifecycle using only React
application pages, including login. This is deliberately broader than the previous vendor-search
proposal; preserve the full scenario's assertions rather than calling a smaller
search test equivalent. The live safety and authentication gates still apply.

**Selected target:** source `cypress/e2e/base/e2e.feature`, scenario `e2e`, to
existing target `tests/features/base/e2e.feature`, scenario `E2E`, tag `@base-e2e`.
Reuse/repair `tests/steps/base/e2e.steps.ts`; do not create a duplicate flow.
The source starts with `staff` on Automation Client 1. Retain NVP's separate
`nvp_service` identity for NVP-specific coverage; do not silently expand this
first scenario into a second-company matrix.

Map and preserve:

1. Full UI login through the React page with the configured `staff` account and matching company.
2. A test-owned vendor linked to the agreed subsidiary, including its required
   approval and payment-method preconditions.
3. A linked approved/open purchase order and invoice with controlled amounts.
4. An available credit memo, its required approval, and its application to the
   invoice using the source's intended values.
5. Payment-run submission, run approval, and any required linked-payment
   approvals through the equivalent React workflows where available.
6. Final outcomes: payment run `CLOSED`, payment `PROCESSING`, invoice `PAID`,
   purchase order `CLOSED`, and credit memo `AVAILABLE`, checked against the
   exact created record identities.

Trace every action, including login, to a supported React route and source-backed
selectors. An unavailable React action blocks acceptance; there is no Angular
fallback or pre-approved exception. Confirm
the required company page-routing profile before login and restore it after
the run. Do not force terminal statuses or disable business/security rules to
make the test pass. Handle the payment processing-date/cutoff precondition
explicitly; more retries cannot make a next-business-day run close today.

Before execution, repair the old port's scenario-global state, saved-auth
fallback, and missing lifecycle cleanup using the Phase 0 fixtures and live
session authentication. Agree safe sandbox payment destinations and supported
cleanup or compensating actions for linked/processed records; do not assume
every payment record can be deleted. No live flag or payment mutation is
authorised merely by listing this implementation target.

Once the data, flag profile, and live safety gates are proven, validate exactly
this one scenario with `--workers=1 --repeat-each=3 --retries=0` through the
shared reporting runner. Confirm the exact selection with `--list` first.

**Exit gate**
- [x] E2E is selected as the first target, using the existing automation companies and named-user identities. It is React-only on every page, including login, with no Angular fallback.
- [ ] E2E controlled data, approvals, sandbox payment destinations, provisioning, and recovery agreed before execution.
- [ ] Isolation and same-scenario UI login/API authentication proven before pilot validation.
- [ ] Existing E2E target reused/repaired; every source precondition, approval, relationship, and final status assertion mapped.
- [ ] Exactly one selected case passes all three executions with `--repeat-each=3 --workers=1 --retries=0`, no skips, and full UI login each time.
- [ ] Login and every application page use the React origin, with source/deployment evidence for all selectors and no Angular fallback.
- [ ] Original company flag profile and shared state restored on success/failure, and interrupted-run recovery verified.
- [ ] All final assertions use the exact test-owned record identities and independent expected values.
- [ ] No `cy.wait` equivalents, retry counters, or conditional branching survived the port.
- [ ] `AgGrid` was used rather than hand-rolled row locators.
- [ ] Prompt corrections folded back into the prompt file — not just fixed in place.

> If the prompt needed significant hand-holding, fix the prompt before Stage 3.
> Every later port inherits its quality.

---

## Stage 2 — Triage inventory

**Goal:** decide which behaviours still need work, not commission a new port
for every source file. The review snapshot reported **51 existing target
feature files**; recount them at the recorded checkout. File count is not
scenario count or evidence of equivalent assertion coverage.

Start the preliminary inventory during Stage 0 using read-only file analysis.
Stage 2 completes and signs off that same inventory after the pilot, including
what the pilot revealed. Inventory work can overlap foundation work without
running competing live suites or marking unverified ports as passing.

Build a checklist joining four evidence sources:

1. Every source scenario in `cypress/e2e/**/*.feature`, with Scenario Outline
   examples expanded, plus setup and assertions implemented by its steps.
2. Every existing target scenario in `tests/features/**/*.feature`, its steps,
   page objects, assertions, and any execution evidence. Map source behaviours
   to target assertions, not merely similar titles. Allow one-to-many mappings
   and record missing, partial, duplicate, and target-only coverage.
3. `docs/tests_status.md`, which is a **payment-run-specific ledger**, not a
   passing/failing baseline for the whole suite. Use it only for the cases and
   revisions it actually describes.
4. `docs/self-heal-flaky-history.json`, which records **failure occurrences and
   signatures**, not a measured flake rate. Without total attempts and comparable
   outcomes in a defined window, a frequency or percentage cannot be inferred.

Columns: source file/scenario/example · tags · source assertions/preconditions ·
existing target path/scenario/assertions · coverage gaps · source/target evidence
state · evidence date/environment/revision/report · recorded failure occurrences
and signature · account/data/cleanup requirements · `@setupEnvironment` dependency ·
**decision and owner**.

Evidence states are **Verified passing**, **Verified failing**, **Blocked**, and
**Unknown**. Missing, stale, or unmatched history stays **Unknown**, never
implicitly Passing. Distinguish historical Cypress evidence from current
Playwright acceptance. Only report a measured flake rate when the run window,
attempt count, and first-attempt outcomes are known; otherwise leave it Unknown.

Decisions: **Reuse** (equivalent existing coverage, still needs acceptance) ·
**Repair existing** (fill gaps/fix the existing port) · **Port new** (no suitable
target) · **Defer** (record reason/owner; app bugs need tickets) · **Drop**
(explicitly approved obsolete/low-value coverage) · **Redesign** (for example,
an unsafe `@setupEnvironment` dependency). Unknown evidence is a reason to
investigate, not an automatic reason to drop a scenario.

> This stage is the one that most needs the original author's judgement rather
> than tooling. Mechanical analysis can propose the Drop and Defer candidates —
> particularly the combinatorial Scenario Outline expansion in
> `payment_run.feature` — but only someone who knows why each scenario was
> written can confirm them. Expect to overrule the mechanical suggestions.

**Exit gate**
- [ ] Every expanded source scenario/assertion maps to existing target coverage or an explicit gap; target-only and duplicate cases recorded.
- [ ] Every scenario carries an owned decision; reuse/repair considered before a new port.
- [ ] Missing history remains Unknown; payment-run evidence is not extrapolated and failure occurrences are not labelled a flake rate.
- [ ] Deferred app bugs have tickets; all deferrals and drops have an owner and explicit approval.
- [ ] Agreed scope, with a real scenario count, signed off with Dan.

---

## Stage 3 — Port in vertical slices

Use review-sized batches within each domain, with verified passing scenarios
first. A domain such as Invoices or Payments can span multiple PRs; the table
below orders domains, not PR sizes. Each PR should cover one bounded workflow
or a small related scenario group whose assertions, setup/cleanup, and evidence
can be reviewed together. Split the batch when those concerns cannot be reviewed
clearly as one change; no arbitrary scenario-count quota is required.

Investigate Unknown cases rather than assuming they are green. Reuse or repair
existing ports before adding files. Every mutating slice must pass the
account/data safety gate before its first live run.

| # | Area | Notes |
|---|---|---|
| 1 | Vendors | Proves out `AgGrid`. Account allocation and failure-restoration proof required before role changes or writes. Vendor-create API contract is unconfirmed; use only approved UI provisioning until supplied. |
| 2 | Purchase Orders | `POST /purchase_orders` and `/po_requests` both confirmed. Capture a real request body first. |
| 3 | Invoices | Best seeding story — `POST /invoices` with a fully typed payload. Large (~60 scenarios). |
| 4 | Credit Memos | Depends on invoice seeding from slice 3. Use the deployed React equivalent where supported; document and ground any genuinely unavailable React capability before using a legacy exception. |
| 5 | Users / Subsidiaries | **Gated on blocker #4** — no user-create endpoint is visible client-side. |
| 6 | Payments | Largest and most red. 16 feature files. Last, once everything else is proven. |
| — | Expenses, Cards, Approvals, Dashboard, Reports, Profile, Administration | Sequence by business priority. |

Per PR:
1. Choose a review-sized batch and record its exact source/target cases and
   assertions. Reuse, repair, or port via the prompt; verify account/data
   ownership and failure-safe restoration before mutations.
2. Add `bdd:<tag>:qa` / `bdd:<tag>:stage` scripts for any new feature tag.
3. `npx bddgen && npx tsc --noEmit`, Prettier on changed files.
4. List the accepted selection, then pass exactly 3 x N executions with
   `--repeat-each=3 --workers=1 --retries=0`, no skips or conflicting overlap.
5. Once CI integration resumes, add only the accepted scenario selection to
   the chosen provider using the same strict flags and count checks; a whole
   tag may contain unaccepted mutations. Keep local and CI acceptance separate.
6. Record selector gaps in the Phase 0.6 audit list. Land and deploy required
   app/library hooks before validating or accepting the affected slice.

**Exit gate per slice**
- [ ] All in-scope scenarios ported or explicitly deferred.
- [ ] Review-sized batch has a reviewed source-to-target behaviour/assertion map; omissions and uncovered source cases remain explicit.
- [ ] Accepted selection passes `--repeat-each=3 --workers=1 --retries=0` with exact counts and no skips.
- [ ] Account allocation/non-conflicting window and test-data ownership verified before live mutations.
- [ ] Test-owned data cleanup and shared-state restoration verified, including failures.
- [ ] Matching source access and deployed selector uniqueness verified; all required upstream hooks already landed/deployed.
- [ ] Deferred until CI integration resumes: accepted selection running on the chosen provider with the same strict flags/counts and retained reports.
- [ ] Banned-pattern scan clean: `waitForTimeout`, `setViewportSize`, `xpath=`,
      `nth-child`, `:eq(`, `.css-`, `console.log`, `process.env.` outside `utils/env.ts`.

---

## Stage 4 — Hardening and cutover

1. Verify the agreed Playwright coverage against expected application behaviour,
   with repeated sequential passes and comparable scheduled CI evidence under
   the agreed observation window. Record scheduled run/report links, environment,
   revisions, selection/counts, and `--workers=1 --repeat-each=3 --retries=0`.
   A local pass or one manually triggered CI pass does not replace this evidence.
   A mandatory Cypress overlap or matching Cypress results is not a cutover gate.
2. Extend the existing account/data safety controls before enabling any parallel
   runs; do not introduce those controls for the first time here. Classify by
   shared accounts, company settings, policies, and records, including tests
   that depend on those resources without changing them. Known candidates
   include vendor/user roles, profile updates, expense/payment defaults,
   shared expense policies, user metadata, and bulk-import result tracking.
   Sequential execution can remain the default; design parallel execution
   only if needed as a separate hardening task.
3. Retire each Cypress feature after its agreed replacement coverage passes
   the Playwright review, repeat-run, scheduled-evidence, and coverage-owner
   sign-off gates. Review the source-to-target assertion map, not just file
   counts, similar titles, or a green domain tag. Record dropped/deferred
   scenarios explicitly and keep any still-required uncovered behaviour in
   the owned Cypress remainder. A smaller pilot does not replace untested
   source behaviours; no side-by-side execution comparison is required.
4. Finish only the remaining, non-blocking hook improvements from Phase 0.6.
   `DropdownToggle` and toast `role="alert"` hooks, when required by a slice,
   must already have landed and been deployed before that slice was accepted.
5. Recheck the deployed app/library revisions recorded in Stage 0 and for each
   accepted batch. The local library and app dependency both read `1.2.4`; the
   historical 1.2.2/1.2.3 skew no longer exists locally. This final drift check
   does not defer deployed-version verification until cutover.
6. Rename `@vendorAddEdit` / `@invoiceAddEdit` to kebab-case once their features
   are ported, updating any scripts that reference them.
7. Decide whether to port the self-heal pipeline. A stable suite needs far less
   of it; `corpay-playwright/docs/self-healing-tests-plan.md` already scopes the
   Playwright equivalent if it's wanted.

**Exit gate**
- [ ] Agreed scope covered and green in comparable scheduled CI runs for the approved observation window, with exact counts and zero retries/skips.
- [ ] Reviewed source-to-target assertion map accounts for every behaviour being retired; a reduced pilot does not count as full source equivalence.
- [ ] Default execution mode documented; any enabled parallel runs have verified shared-resource isolation.
- [ ] Coverage owner approves retirement; every omitted behaviour is explicitly dropped/deferred and any remaining Cypress coverage has an owner.
- [ ] No scenario depends on `@setupEnvironment`-style live config assertions.

---

## Decision log

Decisions and remaining open questions, to be updated as the work proceeds.

| # | Question | Needed by | Status |
|---|---|---|---|
| 1 | Does `POST /vendors` exist server-side? Any user-create endpoint? | Stage 3 slices 1 and 5 | Open — needs a backend engineer |
| 2 | How should API helpers obtain the live session's token and company nonce? | Stage 0 | Implemented and verified for the new read-only BDD client on React: active-origin `Token` plus hydrated `userDetails.company.id`; no saved-session fallback. Existing domain helpers and Angular live coverage remain separate work. |
| 3 | Coverage scope: everything, or a prioritised subset? | Stage 2 | Open (recommend subset); matching Cypress execution results is not required |
| 4 | Keep Jira tags (`@PAY-961`, `@EXP-393`)? | Stage 3 | Recommend keep |
| 5 | Account allocation, owned data, and shared-state cleanup | Stage 0 foundation allocation; before the first mutating E2E run | Existing Automation Client 1/NVP companies and Cypress named identities retained; 12 complete QA/Stage credential pairs verified locally on 2026-09-24. Confirm the owner/exclusive window and prove exact-record cleanup/restoration before live mutations; a correct credential mapping alone does not establish isolation. |
| 6 | Two-company matrix ("Automation Client 1" / "Automation Client NVP")? | Stage 2 | Decided 2026-09-24: retain each scenario's existing company, users, and credentials. Use Rails Console MCP for reviewed company-scoped React/legacy page flags between exclusive batches; snapshot, verify, and restore state. No company switching or credential remapping. See the procedure below. |
| 7 | Formatting-only commit, or scope Prettier to changed files? | Stage 0 | Decided 2026-09-22: Prettier on changed files only; no repo-wide formatting commit |
| 8 | Port the self-heal pipeline at all? | Stage 4 | Open |
| 9 | BDD authentication model | Stage 0 | Decided 2026-09-21: full UI login per scenario requiring authentication; no saved-session reuse or separate anonymous lane |
| 10 | Migration validation versus parallel-execution design | Stages 0-3 / Stage 4 | Sequential migration validation and no conflicting overlap remain required; strict acceptance adds zero retries and exact counts. Basic account/data safety applies now; only parallel-execution tooling is deferred. No mandatory Cypress overlap window. |
| 11 | First migration target | Stage 1 | Decided 2026-09-24: base E2E payment lifecycle, reusing/repairing the existing port. React-only on every page, including login, with no Angular fallback (updated 2026-09-25, see #14). This replaces the vendor-search pilot. Confirm the lifecycle's controlled data, approval paths, safe payment destinations, flag profile, and cleanup/compensation before live execution. |
| 12 | CI provider, QA/Stage usage, and integration | Before hosted live CI acceptance; not local Phase 0 | Updated 2026-09-25: provider choice and QA/Stage usage, schedules, and required checks remain undecided. Retain GitHub Actions and AWS CodePipeline/CodeBuild as options. PR/push checks are offline-only; live GitHub runs require explicit manual `run_live` opt-in. No AWS deployment or live dispatch is part of the foundations PR. Verify the chosen live integration when work resumes. |
| 13 | Dedicated Okta/TOTP case | Stage 0 | Decided 2026-09-25 by the user: exclude due to setup issues; retain with `@skip @incomplete`. No Okta/MFA setup or dummy password provisioning is required. Five ordinary login cases plus one protected-page case remain the six-case/18-execution foundation; normal full UI login is unchanged. |
| 14 | React-only E2E including login | Before any E2E acceptance | Updated by the user 2026-09-25: all application pages, including login and post-login landing, must use React. No Angular exception or fallback; missing React capability blocks the affected action. Preserve full UI credential entry and current-session API authentication. This supersedes the earlier Angular-login allowance. |
| 15 | Existing target mapping and Unknown evidence | Start in Stage 0; scope sign-off in Stage 2 | Preliminary inventory runs alongside foundation reconciliation; map the pilot in detail before implementation and extend the same map across existing targets. Absent evidence is Unknown and failure occurrences are not a measured flake rate. Dan signs off scope and intentional omissions. |
| 16 | Scheduled evidence before Cypress retirement | Before Stage 4 retirement | Pending approval: recommend three consecutive scheduled Stage acceptance runs with comparable scope/data and strict flags/counts. Confirm the cadence/window and coverage owner; this does not require Cypress parity. |
| 17 | Batch size and mapped coverage before cutover | Stage 3 / Stage 4 | Review-sized batches within one domain per PR, not whole-domain rewrites. Every batch needs mapped assertions and retry-free evidence; retire only mapped accepted coverage or explicitly approved omissions. |
| 18 | Rails MCP flag-change owner | Before any affected live run | Decided 2026-09-25: the other Rails MCP chat owns reviewed company-scoped page-flag changes and restoration. This implementation chat publishes requirements and consumes the snapshot/delta/effective-state/run-window handoff; no duplicate writer or assumption that a baseline means flags are applied. |

### Company and page setup through Rails Console MCP

Use the private operational runbook and reviewed per-batch profile for the
source-backed page-routing candidates. This public repository does not contain
live flag values, company/user identifiers, or environment snapshots. Source
expectations are not a restore point; writes remain blocked until the original
state, company/test-user identities, and run window are verified privately.

**Cross-chat handoff:** the other Rails MCP chat is the single flag-change
operator. Record its handoff in the shared runbook/manifest: exact environment,
company and test-user identities, selected flow, original snapshot reference,
reviewed inserts/updates/no-ops, verified effective flags, ready run window, and
restoration owner/actions. This chat must confirm that handoff before browser
validation and report when the batch has finished so the operator can restore
and verify the original state. Until that evidence is present, continue only
local work; do not claim flags changed, start a conflicting run, or independently
toggle flags from tests, shell scripts, or another MCP session.

- Keep each scenario's current assignment to "Automation Client 1" or
   "Automation Client NVP". Keep users, credentials, card programs, roles, and
   business settings unchanged unless separately approved.
- Authenticate the MCP's selected AWS profile through the approved AWS login
   flow. MCP startup is not authentication; live access also requires ECS Exec
   permission. Do not put AWS tokens in test config or change test credentials.
- Before writes, confirm the environment, unique company IDs, selected pages,
   desired React/legacy state, and an exclusive window across Cypress,
   Playwright, CI, and manual testing. Company choice alone does not establish
   isolation, and running batches must not have their flags changed underneath
   them.
- Inspect with `rails_exec` in `read` mode. Build an explicit per-page allowlist
   from deployed behavior, not a blanket toggle of every flag or an assumed
   single React switch. Capture the presence and value of company overrides,
   inspect the test users' higher-priority overrides and relevant module
   settings, and stop if these conflict with the intended page selection.
- Use `write` mode only for the reviewed company-scoped changes. Preserve model
   validations and audit attribution; never change global flag defaults. User
   overrides or module-setting changes need explicit scope approval. Verify the
   effective flags for each test user and the expected routes after fresh UI
   login before starting the selected sequential batch.
- Restore the exact captured state after success or failure, deleting an
   override only when this batch created it and it was originally absent.
   Recheck current values before restoration so another actor's changes are not
   silently overwritten. Verify restoration, retain the before/after evidence,
   and block conflicting runs after interrupted or failed cleanup until state
   is reconciled.
- This is an operator-controlled setup workflow through MCP, not a replacement
   test runner, a per-scenario flag-toggling hook, or a port of hardcoded
   `@setupEnvironment` assertions. Live company IDs, flag allowlists, and run
   windows still need verification before the first mutation.
