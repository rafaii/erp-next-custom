---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-09-05
updated: 2026-09-06
related_adr: ["0025-landlord-cheque-plan-decoupling"]
---

<!-- Deployed to all 4 real_estate_os tenant sites (realestate/rastec/test/us.nnuggets.com) 2026-09-06, bench migrate + clear-cache clean on all four. Core decoupling + advance sweep verified live end-to-end on realestate.nnuggets.com against fully isolated disposable test data (own throwaway Landlord/Supplier, not the real Ooredoo Properties tied to Rastec 20/21) — confirmed real Rastec 20/21 Head Leases, Purchase Invoice ACC-PINV-2026-00003, and Payment Entry ACC-PAY-2026-00001 unchanged throughout. On-account balance UI shipped and smoke-tested live same day.

2026-09-06, same day, live bug found by Imran clicking Mark Cleared on Rastec 21's real (pre-ADR-0025, legacy 1:1) cheques: `_settle_outgoing_pdc` had `paid_from`/`paid_to` backwards for the zero-outstanding-invoices case (ERPNext's `get_payment_entry` sets `paid_from=bank`, `paid_to=payable` for payment_type "Pay" — reversed from what's intuitive), and had no path left for a legacy 1:1 PDC (linked via `head_lease_schedule.pdc_entry`, on a Head Lease with empty `cheque_plan`) to ever get its invoice created at all — `_post_payable_for_outgoing_pdc` was removed with nothing replacing it for that case. Both fixed: `_settle_outgoing_pdc` now special-cases an unlinked schedule row still pointing at the clearing PDC (posts that one invoice first, exactly like the old code), and the `paid_from`/`paid_to` swap is corrected with a code comment explaining ERPNext's convention. Verified live with two isolated test scenarios before deploying. -->


# Landlord Cheque Plan Decoupling

## Summary

Splits landlord payables' current rigid 1:1 chain (one accrual period = one
cheque = one invoice, ADR-0011) into three independent, reconciled pieces:
an admin-entered cheque plan (`PDC Entry`), an accrual schedule that posts
`Purchase Invoice`s on its own due-date cadence regardless of cheque count
(`Head Lease Schedule`), and settlement (`Payment Entry`, with any
over-cleared cash held on-account and auto-swept oldest-invoice-first).
Root-cause fix for `/overview`'s Margin by property showing Rastec 21
(`BLD-00406`) at landlord cost = $0 (see ADR-0025 Context).

## Requirements

- A `Head Lease` on the `PDC` payment method cannot reach
  `head_lease_status = "Active"` until its cheque plan is complete —
  validated, not merely encouraged. Existing `Bank Transfer` buildings are
  unaffected (no cheque plan to complete).
- Cheque-plan input is free-form: any number of cheques, any amount, any
  date per cheque — no assumption it equals `monthly_rent × payment_frequency`
  or lines up 1:1 with `head_lease_schedule` rows.
- `head_lease_schedule` (accrual) posts a `Purchase Invoice` per due row on
  the daily scheduler for **every** Building regardless of
  `landlord_payment_method` — the PDC-vs-Bank-Transfer distinction no
  longer gates *when* an invoice posts, only how the landlord is paid.
- `mark_pdc_cleared` (Outgoing) never creates a Purchase Invoice. It
  allocates the cheque amount against the landlord's outstanding Purchase
  Invoices, oldest `posting_date` first, and posts any remainder as an
  on-account (`unallocated_amount`) Payment Entry against the Supplier.
- A daily scheduler step, after `generate_due_payables` posts each
  period's invoice, automatically sweeps any on-account balance for that
  invoice's supplier against it (oldest-invoice-first), using ERPNext's
  `Payment Reconciliation` tool. No staff confirmation for the sweep step
  itself. **Deliberately deferred, see Implementation Plan** — shipping an
  untested integration against ERPNext's Payment Reconciliation internals
  risked writing bad GL with no way to verify it live this round; until
  it lands, on-account balance just sits unswept (a known, non-destructive
  state — see Consequences in the ADR).
- No retroactive backfill: only Head Leases created after this ships use
  the new model. Already-elapsed periods on existing PDC-path Head Leases
  (Rastec 21 included) are a manual one-time data-entry task, out of scope
  here. `ACC-PINV-2026-00003` (Rastec 20) is left untouched.
- On-account (unreconciled advance) balance per landlord/building is
  surfaced somewhere in the portal (Head Lease page and/or Building
  Profitability) so an invoice sitting `Outstanding` while cash is already
  on hand doesn't read as a real receivable-side risk.

