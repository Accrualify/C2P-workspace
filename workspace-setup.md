# Workspace Setup

How to set up the **C2P multi-root VS Code workspace** and point it at your own
local clones.

**This is not a guide to setting up `corpay-playwright` itself** — that repo has
its own `README.md` covering install, `.env`, and running tests. This document
only covers the workspace that ties the repos together.

---

## Why bother with a multi-root workspace

Porting a test means grounding every selector in the real application source.
With all repos open in one workspace, that is an exact local `grep_search` for
`data-testid=`. Without it, the agent falls back to the remote `githubRepo`
tool, which can silently return nothing on a private repo — and a silent empty
result is what makes an agent start guessing selectors.

It is the difference between "find the hook" and "invent a hook".

---

## Prerequisites

- **VS Code** with **GitHub Copilot** and **Copilot Chat**.
- **Git**, with access to the Accrualify GitHub org.

Node, npm and Playwright browsers are only needed to *run* tests — see
`corpay-playwright/README.md`.

---

## 1. Get the repos

Five repos are involved. The workspace template assumes this layout:

```
repos/
├── Accrualify/
│   ├── accrualify-reactjs/                  # React app — /ap/* pages
│   ├── accrualify-test-automation/          # Cypress suite — the source
│   ├── corpay-playwright/                   # Playwright framework — the target
│   └── corpay-react-components-library/     # shared UI components
└── dev/
    └── C2P-workspace/                       # this repo
```

You do **not** have to use this layout — step 3 covers pointing at wherever your
clones actually live. But if you already have the Accrualify repos checked out
somewhere, note the paths now.

> The selected E2E uses **React for every application page, including login**.
> The Angular source is not required for that flow. Add it separately when
> reviewing legacy coverage; it is not an allowed fallback for the React E2E.

---

## 2. Create your workspace file

```bash
cd C2P-workspace
cp C2P.code-workspace.template C2P.code-workspace
```

`C2P.code-workspace` is gitignored. It is per-machine, and VS Code rewrites it
whenever you add a folder or change a setting — so edit yours freely, it will
never show up in `git status`.

---

## 3. Point it at your clones

Open `C2P.code-workspace` and edit each `folders[].path` to match where your
repos actually are. Relative paths are resolved from the workspace file;
absolute paths work too.

```jsonc
{
  "folders": [
    {
      "name": "Corpay-Playwright repo",        // ← do not change
      "path": "../../Accrualify/corpay-playwright"  // ← change to match your machine
    }
    // …four more
  ]
}
```

> ⚠️ **Change `path`. Never change `name`.**
> The `AGENTS.md` and `.github/copilot-instructions.md` files inside the other
> repos refer to these folders by those exact labels. Renaming them breaks the
> agent's ability to resolve cross-repo references.

Expected `name` values, all five:

| `name` | Repo |
|---|---|
| `Corpay-Playwright repo` | `corpay-playwright` |
| `Accrualify Test Automation repo` | `accrualify-test-automation` |
| `React repo` | `accrualify-reactjs` |
| `Component Library repo` | `corpay-react-components-library` |
| `C2P Workspace` | this repo (`.`) |

---

## 4. Open it

```bash
code C2P.code-workspace
```

Or: **File → Open Workspace from File…** and pick `C2P.code-workspace`.

All five folders should appear in the Explorer under those names. If a folder
shows as missing, its `path` is wrong — fix and reload the window.

---

## 5. Verify the agent customizations loaded

The migration depends on instruction and prompt files inside
`corpay-playwright`. Confirm VS Code picked them up:

1. **Prompts** — open Copilot Chat and type `/`. You should see
   `migrate-cypress-feature`, plus `new-feature`, `new-page-object`,
   `new-api-client`, and `new-tag-script`.
2. **Agent** — `playwright-agent` should be available in the agent picker.
3. **Instructions** — ask Copilot Chat: *"Which instruction files apply when I
   edit `tests/steps/vendors/vendor.steps.ts`?"* It should name
   `cucumber-bdd.instructions.md` and the repo-wide `copilot-instructions.md`.
4. **Cross-repo grounding** — ask: *"Find `data-testid=\"quick-filters-button\"`
   in the React repo."* It should find it in
   `src/components/datagrid/gridFilterDropdown.jsx`. If it can't, the React repo
   isn't in your workspace or its path is wrong.

Step 4 is the one that matters most — it's the capability the whole porting
workflow rests on.

---

## 6. Read in this order

1. [cypress-to-playwright-migration-plan.md](cypress-to-playwright-migration-plan.md)
   — the strategy, and §0 for exactly where things stand.
2. [migration-execution-plan.md](migration-execution-plan.md) — what to do next.
3. `corpay-playwright/AGENTS.md` — the repo's conventions contract.

---

## 7. Optional Rails Console MCP authentication

Register the sibling `rails-console-mcp` clone through **MCP: Open User
Configuration** to make it available across local repositories in the same
VS Code profile. The clone does not need to be a workspace folder.

The MCP process running successfully does **not** mean AWS is authenticated.
It delegates to the AWS CLI and opens a Rails console lazily on the first
execution. AWS sign-in belongs in the terminal/browser, not in chat, MCP
configuration, or test credentials. Application/Active Admin login is separate.

Use the organization's approved authentication method and a separate named
AWS profile for each intended environment. Obtain account, region, role, and
access-portal details through approved internal channels; do not guess them or
publish them in this repository. Do not overwrite another environment's profile.

Complete sign-in in the terminal/browser, then verify that the profile belongs
to the intended environment. If access is unavailable, request it from the AWS
administrator rather than substituting a different environment. Only after
verification, select that profile in the corresponding user-level MCP mapping,
preserving all other servers and mappings. No AWS keys or tokens belong in MCP
configuration, source control, or chat.

Use **MCP: List Servers** to restart `rails-console`, then ask Copilot to execute
`RUBY_VERSION` in the intended environment with `mode: "read"`. Successful STS
authentication alone does not prove ECS Exec access: the role must also be
authorized for the target service, and that service must have an Exec-enabled
running task. Resolve access/target errors before attempting any writes.

Company-specific React/legacy switching follows the
[company and page setup procedure](migration-execution-plan.md#company-and-page-setup-through-rails-console-mcp).
Keep the approved test-company and account assignments unchanged. Authenticate,
verify the exact company and page controls, and reserve an exclusive run window
before applying or restoring any flag changes. Store actual company/user IDs,
flag snapshots, and restore records in the private operational handoff.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| A folder shows as missing in the Explorer | Wrong `folders[].path`. Fix it, then **Developer: Reload Window**. |
| Agent can't find selectors in the React app | React repo not in the workspace, or its `path` is wrong. Verify with step 5.4. |
| Agent references `CustomWorld`, `cucumber.js`, or `test:cucumber:*` scripts | It's working from a stale cache. Reload the window. None of those exist — the repo uses `playwright-bdd`. |
| `/migrate-cypress-feature` doesn't appear in chat | `corpay-playwright` isn't in the workspace, or the Phase 0 work hasn't been pulled — the prompt file is recent. |
| `C2P.code-workspace` shows up in `git status` | You edited the `.template` instead of your copy. Revert the template; edit `C2P.code-workspace`. |
| VS Code keeps rewriting your workspace file | Expected. It's gitignored for exactly this reason. |
