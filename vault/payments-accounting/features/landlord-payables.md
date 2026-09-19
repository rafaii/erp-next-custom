---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-24
updated: 2026-09-06
related_adr: ["0011-head-lease-landlord-payables", "0025-landlord-cheque-plan-decoupling"]
---

<!-- 2026-09-06: ADR-0025 supersedes the PDC-path details below — accrual invoices now post on schedule for every Building regardless of landlord_payment_method (the "PDC path posts on PDC clearance" design here no longer holds), and mark_pdc_cleared no longer creates the Purchase Invoice it settles. See landlord-cheque-plan-decoupling.md for the current design; this file's history stays for context on how the gap (Rastec 21 at $0 landlord cost) was found. -->


# Landlord Payables

## Summary

Generates and posts what we owe landlords. Per ADR-0011, the default and
requested path is **outgoing PDC**: cheques we issue to the landlord,
reusing the existing `Cheque Bank`/`PDC Entry`/reconciliation machinery
from ADR-0010, extended with a `direction` field (`Incoming` / `Outgoing`)
instead of forking a parallel DocType. `Building.landlord_payment_method`
(`PDC` default, or `Bank Transfer`) makes this configurable per building —
set at Building creation, editable later.

This is the mirror of `recurring-invoicing.md` (tenant side) and is what
makes margin per building computable — see gaps G2/G4 in the findings doc.

## Requirements

