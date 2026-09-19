---
status: analysis
owner: developer-1
domain: payments-accounting
created: 2026-08-30
related_adr: ["0015-portal-native-accounting-reports", "0017-platform-modularization"]
---

# Accounting UI Gap Analysis vs. Expert Wireframe

Compares `vault/expert-suggestions/accounting-ui-wireframe.md` (a generic
ERPNext-based SMB accounting-UI spec, written for a multi-vertical SaaS)
against what's actually built in `accounts_portal` +
`real_estate_os/ui/src/pages/Dashboard.tsx` today. Every finding below was
verified against the live code (file:line), not assumed from memory —
see the fact-finding pass this analysis is based on.

## Gap table

| # | Wireframe capability | Status | Evidence |
|---|---|---|---|
| 1 | Chart of Accounts (tree, group/ledger, balances) | ❌ Missing | No COA screen anywhere. `get_account_options()` (`accounts_portal/reports.py:59-78`) returns a flat leaf-only list (`is_group=0`) for dropdown filters — no tree, no group nodes, no per-account balance. |
| 2 | JE — live balance check, disable submit until balanced | ✅ Met | `NewJournalEntryDialog` shows running Debit/Credit totals; Create button disabled until they match. |
| 3 | JE — templates (depreciation, payroll, etc.) | ❌ Missing | No template concept anywhere. |
| 4 | JE — Party field on AR/AP rows | ❌ Missing (by design) | Receivable/Payable accounts are deliberately excluded from the account picker (`Dashboard.tsx:4705-4711`) since Party isn't collected — a real limitation, not an oversight: ERPNext throws on insert otherwise. |
| 5 | JE — "Reverse Entry" (new dated reversing JE) | ❌ Missing | Only plain ERPNext `cancel()` exists (`journal_entry.py:82`) — internally reverses GL Entries but creates no new visible document. |
| 6 | JE — keyboard-first entry, auto-suggest offsetting account | ❌ Missing | Standard independent Select/Input fields (`updateRow`, `Dashboard.tsx:4725-4727`); no Tab/Enter flow, no suggestion logic. |
| 7 | JE — submitted transactions immutable, reversal not silent edit | ✅ Met | Draft-then-submit by design (see `accounting-reports.md`) — a deliberate choice made explicitly to avoid the one-way-door class of bug that hit `ACC-PAY-2026-00001`/PDC-00369. |
| 8 | GL — account/date/cost-center filters | ✅ Met | `GeneralLedgerPanel` (`Dashboard.tsx:4242-4340`). |
| 9 | GL — Party filter | ❌ Missing | No party dropdown in the panel. |
| 10 | GL — drill-down to source voucher | ❌ Missing | `ReportTable` (`Dashboard.tsx:4146-4181`) renders plain non-interactive rows — no onClick, no navigation. |
| 11 | GL — Export | ❌ Missing | No export button anywhere in the panel. |
| 12 | Trial Balance — prior-period/prior-year comparison | ❌ Missing | Single-period only (`TrialBalancePanel:4345-4382`). |
| 13 | P&L — monthly/quarterly/yearly toggle | ❌ Missing (by design) | `_statement_filters` hardcodes `periodicity="Yearly"`, collapsed to one "Total" column (`reports.py` ~99-141) — a deliberate MVP simplification, not an oversight. |
| 14 | P&L — % of total column, trend arrow, vs-prior-period | ❌ Missing | `formatReportCell` (`Dashboard.tsx:4183-4187`) only formats currency/passthrough. |
| 15 | P&L — building/cost-center filter | ✅ Met, **beyond** the wireframe | Real-estate-specific per-building P&L the generic wireframe doesn't even consider — a genuine strength, not a gap. |
| 16 | Balance Sheet — side-by-side Assets \| Liabilities+Equity | ❌ Missing | Flat single vertical list via plain `ReportTable`, in whatever order ERPNext's `balance_sheet.execute()` returns. |
| 17 | AR/AP Aging & Party Ledger | ❌ Missing | Zero code. Already tracked separately as [rent-roll-and-arrears-report](features/rent-roll-and-arrears-report.md) (`status: planned`) — covers the AR/tenant-arrears half; no symmetric AP/landlord aging exists or is planned yet. |
| 18 | Bank Reconciliation (bank feed ↔ GL matching) | ✅ Met | Done 2026-08-30 (Phase 5b). `accounts_portal.bank_reconciliation` + `BankReconciliationPanel` (Reports page). Manual transaction entry only — no CSV/statement import yet. |
| 19 | Multi-company / company switcher | **N/A** — architecture difference | Tenant-per-site (ADR-0003): every site has exactly one Company. A switcher would be dead UI here, not a gap to fill. |
| 20 | Role-based terminology (business label vs. accounting label) | ✅ Substantially met | Staff nav already uses business language ("Contracts", not "Sales Invoice"); the Accountant-gated Reports section is the only place raw accounting terms (GL, Trial Balance, Journal Entry) appear — this is effectively the same split the wireframe describes, achieved by RBAC + information architecture rather than an explicit label-mapping layer. |
| 21 | Vertical app never writes GL directly — routes through Sales/Purchase Invoice | ✅ Fully met | `invoicing.py` (rent), `landlord_payables.py` (landlord bills), and PDC reconciliation (`_reconcile_invoice`) all create real Sales Invoice / Purchase Invoice / Payment Entry documents — exactly the architecture the wireframe prescribes as non-negotiable. |
| 22 | "View Accounting Impact" link on vertical documents | ❌ Missing | No such link/panel exists on the Lease Agreement or Head Lease detail pages. GL is only reachable via the standalone Reports page, with no per-document deep link. |
| 23 | Audit trail surfaced in JE UI (who posted, when) | ⚠️ Partial | The data exists (Frappe's own `owner`/`creation`/`modified_by` fields on every Journal Entry) but isn't fetched or shown in `JournalEntryPanel`'s table today. |
| 24 | Industry-template Chart of Accounts seeding | ❌ Missing as a general concept, **N/A as stated** | The wireframe's "template" concept assumes one ERPNext instance serving multiple verticals that each pick a COA template. Our architecture is one OS app per site (ADR-0003) — real estate's own accounts (Rental Income, Landlord Rent Expense) are already seeded automatically via `rename_default_income_expense_accounts` at install, which *is* the equivalent of "the real estate industry template," just not a user-facing picker. |

## What's already strong (don't re-litigate these)

- **Vertical-to-GL architecture (#21)** — this is the wireframe's single
  most-emphasized principle ("never let a vertical app write GL entries
  directly"), and it's already fully how this platform works. Nothing to
  build here.
- **Per-building P&L (#15)** — genuinely exceeds the wireframe's own
  scope, which never considers a sub-company reporting dimension.
- **Draft-then-submit JE immutability (#7)** — matches the wireframe's
  "never edit a posted transaction" principle exactly, and was a
  deliberate design choice this session, not an accident.
- **Two platform extractions (`inbuilt_esign`, `accounts_portal`)** — the
  wireframe doesn't address app/module architecture at all, but this
  platform is already ahead of a typical single-app SMB tool in that
  dimension.

## Implementation plan

Phased by cost-to-value, not by the wireframe's own document order. Each
phase is independently shippable — nothing here blocks anything else.

### Phase 1 — Traceability & trust (small, high value) — ✅ done 2026-08-30

The wireframe's own framing: *"every number... must be clickable... this
traceability is what makes an SMB owner trust the system."* This is the
cheapest phase and addresses the wireframe's most-repeated principle.

- [x] **GL drill-down**: `ReportTable` rows with `voucher_type`/
      `voucher_no` now open a new `VoucherPreviewDialog` — a lightweight
      read-only field list via `get_doc_detail`, deliberately *not* the
      full `DetailView` (which carries edit machinery that has no place
      in a glance-and-close preview). Works for any voucher doctype
      generically; needed no new routes.
- [x] **Party filter on GL**: `accounts_portal.reports.get_general_
      ledger` now accepts `party_type`/`party`, confirmed live that
      ERPNext's own engine expects `party` as a list (its
      `validate_party` iterates it) before wiring the frontend dropdown.
- [x] **"View Accounting Impact" on Lease Agreement / Head Lease**: new
      collapsed `AccountingImpactPanel` — Lease Agreement scopes to the
      tenant's own Customer party ledger; Head Lease scopes to the
      building's Cost Center (resolved via `getDocDetail("Building",
      ...)`, since Head Lease's own fields only carry the Building
      Link) — the same reporting dimension this app already uses
      everywhere else, more useful than a single Supplier ledger. Both
      reuse `get_general_ledger` + `VoucherPreviewDialog`, no new report
      engine.
- [x] **Audit info in the Journal Entry list**: `get_journal_entries`
      now resolves `owner`/`modified_by` to full names server-side
      (`posted_by`/`last_action_by`); shown as a new column.

Verified live end-to-end via `bench console` against the actual deployed
endpoints (not just unit logic) before and after deploy, real-estate
PR #53 + [accounts-portal@4985d03](https://github.com/rafaii/accounts-portal/commit/4985d03).

### Phase 2 — Financial statement presentation

- [ ] **Balance Sheet side-by-side layout**: the data ERPNext's
      `balance_sheet.execute()` returns already carries each row's
      root type; this is a frontend-only reflow of `BalanceSheetPanel`
      into two columns, no backend change.
- [ ] **Trial Balance prior-period comparison**: verify first whether
      `trial_balance.execute()` supports a comparison natively via its
      own filters before assuming a double-call-and-merge approach is
      needed (same "verify before assuming" discipline this session
      applied to `get_payment_entry`'s `bank_account` param and Frappe's
      hook-merge behavior).
- [ ] **P&L "% of total" column**: pure frontend — each row's value
      divided by the already-returned Total Income/Expense, no backend
      change.
- [ ] **CSV export for GL/TB/P&L/BS**: client-side blob download from
      already-fetched `report.data` — no backend endpoint needed.
- [ ] **P&L period toggle (Monthly/Quarterly/Yearly)**: relaxes the
      current hardcoded-Yearly collapse; needs `_statement_filters` to
      accept a periodicity parameter instead of hardcoding it, plus a
      frontend selector. Medium effort — verify `_relabel_period_column`
      still behaves sensibly once there's more than one real period
      column to show.

### Phase 3 — Chart of Accounts viewer

- [ ] New `accounts_portal` endpoint returning the full COA as a tree
      (`is_group`, `parent_account`, computed balance per node via
      `get_balance_on` or a single GL aggregation query) — genuinely new
      backend work, not a wrapper around an existing report the way
      everything else in `accounts_portal` is.
- [ ] New frontend tree component (collapsible group/leaf rows with
      running balances) — no existing tree UI in this codebase to reuse;
      the largest single frontend investment in this plan.

### Phase 4 — Journal Entry maturity

- [ ] **Reverse Entry action**: new backend function creating a new
      Journal Entry with every row's debit/credit swapped, dated today,
      referencing the original via `user_remark` or a new field —
      well-defined, moderate effort.
- [ ] **Party support on JE rows**: conditionally show Party Type/Party
      fields when a Receivable/Payable account is selected in a row,
      mirroring ERPNext's own dynamic form behavior — needed before a
      Journal Entry can correct an AR/AP balance directly.
- [ ] **JE templates**: no native ERPNext "Journal Entry Template"
      doctype exists to lean on — this would need either a small new
      doctype (saved row-sets) or a simpler "duplicate this entry" convenience
      action as a cheaper first cut.
- [ ] **Keyboard-first row entry / auto-suggest offsetting account**: UX
      polish, lowest priority in this phase.

### Phase 5a — AR/AP Aging (cross-reference, don't duplicate)

The AR/tenant-arrears half is already the planned
[rent-roll-and-arrears-report](features/rent-roll-and-arrears-report.md).
A symmetric AP/landlord-aging view is natural to add when that ships,
not before.

### Phase 5b — Bank Reconciliation: promoted to must-have, re-scoped (✅ done 2026-08-30)

**Correction (2026-08-30, Imran)**: this was originally framed as
optional here on the theory that this platform runs on PDC/cheques, not
bank feeds. Wrong assumption, corrected directly: *"not all verticals
work on PDC. For real estate, PDC might be an option. But there may be
other use cases even within real estate, where they e-transfer every
month. In that case, we need a way to reconcile using bank feed."*
PDC is one collection method this platform happens to support well, not
the only one a real business will actually use — a lease that gets paid
by e-transfer or wire has no PDC Entry to clear, so it needs a genuinely
different reconciliation path.

**Re-scoped smaller than first estimated** once checked: ERPNext already
ships a full, mature reconciliation engine — `Bank Transaction`, `Bank
Reconciliation Tool`, `Bank Clearance`, `Bank Statement Import` doctypes
all exist in this instance already (confirmed live). The tool's own
module (`erpnext.accounts.doctype.bank_reconciliation_tool.
bank_reconciliation_tool`) already exposes `get_bank_transactions`,
`get_linked_payments` (matching suggestions), `reconcile_vouchers`, and
`create_payment_entry_bts` (create-and-match a Payment Entry straight
from a bank line in one step — exactly the incoming-e-transfer case).
Same pattern as every other accounting screen this session: **expose the
existing ERPNext engine through a portal-native UI, don't reimplement
matching logic.**

This closes two gaps at once, not one — a genuinely pre-existing gap was
found while re-scoping this: **the landlord-side "Bank Transfer" payment
method (`Building.landlord_payment_method`) already exists and already
generates Purchase Invoices on schedule (`landlord_payables.
generate_due_payables`), but nothing has ever reconciled them** — they
sit Outstanding forever today, with no path in this app to mark them
paid. Bank Reconciliation fixes both the tenant (Sales Invoice) and
landlord (Purchase Invoice) sides in one build, since ERPNext's own
matching query already handles both voucher types.

**Two parts, both needed:**

- [x] **Part A — model non-PDC rent collection** — done 2026-08-30
  (real-estate PR #54). Added `Lease Agreement.payment_method` (`PDC` /
  `Bank Transfer`, default `PDC`), mirroring `Building.
  landlord_payment_method`'s naming exactly. New `generate_bank_transfer_
  schedule` (`pdc.py`) builds `rent_schedule` directly — same
  `_period_due_dates`/`monthly_rent` math `generate_pdc_schedule` already
  uses, minus PDC Entry creation. `generate_due_invoices` needed no
  change at all, confirmed by reading `_link_pdc_to_invoice` (already a
  no-op when `row.pdc_entry` is unset) before assuming otherwise. New
  `BankTransferSchedulePanel` component kept deliberately separate from
  `PaymentSchedulePanel` — no shared internals, so this can't regress the
  live PDC flow real leases already depend on. `create_lease`/
  `NewLeaseDialog` gained a Payment Method selector.
  **Verified live** (throwaway test lease, created → schedule generated
  → confirmed 3 correctly-dated Pending rows with no `pdc_entry` →
  deleted → confirmed gone and the Unit released back to Vacant in a
  fresh process).
- [x] **Part B — Bank Reconciliation UI** — done 2026-08-30
  (real-estate PR #55; `accounts_portal`@636aa56). Entirely generic,
  zero real-estate concepts involved, in `accounts_portal/
  bank_reconciliation.py`:
  - [x] `get_bank_transactions(bank_account, from_date, to_date)` —
        thin wrap of ERPNext's own function; it already filters to
        `unallocated_amount > 0`, so the "working queue" framing is free.
  - [x] `get_matching_invoices(bank_transaction, invoice_doctype)` —
        **not** ERPNext's own `get_linked_payments`. Read its source
        live before shipping and found it only matches invoices already
        flagged paid *through that specific bank account* (a `Sales
        Invoice Payment` row, or `Purchase Invoice.is_paid=1`) — the POS
        same-document cash-sale pattern. Every invoice this app creates
        (rent, landlord payables) is inserted Outstanding and settled
        later by a separate bank line, so that query would always have
        returned nothing for the actual use case here. Queries open
        invoices directly instead, ranked by closeness of amount.
  - [x] `match_transaction_to_invoice(bank_transaction, invoice_doctype,
        invoice_name)` — also **not** ERPNext's `create_payment_entry_
        bts`: that builds an unallocated party-level Payment Entry with
        no `references` row, which would leave the invoice still
        Outstanding even after the money arrived. Uses `get_payment_
        entry` instead (the same function this app's own PDC clearance
        already relies on) to get a properly invoice-allocated Payment
        Entry, submits it, then calls ERPNext's `reconcile_vouchers` to
        link it to the bank line — folds the originally-separate
        "create" and "reconcile" steps into one call, since a portal
        user only ever wants both together.
  - [x] `create_bank_transaction(...)` — manual entry v1, as planned.
        CSV import (`Bank Statement Import`) remains a fast-follow, not
        built.
  - [x] New portal panel (`BankReconciliationPanel`) — landed as a sixth
        card in the existing Reports page's report-picker grid (next to
        General Ledger, Trial Balance, etc.) rather than a fully separate
        top-level screen — the existing "pick a report → full-panel view
        with a Back button" navigation shape already fit a working queue
        well enough that a new nav concept wasn't needed. Per-Bank-
        Account transaction list, a "Match" button per line opening a
        dialog of ranked open-invoice candidates, and a "Record bank
        line" manual-entry dialog.
  - **Verified live** before writing any frontend code: every ERPNext
    function signature and `Bank Transaction`/`Bank Account` field name
    used here was checked against the live site via `bench console`
    first (this is what surfaced the `get_linked_payments`/
    `create_payment_entry_bts` mismatches above). Then verified the full
    write path end-to-end with a throwaway Sales Invoice: created →
    matched → `match_transaction_to_invoice` correctly took the invoice
    to `outstanding_amount: 0` / `Paid` and the Bank Transaction to
    `Reconciled` / `unallocated_amount: 0` → invoice, Payment Entry, and
    Bank Transaction all cancelled and force-deleted afterward, confirmed
    gone in a fresh console process, with the site's real invoice/
    transaction counts unaffected.
  - **Same-day fix, caught by advisor review before reporting done**: the
    first cut of `match_transaction_to_invoice` always called
    `get_payment_entry` with no `party_amount`, which builds the Payment
    Entry for the invoice's *entire* outstanding regardless of how much
    the bank line actually carries — the initial live test happened to
    use an exact-amount match, which structurally couldn't expose this.
    A bank line smaller than the invoice (e.g. deposit 3000 against an
    invoice outstanding 5000) submitted a Payment Entry for the full
    5000 and marked the invoice Paid despite only 3000 landing — a
    books-corrupting bug. Fixed by computing
    `settle_amount = min(bank transaction unallocated, invoice
    outstanding)` and passing it as `party_amount`, which flows straight
    into the Payment Entry reference row's `allocated_amount`. Also
    fixed `get_matching_invoices` ranking only the 100 most-recently-
    posted open invoices before sorting by amount closeness (an
    exact-amount match older than that window was invisible) — now
    scoped by company/currency and ranks the full open set before
    capping at 100. Re-verified live: the same 3000-vs-5000 scenario now
    correctly leaves the invoice `Partly Paid` at 2000 outstanding, and a
    second bank line for the remainder settles it to `Paid`.
  - **Not yet exercised in a browser**: `tsc`/`vite build` confirm the
    frontend compiles; the live end-to-end verification above is the
    backend write path via `bench console`, not the
    `BankReconciliationPanel`/`MatchInvoiceDialog`/
    `NewBankTransactionDialog` UI itself. Worth a manual click-through
    before relying on it for real reconciliation.

## Related

- Source: `vault/expert-suggestions/accounting-ui-wireframe.md`
- `vault/payments-accounting/features/accounting-reports.md`
- `vault/payments-accounting/features/bank-account-setup.md`
- `vault/payments-accounting/features/rent-roll-and-arrears-report.md`
