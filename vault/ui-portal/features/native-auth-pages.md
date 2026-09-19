---
status: done
owner: developer-1
domain: ui-portal
created: 2026-09-02
updated: 2026-09-05
related_adr: ["0006-admin-portal-approach", "0007-admin-portal-react-frontend"]
---

# Native Login / Set Password Pages (Admin Portal)

## Summary

Imran, after the earlier Website-Settings branding fix: "extend the admin
UI to create these login page. That way the UI will look integrated and
we can use API to connect to the backend." `/login` and `/update-password`
are still Frappe's own stock website pages — separate from the React
admin portal (`ui/src`), which only ever renders once a session already
exists (`admin-portal-ui.md`). The earlier fix (Website Settings: logo,
favicon, footer) re-skinned those Frappe pages; this replaces them
outright with real pages inside the same portal, calling Frappe's
whitelisted APIs directly, so branding, layout, and interaction patterns
are consistent with the rest of the product.

This is a genuinely risky change to plan carefully, not just implement:
`/login` is the ONE page every unauthenticated visit to ANY portal route
(and the Desk at `/app`) gets redirected to. `real_estate_os/hooks.py`
already carries a scar from getting this family of routing wrong once
before — a prior `/<path:app_path>` catch-all rewrote every unmatched
path (including `/login` itself) to the portal shell, and since the
portal shell's `get_context` redirects any Guest straight back to
`/login`, that produced a real, live infinite redirect loop and broke
guest e-sign links at the same time. Confirmed live (not hypothetical)
that naively adding `/login` to `website_route_rules` today would
reproduce exactly that failure mode, because `www/portal/index.py`'s
`get_context` still unconditionally redirects every Guest request away.
Closing that loop correctly is the core of this design, not an
afterthought.

## Requirements

- `/login`: email + password sign-in, calling Frappe's real `login`
  whitelisted method (`usr`/`pwd` fields) directly — no Frappe website
  chrome, matches the rest of the portal's look.
