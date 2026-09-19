---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-27
updated: 2026-09-05
related_adr: []
---

# Own Building Landlord (self-owned placeholder)

## Summary

Imran created a Building through the New Building dialog, typed a new
landlord name into the inline "+ Add landlord" field, but clicked
"Create building" instead of the separate "Add" button next to it — the
typed name was silently discarded (nothing bound it to the form) and the
Building was created with no landlord at all, an easy-to-miss failure mode
since it looked like a normal successful creation.

Rather than just fix the click-through, Imran asked for a formal concept:
a default "Own Building" Landlord that's "as good as no landlord" — no
Head Lease, no rent payment logic — plus made `landlord` mandatory on
Building so this class of accident becomes a loud validation error instead
of a silent bad record.

## Requirements

- `Building.landlord` is mandatory — every Building must point at a real
  Landlord or the self-owned placeholder, never blank.
- A shared "Own Building" Landlord record exists, seeded automatically.
- Everywhere the app used to treat "no landlord" as "owned outright, skip
  the landlord-cost side," it now treats "Own Building" the same way:
  no Head Lease requirement, no Head Lease setup nag, no Supplier
  auto-created for it, no "New Head Lease" action offered.

## Design

- `real_estate/doctype/building/building.json`: `landlord` field now
  `reqd: 1`.
- `real_estate/doctype/landlord/landlord.json` already had a `landlord_type`
  Select with `Individual / Company / Self` — "Self" existed in the schema
  and in `demo_data.py`'s seed data, but nothing in the app's actual gating
  logic ever checked it; it was purely decorative before this. Reused it
  rather than adding a new field: the "Own Building" Landlord is
  `landlord_type: "Self"`.
- `patches/v0_0/seed_own_building_landlord.py` (new, `post_model_sync`):
  idempotently creates one Landlord named "Own Building",
  `landlord_type: "Self"`. All self-owned Buildings link to this one
  shared record — there's no per-building data a placeholder landlord
  would ever need.
- `Unit.validate_head_lease_set_up`: the "must have a Head Lease before
  adding Units" check now also returns early when the Building's Landlord
  has `landlord_type == "Self"`, exactly as it used to return early when
  `landlord` was empty.
- `Landlord.after_insert`: skips `ensure_landlord_supplier` for
  `landlord_type == "Self"` — a Self landlord will never have a Head
  Lease, so it will never need a Purchase Invoice, so there's no reason
  to create a phantom Supplier record standing in for "no landlord."
- `api.get_doc_detail`: for `doctype == "Building"`, adds a computed
  (not stored) `landlord_type` key to the response — a one-hop
  `frappe.db.get_value("Landlord", ...)` lookup — so the portal's Building
  Setup panel can tell "real landlord, needs a Head Lease" apart from
  "Own Building, owned outright" without a second round-trip. Mirrors the
  same one-hop-lookup idiom already used for list columns
  (`landlord.landlord_name` etc., see `portal-list-column-fixes.md`), just
  for the single-document detail view instead of a list.
- `ui/src/pages/Dashboard.tsx`:
  - `NewBuildingDialog`: pre-submit check that `landlord` is set (client-side
    echo of the new server-side `reqd`), "Landlord *" label. `LandlordSelect`
    no longer offers a "—" (none) option, and its placeholder now points
    users at "Own Building" for the self-owned case.
  - Building detail page's Setup panel: `buildingIsSelfOwned` (from the new
    `landlord_type` field) replaces the old "`!buildingLandlord`" check for
    deciding whether to show the Head Lease nag / "New Head Lease" action.

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed + `bench migrate` clean (schema change + new patch ran with
      no errors)
- [x] Verified live: "Own Building" Landlord seeded (`LLD-00321`,
      `landlord_type: Self`, no Supplier created for it); a Unit created
      on a Building whose landlord is "Own Building" succeeds with no
      Head Lease (created and cleaned up as a probe); creating a Building
      with no landlord is correctly rejected
      (`[Building, BLD-00323]: landlord`); `get_doc_detail("Building", ...)`
      returns `landlord_type: "Self"` for it.
