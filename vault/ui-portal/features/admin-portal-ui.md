---
status: done
owner: ui-designer-1
domain: ui-portal
created: 2026-08-19
updated: 2026-08-30
related_adr: ["0006-admin-portal-approach", "0007-admin-portal-react-frontend", "0011-head-lease-landlord-payables"]
---

# Admin Portal UI (custom SPA, headless-hybrid)

## Summary

Replace the default ERPNext Desk skin with a purpose-built Admin Portal — a
static SPA served same-origin via Frappe `www/`, with a sidebar (aetris logo +
9 nav links: Overview, Properties, Contracts, Tenants, Maintenance, Landlords,
Accounts, Reports, Settings). Frappe stays as the headless backend (auth, RBAC,
DocTypes, REST API); the Desk remains only as a System Manager fallback at `/app`.

## Requirements

- Custom sidebar with logo + 9 nav links; no ERPNext chrome.
- **Config-driven sidebar** — nav items NOT hardcoded. Each app declares its nav
  + section metadata (label/route/icon/API source) via the `portal_nav_items`
  hook in `hooks.py`; the shell renders it dynamically, so the same shell is
  reusable across future industry apps.
- Session-cookie auth (custom login → `/api/method/login`, `sid` cookie), RBAC
  via existing roles.
- 9 sections, each backed by Frappe REST + whitelisted methods:
  - Overview — KPIs (occupancy %, lease expiries, PDC due, open maintenance).
  - Properties — Building + Unit (occupancy map entry point).
  - Contracts — Lease Agreements (status, rent, esign status).
  - Tenants — tenant/Customer list + lease status.
  - Maintenance — maintenance requests (submit, track, allocate cost).
  - Landlords — landlord party list (+ lease terms per landlord). Landlord =
    building owner; when the company owns the building itself, Landlord links the
    Company (self-owned). Backed by the `Landlord` DocType (see landlord-doctype).
  - Accounts — invoices, payments, PDC, profitability.
  - Reports — aggregated reporting views.
  - Settings — app/system settings (e-sign settings, cost centers).
- Portal is the site landing; `/app` gated to System Manager.

## Design

- **SPA stack**: React 19 + Vite + TypeScript + Tailwind v4 + shadcn/ui (see
  ADR-0007; replaces the earlier Vue 3 default that was only a placeholder).
- **Source location**: the SPA source lives in `ui/` at the META repo root (NOT
  inside the app). Convex / VLY / Freebuff backends stripped; Frappe is the only
  backend.
- **Layout**: built bundle emitted to
  `apps/real_estate_os/real_estate_os/public/portal/` +
  `real_estate_os/www/portal/index.html` as the thin Jinja shell that loads it.
  Served same-origin → no CORS, no token plumbing.
- **Config-driven shell**: the portal shell is app-agnostic; it reads nav +
  section config declared by the app via the `portal_nav_items` hook at boot (no
  hardcoded sidebar). Extractable to a shared `portal-framework` app once a second
  industry app exists.
- **Auth**: custom login form → `POST /api/method/login`; rely on the `sid` cookie.
- **API**: whitelisted `@frappe.whitelist()` methods in `real_estate_os/api.py`
  for each section (aggregated reads + mutations), plus direct
  `/api/resource/<doctype>` calls for standard CRUD. Detail views use
  `get_doc_detail` / `get_linked_records` to load a record + its linked records.
- **Routing**: `home_page` hook so the portal is the default landing for
  non-admin roles.
- **RBAC**: portal-only roles (Property Manager, Leasing Agent, Maintenance,
  Accounts) get no Desk access; System Manager keeps `/app`.

## Implementation Plan

