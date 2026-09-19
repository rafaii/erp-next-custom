# ADR-0027: Selectable Admin UI — React (default) + Vue "Classic" Skin

- **Status**: accepted
- **Date**: 2026-09-17
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

`0006-admin-portal-approach` and `0007-admin-portal-react-frontend` settled
the Admin Portal on a headless-hybrid custom SPA — React 19 + Vite + TS +
Tailwind v4 + shadcn/ui, served same-origin via Frappe `www/portal`,
config-driven sidebar (`portal_nav_items` in `hooks.py`). `0022` reused the
identical stack/pattern for `platform_console`. Both ADRs explicitly
evaluated and rejected Vue for the primary portal stack.

Imran wants a second, switchable admin UI in the visual style of
[vue-element-admin](https://github.com/PanJiaChen/vue-element-admin)
(collapsible sidebar, tags-view, breadcrumbs, dense tables), selectable from
Settings — not a reversal of 0007/0022, an optional second skin alongside the
React default.

Two facts shape the scope:

1. **vue-element-admin itself is not a viable base.** Confirmed via the
   GitHub API: Vue 2.6.10 + Element UI 2.13.2, last commit 2024-10-24, no
   Vue 3 branch. Vue 2 has been EOL since December 2023. Shipping new
   production surface on it means building on a dead framework version.
   `vbenjs/vue-vben-admin` (Vue 3 + TypeScript + Vite, actively maintained,
   pushed the same day this ADR was written) sits in the same admin-template
   family — same layout DNA, current framework — and is the reference/
   scaffold used instead of the literal repo.
2. **The cost is not evenly spread across the 9 nav sections.** Measured
   directly in `ui/src/pages/Dashboard.tsx` (10.8k lines): Properties,
   Tenants, Maintenance, and Landlords render through the generic
   `ResourceListView(item, query)` component, driven entirely by
   `portal_nav_items`' `{doctype, columns, api}` config — cheap to
   reimplement in a second stack. Overview, Actions, Accounts (GL, Trial
   Balance, P&L, Balance Sheet, Journal Entries, Bank Reconciliation, rent
   roll), Settings (banks, email accounts, modules), and Contracts (lease
   signing, PDC schedules, the e-sign template designer with drag-drop field
   placement) are large, hand-built, bespoke components with no shared
   render contract. Committing to full parity across all 9 sections upfront
   means building and maintaining five substantial custom UIs twice, forever.

## Decision

Add a second, optional admin UI ("**Classic**" — Vue 3 + Vite + TypeScript,
in the vue-element-admin/`vue-vben-admin` visual family — dark collapsible
sidebar, icon+label nav — hand-built rather than scaffolded from either
repo directly, since Phase 1's 4-section scope doesn't justify vendoring
either project's full build)
alongside the existing React default ("**Modern**", unchanged). React remains
the only complete, fully-supported implementation; Classic is additive and,
for now, partial.

Mechanics:

- **Selection is per-tenant, not per-user.** A new `admin_ui` Select field
  (`Modern` default / `Classic`) on `Real Estate Settings` — the existing
  Single doctype already holding tenant-wide config (`amenity_options`,
  `default_labor_rate`) — same Tier-1 custom-field-on-an-existing-settings-
  doctype pattern `0017-platform-modularization` established for
  `Signature Settings.esign_mode`. Editable from the portal's Settings page.
  Chosen over a per-user toggle because this is a business-wide look-and-feel
  decision, not a personal preference, and matches how `0017` already
  resolved an equivalent tenant-facing choice.
- **One `www/portal` page, not a second one.** `hooks.py` rewrites ~15 slugs
  plus their `/<path:name>` variants to `to_route: "portal"`, and `/login` +
  `/update-password` are routed through that same page. A `www/portal-v2/`
  would force duplicating that entire route table and the
  `safe_redirect`/`_DESK_SLUGS`/guest-bootstrap logic in
  `real_estate_os/www/portal/index.py` — precisely the re-derivation
  `0006`'s "Required convention" section warns against (it's what caused the
  earlier live `/desk`-redirect leak). Instead, `get_context` reads
  `Real Estate Settings.admin_ui` and adds it to `portal_bootstrap`; the
  Jinja shell picks which bundle's script tag to emit based on that value.
  Auth, routing, RBAC, and the desk-redirect guards stay single-sourced.
- **Auth pages stay React-only.** `Login.tsx`/`SetPassword.tsx` (and their
  `isDeskPath` guards) are not duplicated into Vue. A Classic-preference
  tenant still logs in and resets passwords through the existing React
  flow; only the authenticated shell after login switches. This keeps the
  security-sensitive redirect-hardening code in exactly one place.
- **Shared render contract, not a hand port.** Both UIs consume the same
  `portal_nav_items` config and the same REST/whitelisted-method API
  surface (the pattern in `ui/src/lib/api.ts`, already confirmed
  backend-agnostic by `0022`). Classic implements its own generic list/detail
  component reading `{doctype, columns, api}` from nav config — so a future
  section added to `portal_nav_items` appears in both UIs for free, as long
  as it's generic. This is also the second real consumer `0017` said would
  be needed to validate extracting a vertical-agnostic render contract out of
  the React portal; not done now (still only one OS app), but named as the
  eventual extraction boundary.
- **Phased rollout with an explicit gate.** Phase 1 ships Classic for the 4
  generic sections only (Properties, Tenants, Maintenance, Landlords). The
  other 5 sections show a "not available in Classic yet" panel linking back
  to the React view. Porting those (Phase 2) requires a fresh go/no-go from
  Imran once Phase 1 is live — not an assumed continuation.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Theme/token reskin of the existing React portal to visually mimic vue-element-admin (collapsible sidebar, tags-view, dense tables) | No second framework, no second build tree, instant full parity since it's the same codebase | Doesn't satisfy "a second UI" literally — same React tree under different CSS/layout tokens | rejected (cheaper, but not what was asked) |
| B. Literal vue-element-admin (Vue 2.6 + Element UI) | Exact visual match to the linked repo, fastest to crib from directly | Builds new production surface on a framework EOL since Dec 2023, with no upgrade path in that repo | rejected |
| C. Vue 3 successor in the same visual family (`vue-vben-admin` as scaffold), full parity across all 9 sections upfront | Complete second UI from day one | 5 of 9 sections are large bespoke builds (Accounts alone: 6 sub-panels); doubles ongoing maintenance immediately with no validation of demand first | rejected (scope, not stack) |
| D. Vue 3 successor, phased — 4 generic sections first, gated Phase 2 | Ships the switchable-UI capability at low cost; validates demand before paying for the 5 expensive ports; keeps React as the sole complete implementation | Classic is visibly incomplete until/unless Phase 2 is approved | **chosen** |

## Consequences

### Positive

- Delivers the actual ask — a genuinely different, switchable second
  frontend — without committing to porting the most expensive, fastest-
  changing parts of the portal (Accounts, e-sign designer) before there's
  any signal it's worth it.
- Reuses every existing backend decision (session auth, RBAC, `portal_nav_items`,
  the REST/whitelisted-method API) unchanged — no new auth stack, no new
  permission model, no backend rewrite.
- Establishes a real render contract (`{doctype, columns, api}` → generic
  list/detail) as a byproduct, which is exactly the extraction `0017`
  anticipated needing a second consumer to validate.

### Negative

- A second `ui-classic/` build tree and a second committed bundle
  (`apps/real_estate_os/real_estate_os/public/portal-classic/`) — the same
  "`bun run build` + commit the bundle" operational overhead already
  accepted for `ui/`, now doubled.
- Two frontend stacks to keep in sync with every backend API change for as
  long as Classic exists, even in its partial Phase-1 form.
- Classic is incomplete by design until a Phase 2 decision — 5 of 9 sections
  fall back to a "not available yet" panel, which is a visible product gap
  a business could notice right after switching.

## Implementation

- Feature file: `vault/ui-portal/features/selectable-admin-ui.md`
- Phase 1: `Real Estate Settings.admin_ui` field, Settings-page toggle,
  `ui-classic/` scaffold (4 generic sections), `www/portal/index.py` bundle
  selection.
- Phase 2 (gated, not started): port Overview, Actions, Accounts, Settings,
  and Contracts's bespoke panels into Classic — requires a fresh decision
  from Imran after Phase 1 ships.