- [x] Fixed the actual accidental Building from this incident
      (`BLD-00320` "Rastec 20") — assigned it to "Own Building" (its
      Cost Center was auto-created on that save, as expected).
- [ ] Not click-tested in an actual browser — no browser tool available in
      this environment.

## Follow-up (PR #22)

Building this surfaced that `/landlords` had no working way to create a
Landlord at all — the "Add" button was disabled (`canCreate` didn't list
`"Landlord"`) and no `NewLandlordDialog` existed; landlords could only be
created inline from the Building form's "+ Add landlord" widget (name
only, no type/contact/bank fields). Fixed: added `NewLandlordDialog`
(name, type, email, phone, address, bank name/account/IBAN) and wired
`"Landlord"` into `canCreate`, matching Building/Customer/Lease
Agreement/Cheque Bank's existing pattern. Verified live: created and
cleaned up a probe Landlord end-to-end through the same `createResource`
call the dialog makes.

## Follow-up (2026-09-05): seed patch never ran on tenants provisioned via the console

Imran, on a newly onboarded RealEstate OS account: creating a Building
showed an empty Landlord dropdown, even though the placeholder text says
to pick "Own Building." Confirmed live: `realestate.nnuggets.com`
(migrated directly, many times) had the seeded "Own Building" Landlord;
both `rastec.nnuggets.com` and `test.nnuggets.com` — provisioned through
`platform_console`'s `bench new-site --install-app` flow — did not.

Root cause: `seed_own_building_landlord.py` is a `post_model_sync` patch,
which only actually executes via `bench migrate` on an already-existing
site. `bench new-site --install-app` (how every tenant is created today)
does not run it — a real gap between how the app is developed (iterated on
via `migrate` on long-lived dev/test sites) and how it's actually deployed
to new customers (fresh `new-site`, never migrated before going live).

Fixed by calling the seed directly from
`real_estate_os.provisioning.finalize_new_tenant` — the function that
already exists specifically to handle "things a fresh site needs that
`bench new-site --install-app` alone doesn't provide" (Company, Fiscal
Year, admin User, Aetris branding). Extracted the patch's logic into an
importable `ensure_own_building_landlord()`; the patch file's own
`execute()` now just calls it, so existing sites getting `migrate`d keep
working exactly as before.

Backfilling the two already-broken sites needed one more discovery:
`bench --site <host> migrate` on `rastec.nnuggets.com`/`test.nnuggets.com`
completed with no errors but still didn't create the Landlord — because
`bench new-site --install-app` doesn't just skip running patches on a
fresh site, it marks them as already-applied in that site's `Patch Log`
(a reasonable default: a new site already has the latest schema, so most
patches genuinely have nothing to do). `migrate` therefore correctly saw
this patch as "already run" and skipped it again. Backfilled both sites by
calling `ensure_own_building_landlord()` directly via `bench console`
instead — safe since it's idempotent regardless of Patch Log state.

A third site, `us.nnuggets.com` ("US Real Estate"), was provisioned at
2026-09-05 18:05 UTC — before the fix's 23:17 UTC deployment — so it also
had zero Landlord records at all; not a new bug, just a fourth pre-fix
site. Backfilled the same way. All four Live tenants
(`realestate`, `rastec`, `test`, `us`) now have "Own Building" seeded;
every tenant provisioned from here on gets it automatically via
`finalize_new_tenant`.

**General lesson, not fully addressed here**: any future one-time seed
data added via a `post_model_sync` patch will have this exact same gap
for every tenant provisioned through the console, not just this one — the
patch mechanism assumes a site's history includes a `migrate` step, which
a freshly `new-site`'d tenant's never had. `finalize_new_tenant` is the
right place for anything a *new* tenant needs from day one; patches
remain right for fixing *existing* tenants' data. Worth a broader audit if
this pattern recurs.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Feature: `landlord-doctype.md`, `head-lease-doctype.md`,
  `building-setup-checklist.md`