- [x] Phase 0 — ADR-0006 approved (headless-hybrid). Frontend stack confirmed in ADR-0007: React 19 + Vite + TS + Tailwind v4 + shadcn/ui.
- [x] Phase 1 — Scaffold SPA (Vite build wired to emit into `public/portal/`), thin `www/portal/index.html` shell.
- [x] Phase 2 — Auth: login page + session bootstrap (`/api/method/frappe.auth.get_logged_user`), logout, 401 handling.
- [x] Phase 3 — Shell: sidebar + aetris logo + client-side router for the 9 links; responsive layout.
- [x] Phase 4 — Backend API: whitelisted methods in `api.py` for each section's aggregate data.
- [x] Phase 5 — Section screens: Overview → Properties → Contracts → Tenants → Maintenance → Landlords → Accounts → Reports → Settings (list + form + filters), incl. detail views with linked records (`get_doc_detail` / `get_linked_records`).
- [x] Phase 6 — RBAC + landing: portal roles, restrict Desk to System Manager, set `home_page`.
- [x] Phase 7 — Deploy: rebuilt custom image → VPS `git pull` → `docker compose up -d` → verify portal at https://realestate.nnuggets.com.

## What was implemented

- Same-origin `www/portal` shell with `portalBootstrap` Jinja data (renamed from
  `__PORTAL__` to avoid the jinja illegal-template error).
- `home_page` hook so the portal is the site landing.
- Login + session bootstrap via the Frappe `sid` cookie.
- Config-driven 9-section sidebar with the aetris logo (`aetris.svg`,
  `aetris.png` fallback), driven by `portal_nav_items` in `hooks.py`.
- Section screens incl. detail views with linked records (`get_doc_detail` /
  `get_linked_records` whitelisted methods in `api.py`).
- Demo seed data (`demo_data.py`) for buildings, units, leases, landlords,
  tenants, contracts.
- RBAC via existing roles; Desk gated to System Manager.
- Deployed live at https://realestate.nnuggets.com (custom image
  `realestate-custom:v15.119.3`).

## Post-launch extensions (2026-08-21 to 2026-08-23)

The v1 above shipped 2026-08-20; the portal has since been hardened against
real usage on the live Rustic tenant (`realestate.nnuggets.com`). See
`CHANGELOG.md` for the full commit-by-commit trail. Highlights:

- **Real URLs**: `BrowserRouter` replaces `HashRouter`; `website_route_rules`
  rewritten from a dangerous catch-all (was breaking `/login`/`/esign` with a
  redirect loop) to an explicit list derived from `portal_nav_items`.
- **Editing**: records editable directly from the detail view across sections.
- **Tenant/lease split flow**: creating a tenant and creating a lease for an
  existing tenant are separate actions (New Lease dialog: Manual or In-built
  e-sign only); `LeaseSigningPanel` drives Draft → Sent → Signed, surfaces
  send failures + a resend action, and shows an activity timeline.
- **Field visibility**: `Portal Field Visibility` DocType — drag-to-reorder,
  show/hide fields per doctype from a view-settings icon, persisted server-side.
- **Customer codes**: `CUST-2026-#####` naming (via Selling Settings
  `cust_master_name`) replaces name-in-URL tenant links.
- **Lease integrity**: signed leases are immutable (Cancel-only); editing a
  Sent lease auto-invalidates and resends the signing link with a confirm
  dialog; `monthly_rent`'s `fetch_from` was removed after it was found to
  silently revert admin-entered rent to the unit's reference rent on every save.
- **E-sign hardening**: OTP verification is a 5-minute session gate (IP-bound),
  not a one-time unlock; `esign_envelope_id` is now set by the real signing
  flow; a "View signed contract" download replaces the dead-end 404 on the raw
  E-Sign Document route.
- **Settings**: Business Signatory and SMTP/IMAP Email Accounts are now
  configurable from the portal Settings page instead of only in the Desk.
