# ADR-0006: Admin Portal UI — Headless-Hybrid Custom SPA (Config-Driven)

- **Status**: accepted
- **Date**: 2026-08-19
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

ERPNext ships with the **Desk** — a generic admin UI (Vue 3 SPA at `/app`, built
on `frappe-ui`) with its own sidebar, navbar, and module/workspace-driven
navigation. Imran wants to replace this default skin with a purpose-built
**Admin Portal**: a clean sidebar (logo + links: Dashboard, Tenants, Landlords,
Properties, Maintenance, Accounts, Settings) tailored to real-estate arbitrage.
The same portal layout must later be reused for **other industries** (different
apps), so the sidebar must not be hardcoded to real-estate concepts.

Two families of approach exist: (1) **reskin** the Desk via CSS/JS/workspace
overrides, or (2) run ERPNext **headless** and build a custom frontend over the
Frappe REST API.

## Decision

**Headless-hybrid: a custom Admin Portal SPA served as Frappe `www/` pages
(same-origin), with the Desk retained only as a System Manager fallback at
`/app`.** Not a CSS reskin; not a separate-origin frontend.

Mechanics:

- **App-agnostic, config-driven sidebar.** The portal is a generic shell; sidebar
  items are **not hardcoded**. Each app declares its nav items + section metadata
  (label, route, icon, API source) via a hook (e.g. `get_portal_nav_items()` in
  `hooks.py`, or a `portal` config block). The shell reads this at boot and
  renders the sidebar dynamically, so the same shell serves `real_estate_os`
  today and other industry apps later.
- **Packaging.** The shell lives inside `real_estate_os` as a self-contained,
  app-agnostic package (`real_estate_os/portal/`), driven only by declared config.
  Extraction into a shared `portal-framework` app is deferred until a second
  industry app exists.
- **Serving.** Static SPA bundle under `real_estate_os/public/portal/`, loaded by
  a thin `www/portal/index.html` shell. Same-origin → reuses the Frappe session
  cookie (`sid`), so **no CORS, no token plumbing**.
- **Auth.** Custom login → `POST /api/method/login` → `sid` cookie. RBAC enforced
  by Frappe's per-DocType role permissions (the same roles the Desk uses).
- **Data.** `GET/POST /api/resource/<doctype>` for CRUD + whitelisted
  `@frappe.whitelist()` methods in `api.py` for aggregated section data.
- **Landing.** Portal is the site landing (`website_route_rules` / `home_page`);
  the Desk (`/app`) is gated to System Manager.

**Landlord** — a `real_estate_os`-app doctype ONLY (not part of the portal
shell/framework): a Landlord is a **building owner** from whom the company leases
a building. When the company owns the building itself, the Landlord references
the operating Company (self-owned). Modeled as a new `Landlord` DocType in the
real_estate_os app; `Building` links to it via a `landlord` field. The portal's
"Landlords" nav item is itself a real_estate_os-declared entry in its nav config.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Reskin Desk (CSS/JS + workspace) | Zero rebuild — reuses list/form/report/permission UI | Cannot truly remove the skin; desk shell core + brittle to override; workspace-driven sidebar can't be a free link list; breaks on ERPNext upgrades; chrome persists | rejected |
| B. Full headless (standalone SPA, separate origin) | Cleanest separation; any stack; desk optionally disabled | 2nd host + CORS + token/API-key auth; rebuild all CRUD; more infra + auth surface | viable but heavier |
| C. Headless-hybrid: custom SPA via `www/` + desk as fallback | Full sidebar/logo/nav control; same-origin session auth; reuses Frappe auth/RBAC/API; desk kept for admins | Must build all portal CRUD UI; two surfaces to keep consistent | **chosen** |

## Consequences

### Positive

- Exact Admin Portal look (sidebar + logo + links) with no ERPNext chrome.
- Backend superpowers free: Frappe auth, RBAC, DocTypes, permissions, workflows,
  audit, validation, print formats.
- No CORS/token infra; session-cookie auth works same-origin.
- Decoupled from desk internals → low upgrade risk vs CSS reskin.
- Desk remains a safety valve for admins/devs.
- Config-driven shell → one reusable layout across future industry apps.

### Negative

- Must build CRUD UI for all 7 sections (list, form, filters, file/print).
- Two UI surfaces (portal + desk) to keep consistent.
- New frontend stack + build tooling (app is currently pure Python + Jinja).

## Required convention for every headless-hybrid portal (real_estate_os and any future OS module)

Found live (2026-09-05, both `real_estate_os` and `platform_console`): Frappe's
own `frappe.core.doctype.user.user.update_password` returns
`get_default_path() or "/desk"` for a System User — **not** the string
`"/app"` an earlier version of this pattern's comments assumed. Since no
tenant/console app registers an `add_to_apps_screen` workspace,
`get_default_path()` is always `None`, so `update_password` always returns
the literal string `"/desk"` after a password reset. A naive exact-string
check (`redirectPath !== "/app"`) lets `"/desk"` straight through, and the
browser follows it into raw Frappe Desk (`/desk` aliases to `/app`, whose own
client router then lands on `/app/home`) — exactly the automatic-Desk-default
this ADR's "headless-hybrid" decision exists to prevent. The same gap exists
in the login flow's own `redirect-to` query param (`?redirect-to=/app`).

This does **not** reverse the decision above — `/app` stays an intentional,
role-gated System Manager/Platform Admin fallback, reachable by deliberate
navigation. The fix is narrower: **no automatic redirect (post-login,
post-password-reset) may ever default into `/app` or `/desk`.** Every OS
module's portal/console `ui/` must carry both of these, copied from
`real_estate_os/ui` (or `platform_console/ui`) rather than re-derived:

- `SetPassword.tsx`: an `isDeskPath(path)` check (matches `/app`, `/desk`,
  and any subpath of either) guarding what `update_password`'s return value
  is allowed to redirect to — never a bare `!== "/app"` string comparison.
- `Login.tsx`'s `safeRedirect()`: the same `isDeskPath`-equivalent check
  applied to the `redirect-to` query param, alongside the existing
  cross-origin and `/login`/`/update-password` guards.
- The server-side mirror in `www/<page>/index.py`'s `safe_redirect()`
  (the already-authenticated-user branch for `/login`/`/update-password`)
  needs the identical `_DESK_SLUGS = {"app", "desk"}` check.

## Implementation

- Feature file: `vault/ui-portal/features/admin-portal-ui.md`
- Feature file: `vault/custom-module/features/landlord-doctype.md`
- Fix: `vault/ui-portal/features/native-auth-pages.md`'s "Follow-up (2026-09-05)"
  section (desk-redirect hardening, applied to both `real_estate_os/ui` and
  `platform_console/ui`)
