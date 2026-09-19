---
status: in-progress
owner: developer-1
domain: multi-tenancy
created: 2026-09-05
updated: 2026-09-05
related_adr: ["0022-console-admin-ui"]
---

# Console Admin UI

## Summary

Replaces `platform_console`'s plain Frappe desk form with a purpose-built React
admin UI (same stack/conventions as `apps/real_estate_os/ui`) for managing every
business on the SaaS platform: onboarding, a searchable business list/detail
view, subscription tracking, and platform-wide settings.

## Requirements

- Businesses list: search + filter by provisioning status, subscription status,
  module, country.
- Business detail view: tenant info, integrations, subscription panel (plan,
  status, Suspend/Reactivate/Cancel actions with confirmation), provisioning
  error log when `status = Failed`.
- New Business onboarding wizard replacing the raw `Tenant Site` form: company
  info → module & country → admin email → integrations → plan selection →
  review & create. Wraps the existing `provision_tenant_site` flow — does not
  change how provisioning itself works.
- Dashboard: counts by provisioning status / subscription status / module,
  recent onboarding activity.
- Settings page: `Platform Settings` doctype fields + `Platform Plan` catalog CRUD.
- Gated to the `Platform Admin` role, same trust model as today.

## Design

- New `ui/` directory at the `platform_console` repo root, mirroring
  `apps/real_estate_os/ui/` exactly: Vite + TS + React 19 + React Router v7 +
  Tailwind v4 + shadcn/ui (`new-york`/`neutral`), same `components.json`.
  Garet font + Aetris logo vendored in from the repo's `asset/` directory.
- `platform_console/www/console/index.py` (new) injects `window.consoleBootstrap`
  (user, csrf_token, nav) — same pattern as
  `real_estate_os/www/portal/index.py` → `window.portalBootstrap`.
- `ui/src/lib/api.ts` copies the `call()` / `listResource()` / `createResource()`
  / `updateResource()` wrappers from `real_estate_os/ui/src/lib/api.ts` (already
  backend-agnostic) plus new typed functions for the endpoints below.
- Build output committed to `platform_console/platform_console/public/console/`
  (`bun run build`, no build step needed on `install-app`).
- New backend doctype `Platform Plan` (internal catalog only — no payment gateway
  fields): `plan_name`, `price`, `billing_period`, `features` (free text).
- New/extended whitelisted methods in `platform_console` (guarded by the
  existing `_require_platform_admin()` pattern in `provisioning.py`):
  - Businesses list/detail: Frappe's own `/api/resource/Tenant Site` REST
    list/get (matches `listResource()`/`getResource()`'s existing contract,
    including the `integrations` child table) — no new endpoint needed.
  - Plan CRUD via the standard `/api/resource/Platform Plan` REST endpoints.
  - Subscription/lifecycle actions: see `tenant-lifecycle-enforcement.md` —
    this feature's UI calls `suspend_tenant_site`, `reactivate_tenant_site`,
    `cancel_tenant_site`, `set_plan` defined there.
- Pages (`ui/src/pages/`): `Login.tsx`, `Dashboard.tsx`, `Businesses/List.tsx`,
  `Businesses/Detail.tsx`, `Onboarding/NewBusiness.tsx`, `Settings.tsx`.
- **Resend admin invite** (added 2026-09-05, Imran: "if a business owner
  tries [the invite link] after expiring, how do we resend that email?"):
  `resend_admin_invite(name)` in `platform_console/provisioning.py` reuses
  the exact bench-console-exec mechanism (`_finalize_new_tenant`) and
  branded-email sender (`_send_branded_welcome_email`) already built for
  onboarding — a new `real_estate_os.provisioning.resend_admin_invite`
  regenerates the User's reset link (`_reset_password(send_email=False)`),
  platform_console re-sends the same Aetris-branded email. A "Resend
  Invite" button on `Businesses/Detail.tsx` triggers it for any Live
  tenant. Also fixed the guest-facing side: an expired/used
  `/update-password` link now shows a distinct "this link expired" state
  with a link to sign-in's existing "Forgot your password?" flow, instead
  of a generic error inviting a retry that could never succeed.

## Implementation Plan

- [x] Scaffold `platform_console/ui/` (Vite + TS + React 19 + RRv7 + Tailwind v4
      + shadcn `new-york`/`neutral`), matching `real_estate_os/ui`'s
      `components.json` and `index.css` token structure; vendor Garet font +
      Aetris logo from `asset/`.
- [x] Add `platform_console/www/console/index.py` + `index.html` bootstrap
      page, injecting `window.consoleBootstrap`; register the SPA fallback
      route in `hooks.py` (`website_route_rules`), mirroring `real_estate_os`.
      **Gotcha found live**: `www/` and `public/` must sit directly under the
      app package dir (`platform_console/www/`, `platform_console/public/`),
      NOT nested one level deeper under the module-name subfolder (easy
      mistake here specifically because this app's one module is also named
      "platform_console") — Frappe's `PathResolver` 404'd until this was
      fixed, confirmed via `docker inspect`/live `curl`.
- [x] Port `src/lib/bootstrap.ts` and `src/lib/api.ts` core wrappers from
      `real_estate_os/ui`; add console-specific typed calls.
- [x] `Login.tsx` — session login gated to `Platform Admin` (reuse
      `real_estate_os/ui`'s Login page pattern/visual design).
- [x] `Platform Plan` doctype (backend) + Settings page Plan CRUD UI.
- [x] `Dashboard.tsx` — status/module counts, recent activity feed.
- [x] `Businesses/List.tsx` — table + search/filter (status, subscription
      status, module, country).
- [x] `Businesses/Detail.tsx` — tenant info, integrations, subscription panel
      wired to the lifecycle actions (see `tenant-lifecycle-enforcement.md`).
- [x] `Onboarding/NewBusiness.tsx` — multi-step wizard calling the existing
      `provision_tenant_site`, `get_coa_options_for_country`,
      `get_currency_for_country`, `get_available_integrations` endpoints.
- [x] `bun run build`, commit the bundle, deploy to `console.nnuggets.com` —
      live and verified reachable 2026-09-05 (image rebuilt, containers
      recreated, `bench migrate` + `clear-cache` + `clear-website-cache` run;
      also required a fix to the VPS's untracked `apps/Dockerfile`, which had
      no `/assets/platform_console` symlink line at all before this).

## Acceptance Criteria

- [x] Logging into `console.nnuggets.com` shows the new React UI (`/login`
      renders `window.consoleBootstrap` + the SPA bundle), not the Frappe
      desk list view — verified via `curl` against the live site.
- [x] A non-authenticated (Guest) user is redirected to `/login`, never shown
      console content — verified live (`/`, `/dashboard` both 301 to
      `/login?redirect-to=...` as Guest).
- [ ] A logged-in user without `Platform Admin`/`System Manager` gets a 403,
      not the console — not yet tested against a real non-admin user.
- [ ] New Business wizard successfully provisions a real tenant site end-to-end
      — **pending Imran's live test** (create + delete a tenant).
- [ ] Businesses list correctly filters/searches; detail view shows accurate
      provisioning + subscription state — pending live click-through.
- [ ] Settings page can create/edit a Plan and it's selectable during
      onboarding and on the subscription panel — pending live click-through.

## Related

- Domain index: `vault/multi-tenancy/multi-tenancy.md`
- ADR: `vault/decisions/0022-console-admin-ui.md`
- Companion feature: `vault/multi-tenancy/features/tenant-lifecycle-enforcement.md`
- Design reference: `apps/real_estate_os/ui/` (stack + conventions to mirror)