## Design

Files touched (relative to `apps/real_estate_os/` unless noted):

- **New** `real_estate_os/real_estate/doctype/head_lease_cheque_plan/` —
  child doctype `Head Lease Cheque Plan`: `check_number`, `amount`,
  `check_date`, `pdc_entry` (Link, read_only).
- `real_estate_os/real_estate/doctype/head_lease/head_lease.json` — new
  `cheque_plan` Table field (options `Head Lease Cheque Plan`).
- `real_estate_os/real_estate/doctype/head_lease/head_lease.py` —
  `validate_cheque_plan_complete()`: blocks `head_lease_status` reaching
  `Active` when `Building.landlord_payment_method == "PDC"` and
  `self.cheque_plan` is empty.
- `real_estate_os/payments/pdc.py`:
  - `generate_pdc_schedule_for_head_lease` replaced by
    `set_head_lease_cheque_plan(head_lease_name, cheques)` — delete-and-
    regenerate (same in-flight guard as `generate_pdc_schedule`: blocked
    once any existing cheque has left Pending), consuming admin-entered
    `cheques` directly instead of deriving one per `head_lease_schedule`
    row.
  - `_post_payable_for_outgoing_pdc` replaced by `_settle_outgoing_pdc`:
    allocates the cheque amount across the Supplier's outstanding
    Purchase Invoices (oldest `posting_date` first) in one Payment Entry
    (`get_payment_entry` scaffolds the first reference, subsequent
    invoices appended to `references`); any remainder becomes that
    Payment Entry's on-account `unallocated_amount` automatically
    (`Payment Entry.set_unallocated_amount`, no manual computation
    needed). If nothing is outstanding at all, builds the on-account
    Payment Entry directly (no invoice to scaffold from).
  - `mark_pdc_cleared`: Outgoing branch now calls `_settle_outgoing_pdc`
    instead of the old create-invoice-then-reconcile path; Incoming
    branch unchanged.
- `real_estate_os/payments/landlord_payables.py`:
  - `_bank_transfer_head_leases()` → `_live_head_leases()`: now includes
    every Bank Transfer Head Lease **plus** every PDC Head Lease that
    already has at least one `Head Lease Cheque Plan` row — this is the
    forward-only cutover gate the ADR requires. A PDC Head Lease with an
    empty cheque plan (legacy, pre-cutover) is excluded, so it doesn't
    get months of back-dated invoices the day this ships.
  - `_generate_due_for_head_lease`: dropped the `not row.pdc_entry`
    guard — idempotency is `row.status == "Pending"` +
    `due_date <= today` only (unset `row.purchase_invoice` is the
    idempotency key, same as before).
- `head_lease_schedule.json` (child table) — `pdc_entry` field left in
  place (still populated informationally by `invoicing.py`'s tenant-side
  equivalent pattern is N/A here; on the Head Lease side nothing writes
  it anymore under the new model) but no longer read as a gate anywhere.
- Frontend (`ui/src/pages/Dashboard.tsx`, `HeadLeasePanel`): the old
  "Generate outgoing cheques" (start-check-number/payment-day) mini-form
  replaced with a proper cheque-plan editor (add/remove rows of check
  number/amount/date, calls `setHeadLeaseChequePlan`) in its own
  "Outgoing cheque plan" card, separate from the accrual "Payment
  schedule" table above it. Editable only while every existing cheque is
  still Pending (`chequePlanLocked`), else shown read-only.
- **Not yet built** (see Implementation Plan): the advance-sweep
  scheduler step, and surfacing on-account balance in the portal.

## Implementation Plan

- [x] ADR approved — `0025-landlord-cheque-plan-decoupling`
- [x] `Head Lease Cheque Plan` child doctype + field on `Head Lease`
- [x] `head_lease.py::validate()` — block `Active` transition without a
      complete cheque plan (PDC buildings only)
- [x] Replace `generate_pdc_schedule_for_head_lease` with
      `set_head_lease_cheque_plan` consuming the cheque plan directly
- [x] Generalize `generate_due_payables`/`_live_head_leases` to post for
      Bank Transfer + cheque-planned PDC buildings; drop the `pdc_entry`
      guard; exclude legacy empty-cheque-plan PDC head leases (cutover)
- [x] Rework `mark_pdc_cleared` (Outgoing) via `_settle_outgoing_pdc`:
      oldest-outstanding-invoice allocation + on-account remainder, no
      invoice creation
