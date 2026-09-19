# ADR-0022: Platform Console Admin UI — Custom React Frontend

- **Status**: accepted
- **Date**: 2026-09-05
- **Deciders**: Imran
- **Supersedes**: refines the UI-layer scope of `0008-tenant-provisioning-control-plane` (that ADR's backend/provisioning mechanics — `bench new-site`, per-module finalize, Traefik routing — are unchanged)
- **Superseded by**: -

## Context

`0008-tenant-provisioning-control-plane` deliberately scoped `console.nnuggets.com`
(app `platform_console`) to v1 onboarding only, on plain Frappe desk forms: *"One
DocType, `Tenant Site` ... this *is* the UI: a Frappe list/form, no custom frontend
needed for v1."* That scope cut was correct at the time — there was exactly one
DocType and one action (Provision).

The console's job has since grown beyond that v1 slice. It needs to be the actual
back-office for the SaaS: a searchable/filterable list of every business on the
platform, a business detail view showing subscription state alongside provisioning
state, a guided onboarding flow (not a raw DocType form with dependent Select
fields), lifecycle actions (Suspend/Reactivate/Cancel) with confirmation UX, and
platform-wide settings (branding, plan catalog). Frappe's desk list/form UI can
technically grow to cover all of this, but it does so as a generic DocType editor
— not as a purpose-built operator console, and it diverges visually/UX-wise from
the tenant-facing Admin Portal (`apps/real_estate_os/ui`) the platform already
built and standardized on for exactly this kind of work.

We already have a working, proven pattern for "custom SPA served same-origin by a
Frappe app, authenticated by the existing session, no new auth stack" —
`0006-admin-portal-approach` and `0007-admin-portal-react-frontend`, live at
`realestate.nnuggets.com` today. Reusing it for the console avoids re-litigating
those same tradeoffs (headless-hybrid vs. Frappe Desk, React vs. Vue, build/deploy
mechanics) a second time.

## Decision

`platform_console` gets a custom React frontend, built and wired the same way as
`apps/real_estate_os/ui`:

- **Stack**: Vite + TypeScript + React 19 + React Router v7 + Tailwind v4 + shadcn/ui
  (`new-york` style, `neutral` base, CSS-variables mode) + Lucide icons — identical
  `components.json` conventions to `real_estate_os/ui`.
- **Branding**: Garet font + Aetris logo, sourced from the repo's `asset/`
  directory, vendored into `platform_console`'s own `ui/src/assets/` the same
  way `real_estate_os/ui` already vendors its own copy.
- **Auth/data**: no new auth stack. Runs inside an authenticated Frappe session,
  gated server-side to the `Platform Admin` role (already exists, from ADR-0008's
  patch). A `www/console/index.py` injects `window.consoleBootstrap` (user, CSRF
  token, nav) the same way `real_estate_os/www/portal/index.py` injects
  `window.portalBootstrap`. Data access reuses the same `frappe.call()` /
  `/api/resource/*` REST wrapper pattern already in `real_estate_os/ui/src/lib/api.ts`
  (that file is already backend-agnostic, not `real_estate_os`-specific).
- **Build/deploy**: `bun run build` writes into
  `platform_console/platform_console/public/console/`; the compiled bundle is
  committed to the `platform_console` repo, no build step required on
  `bench get-app`/`install-app` — same acceptance-test-friendly mechanics as
  `real_estate_os/ui`.
- **Scope of pages for this phase**: Dashboard, Businesses (list + detail),
  New Business onboarding wizard (replaces the raw `Tenant Site` form), Settings
  (Platform Settings + Plan catalog). Detailed in the feature file, not here.

This formally extends ADR-0008 rather than reversing it: the underlying
Tenant Site provisioning mechanism is unchanged. Only the operator-facing UI
layer changes from Frappe Desk forms to a custom SPA.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Keep growing the Frappe Desk form (more fields, more Select-driven client scripts) | Zero new frontend to build/maintain | Gets unwieldy fast for list filtering, multi-step onboarding, and confirm-before-destructive-action UX; permanently diverges visually from the tenant Admin Portal | rejected |
| B. Custom React SPA, same stack/pattern as `real_estate_os/ui` | Proven pattern already in production; consistent operator experience across every "admin" surface in the platform; reuses existing `api.ts`/bootstrap conventions | One more `ui/` tree + build step to maintain, in a second repo | **chosen** |
| C. Adopt the `saas-admin-ui-main` scaffold (Convex-backed AI-generated template) as a starting point | Already has a fleshed-out dashboard/accounts UI to crib visual ideas from | Backend is Convex — architecturally incompatible with this Frappe-hosted, session-cookie-authenticated platform; would require ripping out and replacing its entire data layer | rejected (visual reference only, not a code base) |

## Consequences

### Positive

- One consistent design language (Tailwind v4 + shadcn "new-york" + Garet +
  Aetris) across every admin-style surface in the platform — tenant Admin Portal
  and the SaaS console.
- No new auth/session mechanism; console access is just "log in as a user with
  the `Platform Admin` role," same trust model as today.
- `platform_console`'s existing whitelisted methods (`provision_tenant_site`,
  `get_coa_options_for_country`, etc.) are reused as-is by the new frontend —
  no backend rewrite forced by this decision.

### Negative

- A second `ui/` build tree (separate repo, separate `bun run build` step) to
  keep in sync with backend changes — same operational overhead already
  accepted for `real_estate_os/ui`.
- Console now has a client-side bundle exposed to whoever can reach
  `console.nnuggets.com` (even if they can't log in) — no change in practice
  since Frappe Desk already exposes similar surface area, but worth naming.

## Implementation

- Feature file: `vault/multi-tenancy/features/console-admin-ui.md`
