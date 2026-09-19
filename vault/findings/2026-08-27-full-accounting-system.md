---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-27
updated: 2026-08-27
related_adr: []
---

# Finding — Full-Fledged Accounting (P&L, Balance Sheet, GL, Journal Entry)

## Method

Investigated live, not assumed: read `MASTERPLAN.md`'s original accounting
section, then checked the actual live site's Chart of Accounts, GL data,
and called ERPNext's own report engines directly (`bench console`) against
real production data — Buildings BLD-00320, Head Lease HL-2026-00368,
Lease LSE-2026-00328/00382, and the real Sales/Purchase Invoice/Payment
Entry activity from this session's testing.

## The one-line answer

**The full accounting engine already exists and is already correctly
posting real double-entry bookkeeping underneath every action this app
takes.** Nothing needs to be built on the accounting-calculation side.
What's actually missing is a way to *see* it without leaving this app's
own portal — today the only way to view a P&L, Balance Sheet, General
Ledger, or post a Journal Entry is Frappe's native Desk UI, which nobody
outside this session has ever used and which visibly breaks this
product's own "never show ERPNext" principle (the same reasoning ADR-0013
just established for the Lead/CRM work).

## What the original plan already assumed

`MASTERPLAN.md` §3.5, written before any of this was built: **"Out-of-box:
ERPNext Accounting Module (GL, P&L, Balance Sheet)."** Only the
per-building P&L ("Building Profitability") was ever scoped as custom
work — the plan's own assumption was that a business user reaching for a
company-wide P&L or Balance Sheet would use ERPNext's native reports
directly. That assumption predates this app's evolution into a fully
custom, ERPNext-invisible portal (ADR-0006/0007, then explicitly reaffirmed
for Leads in ADR-0013). It was never revisited for accounting specifically
— this finding is that revisit.

## What's actually there, verified live

- **Chart of Accounts**: ERPNext's "Standard" template, 81 accounts,
  correctly structured across all 5 root types (Asset: 27, Liability: 13,
  Equity: 6, Income: 5, Expense: 30). Fiscal Year 2026 configured.
- **Real GL data already posting correctly**: 10 GL Entries exist right
  now from this session's real test transactions (a Sales Invoice, a
  Purchase Invoice, a Payment Entry) — every one double-entry balanced.
  Confirmed via the Trial Balance report itself: total debit = total
  credit = QAR 8,000. `Cost Center` is correctly tagged on the
  income/expense lines (`BLD-00320 - Rastec 20 - RRE`), which matters
  below.
- **`Journal Entry`: zero, ever.** No manual entry has been made — and
  there's no way to make one from this portal today. A growing business
  will need this soon: opening balances, owner's capital contribution,
  accruals, write-offs, corrections.
- **ERPNext's own report engines work correctly when called directly**,
  confirmed by actually running them against real data, not just checking
  they import:
  - `general_ledger.execute(filters)` and `trial_balance.execute(filters)`
    — simple `(columns, data)` return, both returned complete, correct
    real rows immediately (account, debit, credit, running balance,
    voucher references, cost center, party — everything a GL viewer needs).
  - `profit_and_loss_statement.execute(filters)` and
    `balance_sheet.execute(filters)` — **also work**, but return a richer
    6-tuple (`columns, data, message, chart, report_summary, net_result`),
    not the naive 2-tuple. Confirmed this by inspecting the actual return
    value after a first naive-unpack attempt raised `ValueError`. Real P&L
    output for this site right now: Income 2,000 / Expense 3,000 / Net
    -1,000 for the year to date — correct, and matches what
    `get_cash_flow_projection` already independently computed from the
    schedule side.

## Two real gaps found, not assumed

1. **Account naming doesn't fit a real-estate business.** The Standard
   CoA's generic names are being used as-is: the Company's
   `default_expense_account` — the account every landlord rent payment
   posts against — is literally **"Cost of Goods Sold - RRE"**, a
   manufacturing/retail term with no meaning to a property operator.
   Income posts to **"Sales - RRE"**. A P&L or Balance Sheet shown to
   Imran or a future business owner today would read confusingly even
   though the numbers underneath are completely correct. This is a
   labeling problem, not a data problem — fixable by renaming/restructuring
   the relevant accounts (e.g. "Rental Income", "Head Lease Rent Expense"),
   not by touching any transaction logic.
2. **The Balance Sheet will look wrong for a brand-new company until this
   is addressed**: verified live, it currently shows Total Assets as
   *negative* QAR 1,000 with zero Liability, because no owner's-equity /
   capital contribution has ever been posted — cash has only ever gone
   *out* (a landlord payment) with nothing coming in as a bank deposit yet.
   This is realistic double-entry behavior given the data, not a bug, but
   it means a Balance Sheet page shipped today would confuse a first-time
   viewer unless there's guidance (or a real opening-balance Journal
   Entry) alongside it.

## Where this connects to still-open gaps from the 2026-08-24 findings doc

A P&L is only as complete as what actually posts to the GL. Two gaps from
that earlier finding are direct blockers to a *complete* P&L, not just
nice-to-haves:

- **G9 — maintenance cost posts nowhere** (`maintenance-cost-allocation.md`,
  still `planned`). `Maintenance Request.total_cost` is computed but never
  becomes a GL entry. Any Expense report will silently omit real
  maintenance spend until this ships.
- **G10 — security deposits not in the GL as a refundable liability**.
  Held tenant deposits are tracked in the `Security Deposit` doctype but
  never posted to a Liability account. A Balance Sheet will understate
  liabilities until this ships.

Neither blocks *shipping a GL/P&L/Balance Sheet viewer* — the reports will
correctly show whatever *has* posted — but both should be sequenced before
calling the accounting picture "complete," since otherwise the reports
this work adds will look authoritative while quietly missing two real
categories of cost/liability.

## Recommendation

Approved as `vault/decisions/0015-portal-native-accounting-reports.md`:
build custom portal pages (General Ledger, Trial Balance, P&L, Balance
Sheet viewers + a Journal Entry create/list UI) that call ERPNext's own
report engines and Journal Entry doctype on the backend — reusing the
already-correct calculation logic — rather than either (a) giving business
users Frappe Desk access (fast, but breaks the "never show ERPNext"
principle this product has held to everywhere else), or (b) reimplementing
GL aggregation from scratch (reinvents what's already correct, same
mistake the CRM finding warned against for leads).

## Related

- ADR: `vault/decisions/0015-portal-native-accounting-reports.md`
- Finding: `vault/findings/2026-08-24-ideal-product-vs-current-state.md` (G9, G10)
- Feature: `vault/payments-accounting/features/cost-center-per-building.md`,
  `vault/payments-accounting/features/building-profitability-report.md`,
  `vault/maintenance/features/maintenance-cost-allocation.md`
- PRD: `vault/os/REALESTATE_MASTERPLAN.md` §3.5 (formerly `MASTERPLAN.md`)
