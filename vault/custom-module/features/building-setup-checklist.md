---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-25
updated: 2026-08-25
related_adr: ["0011-head-lease-landlord-payables"]
---

# Building Setup Checklist (Head Lease + Units gating)

## Summary

Two related bugs Imran hit testing the New Building flow: (1) entering
"Total Units: 10" doesn't auto-create 10 Units (expected it to), and (2)
choosing a Landlord doesn't set up a Head Lease or ask for rent — a
Building with a Landlord is leased, not owned, and had no path in the UI
to configure what it owes that landlord.

## Requirements

- Choosing a Landlord on a Building means it's leased (a Head Lease is
  needed); no Landlord means it's owned outright (no Head Lease). This
  distinction already existed conceptually (ADR-0011) but nothing in the
  UI acted on it.
- `Total Units` is a target count, not an auto-generation trigger — real
  units differ in type/rent, so blindly generating N identical units from
  just a count would create wrong data. Instead: make the next step
  (Generate Units, with the count pre-filled) obvious and hard to miss.
- A leased Building's Units can't be created until its Head Lease exists —
  enforced server-side (`Unit.validate_head_lease_set_up`), not just a UI
  suggestion, so this holds regardless of entry point (portal, API, bench
  console). **Scoped to creation only, not edits to already-existing
  Units** — 3 live Buildings already had real Units before this rule (and
  before Head Lease itself) existed; blocking edits to those would lock
  staff out of pre-existing data over a gap that predates this feature.
- The required next steps must be clearly visible to the admin, not
  hidden — a "Setup" panel on the Building detail page, not a silent
  block.

## Design

- `real_estate_os/real_estate/doctype/unit/unit.py`:
  `validate_head_lease_set_up` — if `self.is_new()` and the Building has a
  `landlord` but no `Head Lease` row exists for it yet, `frappe.throw`.
  Runs through every entry point (`createResource`, `generate_units`,
  bench console) since it's in the DocType controller, not the API layer.
- `real_estate_os/demo_data.py`: new `_create_head_leases(buildings)`,
  called after `_create_buildings` and before `_create_units` — every demo
  Building has a Landlord, so without this the demo seed itself would now
  fail the same guard. Rents picked as ~60-70% of each building's implied
  rent roll (the arbitrage margin the rest of the demo data implies).
- `ui/src/pages/Dashboard.tsx`:
  - New `NewHeadLeaseDialog` — minimal (monthly rent + start/end date,
    defaults today/+1yr) — just what `Head Lease.after_insert` needs to
    build its own payment schedule automatically. Everything else
    (payment method, escalation, PDC generation) stays a Head-Lease-detail-
    page follow-up, same "quick create, refine later" pattern as
    `NewUnitDialog`/`GenerateUnitsDialog`.
  - New "Setup" panel on the Building detail page: Head Lease row (only
    if Landlord is set) with a "Set up Head Lease" action; Units row
    (only if Total Units is set) showing `{count} of {total}` with a
    "Generate" action, pre-filled with the target count via
    `GenerateUnitsDialog`'s new `initialCount` prop.
  - `requireHeadLeaseSetUp()` — client-side echo of the same server rule,
    guarding the "Add unit"/"Generate units" buttons with an immediate
    toast instead of a round-trip failure. The server check is what
    actually enforces it; this is just faster feedback.
  - Building creation now navigates to the new Building's detail page
    (previously just refreshed the list) so the Setup panel is
    immediately visible — matches the existing precedent of
    `NewLeaseDialog` navigating to the new lease from the Customer page.

## Implementation Plan

- [x] `Unit.validate_head_lease_set_up`, scoped to `is_new()`
- [x] `demo_data.py::_create_head_leases`, wired into the seed order
- [x] `NewHeadLeaseDialog`
- [x] Building detail page "Setup" panel (Head Lease + Units rows)
- [x] `requireHeadLeaseSetUp()` guard on both sets of Add/Generate unit buttons
- [x] Building creation navigates to the new Building's detail page

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed and verified live on 2026-08-25 with a real (then
      cleaned-up) test: creating a Unit on a landlorded Building with no
      Head Lease raised the exact intended message; creating the Head
      Lease then let Unit creation succeed; a real existing Unit on
      BLD-00004 (one of the 3 affected live Buildings) was confirmed
      still editable end-to-end; a real owned Building (BLD-00032, no
      landlord) confirmed identifiable/unaffected.
- [ ] **Known live gap, not fixed here**: BLD-00004, BLD-00005, BLD-00006
      are real, already-live Buildings with real Units and a Landlord but
      still no Head Lease — this PR doesn't create one for them (their
      real rent terms aren't something to guess at), so their landlord
      cost side remains untracked until Imran sets one up for each
      through the new "Setup" panel.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0011-head-lease-landlord-payables.md`
- Feature: `payments-accounting/features/landlord-payables.md`,
  `head-lease-doctype.md`
