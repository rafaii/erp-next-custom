# ADR-0001: App & Git Repository Structure

- **Status**: accepted
- **Date**: 2026-08-16
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

The GitHub repo (`https://github.com/rafaii/real-estate.git`) is new/empty. The
local machine is a full Frappe Bench install at `~/Development/erpnext`
containing `apps/frappe`, `apps/erpnext`, `sites/`, `env/`, `config/` — none of
which are our code. We must decide what gets committed so that a fresh ERPNext
installation reproduces the product with no manual steps.

## Decision

1. **Create a custom Frappe app** `real_estate_os` under `apps/real_estate_os`.
2. **Initialize git ONLY at `apps/real_estate_os/`** and point it at the GitHub remote.
3. The bench root, `apps/frappe`, `apps/erpnext`, `sites/`, `env/`, `config/`
   are **never** committed — they are upstream/untracked.
4. The canonical vault (`~/Development/erpnext/vault`) stays **local-only**,
   outside the app repo.
5. Everything the product needs (DocTypes, Custom Fields, Print Formats, Roles,
   Server Scripts, Web Views, scheduler jobs, whitelisted methods) is declared
   inside the app via `hooks.py` + fixtures, so a clone reproduces everything.

Acceptance: `bench get-app https://github.com/rafaii/real-estate.git && bench --site <site> install-app real_estate_os` yields a working instance.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Git at bench root (`~/Development/erpnext`) | Single repo | Commits frappe+erpnext source, sites DB config, `.env` risk, huge repo, merge hell with upstream | rejected |
| Monorepo with frappe/erpnext as submodules | Clean-ish | Submodule churn, overkill for one app | rejected |
| Custom app as its own repo (`real_estate_os`) | Standard Frappe pattern, clean clone, self-contained deliverable | Vault/docs live outside repo | **chosen** |

## Consequences

### Positive

- Clone → `bench get-app` → `install-app` reproduces everything (standard Frappe flow).
- No risk of leaking `.env` / site DB config / upstream code.
- Small, reviewable repo containing only our IP.

### Negative

- Vault (planning docs) is not version-controlled (accepted: local-only).

## Implementation

- Feature file: `vault/custom-module/features/app-scaffold.md`
