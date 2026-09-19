---
status: in-progress
owner: developer-1
domain: maintenance
created: 2026-08-16
updated: 2026-09-01
related_adr: []
---

# Maintenance Cost Allocation

## Summary

On resolution, post spares + labor cost to the building's Cost Center via an
Expense Claim (or Stock Entry for inventory spares).

**2026-09-01 scope check**: this plan predates `get_building_profitability()`
and `get_owner_overview()`, both shipped later in the same effort as
Cost Center per Building / Head Lease payables and both already reading
`Maintenance Request.total_cost` by its own `cost_center` field directly —
"real cost data, just not GL-sourced like the other two legs" (their own
code comment). Per Imran's answer to a scope question, GL posting via
Expense Claim/Stock Entry was **deliberately deferred**, not built: it would
need Warehouse/Item stock setup and Expense Claim Type/Employee records this
app doesn't otherwise use, for a number the two live P&L-style reports
already consume in its simpler form. The gap actually fixed instead: `total_cost`
only ever summed `spares_used` — `labor_hours` existed as a field but was
never priced into anything, silently understating every maintenance-cost
figure derived from it.

## Requirements

- ~~Spares (Inventory) → Expense Claim / Stock Entry → GL Entry.~~ Deferred —
  see scope check above.
- Cost posts to the building's Cost Center — already true since ADR-0018's
  `_resolve_cost_center_from_unit` (Unit → Building → Cost Center), not new.
- Security deposit refunds remain a liability (not expensed) — untouched by
  this change, no Security Deposit code was modified.
- **New**: `total_cost` reflects labor, not just spares.

## Design

- ~~`maintenance_request.py::on_resolve` → create Expense Claim / Stock Entry~~
  (superseded — see scope check above).
- **New**: `Real Estate Settings.default_labor_rate` (Currency, Desk-only,
  System Manager) — site-wide hourly rate.
- **New**: `Maintenance Request.labor_rate` (Currency, permlevel 1) defaults
  from the settings value the first time `labor_hours` is set on a request,
  but can be overridden per request (e.g. a contractor billed at a different
  rate) — same per-row-override shape as `Maintenance Spare.rate`.
- **New**: `Maintenance Request.labor_cost` (Currency, read-only, permlevel 1)
  = `labor_hours * labor_rate`; `total_cost` = spares total + `labor_cost`.
  Computed in `validate()`, same place spares were already summed.

## Implementation Plan

- [x] ~~Implement `on_resolve` auto-posting~~ — deferred, not built (see
      scope check).
- [x] ~~Route spares via Stock Entry (inventory) or Expense Claim~~ — deferred,
      not built (see scope check).
- [x] Confirm cost center propagation — already worked (ADR-0018), verified
      unaffected by this change.
- [x] Price labor into `total_cost` — 2026-09-01. `default_labor_rate` added
      to `Real Estate Settings`; `labor_rate`/`labor_cost` added to
      `Maintenance Request`; `validate()` now computes
      `total_cost = spares_total + labor_cost`. `bench migrate` run;
      verified live via `bench console` (in-memory `validate()` calls, no
      real records existed yet on this site to touch): default-rate fallback
      and per-request override both compute correctly.
- [x] **Found and fixed a pre-existing data leak while adding
      `labor_rate`**: `get_doc_detail()` (the generic detail-page endpoint
      every portal detail page uses) never filtered fields by DocField
      `permlevel` — it only checked document-level read permission once,
      then returned every field's value regardless of permlevel. Maintenance
      Request's internal fields (`assigned_to`, `work_order`, `spares_used`,
      `labor_hours`, `total_cost`, `cost_center` — all permlevel 1, Tenant
      has no permlevel-1 grant) were already exposed to a Tenant viewing
      their own request's detail page; adding `labor_rate`/`labor_cost` at
      the same permlevel would have handed a Tenant the actual hourly cost
      basis, not just an aggregate. Fixed generically in `get_doc_detail`
      via `meta.get_permlevel_access("read", user=...)`, gating both the
      scalar-fields loop and the child-tables loop. Verified this has zero
      effect on every other doctype `get_doc_detail` serves (`Building`,
      `Customer`, `Head Lease`, `Landlord`, `Lease Agreement`, `PDC Entry`,
      `Security Deposit`, `Unit` — grepped `hooks.py` for every doctype
      reachable through it, confirmed Maintenance Request is the only one
      with any permlevel>0 field), and verified live end-to-end with a
      scratch Tenant-role test user + scratch request (both deleted after):
      Tenant's `get_doc_detail` response drops to
      `[customer, unit, issue_type, priority, status, description]`;
      Administrator/System Manager/Leasing Agent/Maintenance Staff are
      unaffected (all already hold explicit permlevel-1 read grants — see
      `patches/v0_0/create_staff_roles_and_permissions.py`). Checked the
      sibling function `get_linked_records` for the same class of gap: it
      also fetches `columns` with no permlevel awareness, but no configured
      `portal_detail_links` entry today names a permlevel>0 column (the only
      such doctype, Maintenance Request, has `links: []` there) — left a
      code comment rather than speculative filtering code, since there's
      nothing live to fix.

## Acceptance Criteria

- [ ] ~~Resolved request posts cost to correct building Cost Center~~ —
      superseded; no GL posting is built. See instead:
- [x] `Maintenance Request.total_cost` includes labor cost, not just spares,
      and is attributed to the correct building via `cost_center` (already
      true pre-existing) — verified live.
- [x] A Tenant viewing their own Maintenance Request's detail page cannot see
      `labor_rate`, `labor_cost`, `total_cost`, `cost_center`,
      `spares_used`, `assigned_to`, or `work_order` — verified live with a
      scratch test user.
- **Caveat, not yet true in practice**: `Real Estate Settings.default_labor_rate`
  is unset on the live site as of this writing. Until a System Manager sets
  it in Desk, `labor_rate` stays unset for any request logged without an
  explicit per-request override, so `total_cost` remains spares-only —
  same number as before this change, not a regression, just not yet the
  improvement in effect.

## Related

- Domain index: `vault/maintenance/maintenance.md`
- Feature: `maintenance-request-doctype.md`; `payments-accounting/features/cost-center-per-building.md`
- `role-based-access.md` (permlevel grants this depends on), `building-profitability-report.md` (the report this feeds)
