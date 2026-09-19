---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-26
updated: 2026-08-26
related_adr: []
---

# Portal List Fixes: Readable Columns on Properties, Maintenance, Contracts

## Summary

Follow-on to `contracts-page-fixes.md` (#16) — Imran clarified the original
raw-ID complaint was actually about `/maintenance` and `/properties`, not
`/contracts` (which had already been fixed). Also asked for the Contracts
Unit column to go one step further: `<Building Name> - <Unit Name>`
instead of just the unit's own name, since unit numbers repeat across
buildings and aren't distinguishable alone.

## Requirements

- `/properties` (Building list): `landlord` column shows the landlord's
  name, not the raw Landlord ID.
- `/maintenance` (Maintenance Request list): `unit` column shows the
  unit's number, not the raw Unit ID.
- `/contracts` (Lease Agreement list): Unit column shows
  `<Building Name> - <Unit Name>`, not just the unit name/number.

## Design

- `hooks.py::portal_nav_items`: `landlord` → `landlord.landlord_name`
  (Properties), `unit` → `unit.unit_number` (Maintenance) — same one-hop
  dotted-fetch pattern as #16.
- Contracts needs a name two hops away (Lease Agreement → Unit →
  Building), which Frappe's list API does not support in one dotted
  field — verified live via `bench console`: a fields list containing
  `"unit.building.building_name"` doesn't error, it just silently drops
  that field from the result. Confirmed the one-hop equivalents both
  work: `"unit.building"` (raw Building ID) and `"building.building_name"`
  fetched directly on `Unit`.
  - Contracts' columns now also fetch `unit.building` (raw ID) purely as
    data.
  - `ui/src/pages/Dashboard.tsx::ResourceListView`: for Lease Agreement,
    separately fetches a `Building` name map (`listResource("Building",
    ["building_name"])`), drops the raw `building` column from what's
    *displayed* (it's fetch-only), and combines it into the `unit_number`
    cell client-side as `"<Building> - <Unit>"`.

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean on `hooks.py`
- [x] Deployed and verified live on 2026-08-26 via `bench console`:
      Properties/Maintenance dotted fetches resolve real names (e.g.
      `landlord_name: "Ooredoo Properties"`, `unit_number: "030"`);
      Contracts combine produces e.g. `Rastec 25 - 001` for
      `LSE-2026-00020`.
- [ ] Not click-tested in an actual browser — no browser tool available
      in this environment.

## Deployment incident (see AGENTS.md for the full writeup)

Deploying this required rebuilding the custom app image directly (not via
`docker compose --build`, which is a no-op for a fixed-tag image) and
recreating containers with the full `-f` override list. An earlier
attempt with an incomplete `-f` list plus `--remove-orphans` deleted the
running `db`/`redis-cache`/`redis-queue` containers as "orphans" (they're
only defined in override files that weren't included). No data was lost
— they're named volumes — but it was recoverable by luck, not by design.
Recreated them against the same volume names using the correct full
compose invocation; verified `Building` count unchanged (5) before and
after. AGENTS.md now documents the exact required `-f` file list.

## Follow-up (PR #18)

Imran asked for the same `<Building> - <Unit>` combine (not just the
name-fix) on `/maintenance` too. Generalized the Contracts-only
`doctype === "Lease Agreement"` checks in `ResourceListView` into a
shared `combinesBuildingIntoUnit` flag covering both `Lease Agreement`
and `Maintenance Request`, rather than duplicating the combine logic.
`hooks.py`'s Maintenance columns also fetch `unit.building` now.
Deployed clean this time using the corrected full deploy procedure
(rebuild image separately, full `-f` override list, `-p realestate`) —
no incident.

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `contracts-page-fixes.md`
