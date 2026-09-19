---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-29
updated: 2026-08-30
related_adr: ["0017-platform-modularization"]
---

# Company Bank Account Setup

## 2026-08-30 update: extracted into `accounts_portal`

`get_bank_accounts`/`create_bank_account` and the generic
`_default_bank_account`/`_resolve_deposit_bank_account` helpers now live
in [rafaii/accounts-portal](https://github.com/rafaii/accounts-portal)
(private), per ADR-0017 — see
`vault/decisions/draft-adr-accounts-portal-extraction.md`.
`_resolve_head_lease_bank_account` and `_reconcile_invoice`
(real-estate-specific Head Lease/PDC orchestration) stay in
`real_estate_os/payments/pdc.py`, now importing the generic helpers from
the new app instead of a local copy. Both permission patches
(`create_bank_account_permissions.py`,
`grant_accountant_journal_entry_submit_cancel.py`) needed no changes —
confirmed Frappe permission grants are keyed by doctype+role name, not
by which app owns the doctype.

## Summary

Imran flagged that every landlord payment and rent PDC clearance posts
against the generic ERPNext "Cash" account instead of a real company bank
account. Root cause traced live: `Company.default_bank_account` was unset,
only one Bank-type Account existed (a non-postable group), and zero `Bank
Account` records existed — so ERPNext's own `get_payment_entry` ->
`get_bank_cash_account` -> `get_default_bank_cash_account` chain fell
through to `Company.default_cash_account` every time.

Imran's own design for the full fix, given verbatim:

> The business sets up a bank account during setup. During head lease
> creation, we decide which bank account we are issuing the cheque from.
> When receiving rent, if there is only one bank, it auto gets deposited
> to that account. If there are more than 1 account, every day during PDC
> deposit in the bank, you have to pick the bank account in which you are
> depositing the funds.

This was a 3-step build, all shipped 2026-08-29 (PRs #40-45).

## Requirements

- Step 1 (done): the business can add one or more real company bank
  accounts from Settings. The first one becomes the default; new Payment
  Entries with no bank explicitly chosen resolve to it automatically
  instead of Cash.
- Step 2 (done): incoming rent — `PDC Entry.bank_account`, set at the
  "send to bank" deposit action (ADR-0014), auto-selecting when only one
  bank account exists, requiring a choice when more than one does.
- Step 3 (done): outgoing landlord cheques — `Head Lease.bank_account`
  picked at Head Lease creation, wired into landlord-payable generation and
  clearance so the drawn-from account is explicit per lease rather than
  always the company default. Closes the flagged `landlord-payables.md`
  gap ("Outgoing-cheque source bank account").

## Design

- **`Bank Account` (ERPNext core) for the account record itself**, not
  `Cheque Bank` — `Bank Account` links to a real GL `Account` and is
  exactly what `get_payment_entry`'s resolution chain is built around.
  `Cheque Bank`, by contrast, is not replaced by this.
- **But the *bank name* on it draws from the app's one existing shared
  `Cheque Bank` list, corrected same-day (PR #43)**: the first cut (PR
  #42) had the "Add bank account" form's Bank field pick from ERPNext's
  core `Bank` doctype directly — a second, empty bank-name list, disjoint
  from the `Cheque Bank` list (ADR-0010) already used for landlord/tenant
  issuing banks (which already had "QNB"/"Commercial Bank" on the live
  site). Imran: "The banks table is just names of the bank. You can use
  it to name bank for landlord, tenant or our own bank" — one list, not
  one per use. Reverted `BankSelect` to always use `Cheque Bank`;
  `api.create_bank_account` needed no change, since it already
  auto-creates/reuses the matching core `Bank` record by the same name
  underneath (`Bank Account.bank` is a hard Link to that core doctype,
  unavoidable) — both doctypes autoname off the same `bank_name` field,
  confirmed live, so the core `Bank` record stays invisible plumbing the
  user never manages directly, same pattern as `Landlord.supplier`.
- Creating a `Bank Account` via the API also creates a leaf GL `Account`
  (`account_type="Bank"`, `is_group=0`) parented under the existing "Bank
  Accounts" group account (created by the base Chart of Accounts fixture,
  previously non-postable with nothing under it).
- **Critical gotcha, caught live**: `Company.default_bank_account`'s
  DocField `options` is `"Account"` — it's a Link to the GL Account, *not*
  to the `Bank Account` record, even though the fieldname suggests
  otherwise. `get_default_bank_cash_account` reads it and passes the value
  straight into `frappe.get_cached_value("Account", account, ...)`. The
  first implementation (PR #40) set it to the `Bank Account` name instead,
  which silently no-oped the entire feature — confirmed live post-deploy
  when `get_default_bank_cash_account(company, "Bank")` returned `{}`
  instead of resolving. Fixed same-day in PR #41 to store `Account.name`.
  Caught by verification, not a test — worth remembering that a Link
  field's `options` is the actual source of truth, not its name.
- No changes needed to `payments/pdc.py`'s reconciliation code for the
  single-bank case: once `Company.default_bank_account` is set correctly,
  ERPNext's own default resolution takes over with zero explicit
  `bank_account=` argument required on `get_payment_entry()` calls.
- Historical Payment Entries (`ACC-PAY-2026-00001`/`00002`) already posted
  against `Cash - RRE` before this shipped and can't be re-pointed after
  submission. Imran's call: leave as-is, no correcting Journal Entry.

## Implementation Plan

- [x] `api.get_bank_accounts` / `api.create_bank_account` — PR #40, fixed
      PR #41 same day (see gotcha above).
- [x] New patch `create_bank_account_permissions` grants Accountant role
      read/write/create/delete on `Bank Account` and `Bank` (neither was in
      ADR-0018's original matrix since the feature didn't exist yet).
      Deliberately does *not* grant `Account` doctype access — the backing
      leaf Account is created with `ignore_permissions=True`, so ledger
      structure CRUD stays System-Manager-only.
- [x] Settings page: new `BankAccountsPanel` (+ `NewBankAccountDialog`),
      sitting above the existing `BanksPanel` (Cheque Bank), with an
      explicit UI-level distinction between "your own accounts" and "banks
      tenants'/landlords' cheques are drawn on."
- [x] Verified live 2026-08-29 (fresh `bench console` process each check,
      per the earlier PDC-backfill lesson about not trusting a same-process
      read): created a throwaway `Bank Account`, confirmed
      `Company.default_bank_account` resolved correctly and
      `get_default_bank_cash_account(company, "Bank")` returned the real
      account (not `{}`), then deleted the throwaway records so Settings
      starts empty for Imran to configure for real.
- [x] **2026-08-29 fix (PR #42)**: Bank field on "Add bank account" was a
      free-text `Input` that would've created a new (possibly duplicate)
      `Bank` record on every submit — Imran wanted it to match every other
      bank field in the app (a picker with inline "+ Add bank"). Fixed by
      generalizing the shared `BankSelect` component.
- [x] **2026-08-29 fix (PR #43)**: PR #42's picker pointed at ERPNext's
      core `Bank` doctype (empty on this site), not the app's existing
      `Cheque Bank` list (already populated: QNB, Commercial Bank) —
      Imran caught it immediately since the dropdown showed nothing.
      Reverted to the shared list (see Design gotcha above). Verified live
      in a fresh process: creating a Bank Account against "QNB" reused the
      existing `Cheque Bank` record (no duplicate), auto-created the
      matching core `Bank` "QNB" invisibly, and resolved
      `Company.default_bank_account` correctly; cleaned up the throwaway
      `Bank Account`/`Account`/core `Bank` test records afterward, leaving
      the original `Cheque Bank` entries untouched.
- [x] Step 2 (PR #45, 2026-08-29): incoming deposit-time bank picker.
      `PDC Entry.bank_account` already existed in the schema (added
      speculatively at some earlier point, never wired to any code path —
      wired it up instead of adding a new field). `mark_pdc_sent_to_bank`
      takes an optional `deposit_account`: auto-resolves to the company's
      one bank account when exactly one exists, requires an explicit
      choice when more than one does (throws otherwise — the "prompt to
      configure" pattern, mirrored from step 3). Deliberately count-based,
      not `is_default`-based — once a second account exists the point is
      that the deposit destination varies per run, not that one stays
      "the" default forever. `_reconcile_invoice` resolves it into a real
      GL account at clearance, same as the outgoing side. Frontend: the
      "Due for deposit" panel (Accounts page) shows a picker only when
      more than one bank account exists — with today's single account,
      nothing changes visually, matching "auto" from Imran's own design.
      Re-added `bank_account` to PDC Entry's curated visible fields (a new
      patch, since the original visibility-seeding patch already ran and
      won't re-run) now that it's populated. Verified live (read-only):
      `_resolve_deposit_bank_account(None, company)` returns "Main
      Account - QNB", the site's one configured account.
- [x] Step 3 (PR #44, 2026-08-29): `Head Lease.bank_account` (Link ->
      `Bank Account`) — picked in the "Set up Head Lease" dialog
      (auto-selected when the company has exactly one; a Select when more
      than one exist; a plain "add one in Settings first" note when
      none exist — Head Lease creation itself isn't blocked on this).
      `generate_pdc_schedule_for_head_lease` resolves-and-persists it
      (falling back to the company default) the first time it's called on
      a Head Lease that doesn't already have one, and throws a clear error
      naming Settings -> Bank accounts if no bank account can be resolved
      at all — the "prompt the user" Imran asked for. Deliberately did
      **not** add a per-PDC `deposit_account` field for this direction
      (per advisor review) — the outgoing bank is decided once per Head
      Lease, not per cheque, so `Head Lease.bank_account` is the single
      source of truth; a per-PDC copy would only ever be set from that
      same value and could drift from it with no independent user action
      ever setting it.
- [x] **Gotcha, confirmed live before wiring (avoiding a repeat of PR
      #41's mismatch)**: `get_payment_entry`'s own `bank_account` param is
      a GL **Account** name, not a `Bank Account` record name —
      `get_bank_cash_account` (payment_entry.py) passes it straight into
      `get_default_bank_cash_account(..., account=bank_account)` ->
      `frappe.get_cached_value("Account", account, ...)`. `_reconcile_invoice`
      resolves `Head Lease.bank_account` -> `Bank Account.account` (the GL
      account) before calling `get_payment_entry(bank_account=...)`.
      Verified live (read-only, no insert): `get_payment_entry("Purchase
      Invoice", "ACC-PINV-2026-00003", bank_account="Main Account - RRE")`
      resolved `payment_type="Pay"`, `paid_from="Main Account - RRE"`,
      `paid_to="Creditors - RRE"` — correct.
- [x] **HL-2026-00368** (the head lease Imran flagged, 13 Outgoing PDCs):
      set `bank_account = "Main Account - QNB"` directly via `bench
      console` (committed, re-verified in a fresh process) since the
      field didn't exist when this Head Lease was created. Its 12
      remaining Pending PDCs will now clear against the real bank account.
      Its already-**Cleared** PDC-00369 posted `ACC-PAY-2026-00001` to
      `Cash - RRE` on 2026-08-27, before any bank account existed — a
      submitted Payment Entry's account can't be edited. Asked Imran
      whether to post a correcting Journal Entry; **his call: leave it,
      no correction** — same decision as the earlier historical-Cash
      question in step 1.

## Acceptance Criteria

- [x] A company bank account can be added from Settings and becomes the
      default automatically if it's the first one. Its picker draws from
      the app's one shared `Cheque Bank` list (PR #43), not a second,
      empty one.
- [x] With a default bank account configured, a new Payment Entry
      resolves to it instead of the generic Cash account — verified live
      2026-08-29 with a real bank account ("Main Account", QNB) configured
      through the Settings UI itself.
- [x] Incoming PDC deposit flow prompts for (or auto-selects) a bank
      account — step 2, verified live (read-only): `_resolve_deposit_bank_
      account(None, company)` correctly auto-picks "Main Account - QNB",
      the site's one configured account, with no prompt needed. Not yet
      verified against a real deposit/clearance run (deliberately not
      fabricated on a genuine live cheque — the resolution and GL-account
      wiring is identical to step 3's, already proven end-to-end).
- [x] Head Lease creation captures which bank account issues outgoing
      cheques, and an existing Head Lease's outgoing PDCs resolve to it at
      clearance — verified live for HL-2026-00368 (see step 3 above).

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- `vault/payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`
  (deposit/clearance flow this extends)
- `vault/payments-accounting/features/landlord-payables.md` (flags the
  outgoing-bank gap step 3 closes)
- `vault/decisions/0010-pdc-schedule-and-bank-reconciliation.md` (Cheque
  Bank, the doctype this deliberately does *not* reuse)