- **2026-08-30: Modules & integrations panel**: Imran wanted visibility
  into what's installed on the site — a new Settings panel lists this
  OS's extension apps via Frappe's own `get_versions()` (excluding core
  `frappe`/`erpnext`), plus an Integrations table showing the active
  e-sign provider and whether the `esign_provider` hook actually has
  something registered for it (the same check `e_sign/dispatch.py` uses,
  so it can't falsely claim "Connected"). Rides the existing
  `get_settings_data()` call, no new endpoint.
  **Same-day fix**: Imran caught `inbuilt_esign` showing up as both a
  generic Module and the resolved E-Sign integration — the same fact
  twice under different labels, which would have doubled up again once a
  Zoho/DocuSign provider existed. `get_installed_modules` now excludes
  any app registered as a provider for a swappable integration slot
  (`_PROVIDER_HOOKS`, currently just `esign_provider`); `get_integrations`
  resolves the provider's own app title ("Inbuilt E-Sign") instead of
  echoing the raw `esign_mode` string ("In-Built"). Modules now correctly
  shows just `Real Estate OS`/`Accounts Portal`.
- **2026-09-01: Aetris-branded website auth pages**: found while
  chasing an onboarding-email complaint (see
  `multi-tenancy/features/tenant-provisioning-console.md`) that `/login`,
  `/update-password`, and `/forgot-password` were all still stock
  ERPNext branding (erpnext logo, "Powered by ERPNext" footer) on every
  tenant including this live one — the portal above only takes over
  post-login, so none of Frappe's own website auth pages were ever
  covered. Fixed via `Website Settings` (`app_logo`/`banner_image`/
  `favicon`/`footer_powered`), the single source those pages read from —
  reuses the same `aetris.svg`/`.png` this portal's own sidebar already
  uses. Set once per tenant in `real_estate_os.provisioning
  .finalize_new_tenant`; backfilled onto this already-live tenant
  directly.
- **2026-09-02: Per-tenant outgoing mail (Aetris-managed or own SMTP)**:
  found right after the branding fix above — clicking "reset password"
  sent nothing, on every tenant, since no site had ever had a working
  Email Account. `real_estate_os.brevo_mail.dispatch_outgoing_mail`
  (registered as `override_email_send`, which pre-empts Frappe's real
  SMTP delivery entirely) now routes every outgoing email: through the
  business's own Email Account if they've configured a real, working one
  on this Settings page — no new toggle, that configuration action
  itself is the opt-in — otherwise through Brevo, billed to Aetris
  (credits/billing model deferred). See
  `multi-tenancy/features/tenant-provisioning-console.md` for the full
  mechanism (the bench-wide placeholder Email Account, the
  `_from_site_config` discriminator).
- **2026-09-02: Native React `/login` + `/update-password` pages** —
  replaces Frappe's stock website auth pages outright (the 2026-09-01
  entry above only re-skinned them). Full design, live-verification
  detail, and the redirect-loop risk this closes: see the dedicated
  `native-auth-pages.md` feature doc.
- **Head Lease / landlord payables (PR #12, ADR-0011)**: Building's linked
  records now include a "Head Lease" row (1:1, so a single clickable entry,
  not a separate top-level nav item — confirmed with Imran). Head Lease's
  own detail page gets a `HeadLeasePanel`: the payment schedule table (via
  `get_doc_detail`'s existing generic child-table extraction), a "Generate
  outgoing cheques" action, and a "Bank confirmation needed" panel for
  Deposited outgoing cheques — the latter closes a real gap, since the
  existing Accounts-page "Deposited cheques" panel filters to
  `direction=Incoming` only (PR #7) and had no path for Outgoing cheques at
  all. See `payments-accounting/features/landlord-payables.md`.

## Acceptance Criteria

- [x] Fresh `get-app` + `install-app real_estate_os` reproduces the portal (no manual bench/site edits).
- [x] Login lands on the portal (not the Desk) for non-admin roles.
- [x] Sidebar shows logo + 9 links; each section loads real data via API.
- [x] `/app` reachable only by System Manager.
- [x] Tenant-facing data still governed by existing role permissions (no privilege escalation).

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- ADR: `vault/decisions/0006-admin-portal-approach.md`, `vault/decisions/0007-admin-portal-react-frontend.md`
- Related feature: `custom-module/features/landlord-doctype.md`
- Existing portal features: `tenant-portal-ui.md`, `maintenance-portal.md`, `occupancy-dashboard.md`
