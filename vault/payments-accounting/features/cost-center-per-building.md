---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-27
related_adr: []
---

# Cost Center Per Building

## Summary

Auto-create a Cost Center on Building creation and route rent income, PDC
deposits, and maintenance expenses to it. This is the prerequisite for
margin-per-building being computable at all — gap G3 in
`vault/findings/2026-08-24-ideal-product-vs-current-state.md`. Uncommenting
the `Building.after_insert` hook alone is not sufficient: cost centers with
nothing posted to them produce zero visibility, same as today. The hook
*and* the invoice-line `cost_center` wiring must ship together.

## Requirements

- On Building save: create Cost Center (name = building) if absent.
- `real_estate_os/payments/invoicing.py::create_invoice_for_lease` sets
  `cost_center` on the Sales Invoice item line (it currently posts to
  `default_income_account` with no cost center at all — verified by
  reading the function body).
- Existing Buildings (created before this hook existed) get a backfill
  patch, not just new ones going forward.
- Security deposits post to a refundable liability account (tracked
  separately — see gap G10; not blocking this feature).

## Design

- `real_estate_os/building/utils.py::after_building_insert` (already
  referenced, commented out, in `hooks.py::doc_events`) → create Cost
  Center (frappe Cost Center DocType), named after the Building.
- `invoicing.py::create_invoice_for_lease`: resolve the lease's Unit →
  Building → Cost Center, set on the Sales Invoice Item row.
- Patch under `real_estate_os/patches/` to backfill Cost Centers + relink
  for Buildings that predate the hook.

## Implementation Plan

- [x] Uncomment + implement `Building.after_insert` → Cost Center hook
- [x] Backfill patch for existing Buildings
- [x] Propagate cost center to rent invoice line items (`invoicing.py`)
- [ ] Propagate to maintenance expense claims (depends on
      `maintenance-cost-allocation.md`, Phase 4 — not blocking this feature)

Code complete on `agent/developer-1/cost-center-per-building`
([PR #2](https://github.com/rafaii/real-estate/pull/2)). No live bench site
in the dev environment this was written in — `py_compile` clean, but
`bench migrate` + a real Building/lease/invoice cycle needs to run on the
actual dev site before this flips to `done`.

Review round caught two bugs before either branch was reviewed by Imran:
`_get_parent_cost_center` would have thrown on the first Building
(`Company.cost_center` is a leaf, not a group — Cost Center insert
requires a group parent); the backfill's `cost_center == ""` filter would
have silently matched zero pre-existing Buildings (a fresh Link column
defaults to `NULL`, not `""`). Both fixed. Also hardened: the hook and
patch now `frappe.log_error` + continue on a Cost-Center-creation failure
instead of blocking Building creation / aborting `bench migrate`.

## Acceptance Criteria

- [x] Building creation produces a linked Cost Center — verified live
      repeatedly this session (e.g. BLD-00320)
- [x] Rent invoice line items carry the correct Cost Center — verified via
      direct GL Entry query 2026-08-27: Sales Invoice's income line shows
      `cost_center: 'BLD-00320 - Rastec 20 - RRE'`
- [x] Pre-existing Buildings backfilled with a Cost Center after migration
      (patch shipped and run on production)

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `building-doctype.md`, `building-profitability-report.md`
