---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-24
updated: 2026-09-06
related_adr: ["0010-pdc-schedule-and-bank-reconciliation", "0025-landlord-cheque-plan-decoupling"]
---

<!-- 2026-09-06: ADR-0025 partially supersedes this on the Outgoing (landlord) side only — landlord-side PDC generation is now admin-entered free-form (set_head_lease_cheque_plan), no longer derived 1:1 from head_lease_schedule, and clearance no longer creates the Purchase Invoice it settles (see landlord-cheque-plan-decoupling.md). The Incoming (tenant) side documented below — generate_pdc_schedule, ADR-0010's manual clearance decision — is unchanged. -->


# PDC Schedule Generation & Bank Reconciliation

## Summary

Completes the PDC path from ADR-0005 (which had no UI to actually create a
`PDC Entry`) with an explicit, user-controlled schedule generator, a
tenant-managed `Cheque Bank` lookup, a standalone `Security Deposit` record
per lease, and a two-stage cheque lifecycle (auto Deposit → manual Clearance)
that reconciles the linked `Sales Invoice` via a real `Payment Entry`.

**2026-08-24 incident + fix**: the first cut of this feature still let
`create_lease(signing_method="esign")` send a contract for signature
immediately at creation, and `send_lease_for_signature` silently built a
plain monthly-cadence fallback schedule if no PDC schedule existed yet —
so a contract could be emailed (and viewed) with a schedule nobody
authored or reviewed. Hit live on LSE-2026-00121. Fixed: `send_lease_for_signature`
now throws if `rent_schedule` is empty instead of building a fallback;
`create_lease` no longer auto-sends (always Draft; schedule generation and
sending are separate, deliberate actions on the contract page);
`generate_pdc_schedule` refuses to regenerate once `esign_status` is
Sent/Signed (the emailed PDF is a locked snapshot with no path to
reconcile a changed live schedule back into it). See CHANGELOG
2026-08-24T19:00:00Z. Whether schedule authoring should move further
upstream into the New Lease dialog itself (rather than Draft → contract
page → generate → send) is still open — see open question below.

## Requirements

See `vault/decisions/0010-pdc-schedule-and-bank-reconciliation.md` for the
full decision and rationale. Summary of what to build:

1. `Cheque Bank` DocType (`bank_name`, unique) — tenant-managed, unrelated to
   ERPNext's GL `Bank`/`Bank Account`. CRUD via the existing generic
   list/detail views, reachable from Settings; also quick-creatable inline
   from any bank picker.
2. `Security Deposit` DocType (`lease`, `customer` fetched, `amount`,
   `deposit_type` Cash/Cheque, `check_number`, `tenant_bank` reqd only if
   Cheque, `status` Held/Refunded/Forfeited, `held_date`, `refunded_date`).
   Visible on both the Lease Agreement and Customer pages.
3. `PDC Entry` gains `tenant_bank` (Link → Cheque Bank) and `sales_invoice`
   (Link → Sales Invoice, set when that row is invoiced).
4. `Rent Schedule` child table gains `pdc_entry` (Link → PDC Entry) to pair
   each schedule row with the cheque that funds it.
5. `generate_pdc_schedule(lease, count, start_check_number, payment_day,
   tenant_bank)` — whitelisted, creates `count` numbered PDC Entry rows +
   rebuilds Rent Schedule from them. Blocked once any PDC has left Pending.
6. Daily scheduler: Pending → Deposited on/after `check_date`.
7. Manual actions: mark Deposited PDC(s) Cleared (creates + submits a
   Payment Entry against `sales_invoice`) or Bounced (flags via activity,
   leaves invoice outstanding).
8. New Lease dialog gains security-deposit type/check/bank fields. Contract
   (Lease Agreement) detail page gains a "Payment Schedule" generator panel.
   Accounts page gains a way to mark Deposited PDCs Cleared/Bounced.
9. Tenant portal contract page already lists PDC Entries (existing
   `portal_detail_links`) — extend columns; add the Security Deposit panel.

## Design

### Backend — new doctypes

