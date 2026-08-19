# ERPNext Custom — meta repo

Monorepo-style container for the custom ERPNext/Frappe work on the bench at
`~/Development/erpnext`. Contains the canonical vault (design docs, ADRs,
feature plans) and the custom apps as git submodules.

## Contents

| Path | What it is |
|---|---|
| `vault/` | Canonical design documentation: `INDEX.md`, `MASTERPLAN.md`, ADRs in `decisions/`, domain folders with feature plans |
| `AGENT.md` | Operating rules for coding agents (branch/worktree policy, vault workflow) |
| `apps/real_estate_os/` | The custom Frappe app (submodule → `rafaii/real-estate`) |

Bench internals are intentionally NOT here: `apps/frappe`, `apps/erpnext`,
`sites/`, `env/`, `config/`, `logs/` are upstream/untracked (see `.gitignore`).
Secrets live in `.env` and are never committed.

## Cloning

```bash
git clone --recurse-submodules https://github.com/rafaii/erp-next-custom.git
```

The app submodule must be present for the ERPNext acceptance test:

```bash
bench get-app https://github.com/rafaii/real-estate.git
bench --site <site-name> install-app real_estate_os
```

## Workflow

- `vault/` is the single source of truth for architecture (see `AGENT.md`).
- The app repo (`apps/real_estate_os`) keeps its own git history and branch
  policy — never commit app work directly in this meta repo.
- To update the submodule pointer: commit inside the app repo, then
  `git submodule update --remote apps/real_estate_os` and commit here.