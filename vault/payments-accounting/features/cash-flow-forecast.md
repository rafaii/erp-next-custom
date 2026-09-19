---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-24
updated: 2026-08-31
related_adr: []
---

# Cash Flow Forecast

## Summary

Forward-looking cash position: known tenant rent due (from `Rent Schedule`
+ `PDC Entry`) minus known landlord rent due (from `Head Lease Schedule`,
once it exists) over the next N months. This is what lets the operator see
a break-even occupancy threshold and catch a cash crunch before it happens
— gap G5 in the findings doc, and the piece of "ideal product" pillar P7
(Intelligence) that has no code today at all.

Depends on `head-lease-doctype.md` / `landlord-payables.md` for the outflow
side — until those exist this report can only show inflows, which is
already better than nothing but should be labeled partial in the UI.

## Requirements

- Monthly buckets, configurable horizon (default 6 months, matching the
  existing 6-month revenue chart in `get_accounts_data`).
- Inflow: Pending/Deposited `PDC Entry` amounts + unbilled `Rent Schedule`
  rows, by expected date.
- Outflow: Pending `Head Lease Schedule` rows, by due date (once built —
  ship inflow-only first if Wave 2 lands after this).
- Net position per month, running balance.

## Design

- New endpoint `real_estate_os/api.py::get_cash_flow_forecast`, following
  the existing `get_accounts_data` pattern (grouped aggregation, no new
  DocType needed).
- Portal "Reports" nav item (already wired to `get_accounts_data` in
  `hooks.py::portal_nav_items` — this becomes a second reports view or a
  tab within it).

## Implementation Plan

- [x] `get_cash_flow_projection` (v1, narrower than the design above) —
      This Month / Next Month only, both inflow (Rent Schedule) and
      outflow (Head Lease Schedule, which shipped since this file was
      written), summed directly by `due_date` window, `status !=
      Cancelled`. Prompted by Imran hitting exactly the gap this file
      predicted: a brand-new Building/lease showed almost nothing on the
      trailing Revenue vs Expense chart, and asked for a projection.
- [x] Portal UI: `CashFlowProjectionPanel` — two cards (This Month, Next
      Month), each showing income, landlord payments, net. Originally
      shipped on the old plain "Accounts" ops dashboard; that page was
      removed 2026-08-30 (no standalone value per Imran) and its route
      taken over by the renamed former-Reports ledger page. This panel
      moved to the redesigned `/overview` page instead — it's forward-
      looking cash detail, which fits the owner-facing dashboard, not the
      accountant's ledger. Nearly deleted as apparently-orphaned dead code
      during that rename; caught by checking this file before deleting.
- [x] v2 (2026-08-31): `get_cash_flow_projection` now takes a `months`
      parameter (default 6, clamped 1-24), returning a `months` array
      instead of the fixed `this_month`/`next_month` pair, plus a running
      balance and a `crunch_month` flag (the first month the running
      balance goes negative). Running balance starts from today's actual
      cash on hand (same inline Bank Account query as
      `get_owner_overview`, not `accounts_portal`'s hard-throwing
      `get_bank_accounts()`) rather than 0, so a flagged shortfall reads
      as a real projected bank-balance date, not just a directional net
      number — this is the "break-even indicator" the original
      Implementation Plan line called for, scoped as "first month cash
      goes negative" rather than a per-unit occupancy-threshold model
      (a much larger, separate undertaking that wasn't asked for).
      `CashFlowProjectionPanel` (Overview's compact 2-card view) now
      reads `months[0]`/`months[1]` from the same richer shape instead of
      the old fixed keys — no visual change there, still the quick-glance
      version. New `CashFlowForecastPanel` — a "Cash Flow" tab on
      Accounts (not a new sidebar item, per the same "lesser the better"
      pattern already applied to Contracts/Rent Roll) — shows the full
      horizon with a 3/6/12-month selector, a crunch-month callout, and
      the per-month table with running balance.

## Acceptance Criteria

- [x] Verified live against the exact reported scenario (Building
      BLD-00320: QAR 3000/mo Head Lease, 1 of 3 units leased at QAR
      2000/mo): projection returns `{this_month: {income: 2000,
      payables: 3000, net: -1000}, next_month: {same}}` — ties out to
      the real Rent Schedule/Head Lease Schedule rows.
- [x] Verified live via `bench console`: `months` param correctly clamped
      (999 → 24 rows, 0/-5 → 1 row), running balance accumulates
      correctly across a real 6-month window starting from the actual
      live cash-on-hand figure, `crunch_month` correctly `None` when
      every month's net keeps the running balance positive. Advisor
      review before deploy flagged that a QAR 0 starting cash (a
      recurring artifact on this site — no bank feed, historical PDC
      clearances posted to Cash) would make every month read as an
      immediate "cash crunch," which is arithmetically correct but reads
      as a false alarm; added an explicit caveat banner for that specific
      case (`starting_cash === 0`) rather than suppressing the flag,
      since it's honest information once a real balance exists.

## 2026-08-27: why the Revenue/Expense chart looked empty (not a bug)

Imran asked why a freshly-created Building/lease showed nothing on the
Accounts page's Income vs Expense chart. Investigated live rather than
assumed: the numbers were correct, not broken.

- `monthly_revenue`/`monthly_expense` (`get_accounts_data`) are trailing
  **actuals** — a Sales Invoice only exists once a rent period is due and
  actually invoiced; a Purchase Invoice/expense only exists once a
  landlord cheque is manually confirmed Cleared (no bank feed exists to
  automate that — ADR-0010). A portfolio created today has almost none of
  that yet, even though every lease/head lease is already committed to
  real, known future amounts.
- Confirmed the scheduler itself is healthy, not broken: `Scheduled Job
  Type.last_execution` showed `mark_pdc_deposited`,
  `generate_due_payables`, `generate_due_invoices`, `send_reminders`, and
  `transition_lease_statuses` all ran on time at 00:00 today. The specific
  Outgoing PDC for this Head Lease was created *after* that day's run —
  it simply hasn't had its next nightly pass yet (tomorrow it flips
  Pending → Deposited), and posting an expense from it additionally
  requires a human's "Mark Cleared" confirmation on the Head Lease page
  regardless of scheduler timing.

This is exactly the gap this feature file predicted back on 2026-08-24
("the operator cannot see the only number the business runs on" until
enough history exists) — the fix wasn't a bug fix, it was building the
forward-looking projection this file already called for. See the
Implementation Plan above.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `custom-module/features/head-lease-doctype.md`,
  `payments-accounting/features/landlord-payables.md`,
  `ui-portal/features/admin-portal-ui.md`,
  `pdc-schedule-generation-and-reconciliation.md`
