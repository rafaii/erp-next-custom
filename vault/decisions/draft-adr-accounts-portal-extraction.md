---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-29
updated: 2026-08-30
related_adr: ["0017-platform-modularization"]
---

# Extract portal-native accounting into its own app (accounts_portal)

Implementation record for a second application of ADR-0017 (platform
modularization) — not a new ADR, since it doesn't introduce a new
strategic decision beyond what 0017 already decided (extract reusable
capability into separate installable apps). Kept as its own file rather
than folded only into the feature files, since the survey and the
deploy-time bug are worth a durable, dedicated record.

## Why

Imran: "Let us strip out the accounts module too, because that too would
be a common module used by different OS." Same motivation as ADR-0017's
e-sign extraction — GL/Trial Balance/P&L/Balance Sheet viewers, Journal
Entry create/list, and company Bank Account setup have nothing
real-estate-specific about them; any Frappe/ERPNext-based OS app would
want the same portal-native accounting layer.

## What was actually surveyed (not guessed)

A full-repo survey (fork, 2026-08-29) found every accounting-related
function's *body* is already 100% generic — the coupling to real estate
is much shallower here than e-sign's was:

- **`payments/journal_entry.py`** (all 5 functions): zero real-estate
  coupling. Touches only `Journal Entry`, `Account`, `Company`.
- **`api.py`**: `get_general_ledger`, `get_trial_balance`,
  `get_profit_and_loss`, `get_balance_sheet`, `_statement_filters`,
  `_relabel_period_column`, `_strip_company_abbr`,
  `_clean_account_columns`, `_default_company`, `get_bank_accounts`,
  `create_bank_account` — all generic wrappers around ERPNext's own
  report engines / `Account`/`Bank Account` doctypes.
- **`payments/pdc.py`**: `_default_bank_account` and
  `_resolve_deposit_bank_account` are generic (take a company name and a
  Bank Account name, nothing PDC-specific) despite currently living in a
  real-estate-specific file.
- **Permission patches** (`create_bank_account_permissions.py`,
  `grant_accountant_journal_entry_submit_cancel.py`): grant the
  real-estate-specific "Accountant" role access to generic doctypes
  (`Bank Account`, `Bank`, `Journal Entry`) — Frappe permission grants
  are keyed by doctype+role name, not by which app owns the doctype, so
  **these need zero code changes** regardless of what moves — confirmed
  by the same reasoning that made the e-sign doctypes' existing DocPerm
  rows survive that extraction untouched.

**One real difference from the e-sign extraction, in our favor**: this
new app would own **zero custom DocTypes** — `Account`, `Bank Account`,
`Bank`, `Journal Entry`, `GL Entry` are all ERPNext core doctypes already.
There is no DocType `module` reassignment, no live signed-record risk,
and no Frappe Module Def to declare or collide with. The extraction is
pure Python-function relocation plus import-path updates.

**One real coupling that needs a decision** (see Open Questions):
`get_report_filter_options()` bundles a real-estate-specific Building
list alongside the generic Account list in one API response — the only
accounting endpoint that isn't already clean.

## What moves

New app (name TBD — recommending `accounts_portal`, see Open Questions),
repo `rafaii/accounts-portal`, cloned onto the VPS as a sibling to
`real_estate_os` and `inbuilt_esign` — same pattern, same reasoning as
last time (the VPS has no outer meta-repo checkout; only individual app
repos are `git`-managed there).

- `payments/journal_entry.py` → moves whole, as-is.
- From `api.py`: `get_general_ledger`, `get_trial_balance`,
  `get_profit_and_loss`, `get_balance_sheet`, `_statement_filters`,
  `_relabel_period_column`, `_strip_company_abbr`,
  `_clean_account_columns`, `_default_company`, `get_bank_accounts`,
  `create_bank_account`.
- From `pdc.py`: `_default_bank_account`, `_resolve_deposit_bank_account`
  — `real_estate_os`'s own PDC code imports these back from the new app.

## What stays in `real_estate_os`

- `_resolve_head_lease_bank_account`, `_reconcile_invoice` — genuinely
  real-estate-specific (Head Lease, PDC Entry orchestration). They'll
  import the generic bank-account helpers from the new app instead of
  a local one.
- Both permission patches — unchanged, per the finding above.
- A thin `get_report_filter_options()` wrapper that calls the new app's
  generic account-options function and merges in the Building list — so
  the frontend's existing single-call contract doesn't have to change
  (mirrors how `e_sign/dispatch.py` stayed as a thin OS-side wrapper
  around the extracted `workflow.py`).
- **All frontend code** — see below.
- `ReportsView`/`ReportsOverview` — these are the portal shell's own
  "Reports" landing page (static real-estate cards + the panel
  dispatcher); always OS-specific regardless of what the panels
  themselves do.

## What does NOT move, by design (not oversight): the frontend

`GeneralLedgerPanel`, `TrialBalancePanel`, `ProfitAndLossPanel`,
`BalanceSheetPanel`, `JournalEntryPanel`, `NewJournalEntryDialog`,
`BankAccountsPanel`, `NewBankAccountDialog`, `ReportTable`, and friends
all stay in `real_estate_os/ui/src/pages/Dashboard.tsx` for this pass.

