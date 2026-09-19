---
status: in-progress
owner: imran
domain: ui-portal
created: 2026-09-17
updated: 2026-09-18
related_adr: ["0027-selectable-admin-ui", "0006-admin-portal-approach", "0007-admin-portal-react-frontend", "0017-platform-modularization"]
---

> **As-built note (2026-09-17):** Phase 1 merged (PR #87) and deployed live
> to all 4 `real_estate_os` tenant sites (rastec/realestate/test/us.nnuggets.com)
> 2026-09-18 — image rebuilt, containers recreated, `bench migrate` run on
> each site to sync the new `admin_ui` field, `get_settings_data` verified
> returning it correctly on all 4. `ui-classic/` is a lean, hand-built Vue 3
> + Vite + TS app in the vue-element-admin/vue-vben-admin visual family
> (dark sidebar, `lucide-vue-next` icons) — not a literal scaffold/clone of
> `vue-vben-admin`'s monorepo, which is far larger than Phase 1's 4-section
> scope justified. Only `call`/`listResource` were ported from
> `ui/src/lib/api.ts` (no `createResource` — Phase 1 is read-only, matching
> "no create dialogs" below). See "Deliberate Phase 1 simplifications" for
> what was cut versus the original design pass.
>
> **As-built note (2026-09-18):** Overview added to `ui-classic/` on
> `agent/developer-1/classic-overview`, opened as a per-section Phase 2 item
> at Imran's explicit request ("let us build the Overview page") — see
> "Overview (per-section Phase 2)" below. This does NOT reopen Phase 2
> wholesale; Actions/Accounts/Settings/Contracts remain gated.
>
> **Verified live in-browser by Imran (2026-09-18)**, on `test.nnuggets.com`:
> the "not available in Classic yet" fallback renders correctly for the
> gated sections, and Overview (stat cards, occupancy, cash flow projection,
> building profitability, etc.) renders real data correctly. Properties/
> Tenants/Maintenance/Landlords list views have not been separately called
> out as checked — worth a quick look if not already done.

# Selectable Admin UI — React "Modern" (default) + Vue "Classic" skin

## Summary

Add a second, optional admin UI ("Classic" — Vue 3 + Vite + TypeScript, in
the vue-element-admin/vue-vben-admin visual family) alongside the existing
React portal ("Modern", unchanged, still the only complete implementation).
Selectable per-tenant from Settings. Phase 1 covered the 4 nav sections that
already render generically; Overview was added 2026-09-18 as an explicit
per-section Phase 2 item. The remaining 4 (Actions, Accounts, Settings,
Contracts) fall back to a "not available yet" panel until their own
go/no-go. See ADR-0027 for the full rationale.

## Requirements

- **Settings toggle**: `Real Estate Settings.admin_ui` (Select: `Modern` /
  `Classic`, default `Modern`), editable from the portal's Settings page
  (`SettingsView()` in `ui/src/pages/Dashboard.tsx`).
- **Phase 1 scope — Classic implements exactly these 4 sections**, each
  driven by the same `portal_nav_items` config the React portal already
  uses:
  - Properties (`Building`, columns: `building_name`, `status`,
    `total_units`, `landlord.landlord_name`)
  - Tenants (`Customer`, columns: `customer_name`, `customer_type`,
    `email_id`, `mobile_no`)
  - Maintenance (`Maintenance Request`, columns: `status`, `issue_type`,
    `priority`, `unit.unit_number`, `unit.building`)
  - Landlords (`Landlord`, columns: `landlord_name`, `landlord_type`,
    `phone`, `email`)
- **Out of scope for Phase 1** (Overview, Actions, Accounts, Settings,
  Contracts): Classic shows a "not available in Classic yet" panel
  explaining that only a System Manager can switch back to Modern (from
  Modern's own Settings page or the Desk) — no in-app "switch" action, since
  `Real Estate Settings` write is System-Manager-gated the same as
  `esign_mode`/branding already are on that page.
- **Auth pages are not duplicated**: `/login` and `/update-password` always
  render the existing React `Login.tsx`/`SetPassword.tsx`, regardless of
  `admin_ui`. Only the authenticated shell switches.
- **No behavior change for Modern**: existing React portal, all 9 sections,
  unaffected.

## Design

- **One `www/portal` page.** `real_estate_os/www/portal/index.py`'s
  `get_context` reads `frappe.db.get_single_value("Real Estate Settings", "admin_ui")`
  and adds it to `context.portal_bootstrap` (e.g. `"ui": "modern" | "classic"`).
  The guest branch and the `PUBLIC_ROUTES` (`login`, `update-password`)
  branch always use `"modern"` — see Requirements above. `www/portal/index.html`
  conditionally emits the Modern (`/assets/real_estate_os/portal/...`) or
  Classic (`/assets/real_estate_os/portal-classic/...`) script/style tags
  based on that value.
- **`ui-classic/` tree** at the app-repo root (sibling to `ui/`, inside
  `apps/real_estate_os`, same as ADR-0007's "META repo root" — the git repo
  root of the `real_estate_os` app, not the outer bench checkout):
  - Vue 3 + Vite + TypeScript, `lucide-vue-next` icons, dark collapsible
    sidebar — vue-element-admin/vue-vben-admin's visual DNA, hand-built
    rather than cloned (see as-built note above).
  - `src/lib/bootstrap.ts` reads `window.portalBootstrap` — same shape
    `index.py` already injects for Modern (`user`, `csrf_token`, `nav`,
    `is_tenant`). No new bootstrap contract; direct port of
    `ui/src/lib/bootstrap.ts`.
  - `src/lib/api.ts` ports `call`/`listResource` from `ui/src/lib/api.ts`
    (same `/api/method/*` and `/api/resource/*` endpoints, same
    `X-Frappe-CSRF-Token` header, same `_server_messages` unwrapping).
  - `src/views/ResourceListView.vue` — the render-contract component,
    parameterized by `{doctype, columns}` from nav config, covering the 4
    Phase-1 sections. A deliberately smaller subset of the React
    `ResourceListView` (`Dashboard.tsx` ~line 4241): search, sortable
    columns, fixed-page-size pagination, and the Maintenance Request
    building-name join it needs — but no create-record dialogs, no
    per-column filter dropdowns, no per-row detail navigation. See
    "Deliberate Phase 1 simplifications" below.
  - `src/router/index.ts` builds routes at runtime from
    `getBootstrap().nav` — an explicit `PHASE1_ROUTES` allowlist (not "has a
    doctype", since Contracts has one too but isn't in scope) picks
    `ResourceListView` vs `NotAvailableView` per route.
  - `src/views/NotAvailableView.vue` for the 5 out-of-scope routes (see
    Requirements above).
  - Build output: `apps/real_estate_os/real_estate_os/public/portal-classic/`,
    committed the same way `public/portal/` already is
    ([[real_estate_os_ui_build_step]] — `bun run build`, a required explicit
    step, not implied by typechecking passing).
- **Real Estate Settings schema**: add `admin_ui` field to
  `real_estate_os/real_estate/doctype/real_estate_settings/real_estate_settings.json`
  (existing Single doctype, currently holding `amenity_options` and
  `default_labor_rate`).

## Deliberate Phase 1 simplifications

Cut from the React `ResourceListView`'s feature set to keep Phase 1 to
"cheap, generic sections only" — not oversights, but choices that trade
Classic's completeness for staying inside the phase's cost budget. Revisit
if/when Phase 2 is approved:

- **No create-record dialogs.** Modern's `ResourceListView` has an "Add"
  button opening a bespoke New*Dialog per doctype (`NewBuildingDialog`,
  `NewTenantDialog`, etc.) — those are per-doctype, not part of the shared
  render contract. Classic is read-only in Phase 1.
- **No per-record detail page.** Clicking a row in Modern navigates to a
  detail view with linked records and inline editing (`EditableField`).
  Classic's rows are display-only; there is no `/properties/<name>`
  equivalent in Classic yet.
- **No per-column filter dropdowns**, only the free-text search box (which
  searches the same displayed columns Modern's search does).
- **No "switch back to Modern" action inside Classic.** Deliberate, not a
  gap: `Real Estate Settings` write is System-Manager-gated, so only a
  System Manager can change `admin_ui` at all — and a System Manager
  already has the Desk (`/app`) fallback per ADR-0006. Building an in-app
  toggle inside Classic to work around a permission Classic's own users
  don't have would be inconsistent with how esign_mode/branding already
  work on the Modern Settings page.
- **No logout affordance.** Neither does Modern's `Dashboard.tsx` today
  (`LogoDropdown.tsx`/`use-auth.ts` exist but aren't wired into the actual
  portal) — Classic matches Modern's real current behavior, not a
  hypothetical better one. Out of scope for this feature.

## Overview (per-section Phase 2)

Opened 2026-09-18 at Imran's explicit request, independent of the other 4
gated sections. Port of `OverviewView()` in `ui/src/pages/Dashboard.tsx`
(~line 937):

- `src/views/OverviewView.vue`, added to the router's `ROUTE_COMPONENTS` map
  (`src/router/index.ts` — replaced the old `PHASE1_ROUTES` `Set` with a
  route→component map, since Overview is `api`-driven, not `doctype`-driven,
  and Contracts still needs to resolve to `NotAvailableView` despite having
  a `doctype`).
- Three whitelisted methods (`get_owner_overview`, `get_building_profitability`,
  `get_cash_flow_projection`), each with its **own** loading/error state via
  a small `useAsync` composable (`src/lib/useAsync.ts`) — a slow/failing
  `get_building_profitability` must not blank the stat cards fed by
  `get_owner_overview`, matching how Modern passes `loading`/`error`/`onRetry`
  into each panel separately rather than gating the whole page on one
  combined fetch.
- `src/components/CashFlowProjectionPanel.vue` and
  `BuildingProfitabilityPanel.vue` port their React namesakes faithfully
  (same numbers, same table columns) since they're data-shaped, not
  chrome-shaped.
- The trend chart (`monthly_income_vs_bills`) is a plain CSS/SVG-free bar
  chart (`src/components/RevenueBarChart.vue`), not a ported `recharts`
  equivalent — Classic stays dependency-light, per Phase 1's existing
  no-chart-library posture.
- Overdue-tenant/renewal/lease-ending rows are **not clickable** (Modern
  navigates to a Customer/Head Lease/Lease Agreement detail page on click;
  Classic has no detail-page equivalent — see "No per-record detail page"
  above). Rendered as plain list rows instead.
- `fmtMoney`/`fmtNum`/`pct` ported into `src/lib/format.ts` — confirmed by
  reading `Dashboard.tsx:423-434` that neither applies a currency symbol
  (tenants use QAR/INR/USD), so nothing to get wrong there.

## Decision gate — Phase 2 (remaining sections)

Actions, Accounts (GL/Trial Balance/P&L/Balance Sheet/Journal Entries/Bank
Reconciliation/rent roll), Settings, and Contracts's lease-signing + e-sign
template designer are **still not started**. Each is a multi-day port on its
own (Accounts alone is 6 sub-panels). Opening Overview above was a
per-section decision, not a wholesale Phase 2 approval — get an explicit
go/no-go from Imran for each remaining section before starting it.

## Implementation

- ADR: `vault/decisions/0027-selectable-admin-ui.md`
- Branch: `agent/developer-1/selectable-admin-ui` (worktree
  `apps/real_estate_os/wt-developer-1-selectable-admin-ui`)
- Phase 1 files:
  - `real_estate_os/real_estate/doctype/real_estate_settings/real_estate_settings.json`
    (new `admin_ui` Select field)
  - `real_estate_os/api.py`: `get_settings_data()` exposes
    `admin_ui: {value, options, can_manage}`; new `set_admin_ui(value)`
    whitelisted method (mirrors `set_esign_mode`)
  - `ui/src/lib/api.ts`: new `setAdminUi(value)`
  - `ui/src/pages/Dashboard.tsx` (`SettingsView()`, ~line 7037): Admin UI
    card (Select + Save), same pattern as the E-Signature card
  - `real_estate_os/www/portal/index.py` + `www/portal/index.html`:
    `context.admin_ui` ("modern"/"classic"), conditional bundle script/style
    tags
  - `ui-classic/` (new tree, Phase 1 scope only — see Design above)
- Verified locally: `bunx tsc --noEmit` (ui/), `bunx vue-tsc -b` +
  `bun run build` (ui-classic/) all clean; Python files `py_compile` clean.
  Deployed live 2026-09-18 (see as-built notes above) — `admin_ui` toggle
  and bundle serving confirmed working; full authenticated-browser render
  confirmed by Imran on `test.nnuggets.com` same day (fallback panel +
  Overview both verified — see note at top of file).
- Overview addition: Branch `agent/developer-1/classic-overview`, worktree
  `apps/real_estate_os/wt-developer-1-classic-overview`. Files:
  `ui-classic/src/views/OverviewView.vue`,
  `ui-classic/src/components/{StatCard,MetricBar,RevenueBarChart,CashFlowProjectionPanel,BuildingProfitabilityPanel}.vue`,
  `ui-classic/src/lib/{overview-types,useAsync}.ts`, `format.ts` additions,
  `router/index.ts` rewrite. No schema change — no `bench migrate` needed
  for this one, just image rebuild + container recreate.
