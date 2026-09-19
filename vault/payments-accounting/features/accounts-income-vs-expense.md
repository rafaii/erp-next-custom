---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-26
updated: 2026-08-26
related_adr: ["0011-head-lease-landlord-payables"]
---

# Accounts Page: Revenue vs Expense

## Summary

Imran asked whether the Accounts page's PDC summary was tenant-only or netted
against landlord payables (it's tenant-only, by design — see ADR-0011's
`direction` split), then asked for the monthly chart to be able to show
revenue vs expense, with a dropdown to switch views rather than always
showing both.

While checking this, a real data-integrity bug surfaced (not what was
asked, but found investigating the same numbers): 3 leases from the
original demo-data seed were counted in "PDC pending" as if they were real
tenant commitments — two were never sent for signature, and one had been
silently superseded by a real, later, properly signed lease for the same
tenant but was never cancelled. See "Stale lease cleanup" below.

## Requirements

- Accounts page's trend chart gets a dropdown: "Revenue" (existing
  single-series view, unchanged default) or "Revenue vs Expense" (Sales
  Invoice vs Purchase Invoice per month, grouped bars).
- `collected` (Accounts page KPI) must only sum tenant-side (Receive-type)
  Payment Entries — it was summing every Payment Entry regardless of
  type, which would have silently pulled in landlord Pay-type entries
  the moment those exist.

## Design

- `real_estate_os/api.py::get_accounts_data`:
  - `collected` now filters `payment_type: "Receive"` explicitly.
  - New `monthly_expense`: same 6-month window as `monthly_revenue`,
    summed from `Purchase Invoice` instead of `Sales Invoice`.
- `ui/src/pages/Dashboard.tsx`: new `AccountsTrendChart` — a `<Select>`
  switcher wrapping the existing single-series `RevenueRechart` (mode
  "Revenue") plus a new grouped two-series bar chart (mode "Revenue vs
  Expense"). Reuses `RevenueRechart` rather than duplicating the
  single-series render path.

## Stale lease cleanup (found while verifying the PDC numbers, not a code change)

Cross-checked "PDC pending: 19 / 104,500" against the actual `PDC Entry`
rows grouped by lease and found 3 of the 4 leases holding those cheques
were never real: `LSE-2026-00018`/`LSE-2026-00019` (Draft, never sent) and
`LSE-2026-00021` ("Pending Signature", but already had 4 cleared cheques
going back months — actually superseded by `LSE-2026-00121`, a real,
later, properly Signed/Active lease for the same tenant that was never
cancelled). All 3 were original demo-seed fixtures (created in the same
second as the initial `demo_data.seed` run). Cancelled all 3 via the
existing `cancel_lease` action (Imran's choice, of 4 options offered) —
not a raw delete: their Pending cheques flip to Cancelled, already-Cleared
history stays intact, and their Units release back to Vacant (verified no
overlap with the real lease's unit before running it).

## Implementation Plan

- [x] `get_accounts_data`: `collected` scoped to Receive; `monthly_expense` added
- [x] `AccountsTrendChart` with Revenue / Revenue vs Expense switcher
- [x] Stale lease cleanup (LSE-2026-00018/19/21 cancelled)

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed and verified live on 2026-08-26: `get_accounts_data`
      returns `monthly_expense` with the correct 6-month shape (all zero
      — no Purchase Invoice exists yet, expected); `collected.total` is 0
      (no Receive-type Payment Entry exists yet). Not click-tested in an
      actual browser for the dropdown UI itself — no browser tool
      available in this environment.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `landlord-payables.md`