- `/login`, forgot-password mode (toggled within the same page, matching
  Frappe's own UX — not a separate URL): single email field, calls
  `frappe.core.doctype.user.user.reset_password`. Always shows the same
  generic "check your inbox" message regardless of outcome — the backend
  is deliberately anti-enumeration; the frontend must not be smarter than
  that.
- `/update-password?key=...`: new-password + confirm fields, calls
  `frappe.core.doctype.user.user.update_password`. Handles the expired/
  used-key case (HTTP 410) and weak-password rejection with real error
  messages, not a crash.
- Already-authenticated visits to either page redirect home instead of
  re-rendering the form.
- **Explicitly out of scope for v1**: the `old_password`-based "change my
  password while logged in" flow (separate settings-page concern); a
  live password-strength meter (server-side validation on submit is
  enough); magic-link ("email me a login link") sign-in — confirmed live
  via `System Settings.login_with_email_link=1` during pre-flight, but
  Imran's call was to drop it for v1 rather than rebuild it now.

## Design

- **`real_estate_os/www/portal/index.py`** — the ONE shared Jinja
  controller every portal route resolves through. Pressure-tested
  finding: `website_route_rules` rewrites the endpoint *before* Frappe's
  renderer cascade ever looks for `frappe/www/login.py` — once the route
  rule exists, that file is not "shadowed," it's unreachable for this
  site. Everything it did has to be explicitly ported into `get_context`:
  an already-authenticated-user redirect, and open-redirect protection.
  Restructured so `/login` and `/update-password` are the two exceptions
  to the existing Guest-redirect (guest gets a minimal bootstrap — no
  `nav`, no `is_tenant` — instead of being bounced away); an
  already-authenticated visit to either gets redirected home via a
  `safe_redirect()` helper (same-origin, path-only, refuses `/login`/
  `/update-password` themselves as targets — the guard Frappe's own
  `sanitize_redirect()` provided before this route became unreachable).
  Deliberately does NOT port `frappe/www/login.py`'s literal
  already-authenticated guard (`if redirect_to != "login"`) — that
  compares a bare string against a value already turned into a full URL
  by that point, dead code in the original, not worth inheriting.
- **Guest CSRF, verified structurally not just assumed**:
  `frappe/sessions.py` never persists a Guest session (no DB row, no
  cache entry), so `validate_csrf_token()`'s first check is always falsy
  for Guest and always short-circuits past CSRF validation for `login`,
  `reset_password`, and `update_password?key=...`. Still calls the
  existing `_csrf_token()` helper unconditionally in the guest branch
  (one-line, always-correct, no reason to special-case it to `""`).
- **`real_estate_os/hooks.py`** — two more static entries in the existing
  explicit, non-catch-all `website_route_rules` list (same shape as
  `_PORTAL_DETAIL_ONLY_SLUGS`): `/login` and `/update-password` →
  `to_route: "portal"`. No `<path:name>` variant needed — neither takes a
  URL path segment; `/update-password`'s `?key=...` is a query string,
  untouched by route matching. `_reset_password()` already hardcodes the
  emailed link's path as `/update-password?key=...` — no change needed
  there.
- **`ui/src/main.tsx`** — two new lazy routes, deliberately outside
  `RequireAuth` (guest-accessible): `/login` → `Login.tsx`,
  `/update-password` → `SetPassword.tsx`.
- **`ui/src/pages/Login.tsx`** (new) — one page, two modes via local
  state. Sign-in → `call("login", {usr, pwd})` (existing `lib/api.ts`
  helper); success is a full-page `window.location.href` navigation
  (not client-side route push) to a `safe_redirect()`-checked
  `redirect-to`, since `getBootstrap()` is server-injected per-page-load
  — matches the existing `signIn`/`signOut` pattern in `use-auth.ts`.
  Forgot-password mode as described in Requirements. A 429 (rate
  limited) gets its own message on both modes, not folded into the
  generic copy. Styled with existing shadcn primitives already in
  `ui/src/components/ui/` (card, input, button, label, alert) — no new
  primitives needed.
- **`ui/src/pages/SetPassword.tsx`** (new) — reads `key` from the query
  string (missing entirely → immediate "invalid link" state, no API
  call). Confirmed live in Frappe's source: an expired/reused/invalid
  key returns HTTP 410 with a plain `{"message": "<string>"}` body, not
  a raised exception — `lib/api.ts`'s existing `serverMessage()`
  fallback already surfaces this correctly as a thrown `Error`, no
  special-casing needed. **Success is not a confirmation screen**:
  `update_password` ends by logging the user in and returning a redirect
  path (`/desk` for a System User, else the tenant's home) — the page
  navigates straight there via `window.location.href`, matching the
  pattern above.

## Pre-flight (checked live before scoping `Login.tsx`)

Checked via `bench --site realestate.nnuggets.com console`, 2026-09-02:

| Setting | Value | Effect on scope |
| --- | --- | --- |
| `System Settings.enable_two_factor_auth` | `0` | No verification-challenge handling needed |
| `System Settings.disable_user_pass_login` | `0` | Password login is the correct primary path |
| `System Settings.login_with_email_link` | `1` (live, rendered on the current page) | Dropped for v1 per Imran's explicit call — see Requirements |
| `Website Settings.route_redirects` | none | No conflicting redirect wins over the new route rule |
| `Social Login Key` (enabled) | none | No OAuth buttons to account for |

## Implementation Plan

- [x] `www/portal/index.py` restructure — deployed ALONE first (real-estate
      PR #72, no `hooks.py` change yet), verified live as a provable
      no-op: `/login` still resolved to stock Frappe, authenticated
      portal unaffected, guest `/overview` still redirected to
      `/login?redirect-to=/overview` exactly as before.
- [x] `hooks.py` route rules + `ui/src/pages/Login.tsx` +
      `ui/src/pages/SetPassword.tsx` + `main.tsx` routes — deployed
      together (PR #73), then `bench clear-cache` on both
      `realestate.nnuggets.com` and `rastec.nnuggets.com`.
- [x] Live verification (see Acceptance Criteria) on
      `realestate.nnuggets.com`, then `rastec.nnuggets.com`.
- [x] Confirmed `console.nnuggets.com` (separate app, `platform_console`,
      never carries these hooks) is unaffected — still Frappe's stock
      `/login` form, 200.
- [x] **Found live during verification, fixed same session (PR #74)**:
      an invalid-login attempt showed the raw exception class name
      (`frappe.exceptions.AuthenticationError`) instead of the real
      message ("Invalid login credentials") — the legacy `cmd=login`
      endpoint doesn't populate `_server_messages`, so `lib/api.ts`'s
      shared `call()`/`serverMessage()` fallback picked the wrong field.
      Fixed locally in `Login.tsx` (a dedicated `loginCall()` that
      prefers `json.message`) rather than changing the shared helper's
      fallback order for every other caller in the app.

## Acceptance Criteria

- [x] Guest `GET /login` → single `200` with the new React page, not a
      redirect loop — curl-verified (`portalBootstrap`/`id="app"` in the
      body, not Frappe's `form-login`), not just browser-inferred.
- [x] Authenticated `GET /login` → single `302` home, not a re-render of
      the form — curl-verified with a real session cookie.
- [x] Valid sign-in → correct home route; verified via a real throwaway
      test user (`auth-test@rafais.com`, deleted after) hitting
      `/api/method/login` directly — `200`, session cookies set.
- [x] Invalid sign-in → inline error, page still usable — verified live
      (see the PR #74 finding above), now shows the correct message.
- [x] Forgot-password → calls the same `frappe.core.doctype.user.user
      .reset_password` endpoint already proven live to deliver via Brevo
      (see the 2026-09-02 CHANGELOG entry) — not re-tested end-to-end
      with a fresh email send in this pass since the delivery mechanism
      itself doesn't depend on which frontend calls it.
- [x] Emailed link → `/update-password?key=...` renders the new page;
      verified with a real, persisted reset key: `update_password`
      returned `/app/home` (the test user was a System User) and the
      user was already logged in — no bounce through `/login`.
- [x] Re-visiting a used/expired link → clear "invalid/used" error —
      verified live: re-using the same key returned the real 410 message.
- [x] System Manager / Desk access at `/app` still works end-to-end —
      curl-verified reachable post-deploy.
- [x] Rollback path (revert the two `hooks.py` lines + `bench
      clear-cache`) was available and unused — step 1's no-op deploy
      proved the restructure alone before the risky route-rule step ran.

## Follow-up (2026-09-02): Setup Wizard leak + Desk redirect after set-password

Found right after this feature shipped: setting a password from the invite
email landed on `https://rastec.nnuggets.com/app/setup-wizard/0` — stock
ERPNext branding, reached through `/app`, not through anything this
feature's own routing touches. Two distinct bugs, both in the *destination*
`update_password` sends a System User to, not in `/login`/`/update-password`
themselves:

1. `frappe.is_setup_complete()` (gates the Setup Wizard on every `/app`
   boot) reads a cache (`Installed Application` table) computed once, at
   `install_app()` time — before `real_estate_os.provisioning
   .finalize_new_tenant` ever creates the Company. Never recalculated
   after, so `/app` stayed permanently "incomplete" even with a real
   Company + admin user in place. Fixed by having `finalize_new_tenant`
   call `Installed Applications.update_versions()` itself once both
   exist — every future tenant self-corrects at provisioning time.
2. `update_password()` hardcodes `/app` as the destination for every
   System User — Frappe's own Desk, still "ERPNext UI" by the same
   standard this whole feature was built to meet. `SetPassword.tsx` now
   treats that one specific value as "no real target" and sends them
   into the portal (`/`) instead, matching `Login.tsx`'s own default.

See `CHANGELOG.md` (2026-09-02T05:00:00Z) for full detail — real-estate
PR #75.

## Follow-up (2026-09-05): the 2026-09-02 fix was itself too narrow

Imran, testing live on `test.nnuggets.com`: "After password reset and
setting password, it redirects to `.../app/home` (which is ERPNext portal) -
that should NEVER happen! All login should only go to our own UI. ERPNext
should be headless!" — the same bug class as 2026-09-02, recurring, because
that fix's exact-string check (`redirectPath !== "/app"`) was already too
narrow at the time it shipped: this very file's own Acceptance Criteria note
above says the verified value was `/app/home`, not `/app` — that variant was
never actually covered, it just happened not to surface again until now.

Root cause, confirmed by reading Frappe's own source this time
(`frappe/core/doctype/user/user.py`, `update_password`) rather than only
observing the browser's end state: for a System User it returns
`get_default_path() or "/desk"` — literally the string `"/desk"` here, since
no tenant app registers an `add_to_apps_screen` workspace. The browser
follows `/desk`, Frappe aliases it to `/app`, and Desk's own client router
lands on `/app/home` — three different literal strings (`/desk`, `/app`,
`/app/home`) for the same underlying "send me to Desk" outcome, and any
exact-string check will always miss whichever one wasn't tested against.

Fixed properly this time with `isDeskPath()` (`ui/src/pages/SetPassword.tsx`)
— matches the *first path segment* (`app` or `desk`), catching every variant
regardless of what Frappe happens to append after it. Also hardened
`Login.tsx`'s `safeRedirect()` and the server-side mirror in
`www/portal/index.py` for the identical `?redirect-to=/app`-style gap on the
login flow itself, which had never been touched by either the 2026-09-02 fix
or this one until now. Per Imran's explicit direction, this must not be
something each future OS module rediscovers on its own — see the new
"Required convention" section added to `0006-admin-portal-approach.md`, and
applied here to `platform_console/ui`'s identical copy of this same pattern
in the same session (found and fixed there before it ever shipped as a live
bug, unlike this file's history).

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Depends on / extends: `admin-portal-ui.md` (the portal shell and
  `RequireAuth`/`use-auth.ts` patterns this reuses), its 2026-09-01
  Aetris-branding entry (the smaller fix this supersedes for these two
  specific pages) and 2026-09-02 per-tenant-email entry (the Brevo
  pipeline `reset_password` depends on)
- ADR: `0006-admin-portal-approach`, `0007-admin-portal-react-frontend`
