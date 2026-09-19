# ADR-0007: Admin Portal Frontend — React SPA (replaces Vue default)

- **Status**: accepted
- **Date**: 2026-08-20
- **Deciders**: Imran
- **Supersedes**: "Vue 3 + Vite + Tailwind (default)" design note in `ui-portal/features/admin-portal-ui.md`
- **Superseded by**: -

## Context

ADR-0006 approved the headless-hybrid Admin Portal architecture (custom SPA served
same-origin via Frappe `www/`, config-driven sidebar, desk kept as System Manager
fallback) but deliberately left the frontend stack TBD. The feature file defaulted
to **Vue 3 + Vite + Tailwind**, and an initial Vue SPA was built at commit `10f7e4a`.
Before the portal went live, Imran re-evaluated the stack and the frontend was
rebuilt in React; the Vue code was removed at commit `6aa7f28`.

## Decision

**React 19 + Vite + TypeScript + Tailwind v4 + shadcn/ui** for the Admin Portal
frontend.

Mechanics:

- **Frontend location**: the SPA source lives in `ui/` at the META repo root
  (NOT inside the app). Convex and VLY/Freebuff backends are stripped; Frappe is
  the only backend.
- **Build**: the built bundle is emitted to
  `apps/real_estate_os/real_estate_os/public/portal/` and served same-origin via a
  thin Jinja shell, so no CORS and no token plumbing.
- **Auth**: session auth via the Frappe `sid` cookie on the same-origin
  `www/portal` page; login page + session bootstrap in the SPA.
- **Data**: `/api/resource/<doctype>` for standard CRUD + whitelisted
  `@frappe.whitelist()` methods in `real_estate_os/api.py` for aggregated data and
  detail views (`get_doc_detail` / `get_linked_records`).
- **Config-driven sidebar**: nav items come from the `portal_nav_items` hook in
  `hooks.py` (9 sections: Overview, Properties, Contracts, Tenants, Maintenance,
  Landlords, Accounts, Reports, Settings), so the shell stays app-agnostic.
- **Logo**: platform logo `aetris.svg` (`aetris.png` fallback) replaces generic
  iconography.
- **Demo data**: `demo_data.py` seeds buildings, units, leases, landlords, tenants,
  and contracts for portal testing.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Vue 3 + Vite + Tailwind (ADR-0006 default) | Initial SPA already built (10f7e4a) | Larger ecosystem churn for admin-grade UI tables/forms; team preference leaned React; placeholder-quality shell | rejected |
| B. React 19 + Vite + TS + Tailwind v4 + shadcn/ui | shadcn/ui component quality + headless control; TS strictness; active ecosystem; fine-grained table/form components | Frontend rebuild after the Vue pass | **chosen** |

## Consequences

### Positive

- Production-grade component library (shadcn/ui) for list/form/filter screens.
- Type-safe frontend over the Frappe API; smaller, faster bundle via Vite.
- Config-driven 9-section sidebar keeps the shell reusable across future apps.
- Same-origin serving keeps auth to the `sid` cookie (no CORS/token infra).

### Negative

- Must keep the `ui/` toolchain (bun, Vite, Tailwind v4) in sync with app deploys.
- Two UI surfaces (portal + desk) to keep consistent, as in ADR-0006.

## Implementation

- Feature file: `vault/ui-portal/features/admin-portal-ui.md`
- Vue portal built at `10f7e4a`, replaced by the React portal at `6aa7f28` (Vue code removed).
- ADR-0006 itself stands (headless-hybrid + config-driven shell unchanged).
