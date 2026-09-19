# C2P Workspace

VS Code multi-root workspace for the Cypress → Playwright migration. Groups the four repos involved, plus this one.

## Start here

New to this? Follow [workspace-setup.md](workspace-setup.md) — it covers getting the repos, creating your workspace file, pointing it at your clones, and verifying the agent customizations loaded.

## Contents

| Document | What it's for |
| --- | --- |
| [workspace-setup.md](workspace-setup.md) | Set up this workspace on your machine. Start here. |
| [cypress-to-playwright-migration-plan.md](cypress-to-playwright-migration-plan.md) | **Strategy** — why the migration is shaped this way, what was found in each codebase, and (§0) exactly where things stand today. |
| [migration-execution-plan.md](migration-execution-plan.md) | **Execution** — what to do next, in what order, and the exit gate for each stage. |

Read them in that order. For the framework's own conventions, see `AGENTS.md` in the `corpay-playwright` repo.

## The repos

| Folder name in the workspace | Repo | Role |
| --- | --- | --- |
| `Corpay-Playwright repo` | `corpay-playwright` | Target — the Playwright framework |
| `Accrualify Test Automation repo` | `accrualify-test-automation` | Source — the Cypress suite |
| `React repo` | `accrualify-reactjs` | Grounding — `/ap/*` pages |
| `Component Library repo` | `corpay-react-components-library` | Grounding — shared UI components |
| `C2P Workspace` | this repo | Coordination and planning docs |

The two grounding repos are what let selectors be found by local search instead of guessed. See [workspace-setup.md](workspace-setup.md#why-bother-with-a-multi-root-workspace).
