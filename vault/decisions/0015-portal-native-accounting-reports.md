# ADR-0015: Portal-Native Accounting Reports (GL, Trial Balance, P&L, Balance Sheet, Journal Entry)

- **Status**: accepted
- **Date**: 2026-08-27
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Imran asked for a full-fledged accounting system — proper P&L, Balance
Sheet, Journal Entry, and General Ledger support — noting the current
Accounts page looks basic, and asked whether ERPNext's underlying data
already supports this.

See `vault/findings/2026-08-27-full-accounting-system.md` for the full
investigation. Short version: **the accounting engine and data already
exist and are already correct** — verified live by calling ERPNext's own
`general_ledger`, `trial_balance`, `profit_and_loss_statement`, and
`balance_sheet` report functions directly against real production data;
all four returned correct, complete output. `MASTERPLAN.md` §3.5 always
assumed this would be "out-of-box ERPNext." What's missing is a way to
*see* any of it without leaving this app's portal for Frappe Desk — which
this product has deliberately never asked a business user to do anywhere
else (ADR-0006/0007's headless-hybrid approach, ADR-0013's explicit "never
reference ERPNext in the frontend").

Two real gaps found alongside this: the default Chart of Accounts uses
generic/retail naming (landlord rent expense posts to an account literally
called "Cost of Goods Sold"), and no owner's-equity/capital entry has ever
been posted, so a Balance Sheet would read confusingly today even though
the numbers are correct.

## Decision

Build new portal pages, backed by new whitelisted `real_estate_os`
methods that **call ERPNext's existing report engines and doctypes**
rather than reimplementing GL aggregation — the same "extend, don't
rebuild" pattern already used for Customer, Supplier, Cost Center, and
(per ADR-0013) Lead:

- **General Ledger viewer** — filterable by account, cost center
  (building), date range, party. Backed by
  `erpnext.accounts.report.general_ledger.execute()`.
- **Trial Balance** — backed by
  `erpnext.accounts.report.trial_balance.execute()`.
- **Profit & Loss Statement** — backed by
  `erpnext.accounts.report.profit_and_loss_statement.execute()`. Verify
  during implementation whether its `cost_center` filter produces a
  correct per-building P&L directly from the native engine — if so, this
  may subsume or cross-check the existing custom `get_building_profitability`
  (which reimplements a narrower version of the same thing via raw SQL).
- **Balance Sheet** — backed by
  `erpnext.accounts.report.balance_sheet.execute()`.
- **Journal Entry** — new create/list UI (genuinely new portal surface —
  nothing today lets staff post a manual entry). Needed for opening
  balances, owner's capital, accruals, and corrections. Uses the standard
  `Journal Entry` doctype directly.

**Phasing** (decided): (1) GL + Trial Balance viewers first — smallest
lift, both already return simple output; (2) P&L + Balance Sheet next;
(3) Journal Entry create/list last — the biggest net-new UI, and the one
most worth gating behind a role once RBAC (gap G8) exists.

**Account renaming** (decided): rename the default accounts actually in
use (`default_expense_account`, `default_income_account`) to real-estate-
appropriate labels **before** shipping the P&L/Balance Sheet pages, so a
business user never sees "Cost of Goods Sold." Needs care — Frappe's
account rename mechanism, not a raw field edit — on a site with live GL
history.

**Cross-module risk**: resolved by `0016-module-aware-provisioning` —
each tenant site has its own independent Chart of Accounts (tenant-per-
site, `0003`), so this renaming can never affect a different client or a
future non-real-estate module. A future OS ships its own account-naming
fixture the same way.

**Opening balance**: a one-time opening Journal Entry (owner's capital)
so the Balance Sheet reads sensibly from day one — done once the Journal
Entry UI exists, or sooner via `bench console` if wanted before then.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A — Grant Frappe Desk access for native Accounts reports | Zero new code | Breaks this product's established principle everywhere else (never show ERPNext); jarring UX next to the custom portal | Rejected |
| **B — New portal pages calling ERPNext's existing report engines** | Reuses already-correct, tested calculation logic; consistent with every other integration in this app; stays inside the branded portal | Real new frontend work; P&L/Balance Sheet's 6-tuple return needs handling | **Chosen** |
| C — Reimplement GL aggregation from scratch | Full control | Reinvents fiscal-year/opening-balance/account-hierarchy logic ERPNext already gets right | Rejected |

## Consequences

### Positive

- Real accounting depth without reinventing anything that already works.
- Closes a real inconsistency: every other ERPNext-backed concept in this
  app is wrapped in portal UI; accounting was the one area still
  implicitly assuming Desk access.
- Surfaces (rather than hides) the two real GL-completeness gaps (G9
  maintenance cost, G10 security deposit liability).

### Negative

- Meaningful new frontend scope — four report-viewer pages plus a
  create/list UI for Journal Entry.
- The account-renaming cleanup needs care on a site with live GL history.
- A Journal Entry creation UI is a real capability expansion — worth
  pairing with RBAC (gap G8, unbuilt) so it isn't available to every
  staff user by default.

## Implementation

- Feature file: `vault/payments-accounting/features/accounting-reports.md` (new, on approval)
- Priority and sequencing: `vault/IMPLEMENTATION-PLAN.md`
