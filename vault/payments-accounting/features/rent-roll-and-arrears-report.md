---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-24
updated: 2026-08-31
related_adr: []
---

# Rent Roll & Arrears Report

## Summary

The operator-facing report that answers "who owes what, and how late are
they": a rent roll (every active lease, its monthly rent, last payment,
current balance) plus an arrears aging bucket (0-30 / 31-60 / 61-90 / 90+
days). Currently zero report definitions exist anywhere in the app
(`find -path "*report*"` returns nothing) — gap G7 in the findings doc.

## Requirements

- One row per lease with `lease_status in (Active, Expiring Soon)`
  (depends on `lease-lifecycle-state.md`).
- Columns: building, unit, tenant, monthly rent, last payment date, current
  outstanding balance, days overdue, aging bucket.
- Collection rate summary (collected / billed) per building and portfolio-
  wide — the KPI called out in the findings doc §1 as the operator's actual
  north star, distinct from the property-manager framing's "collection
  rate for the owner."
- Filterable by building; drill-down to the lease's Rent Schedule / PDC
  Entries.

## Design

**Revised 2026-08-31 (Imran)**: originally spec'd as a Frappe Script
Report — Desk-only, unreachable by the business owner. Every accounting
report built since this file was written (GL, Trial Balance, P&L,
Balance Sheet, Bank Reconciliation) shipped instead as a portal-native
endpoint + tab on the Accounts page, so the business owner never needs
Desk access. Ships the same way here: a new tab on `AccountsView`
alongside those, not a Script Report.

- `real_estate_os.api.get_rent_roll()` — new endpoint (real-estate-
  specific, so it lives in `real_estate_os`, not the generic
  `accounts_portal`, matching how `get_building_profitability` and
  `get_owner_overview` are already split from the generic GL/JE/BS
  endpoints).
- Source: active `Lease Agreement` rows (`lease_status in (Active,
  Expiring Soon)`) joined to their `Customer`'s `Sales Invoice`s.
  Sales Invoice carries no direct Lease/Unit/Building link (same
  resolution gap noted throughout `api.py`), so balance/aging/last-
  payment are attributed via the lease's own Customer — a tenant with
  more than one active lease is rare enough not to special-case, the
  same simplification `get_owner_overview`'s overdue list already uses.
- Aging buckets computed from the customer's oldest still-outstanding
  invoice's `due_date` vs. the server's `today()` (Current / 0-30 /
  31-60 / 61-90 / 90+).
- Collection rate (billed vs. collected, i.e. `billed - outstanding`)
  rolled up per building and portfolio-wide, grouped by each customer's
  *current* lease building — not GL Cost Center — to avoid
  `get_building_profitability`'s own documented "posted before the
  building had a Cost Center" attribution gap for this specific view.
- Building filter: client-side, over the already-fetched rows — same
  pattern every other portal list page already uses, no server param
  needed.
- Drill-down: each row navigates to its Lease Agreement detail page
  (which already shows Rent Schedule / PDC Entries) — reuses existing
  navigation, no new drill-down UI.

## Implementation Plan

- [x] `get_rent_roll()`: rent roll base query + aging buckets
- [x] Collection-rate summary (per building + portfolio)
- [x] `RentRollPanel` — new tab on `AccountsView`, building filter +
      drill-down via existing lease detail navigation

## Acceptance Criteria

- [x] Report lists every active lease with a per-row balance — labeled
      "Tenant balance," not "outstanding balance for this lease," since
      Sales Invoice has no lease link and the figure is genuinely the
      customer's total across all their invoices (advisor review flag).
- [x] Aging buckets verified live against real Sales Invoice due dates
      through an actual day-boundary crossing during testing (not a
      static spot-check): a due-today invoice correctly read "Current,"
      not overdue, and flipped to reflect real state as the server's
      `today()` advanced.
- [x] Collection rate does **not** tie out to `get_accounts_data`'s
      existing totals — deliberately, not a bug. `get_accounts_data`'s
      "collected" is all-time, all-invoices Payment-Entry cash received;
      this report's rate is `billed − outstanding` scoped to invoices
      that have actually come due, which is the standard rent-roll
      definition ("of what's owed so far, how much is in?"). An earlier
      draft used all-time billed with no due-date scoping, which caused
      a tenant who was fully paid-to-date to show a 50% collection rate
      purely because their next period's invoice had already posted —
      caught by advisor review before deploy, not by the original
      acceptance criterion, which was wrong on this point and is
      corrected here instead of chased.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `custom-module/features/lease-lifecycle-state.md`,
  `pdc-schedule-generation-and-reconciliation.md`