- `PDC Entry` gets a `direction` Select field (`Incoming` / `Outgoing`,
  default `Incoming`) plus a `head_lease` Link (used only when
  `Outgoing`) and a `purchase_invoice` Link (the outgoing analogue of the
  existing `sales_invoice`). `lease` stops being `reqd` at the DocType
  level — validated as an XOR with `head_lease` in the controller instead
  (`direction=Incoming` requires `lease`, empty `head_lease`; vice versa).
  No backfill patch needed: a Select field with a schema-level default
  gets that default stamped onto every existing row by `ALTER TABLE ADD
  COLUMN` itself (confirmed the hard way on `lease_status` — PR #5), so
  every live PDC Entry reads `Incoming` the moment the column exists.
- The "Rent" Item (`invoicing.py::_get_rent_item`) becomes
  `is_purchase_item: 1` in addition to `is_sales_item: 1` — one Item
  posts both directions rather than forking a second one.
- Landlord-rent expense posts to `Company.default_expense_account` (the
  payables-side analogue of `default_income_account`).
- For a Building with `landlord_payment_method = "PDC"`: each due `Head
  Lease Schedule` row requires a linked outgoing `PDC Entry`
  (`direction = "Outgoing"`); its Deposited → Cleared/Bounced lifecycle
  reuses `payments/pdc.py`'s existing state machine, generalized to handle
  both directions instead of assuming incoming.
- For a Building with `landlord_payment_method = "Bank Transfer"`: daily
  scheduler job creates a `Purchase Invoice` directly for each due `Head
  Lease Schedule` row (mirrors `invoicing.generate_due_invoices`), no PDC
  involved.
- Either path ends in a `Purchase Invoice` against
  `default_payable_account`, with the Building's `cost_center` on every
  line.
- Landlord payment recorded via `Payment Entry` against the `Purchase
  Invoice` (reuses ERPNext's standard `get_payment_entry`, same pattern
  `payments/pdc.py::_reconcile_invoice` already uses).
- `Head Lease Schedule` row status: Pending → Invoiced → Paid / Overdue.

## Design

- Extend `real_estate_os/real_estate/doctype/pdc_entry/pdc_entry.json`
  with `direction` (Select, default `Incoming`); patch backfills existing
  rows.
- `real_estate_os/payments/pdc.py`: generalize `mark_pdc_deposited`,
  `mark_pdc_cleared`, `mark_pdc_bounced`, `_reconcile_invoice` to branch on
  `direction` — `Outgoing` reconciles against a `Purchase Invoice`
  (`get_payment_entry` in `payment_type="Pay"` mode) instead of a `Sales
  Invoice`.
- New `real_estate_os/payments/landlord_payables.py`:
  `create_payable_for_head_lease(head_lease, schedule_row)`,
  `generate_due_payables()` (Bank Transfer path only — the PDC path posts
  on PDC clearance, not on a due-date scheduler tick).
- Wire `generate_due_payables` into `hooks.py::scheduler_events.daily`.

## Implementation Plan

- [x] ADR approved — `0011-head-lease-landlord-payables`
- [x] `PDC Entry.direction` field (no backfill patch needed — schema
      default confirmed to stamp existing rows on ALTER TABLE)
- [x] Generalize `payments/pdc.py` reconciliation for `Outgoing` direction
      — PR #6 (Head Lease DocType), PR #7 (PDC direction), both live
- [x] `build_payment_schedule(head_lease_doc)` — populates
      `head_lease_schedule` from Start/End Date + Monthly Rent, called
      automatically from `Head Lease.after_insert` (no cheque-number/bank
      choices to make here unlike tenant PDC schedules, so no separate
      user-triggered action needed) — PR #9
- [x] `create_payable_for_head_lease` — Purchase Invoice with cost center
      — PR #9
- [x] `generate_due_payables` scheduler job (Bank Transfer path), wired
      into `hooks.py::scheduler_events.daily` — PR #9
- [x] "Rent" Item purchase-side flag — turned out to already be
      `is_purchase_item: 1` (ERPNext's own Item default), verified live;
      no code change needed
- [x] Verify: Payment Entry reconciliation closes the Purchase Invoice —
      verified live on 2026-08-25 with a real (then cleaned-up) Head
      Lease/Purchase Invoice/Payment Entry cycle: 13 schedule rows
      generated correctly, 3 due rows each posted their own Purchase
      Invoice (supplier/cost-center/credit_to/expense_account all
      correct), `get_payment_entry("Purchase Invoice", ...)` correctly
      inferred `payment_type="Pay"` (never tested before this — the PR #7
      changelog flagged this as an open assumption), and submitting the
      Payment Entry took `outstanding_amount` from 1000 to 0. Verification
      caught and fixed its own bug: the test script only tracked/cleaned
      up the first of the 3 created invoices, leaving 2 live submitted
      Purchase Invoices behind — found by re-querying the DB independently
      instead of trusting the script's own "cleanup: ok" self-report, then
      fixed with a second pass. All test data (Head Lease, 3 Purchase
      Invoices, 1 Payment Entry) confirmed removed and Building
      BLD-00004's `landlord_payment_method` confirmed restored to `PDC`.
- [x] **Fixed a real, confirmed security bug (PR #11)**: PDC Entry's
      Tenant read permission did NOT exclude Outgoing rows. Verified live
      against a real tenant portal login (`khalid@4itrading.com`, no
      relation to the test Landlord/Head Lease) before writing the fix:
      both `frappe.get_list("PDC Entry", ...)` and a direct
      `frappe.get_doc(...)` returned an unrelated Outgoing PDC — landlord
      name, amount, Head Lease reference all exposed. Root cause: Outgoing
      rows have no `customer` value (they settle a Head Lease, not a
      Lease), and Frappe's User Permission scoping only restricts a
      document when the restricted link field is actually *set* — an
      empty field is visible to everyone unless
      `apply_strict_user_permissions` is on (confirmed off on this site).
      Fixed with `permission_query_conditions` (list views) and
      `has_permission` (direct `get_doc`) hooks on PDC Entry, both scoping
      to `direction = "Incoming"` for anyone without System Manager.
      **Deploy gotcha found the hard way**: rebuilding the image and
      recreating containers was NOT enough — the exploit still succeeded
      immediately after that deploy. `bench --site <site> clear-cache` was
      required before the new `hooks.py` entries took effect (Frappe
      caches resolved hooks in Redis per site, independent of the app
      code on disk). Re-verified live after the cache clear: the same
      exploit attempt is now blocked on both `get_list` and `get_doc`, and
      a real tenant with 24 genuine Incoming PDCs (`layla@4itrading.com`)
      still sees all 24 — the fix isn't over-broad. **Any future PR that
      changes `hooks.py` needs a `clear-cache` step in its deploy, not
      just an image rebuild.**
- [x] `generate_pdc_schedule_for_head_lease(head_lease_name,
      start_check_number, payment_day)` (`payments/pdc.py`) — creates one
      Outgoing PDC Entry per not-yet-linked `head_lease_schedule` row for a
      PDC-path Building. Whitelisted, explicit action (mirrors
      `generate_pdc_schedule`'s user-triggered convention) — not yet
      reachable from any UI, since there's no admin-portal page for Head
      Lease; callable via API/bench console today. Fill-in-missing rather
      than delete-and-rebuild, since `head_lease_schedule` itself is never
      regenerated once built. PR #10.
- [x] `mark_pdc_cleared` now creates the Purchase Invoice for an Outgoing
      PDC at clearance time if one doesn't exist yet
      (`_post_payable_for_outgoing_pdc`), then reconciles it the same way
      as before — this is what "the PDC path posts on PDC clearance" in
      the Design section actually required, and was still missing after
      PR #9. PR #10. **Verified live end-to-end on 2026-08-25**: generated
      13 Outgoing PDC Entries for a real Head Lease (correct
      direction/landlord fetch-through/check numbers), forced one due,
      confirmed `mark_pdc_deposited` picked it up, `mark_pdc_cleared`
      correctly created a fresh Purchase Invoice at that moment (not
      before), reconciled it to `outstanding_amount = 0`, and updated the
      linked `head_lease_schedule` row to `Invoiced`. All test data (Head
      Lease, 13 PDC Entries, 1 Purchase Invoice, 1 Payment Entry)
      confirmed removed via independent DB query afterward.
- [x] Review-caught fix: `generate_due_payables` (Bank Transfer path)
      could have double-billed a schedule row that already had an outgoing
      PDC linked to it, if a Building's `landlord_payment_method` was
      switched from PDC to Bank Transfer after cheques were already
      issued (the field is explicitly editable after creation, per
      ADR-0011). Now skips any row with a linked `pdc_entry` regardless of
      the Building's current setting. PR #10.
- [ ] Escalation not yet applied: `build_payment_schedule` is flat-rate;
      `escalation_percent`/`escalation_frequency` are captured on the
      doctype but don't affect schedule amounts yet.
- [x] Which of the company's own bank accounts an outgoing cheque is
      drawn on — closed 2026-08-29 by `Head Lease.bank_account` (step 3 of
      `bank-account-setup.md`): picked at Head Lease creation (auto-filled
      to the company default if there's only one and none was chosen),
      resolved into a real GL account at clearance via
      `_reconcile_invoice`'s explicit `bank_account=` argument to
      `get_payment_entry`.
- [x] Admin-portal UI (PR #12): Head Lease surfaces as a linked record on
      the Building detail page (1:1, so a single clickable row rather than
      a separate top-level nav item — confirmed with Imran via
      AskUserQuestion). Its own detail page gets a `HeadLeasePanel`
      showing the payment schedule table, a "Generate outgoing cheques"
      action (`generate_pdc_schedule_for_head_lease`), and a "Bank
      confirmation needed" mini-panel for Deposited outgoing cheques
      (reusing `markPdcCleared`/`markPdcBounced` — these were previously
      only reachable from the global Accounts page, which filters to
      Incoming only, so Outgoing cheques had no confirmation path in the
      UI at all until this). Wired via `portal_detail_links` in
      `hooks.py`, the same config-driven mechanism every other
      relationship in this app uses — no bespoke fetch code needed for
      the linked-records list itself.

PR #3 of 3 for ADR-0011 landed as PR #9 + PR #10
(`agent/developer-1/landlord-payables`,
`agent/developer-1/outgoing-pdc-generator`).

## Acceptance Criteria

- [x] Existing incoming-PDC flow (tenant rent) verified unchanged after
      the `direction` field shipped — confirmed throughout this session
- [x] A `PDC` Building's due Head Lease Schedule row blocks without a
      linked outgoing PDC; a `Bank Transfer` Building's doesn't
- [x] Purchase Invoice lines carry the correct Building cost center —
      verified via direct GL Entry query 2026-08-27: `ACC-PINV-2026-00003`'s
      expense line shows `cost_center: 'BLD-00320 - Rastec 20 - RRE'`
- [x] Building Profitability report shows non-zero expense side — shipped
      (`building-profitability-report.md`) and verified live with real
      posted landlord expense (QAR 3,000)

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `vault/decisions/0011-head-lease-landlord-payables.md`
- Feature: `custom-module/features/head-lease-doctype.md`,
  `cost-center-per-building.md`, `building-profitability-report.md`,
  `pdc-schedule-generation-and-reconciliation.md`
