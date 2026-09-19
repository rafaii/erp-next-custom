---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-27
updated: 2026-09-03
related_adr: ["0015-portal-native-accounting-reports", "0017-platform-modularization"]
---

# Portal-Native Accounting Reports (GL, Trial Balance, P&L, Balance Sheet, Journal Entry)

## 2026-08-30 update: extracted into `accounts_portal`

`get_general_ledger`/`get_trial_balance`/`get_profit_and_loss`/
`get_balance_sheet` and all of Journal Entry create/list now live in
their own app, [rafaii/accounts-portal](https://github.com/rafaii/accounts-portal)
(private), per ADR-0017 — see
`vault/decisions/draft-adr-accounts-portal-extraction.md` for the full
extraction record, including a real dotted-path bug caught and fixed
same-day post-deploy. `real_estate_os.api.get_report_filter_options`
stays as a thin wrapper merging the new app's generic account list with
a real-estate-specific Building list. Every code reference below that
still says `real_estate_os.api.get_general_ledger` etc. describes where
this design *originated*, not where the code lives today —
`accounts_portal.reports`/`accounts_portal.journal_entry`/
`accounts_portal.bank_accounts` are the current dotted paths.

## Summary

The full accounting engine (Chart of Accounts, GL, Journal Entry,
ERPNext's own P&L/Balance Sheet/Trial Balance/General Ledger report
engines) already exists and already posts correctly underneath every
action this app takes — verified live against real production data
(`vault/findings/2026-08-27-full-accounting-system.md`). Nothing needs
building on the calculation side. What's missing is a way to see any of
it without leaving this app's portal for Frappe Desk.

## Requirements

- General Ledger viewer — filterable by account, cost center (building),
  date range, party.
- Trial Balance viewer.
- Profit & Loss Statement viewer.
- Balance Sheet viewer.
- Journal Entry create/list UI — the only genuinely new capability here;
  nothing today lets staff post a manual entry (opening balances, owner's
  capital, accruals, corrections).
- Default accounts renamed to real-estate-appropriate labels
  (`default_expense_account`/`default_income_account` currently post to
  generic "Cost of Goods Sold"/"Sales") before this ships, so a business
  user never sees ERPNext's generic retail-flavored naming.
- A one-time opening Journal Entry (owner's capital) so the Balance Sheet
  reads sensibly from day one.

## Design

- New whitelisted `real_estate_os` methods that call ERPNext's existing
  report engines directly rather than reimplementing GL aggregation:
  `erpnext.accounts.report.general_ledger.execute()`,
  `.trial_balance.execute()`, `.profit_and_loss_statement.execute()`,
  `.balance_sheet.execute()`. GL/Trial Balance return a simple
  `(columns, data)` tuple; P&L/Balance Sheet return a richer 6-tuple
  (`columns, data, message, chart, report_summary, net_result`) —
  confirmed live, handle accordingly.
- Verify during implementation whether `profit_and_loss_statement`'s
  `cost_center` filter produces a correct per-building P&L directly —
  if so, this may subsume or cross-check the existing custom
  `get_building_profitability`.
- Journal Entry create/list: standard `Journal Entry` doctype
  (`frappe.get_doc({"doctype": "Journal Entry", ...}).insert().submit()`),
  new portal pages (not just a report wrapper).
- Account renaming: Frappe's account rename mechanism (not a raw field
  edit) — needs care on a site with live GL history. Ships as this
  vertical's own Chart-of-Accounts fixture (`PLATFORM_STRATEGY.md` §3),
  applied automatically at install — never a cross-OS shared mapping.

## Implementation Plan

- [x] Rename default accounts (Income/Expense) to real-estate-appropriate
      labels — PR #33: patch `rename_default_income_expense_accounts` uses
      ERPNext's own `update_account_number` rename entry point (validated
      first on a disposable throwaway Account), runs on every install so a
      fresh site gets correct labels automatically, never a one-off manual
      rename. Verified live: `Sales - RRE` → `Rental Income - RRE`,
      `Cost of Goods Sold - RRE` → `Landlord Rent Expense - RRE`, Company
      defaults and all existing GL Entry rows followed the rename
      automatically (Frappe's own Link-rewrite behavior), and a fresh
      throwaway Sales Invoice still posted correctly against the renamed
      income account afterward.
- [x] One-time opening Journal Entry (owner's capital) — posted live via
      `bench console` (data operation, not code — confirmed with Imran
      first: 100,000 QAR, dated 2026-07-01, `is_opening="Yes"` so it reads
      as an opening balance rather than period activity). `ACC-JV-2026-00001`.
      Trial Balance now balances: total closing debit = closing credit =
      108,000.
- [x] General Ledger viewer (portal page + endpoint) — PR #33.
      `api.get_general_ledger` calls `erpnext.accounts.report.
      general_ledger.execute()` directly; portal panel filters by account,
      building (Cost Center), and date range. Verified live: output matches
      a direct `bench console` call with identical filters exactly.
- [x] Trial Balance viewer — PR #33. `api.get_trial_balance` calls
      `erpnext.accounts.report.trial_balance.execute()` directly (resolves
      the Fiscal Year server-side from the date range so the portal only
      deals in dates). Verified live, matches direct call.
- [x] **2026-08-28 fix (PR #35)**: Imran reported `/reports` showed the same
      page as `/accounts`. Root cause: `Accounts` and `Reports` both declare
      `api: "real_estate_os.api.get_accounts_data"` in `portal_nav_items`
      (`Reports`' own overview panel reuses that same data), and the
      route-dispatch check in `Dashboard.tsx` matched on
      `activeItem.api.includes("accounts")` — a substring check that matched
      *both* nav items, so `Reports` never reached its own branch and always
      rendered `AccountsView` instead. The GL/TB work above was correct but
      genuinely unreachable through the UI until this fix. Switched all three
      dispatch checks (Settings/Reports/Accounts) to match on
      `activeItem.label` instead of the shared `api` string. Deployed;
      confirmed `/reports` now serves a 200 (redirects to `/login` only when
      unauthenticated, as expected).
- [x] **2026-08-28 fix (PR #36)**: Imran flagged the GL table as too wide
      (ERPNext's report returns 15 columns — voucher subtype, party type,
      against-voucher type/no, bill no, etc. — accurate for Desk, noisy for
      a portal view) and asked whether an 8,000 "Total closing balance"
      with no account filter was correct. Curated the default GL view down
      to 9 columns (date, account, debit, credit, balance, voucher
      type/no, against, party name). Verified the 8,000 figure directly
      against `erpnext.accounts.report.general_ledger.execute()`: it's
      correct, not a bug — with no Account filter, Total/Closing always
      show equal debit=credit (that's what a balanced ledger means, not a
      net balance), and Opening only computes a real balance once scoped
      to a single Account (confirmed: filtering to `Cash - RRE` alone
      correctly shows the 100,000 Opening balance and a 97,000 running
      balance). Added an inline note in the UI explaining this instead of
      letting it look like an error.
- [x] Profit & Loss Statement viewer — PR #37. `api.get_profit_and_loss`
      calls `erpnext.accounts.report.profit_and_loss_statement.execute()`
      with `periodicity="Yearly"` + `filter_based_on="Date Range"`,
      collapsing to a single "Total" column spanning the whole selected
      range (relabelled from its default period-name label, e.g. "2026",
      which is misleading for a partial-year range — verified live).
      Resolved the open question: **yes**, the `cost_center` filter
      produces a correct per-building P&L — verified live that it scopes
      both the income and expense sides together, matching
      `get_building_profitability`'s numbers for the same building/range.
      Optional Building filter added to the portal panel; `SummaryCards`
      surface the report engine's own Total Income/Expense/Profit numbers
      above the table.
- [x] Balance Sheet viewer — PR #37. `api.get_balance_sheet`, same
      collapsed-single-period approach as P&L. No Building filter — Balance
      Sheet accounts (Cash, Debtors, Equity) aren't building-attributed the
      way invoice lines are, so a per-building slice isn't meaningful.
      Verified live: balances correctly (99,000 Total Asset = 100,000
      Total Equity - 1,000 Provisional Profit/Loss).
- [x] **2026-08-29 fix (PR #39)**: Imran pointed out account names in every
      report carry a pointless " - RRE" suffix, since every tenant is its
      own Frappe site (ADR-0003) with exactly one Company — the suffix
      exists to disambiguate accounts across multiple Companies sharing one
      site, which never applies here. Stripped for display only in
      `_clean_account_columns`, scoped to columns each report itself types
      as `options: "Account"` (not a string-shape guess), so the real
      `Account.name` and every Link field pointing at it stay untouched.
      Verified live: GL/TB/P&L/BS now show "Cash" instead of "Cash - RRE".
- [x] Journal Entry create/list UI — PR #46, 2026-08-29. New
      `payments/journal_entry.py`: `get_journal_entries`,
      `create_journal_entry`, `submit_journal_entry`,
      `cancel_journal_entry`, `delete_draft_journal_entry`. Draft-then-
      submit, not insert-and-submit — a submitted Journal Entry has no
      undo, the same one-way-door class as `ACC-PAY-2026-00001`'s frozen
      Cash posting, so `create_journal_entry` only ever inserts a Draft
      and Submit/Cancel/Delete-draft are separate explicit actions.
      Shipped as a 5th card on the Reports page next to GL/TB/P&L/BS, not
      a new top-level nav item. The account picker excludes
      Receivable/Payable accounts (Debtors/Creditors/Employee Advances on
      this site) since those need a `party_type`/`party` this generic
      form doesn't collect — confirmed the live `account_type` strings
      before wiring the filter, rather than guessing. New patch grants
      Accountant explicit `submit`/`cancel` on Journal Entry — ADR-0018's
      original grant only covered read/write/create/delete, which would
      have let an Accountant create a Draft but never post or undo one.
      **Verified live end-to-end 2026-08-29** through the actual
      whitelisted endpoints (not just `frappe.get_doc`): created a small
      balanced test entry (`ACC-JV-2026-00002`, Cash/Rental Income, 10
      QAR) as Draft, submitted it, confirmed it appeared in
      `api.get_general_ledger`'s own output (the same endpoint the portal
      calls), then cancelled and force-deleted it and confirmed in a
      fresh process that Cash's active (`is_cancelled=0`) balance was
      back to exactly its prior 99,000. **Not verified through an actual
      browser** — this session has no browser-automation tool, so the
      dialog's rendering, button wiring, and client-side balance check
      were reviewed in code but not clicked through; worth a quick manual
      check next time Imran is in the portal.

## Acceptance Criteria

- [x] No account name in the portal reads as generic ERPNext/retail
      terminology (e.g. "Cost of Goods Sold") — both default accounts
      renamed and verified live (see above)
- [x] GL/Trial Balance/P&L/Balance Sheet viewers show real, correct data
      matching a direct `bench console` call to the same ERPNext report
      function — verified live 2026-08-28 for all four
- [x] A manual Journal Entry can be created and appears correctly in
      subsequent reports — verified live 2026-08-29 through the actual
      `create_journal_entry`/`submit_journal_entry` endpoints and
      `api.get_general_ledger`'s own output (see Implementation Plan for
      the full test and cleanup). Browser click-through not yet done —
      see the note there.

## 2026-09-03 update: collapsible P&L tree table

The P&L statement viewer previously rendered a flat table of all accounts
expanded, which was noisy for non-accountant users. Replaced with a
collapsible tree table:
- Top-level groups (Income, Expenses, Profit for the year) visible by default
- Chevron toggle on group rows to drill into sub-accounts
- Hierarchy inferred from ERPNext's leading-space indentation in `account` names
- Backend unchanged — pure frontend change in `Dashboard.tsx`
- Build verified, merged to `origin/main` via `agent/developer-1/collapsible-pnl`

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `vault/decisions/0015-portal-native-accounting-reports.md`
- Finding: `vault/findings/2026-08-27-full-accounting-system.md`
- Feature: `building-profitability-report.md`, `cost-center-per-building.md`
