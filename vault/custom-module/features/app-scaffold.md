---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-16
related_adr: ["0001-app-repo-structure"]
---

# App Scaffold

## Summary

Bootstrap the `real_estate_os` Frappe app under `apps/real_estate_os/` and make
it the git repo linked to `https://github.com/rafaii/real-estate.git`.

## Requirements

- Valid Frappe v15 app structure (pyproject.toml + flit build, hooks.py, modules.txt, config).
- Frappe/ERPNext declared as dependencies.
- Only the app directory is under git; bench root and upstream apps are not.

## Design

Files (relative to `apps/real_estate_os/`):

- `pyproject.toml` — flit build, `frappe >=15.0.0,<16.0.0`, `erpnext >=15.0.0,<16.0.0`.
- `real_estate_os/__init__.py` — `__version__`.
- `real_estate_os/hooks.py` — app metadata, fixtures/scheduler placeholders.
- `real_estate_os/modules.txt` — `Real Estate OS`.
- `real_estate_os/config/desktop.py` — module desktop entry.

## Implementation Plan

- [x] Scaffold app structure (pyproject, hooks, modules, config)
- [x] `git init -b main` at `apps/real_estate_os/`
- [x] `git remote add origin https://github.com/rafaii/real-estate.git`
- [x] Initial commit of scaffold

## Acceptance Criteria

- [x] `bench get-app` + `install-app` flow documented in README
- [x] Only app files committed; `.env` and bench/site files excluded

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0001-app-repo-structure.md`