- [x] Frontend cheque-plan editor in `HeadLeasePanel`, decoupled from the
      accrual schedule display
- [x] **New scheduler step: sweep on-account advance balance against
      newly posted invoices, oldest-first, via ERPNext's Payment
      Reconciliation tool.** Implemented as `sweep_advance_payments()`
      (`landlord_payables.py`), wired into `scheduler_events.daily`
      immediately after `generate_due_payables`. **Verified live**
      2026-09-06 on realestate.nnuggets.com with fully isolated disposable
      test data (own throwaway Landlord/Supplier — not the real Ooredoo
      Properties tied to Rastec 20/21): a $6,000 cheque cleared against a
      single $1,000 invoice left $5,000 on-account; two subsequent $1,000
      invoices, posted and swept one at a time, each correctly dropped
      from $1,000 outstanding to $0 and reduced the Payment Entry's
      unallocated balance by exactly $1,000 each time (5000 → 4000 →
      3000), with `references` growing correctly
      (`[INV1]` → `[INV1,INV2]` → `[INV1,INV2,INV3]`). All test data
      (Landlord, Supplier, Building, Cost Center, Head Lease, PDC Entry,
      3 Purchase Invoices, 1 Payment Entry) cancelled, force-deleted, and
      confirmed removed via independent query afterward.
- [x] Surface on-account balance in the portal — `get_landlord_advance_balance(head_lease_name)`
      (sums unallocated Payment Entry balance for that Head Lease's
      landlord Supplier) shown as an explanatory banner on the Head Lease
      page (`HeadLeasePanel`), reusing the existing "unattributed" amber-
      banner style; refreshed after Mark Cleared, the action that changes
      it. Deployed and smoke-tested live against the real Rastec 20/21
      Head Leases (both correctly return 0.0 — no crash, no false
      balance) 2026-09-06.
- [x] Verify live end-to-end: deployed to all 4 real_estate_os tenant
      sites (realestate/rastec/test/us.nnuggets.com) 2026-09-06 —
      `bench migrate` (new DocType) and `bench --site <site> clear-cache`
      (hooks.py scheduler change) both ran clean on all four, no errors.
      Core decoupling + gate + sweep exercised end-to-end on
      realestate.nnuggets.com per above.
- [x] Confirm existing Bank Transfer path and existing 1:1 PDC Head Leases
      (created before this ships) are unaffected — **verified live**:
      Rastec 20 (`HL-2026-00368`) and Rastec 21 (`HL-2026-00407`), the
      real pre-existing PDC-method Head Leases with empty cheque plans,
      confirmed `head_lease_status` unchanged (`Active`) and completely
      untouched by both test runs and the sweep (`ACC-PINV-2026-00003`
      outstanding still 0.0, `ACC-PAY-2026-00001` unchanged) — the
      cutover gate correctly excludes them from `_live_head_leases()`.
- [x] Update `landlord-payables.md` and
      `pdc-schedule-generation-and-reconciliation.md` status/requirements
      to reflect what this ADR supersedes.

## Acceptance Criteria

- [x] A `PDC`-method Head Lease cannot be set `Active` without a complete
      cheque plan; a `Bank Transfer`-method Head Lease is unaffected —
      verified live (gate blocked, then succeeded once a cheque plan was
      set)
- [x] Accrual `Purchase Invoice`s post monthly on schedule for a PDC-method
      Head Lease even before any cheque has cleared — verified live
      (invoice 1 posted and was outstanding before the cheque cleared)
- [x] A cheque cleared for more than the currently-outstanding invoice
      total posts the remainder as an on-account Payment Entry, not an
      error and not a lost amount — verified live ($6,000 cheque, $1,000
      invoice, $5,000 correctly left on-account)
- [x] A later-posted accrual invoice for the same supplier is
      automatically reconciled against existing on-account balance,
      oldest-invoice-first, with no manual step — verified live across
      two separate scheduler-tick simulations
- [x] Rastec 21 and other pre-existing Head Leases are unchanged by this
      work (no retroactive invoices, no forced cheque-plan backfill) —
      verified live, both real Rastec Head Leases and their financial
      records confirmed untouched

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `vault/decisions/0025-landlord-cheque-plan-decoupling.md`
- Feature: `landlord-payables.md`, `pdc-schedule-generation-and-reconciliation.md`,
  `head-lease-doctype.md`, `building-profitability-report.md`
