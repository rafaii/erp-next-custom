---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-26
related_adr: []
---

# Building Profitability Report

## Summary

Per-building P&L: rent income minus landlord (head-lease) cost minus
maintenance cost, grouped by the building's Cost Center. This is gap G4
from the 2026-08-24 findings doc
(`vault/findings/2026-08-24-ideal-product-vs-current-state.md`) — the
operator's primary KPI ("margin per building"), which the finding said was
unbuildable until Cost Center per Building and Head Lease/landlord payables
existed. Both were confirmed live to already be built (PRs #9-#14), so this
was the next unblocked, highest-ranked gap. Confirmed with Imran via
AskUserQuestion over 3 other open candidates (rent roll/arrears, cash-flow
forecast, RBAC roles) before building.

## Requirements

- Income (rent), landlord cost, maintenance cost, net margin per building.
- Reuses the existing Cost Center per building rather than introducing a
  new grouping key.

## Design

- `real_estate_os/api.py::get_building_profitability` (new whitelisted
  method, same pattern as the existing `get_accounts_data`):
  - Income: `SUM(Sales Invoice Item.base_net_amount)` for submitted
    invoices, grouped by `cost_center`.
  - Landlord cost: same query shape against `Purchase Invoice Item`.
  - Maintenance cost: `Maintenance Request.total_cost` summed directly by
    its own `cost_center` field — **not GL-sourced** like the other two
    legs, because nothing posts a Journal Entry for maintenance cost yet
    (that's a separate, still-open gap, G9 in the findings doc). Real cost
    data, just not accounting-sourced.
  - Occupancy: `Unit.status = 'Occupied'` count per building.
  - Net margin = income − landlord cost − maintenance cost; rows sorted
    weakest-margin-first.
- Frontend: new `BuildingProfitabilityPanel` on `/reports`
  (`ui/src/pages/Dashboard.tsx::ReportsView`), below the existing
  revenue/PDC summary panels — same page, not a new nav item, since
  `/reports` already existed as a mostly-decorative placeholder.

### Unattributed income/cost (found during live verification, not assumed)

Verifying the query against real production data before shipping surfaced
a real data-quality gap: **Cost Center is assigned to a Building lazily**
(`building/utils.py::ensure_building_cost_center`, runs on save — see
`cost-center-per-building.md`), and a Sales/Purchase Invoice Item's
`cost_center` is frozen at submission time and cannot be changed after
(Frappe GL entries are immutable once submitted). So any invoice posted
before its building had a Cost Center is permanently stuck under the
company's default Cost Center (`Main - RRE`) instead of any building.

Confirmed live: 22 of 23 posted Sales Invoices ($119,300) are under
`Main - RRE`; only 1 ($5,000, posted 2026-08-26 after `BLD-00032` got its
cost center on 2026-08-22) is correctly attributed. Without a fix, 4 of 5
buildings would have silently shown "$0 income" — not because they made no
money, but because their invoices predate the feature. Rather than ship
that, `get_building_profitability` also returns `unattributed_income` /
`unattributed_landlord_cost` (total posted minus what's attributed to any
building's Cost Center), and the panel shows this as an explicit amber
banner rather than a silent gap.

This is a real limitation, not a bug in this report: fixing the historical
attribution requires a manual GL correction (e.g. a Journal Entry
reclassifying the old invoice amounts to the correct Cost Center), which is
an accounting operation, not something this report can or should do
automatically.

### Related finding surfaced, not fixed here

Zero landlord payables have posted at all in production: every live
Building uses `landlord_payment_method = PDC`, and per `landlord-payables.md`
/ `pdc-schedule-generation-and-reconciliation.md`, the PDC path only posts a
Purchase Invoice at cheque-clearance time via an Outgoing PDC Entry — but
zero Outgoing PDC Entries exist on the site at all (`unattributed_landlord_cost`
is correctly `0`, not because of an attribution gap this time, but because
the operational step of issuing outgoing cheques hasn't happened yet). Not
fixed as part of this task — it's an operational/data gap in
`landlord-payables.md`, not a defect in the profitability report.

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed and verified live on 2026-08-26: `get_building_profitability()`
      called directly on production returns the expected per-building rows
      (BLD-00032: $5,000 income / 100% margin; the other 4 buildings: $0,
      correctly) plus `unattributed_income: 119300.0`,
      `unattributed_landlord_cost: 0.0` — matches the manually-run SQL used
      to verify the design before writing the endpoint.
- [ ] Not click-tested in an actual browser — no browser tool available in
      this environment.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `cost-center-per-building.md`, `landlord-payables.md`
- Finding: `vault/findings/2026-08-24-ideal-product-vs-current-state.md` (G4)
