---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-25
updated: 2026-08-25
related_adr: []
---

# Building Amenity Options + Inline Landlord Creation

## Summary

Two UX bugs Imran hit while walking through the "New Building" flow for the
first time: (1) the Landlord dropdown had no way to add a new landlord
without leaving the form, and (2) Amenities was a free-text box, when it
should be a business-configurable set of checkboxes (Pool, Gym, Parking,
etc.) with an "Other" free-text field for anything not listed.

## Requirements

- Landlord field on the New Building form: pick an existing Landlord, or
  create one inline (name only — mirrors the existing "add a bank inline"
  pattern already used for `Cheque Bank` in the PDC/deposit flows).
- Amenities: a business-configurable list of checkbox options, plus an
  "Other" text field for anything not in the list. The configured list is
  per-business (site-wide), editable from the Settings page — not
  hardcoded, since different businesses want different amenity sets.
- `Building.amenities` stays the single free-text field it already was —
  no schema change, no data migration. The checkbox UI reads/writes that
  same comma-separated string, so existing data (and any other consumer
  of the field) is unaffected.

## Design

- New Single DocType `Real Estate Settings`
  (`real_estate/doctype/real_estate_settings/`) — one field,
  `amenity_options` (Long Text, JSON array of strings). Mirrors the
  existing `Signature Settings` Single pattern (E-Sign module) exactly.
- `real_estate_os/api.py`: `get_amenity_options()` / `set_amenity_options()`
  (whitelisted; the latter trims/dedupes/drops blanks). `get_settings_data()`
  now includes `amenity_options` in its response.
- Patch `patches/v0_0/seed_amenity_options.py` seeds a starter list (Pool,
  Gym, Parking, Security, Elevator, Balcony, Central A/C, Maid's Room) so
  the checkboxes aren't empty on a fresh install — fully editable
  afterward, never re-seeded once set.
- `ui/src/pages/Dashboard.tsx`:
  - `LandlordSelect` — mirrors the existing `BankSelect` inline-create
    component exactly, for `Landlord` (only `landlord_name` is required to
    create one — see `landlord.py`).
  - `AmenitiesField` — self-fetching checkbox list + "Other" input. Parses
    the existing comma-separated string client-side: any token matching a
    configured option becomes a checked box, anything else lands in
    "Other" (so pre-existing free-text data is preserved, not discarded,
    the first time it's edited through this new widget).
  - Used in `NewBuildingDialog`, and special-cased into the generic
    Building edit-mode field grid (`doctype === "Building" && f.fieldname
    === "amenities"`) so editing an existing Building gets the same
    checkbox UI as creating one.
  - New "Building amenities" card on the Settings page: add/remove
    configured options as chips, Save.

## Implementation Plan

- [x] `Real Estate Settings` Single DocType + seed patch
- [x] `get_amenity_options` / `set_amenity_options` API + `get_settings_data` extension
- [x] `LandlordSelect` inline-create component, wired into `NewBuildingDialog`
- [x] `AmenitiesField` checkbox+Other component, wired into `NewBuildingDialog`
      and the generic Building edit-mode field grid
- [x] Settings page "Building amenities" management card

## Acceptance Criteria

- [x] `tsc -b && vite build` clean
- [x] `py_compile` clean
- [x] Deployed and verified live on 2026-08-25: `bench migrate` created
      `Real Estate Settings` and seeded the starter amenity list
      (confirmed via direct DB query). Created a real (then cleaned-up)
      Landlord via the exact insert `LandlordSelect` performs — confirmed
      its Supplier still auto-creates (PR #6 behavior intact). Created a
      real Building with a comma-separated amenities value matching what
      `AmenitiesField` produces — stored exactly as written.
      `get_amenity_options`/`set_amenity_options` round-tripped correctly.
      Not yet click-tested in an actual browser — no browser tool
      available in this environment; flagging for a real click-through.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Related feature: `ui-portal/features/admin-portal-ui.md`
