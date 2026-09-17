# C2P Workspace

VS Code multi-root workspace for the Cypress → Playwright migration. Groups the four repos involved, plus this one.

## Setup

```bash
cp C2P.code-workspace.template C2P.code-workspace
code C2P.code-workspace
```

`C2P.code-workspace` is gitignored. It is per-machine, and VS Code rewrites it whenever you add a folder or change a workspace setting — so edit yours freely, it will never turn up in `git status`.

## Folder paths

The template uses paths relative to this repo and assumes all clones are siblings:

```
repos/
├── Accrualify/
│   ├── accrualify-reactjs/
│   ├── accrualify-test-automation/
│   ├── corpay-playwright/
│   └── corpay-react-components-library/
└── dev/
    └── C2P-workspace/   ← this repo
```

If your clones live elsewhere, edit the `folders[].path` values in your copy. Leave the `name` values alone — the `.github/copilot-instructions.md` and `AGENTS.md` files in the other repos refer to them by those labels.