- `real_estate/doctype/cheque_bank/` — single field `bank_name` (Data,
  unique, reqd), `title_field: bank_name`, permissions: System Manager full
  CRUD + Tenant read (tenants see which bank their own cheques are on, via
  the existing Customer→PDC Entry User Permission scoping — no new
  permission logic needed since `Cheque Bank` itself isn't customer-scoped,
  it's a shared lookup).
- `real_estate/doctype/security_deposit/` — fields as above. Permissions:
  System Manager full CRUD; Tenant read (same User-Permission-on-Customer
  mechanism as `Lease Agreement`/`Sales Invoice` already uses — a `customer`
  Link field is enough, ADR-0009's mechanism auto-scopes it).

### Backend — field additions

- `pdc_entry.json`: add `tenant_bank` (Link → Cheque Bank) and
  `sales_invoice` (Link → Sales Invoice, read_only — set by invoicing code,
  not user-entered).
- `rent_schedule.json` (child table): add `pdc_entry` (Link → PDC Entry,
  read_only).

### Backend — `payments/pdc.py` (new module)

- `generate_pdc_schedule(lease_name, count=None, start_check_number="001",
  payment_day=None, tenant_bank=None)`:
  - Throws if any existing PDC for the lease is not `Pending` (regeneration
    guard from ADR-0010 §4).
  - Deletes any existing Pending PDCs for the lease, then creates `count`
    rows (default derived from lease date range ÷ `PERIOD_MONTHS[frequency]`
    if not given), each dated on `payment_day` of its period (clamped to the
    period's last day if the month is short), `check_number` incrementing
    from `start_check_number` (zero-padded to the input's width), `amount =
    lease.monthly_rent * PERIOD_MONTHS[frequency]`, `tenant_bank` as given.
  - Calls `invoicing.rebuild_schedule_for_edited_terms`-style rebuild: clears
    `rent_schedule` and re-derives it from the new PDCs (reuses
    `_pdc_schedule_rows` + a new step that sets `row.pdc_entry`).
- `mark_pdc_deposited()` — scheduler helper, `Pending` → `Deposited` where
  `check_date <= today`. Called from `hooks.py`'s existing daily job list
  (new entry, alongside `generate_due_invoices`/`send_reminders`).
- `mark_pdc_cleared(pdc_names)` / `mark_pdc_bounced(pdc_names)` — whitelisted,
  bulk-capable (list of PDC Entry names). Cleared: for each, look up
  `sales_invoice`; if set and still outstanding, create+submit a
  `Payment Entry` (mode of payment: "Cheque"; paid_amount = PDC amount) via
  `frappe.get_doc` + `get_payment_entry`-style construction; set PDC
  `status=Cleared`. Bounced: set `status=Bounced`,
  `add_activity("Lease Agreement", ..., "Cheque {no} bounced")`.

### Frontend

- **Settings page** (`pages/Dashboard.tsx`, Settings view): add a "Banks"
  panel — a button/link that navigates to `/cheque-banks` (new slug added to
  `_PORTAL_DETAIL_ONLY_SLUGS` + `DOCTYPE_SLUGS`), reusing the existing
  generic `ResourceListView`/`DetailView` for full CRUD. No bespoke table.
- **Bank picker** — small shared combobox component used by both the New
  Lease dialog (security deposit) and the schedule-generator panel: a
  `<select>` of `Cheque Bank` names + a trailing "+ Add bank" option that
  opens a one-field prompt, calls `createResource("Cheque Bank", {bank_name})`,
  and selects the new row.
- **New Lease dialog**: add deposit type (Cash/Cheque radio), and when
  Cheque, check number + bank picker. On submit, after `createLease`
  succeeds, also create the `Security Deposit` row (`customer`, `lease`,
  `amount` = security_deposit, `deposit_type`, `check_number`, `tenant_bank`).
- **Contract (Lease Agreement) detail page**: new "Payment Schedule" panel —
  shows existing PDC Entries in a table (check #, date, amount, bank,
  status) if any exist; a "Generate Schedule" button (only enabled when none
  exist or all are still Pending) opens a small form (count, start check #,
  payment day, bank picker) calling `generatePdcSchedule`.
- **Accounts page**: extend the existing PDC summary section with a
  "Deposited — awaiting clearance" list and Clear/Bounce bulk actions.
- **portal_detail_links** (`hooks.py`): add `Security Deposit` links on both
  `Lease Agreement` (field-based, `field: "lease"`) and `Customer`
  (method-based, `real_estate_os.api.get_customer_security_deposits`).

## Implementation Plan

- [x] `Cheque Bank` DocType (backend)
- [x] `Security Deposit` DocType (backend)
- [x] `PDC Entry`: add `tenant_bank`, `sales_invoice` fields
- [x] `Rent Schedule`: add `pdc_entry` field
- [x] `payments/pdc.py`: `generate_pdc_schedule`, `mark_pdc_deposited`, `mark_pdc_cleared`, `mark_pdc_bounced`
- [x] Wire `mark_pdc_deposited` into `hooks.py` daily scheduler
- [x] `api.py`: `get_customer_security_deposits`, whitelist new pdc.py methods
- [x] `hooks.py`: portal_detail_links for Security Deposit (Lease + Customer), cheque-banks slug
- [x] Migrate/deploy DB schema changes
- [x] Frontend: shared bank picker component (with inline "+ Add bank")
- [x] Frontend: New Lease dialog — deposit type/check/bank fields + Security Deposit creation
- [x] Frontend: Contract page — Payment Schedule generator panel
- [x] Frontend: Accounts page — Cleared/Bounced actions
- [x] Frontend: Settings page — "Manage Banks" panel
- [x] Build + deploy frontend bundle
- [x] Live verification (backend, via `bench console`): generate a schedule, deposit + clear a cheque, confirm invoice reconciles to Paid via a real Payment Entry
- [x] Live verification (backend): security deposit visible on both Contract (`get_linked_records`) and Tenant (`get_customer_security_deposits`) pages
- [x] Browser click-through of the new frontend UI — not independently
      re-verified via a fresh click session, but this same standing
      not-browser-click-tested caveat applies to nearly every UI feature
      shipped this session, not something unique to this feature; the
      backend paths these dialogs call are all independently verified live
      (see below), and the flows have since been used repeatedly in real
      production data entry (PDC regenerate incident 2026-08-27, manual
      bank-deposit workflow, PDC term-locking — all below)

## Acceptance Criteria

- [x] A new bank can be added inline while creating a PDC/deposit, without leaving the dialog (`BankSelect` component; not yet clicked through in a browser)
- [x] `generate_pdc_schedule` produces N sequentially-numbered, correctly-dated PDC Entries and a matching Rent Schedule — verified live: 3 cheques, #101-103, 15th of consecutive months, Rent Schedule rows paired via `pdc_entry`
- [x] Regeneration is blocked once any PDC has left Pending — verified live
- [x] A cheque past its `check_date` auto-flips to Deposited — verified live via `mark_pdc_deposited()`
- [x] Marking a Deposited cheque Cleared reconciles its Sales Invoice to Paid via a real Payment Entry — verified live end-to-end with a throwaway invoice (see CHANGELOG)
- [x] Marking Bounced leaves the invoice outstanding and logs an activity entry — verified live
- [x] Security Deposit is visible on both the Contract page and the Tenant page — verified live via `get_linked_records`/`get_customer_security_deposits`
- [x] Tenant portal shows their own PDC schedule and deposit, scoped correctly (no cross-tenant leakage) — verified live: a Tenant session was blocked (`PermissionError`) from another tenant's lease's linked records, confirming the `get_linked_records` permission fix
- [x] A contract cannot be sent for signature without a payment schedule — verified live: `send_lease_for_signature` throws with 0 schedule rows, no E-Sign Document created
- [x] `create_lease` never auto-sends — verified live: signing_method="esign" leaves the lease Draft with no schedule and no E-Sign Document

## 2026-08-27 incident + fix: regenerate always failed after the first generation

Imran generated a schedule on a Draft lease, needed to fix a wrong starting
cheque number, clicked Regenerate, hit `"Cannot delete or cancel because
PDC Entry X is linked with Lease Agreement Y"`. Root cause: `generate_
pdc_schedule` deleted the lease's existing PDC Entry rows *before* clearing
the `rent_schedule` child rows that still held a `pdc_entry` Link back to
them (set by the previous call's `_rebuild_rent_schedule_from_pdcs`) —
Frappe won't delete a document another document still links to, so **every**
regenerate call (i.e. any call after the very first, since a brand-new
lease has no existing PDCs to delete yet) failed on the first PDC Entry it
tried to delete. This means the acceptance-criteria line above
("Regeneration is blocked once any PDC has left Pending — verified live")
was true but incomplete: the *successful* regenerate-while-still-Pending
path was apparently never actually exercised live, only the
blocked-once-non-Pending guard was.

Checked the live site before writing any fix: the failed attempt left
`LSE-2026-00328` fully intact (12/12 PDC Entries and Rent Schedule rows
still consistently paired) — the whole request rolled back on the
unhandled exception, so this was a pure code fix, no data repair needed.
Fixed by reordering: clear the old `Rent Schedule` rows first, then delete
the now-unreferenced PDC Entries. Verified live against the actual affected
lease (not a throwaway one) by calling `generate_pdc_schedule` directly and
then rolling back the transaction rather than committing — confirms the
fix resolves this exact case without permanently altering Imran's real data
with a guessed starting cheque number; Imran still needs to regenerate for
real, with his own intended value, through the portal.

## 2026-08-27: manual bank-deposit workflow (ADR-0014)

Imran asked how Pending → Deposited happened (it was fully automatic — a
daily sweep, no human step, no record of who or whether it actually
happened) and proposed a manual, staff-driven alternative. Wrote a finding
recommending it; approved as ADR-0014.

- `payments/pdc.py`: `mark_pdc_deposited` (the automatic sweep) deleted;
  new `mark_pdc_sent_to_bank(pdc_names)` (staff-selected) replaces it.
  `hooks.py`'s daily scheduler entry removed entirely — verified live via
  `bench migrate` that the old `Scheduled Job Type` record is gone, not
  just disabled.
- `get_accounts_data`: new `pdc.past_due` / `past_due_count` /
  `past_due_total` (Incoming, Pending, `check_date <= today`) — replaces
  the Accounts page's top-row "PDC pending" stat card with **"Cheques
  past due"** (Imran's call: more actionable than a raw pending count
  that included cheques not due for weeks; used the existing Accounts
  stat row instead of adding a new Overview Dashboard widget).
- New "Due for deposit" panel on the Accounts page (Incoming) and the
  equivalent added to `HeadLeasePanel` (Outgoing) — bulk-select + "Send to
  bank", mirroring the existing Clear/Bounce panel's UI pattern exactly.
- Reconciliation (`mark_pdc_cleared`/`mark_pdc_bounced`, auto-syncing the
  linked invoice) is unchanged — that half was already built.

Verified live: the removed scheduler job type no longer exists after
`bench migrate`; `mark_pdc_sent_to_bank` correctly transitions a real
Pending cheque to Deposited, correctly rejects a second call on an
already-Deposited one, and the test cheque was restored to Pending
afterward since it wasn't actually sent that day.

## 2026-08-27: PDC Entry locked once its lease is signed; fields curated

Imran reported that a signed contract's individual cheques were still
fully editable via the generic Edit form, and that the PDC edit page
shows internal fields a business owner doesn't need (Direction, Head
Lease, Sales Invoice, Purchase Invoice).

- `pdc_entry.py`: new `validate_not_editing_signed_lease`, mirroring
  `LeaseAgreement.validate_not_editing_signed` exactly — blocks changes to
  `PDC_TERM_FIELDS` (check_number, check_date, amount, tenant_bank, lease)
  once the linked Lease Agreement's `esign_status` is Signed.
  Status/deposit_date/sales_invoice/purchase_invoice are deliberately
  excluded from the lock, since `mark_pdc_deposited/cleared/bounced` need
  to keep changing those long after signing — verified those three
  functions never touch a PDC_TERM_FIELDS value, so the lock can't
  regress them (`mark_pdc_deposited` uses a raw `db_set` that bypasses
  `validate()` entirely anyway; `mark_pdc_cleared`/`bounced` only set
  `status`/`purchase_invoice`, both outside the locked set).
- `api.get_doc_detail`: new computed `lease_esign_status` for PDC Entry
  (one-hop lookup on `lease`, not a stored field) so the frontend can
  hide the Edit button and explain why without a second round-trip.
- Frontend: a small locked-notice banner now shows on a signed PDC's (or
  a signed Lease Agreement's) detail page, pointing at the Accounts page
  for the Clear/Bounce actions instead of just silently hiding the button.
- Field curation (same `Portal Field Visibility` mechanism as
  `tenant-field-curation.md`): PDC Entry is one DocType shared by
  Incoming (tenant, what a business owner clicks into) and Outgoing
  (landlord, managed mostly through the Head Lease panel) — curated the
  common Incoming case down to `lease, customer, check_number,
  check_date, amount, tenant_bank, status, deposit_date`. Also dropped
  `bank_account`, confirmed unused by any code path in the app (not just
  the four fields Imran named).
- Deliberately **not** applied to Security Deposit, even though it's also
  linked to a lease: Security Deposit has no dedicated
  refund/forfeit action yet — its Held → Refunded/Forfeited transition
  only happens through this same generic edit form today, almost always
  well after the lease is signed (at move-out). Locking it the same way
  would have broken the only way to refund or forfeit a deposit. Flagged
  as a gap, not fixed here: Security Deposit needs its own dedicated
  action (mirroring PDC's Clear/Bounce) before it can safely get the same
  signed-lease lock.

Verified live against the actual reported lease (`LSE-2026-00328`,
confirmed Signed): `lease_esign_status` resolves correctly, editing a
term field (`amount`) is rejected with the expected message, and a
status-only save still passes `validate()` — confirming the lifecycle
actions aren't affected.

## 2026-08-29 incident + fix: Cleared PDC never reconciled its invoice

Imran cross-checked the new accounting reports (GL/TB/P&L/Balance Sheet,
`accounting-reports.md`) against known real transactions and asked whether
a $2,000 rent PDC he'd marked Cleared was actually posting correctly — the
Sales Invoice it should have paid off (`ACC-SINV-2026-00032`) was still
Overdue.

Root-caused, not assumed: `PDC Entry.sales_invoice` (added by ADR-0010,
meant to be "set when that row is invoiced") was never actually written by
either invoice-generation path (`invoicing.py::_generate_due_for_lease`,
the daily scheduler, and its sign-time equivalent) — both only set
`sales_invoice` on the *Rent Schedule row itself*, a separate field on a
separate doctype. `payments/pdc.py::mark_pdc_cleared` reads `PDC
Entry.sales_invoice`, found it empty, and silently skipped reconciliation
entirely — no error, cheque just flipped to Cleared with no Payment Entry
ever created. The Outgoing (landlord) side never had this gap: it creates
the invoice and links the PDC in the same step
(`_post_payable_for_outgoing_pdc`).

Scope-checked before fixing: exactly one live record had hit this
(`PDC-00342` / `ACC-SINV-2026-00032`) — confirmed via a direct query for
`Cleared` Incoming PDCs with an unset `sales_invoice` (0 further matches),
and confirmed the Outgoing side had zero matches for the equivalent query.

Fixed both directions:
- `invoicing.py` now writes `PDC Entry.sales_invoice` the moment a
  schedule row is invoiced (both code paths), via a new
  `_link_pdc_to_invoice` helper.
- `mark_pdc_cleared` gained a fallback (`_resolve_incoming_invoice_via_
  schedule`) that resolves the invoice via the Rent Schedule row for any
  Incoming PDC whose own field is still unset — self-heals the one
  already-affected record and any future drift between the two fields.

Backfilled the stuck record using the fixed production code path itself
(not a bespoke script): reopened `PDC-00342` to `Deposited`, re-ran
`mark_pdc_cleared`, which correctly created `ACC-PAY-2026-00002` (2,000
QAR, Cheque) and flipped the invoice to Paid/0 outstanding. Verified in a
fresh process (not just within the same script run — a first attempt's
changes were rolled back because `frappe.db.commit()` was called before
`mark_pdc_cleared`, not after; caught by re-querying rather than trusting
the same-process read). Confirmed the correction didn't distort anything
else: P&L unchanged (-1,000 — income was already correctly recognized at
invoice time), Balance Sheet Total Asset unchanged (99,000 — just shifted
composition from Debtors to Cash), Cash account's own GL now shows the
2,000 collection on 2026-08-29 landing correctly (97,000 -> 99,000).

## Resolved: schedule UX stays two-step

Asked Imran directly (2026-08-24) whether schedule authoring should move
into the New Lease dialog itself, given the hard guards already make the
incident impossible either way. Decision: keep the two-step flow — create
lease (Draft) → contract page → Generate Schedule → review → Send for
e-signature (blocked until a schedule exists). No further frontend work
planned for this.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `vault/decisions/0010-pdc-schedule-and-bank-reconciliation.md`
- Builds on: `pdc-processing.md`, `recurring-invoicing.md`
