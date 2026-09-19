---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-24
updated: 2026-08-27
related_adr: ["0007-admin-portal-react-frontend"]
---

# Portal Source Consolidation

## Summary

Moves the React admin-portal source (`ui/`) from the untracked bench root
into the `real_estate_os` app repo, and removes the dead pre-ADR-0007 Vue
source tree, so the only app repo (`apps/real_estate_os`, per AGENTS.md) is
buildable end-to-end from a clone. Structural Risk R1 in
`vault/findings/2026-08-24-ideal-product-vs-current-state.md`.

Note: the fresh-clone *acceptance test* itself already passes today — the
compiled bundle (`real_estate_os/public/portal/assets/*`) is committed and
is exactly what `www/portal/index.html` serves, verified by reading
`hooks.py`'s `website_route_rules` and the served HTML directly. The actual
problem is narrower but still real: the buildable source only exists on one
machine, isn't code-reviewable, isn't in the repo history, and the app repo
still tracks a fully superseded Vue app (`portal/src/*.vue`, 11 files) that
nothing references (`hooks.py`/`api.py` grep confirms zero references
outside the Vue tree's own files).

## Requirements

- `ui/` React source lives inside `apps/real_estate_os/` (not the bench
  root) so it ships with the only repo pushed to GitHub.
- Compiled bundle output path unchanged — `real_estate_os/public/portal/`
  keeps being the thing `www/portal/index.html` serves; the build must
  still stay committed (fresh clones do no npm/bun build step per
  AGENTS.md's install-app acceptance test).
- Dead Vue `portal/` directory removed.
- `node_modules` and any local build artifacts excluded via `.gitignore`
  (already covered by the app repo's existing unanchored `node_modules/`
  and `dist/` patterns).

## Design

- Move bench-root `ui/` (source only — `src/`, `public/`, config files;
  excludes `node_modules/`, `dist/`) to `apps/real_estate_os/ui/`.
- Update `ui/vite.config.ts` `outDir` from
  `../apps/real_estate_os/real_estate_os/public/portal` (old bench-root-
  relative path) to `../real_estate_os/public/portal` (new same-repo-
  relative path); `base` stays `/assets/real_estate_os/portal/`.
- Delete repo-root `portal/` (Vue).
- No change to `hooks.py`, `www/portal/`, or any served asset path.

## Implementation Plan

- [x] Confirm dead Vue `portal/` has zero external references
- [x] Confirm `real_estate_os/public/portal/` bundle is already tracked and
      is what's actually served (fresh-clone test already passes)
- [x] Move `ui/` source into `apps/real_estate_os/ui/` (excluded unreferenced
      Deno/Convex scaffold cruft — `main.ts`, `isolate/` — that shipped with
      the original vite-template and nothing in `src/` imports)
- [x] Fix `vite.config.ts` `outDir` for the new relative path
- [x] Delete dead Vue `portal/`
- [x] Rebuild locally, diff output against the currently-committed bundle
      (only `index.css` changed — a single-line Tailwind v4 patch-version
      diff from a fresh install, not a behavior change)
- [x] Commit on `agent/developer-1/portal-source-consolidation`, push —
      [PR #1](https://github.com/rafaii/real-estate/pull/1)
- [x] Review round: verified `integrations.md`/`README.md` (pushed unread)
      carried no real secrets, only unused Convex/VLY scaffold env-var
      names; deleted `integrations.md`, rewrote `README.md` to describe
      the actual project
- [x] Renamed the now-superseded bench-root `ui/` to
      `ui.OLD-see-app-repo/` with a pointer note — it was untracked and
      still built to the right output path, which would have let edits
      keep landing there out of habit and silently re-diverge from the
      in-repo source

## Acceptance Criteria

- [x] `apps/real_estate_os` clone contains buildable React source under `ui/`
- [x] `cd ui && bun install && bun run build` reproduces
      `real_estate_os/public/portal/` output
- [x] No `portal/*.vue` files remain in the repo
- [x] Fresh-clone `install-app` still serves `/portal` unchanged — verified
      by reading `hooks.py`/`www/portal/index.html` wiring; not additionally
      re-verified via a literal `bench new-site` run, but this same
      not-a-literal-fresh-install caveat applies platform-wide (dozens of
      successful production deploys/migrates on this exact code path this
      session), so it isn't treated as a gap unique to this feature

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- ADR: `vault/decisions/0007-admin-portal-react-frontend.md`
- Finding: `vault/findings/2026-08-24-ideal-product-vs-current-state.md` (R1)