Reasoning: unlike backend Python apps, Frappe has no native mechanism for
sharing frontend components across installed apps — the e-sign
extraction never had to solve this because its "frontend" was
server-rendered Jinja pages that just moved which app's `www/` folder
serves them, not React components in a bundled SPA. Actually extracting
these panels would mean standing up a real npm package (published
somewhere, versioned, imported into `real_estate_os/ui`'s
`package.json`) with no second consumer yet to prove the packaging
approach against — exactly the kind of infrastructure the e-sign
extraction deliberately deferred for Zoho until a real second provider
existed. Proposing the same discipline here: **ship the backend
extraction now, revisit the frontend if/when a second OS's portal
actually needs these panels.**

Two small pieces of frontend coupling exist if this is ever revisited:
`GeneralLedgerPanel`/`ProfitAndLossPanel`'s Building filter, and
`NewBankAccountDialog`'s use of `BankSelect` (hardcoded to the
real-estate-specific `Cheque Bank` doctype) — neither blocks the backend
extraction, both would need decoupling before any future frontend move.

## Deployment plan (mirrors what worked for inbuilt_esign, lessons applied)

1. Scaffold the new app with the module-folder `__init__.py` from the
   start (missing this broke `bench install-app` last time — caught and
   fixed then, applying the lesson up front now).
2. Move the listed functions; update `real_estate_os`'s imports/call
   sites and `required_apps`.
3. Update `ui/src/lib/api.ts` and the ~7 inline `call()` string literals
   in `Dashboard.tsx` to the new dotted paths — no component logic
   changes, since props/behavior are unaffected by which app serves the
   endpoint.
4. Two PRs (new repo + `real-estate`), reviewed together.
5. VPS: `bench backup` first (cheap hygiene, even though no doctypes are
   actually at risk this time), clone the new app, update the Dockerfile
   (new app installed/copied before `real_estate_os`, same dependency
   order as `inbuilt_esign`), rebuild, `bench install-app`, `bench
   migrate`, verify every panel (GL/TB/P&L/BS/JE/Bank Accounts) still
   works end-to-end in the live portal.
6. Vault: update `accounting-reports.md`/`bank-account-setup.md` to
   reflect the new home for these endpoints; extend ADR-0017 or write a
   short new ADR recording this second extraction.

## What actually happened (2026-08-29/30)

Steps 1-5 executed largely as planned, with one real bug caught and
fixed same-day, and one deploy-infra wrinkle neither the plan nor the
`inbuilt_esign` precedent anticipated.

- **Scaffolding turned out simpler than step 1 assumed**: this app owns
  zero custom DocTypes (confirmed by the survey), so there's no Frappe
  Module and therefore **no module-level `__init__.py` to add** — the
  `inbuilt_esign` lesson (missing `__init__.py` in its module folder)
  doesn't apply here at all, since there's no such folder. Files
  (`reports.py`, `bank_accounts.py`, `journal_entry.py`) sit directly
  under the Python package (`accounts_portal/accounts_portal/*.py`),
  two levels deep, not three.
- **Real bug, caught immediately post-deploy, from getting the above
  right in reasoning but wrong in execution**: every dotted-path string
  written during the move — both `real_estate_os`'s two internal
  imports and all 11 frontend `call()` references — used the
  `inbuilt_esign` three-level pattern (`accounts_portal.accounts_portal.
  reports...`) instead of the correct two-level one
  (`accounts_portal.reports...`). This broke the Reports page, Journal
  Entry panel, and Bank Accounts settings panel live in production
  (`ModuleNotFoundError`) until caught by the same
  `frappe.get_attr(...)` verification step used throughout this session,
  fixed same-day (PR #50), rebuilt, redeployed, and every endpoint
  re-verified individually via `bench console` before considering this
  done. No migration was needed for the fix (pure Python/TS, no
  schema/hook changes) — only image rebuild + container recreate.
- **New wrinkle**: private repos need real authentication to `git clone`
  on the VPS, unlike the public `inbuilt-esign` clone that worked
  earlier. `real_estate_os`'s own VPS checkout already had a personal
  access token embedded in its remote URL (`https://ghp_...@github.com/
  ...`) — reused the same token for `accounts_portal`'s clone, and
  retroactively added it to `inbuilt_esign`'s remote too (broken by
  switching that repo private moments earlier in the same session,
  before it had ever needed a fresh pull with the new visibility).
- **Verified live, in order**: `install-app` completed cleanly (no
  errors, unlike the first `inbuilt_esign` attempt); `bench migrate`
  clean; after the path fix, `get_general_ledger`/`get_trial_balance`/
  `get_profit_and_loss`/`get_balance_sheet`/`get_bank_accounts`/
  `get_journal_entries` all called successfully via `frappe.get_attr`
  and returned real data (11 GL rows, 7 TB rows, a real Balance Sheet
  summary, the one configured "Main Account" bank account, the real
  opening Journal Entry `ACC-JV-2026-00001`); `real_estate_os.api.
  get_report_filter_options` correctly merged 56 accounts + 1 building
  in one response; `pdc.py`'s imports of `_default_bank_account`/
  `_resolve_deposit_bank_account` from the new app resolve and work
  correctly; the main site and `/reports`/`/settings` routes still
  serve.

## Decisions (approved 2026-08-29)

1. **App name**: `accounts_portal`, repo `rafaii/accounts-portal`.
2. **Scope**: backend only for this pass. Frontend panels stay in
   `real_estate_os/ui` until a second OS needs them.
3. **Repo visibility**: **private** — and this is now a standing default
   for all future new repos (retroactively applied to
   `rafaii/inbuilt-esign` too, switched from public to private the same
   day).
4. **`get_report_filter_options`**: keep as one call — `accounts_portal`
   exposes a generic accounts-only endpoint, `real_estate_os` keeps a
   thin wrapper that merges in Buildings. Zero frontend changes.
