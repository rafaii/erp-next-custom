
# Building an Accounting Module on ERPNext: Architecture and UI Blueprint

ERPNext already gives you a solid double-entry engine (Chart of Accounts → Journal Entry → GL Entry → reports), so your job as frontend architect is to expose that power through interfaces an SMB owner (not an accountant) can actually use, while keeping every screen traceable back to a balanced ledger entry. Below is a practitioner's breakdown of what's critical, how to design the core screens, and wireframes you can hand to your dev team.github+1

## What's Non-Negotiable for SMBs

An SMB accounting module fails or succeeds on five things: speed of data entry, error prevention, real-time visibility, auditability, and a smooth link to the operational documents (sales/purchase invoices) your vertical apps generate.frappe+1

|Capability|Why SMBs need it|ERPNext backing|
|---|---|---|
|Chart of Accounts (COA)|Foundation for every report; must support industry-specific templates (preschool vs. auto workshop)|Group vs. Ledger accounts, 5 root types github+1|
|Journal Entry (JE)|Manual override for accruals, corrections, depreciation, payroll|Debit/credit rows, must balance before submit frappe+1|
|GL Entry / Ledger view|Single source of truth; every transaction (invoice, payment, JE) writes here|Auto-generated GL Entry per voucher xero+1|
|Trial Balance|Sanity check before closing books|Sum of all ledger debits/credits per account|
|P&L (Income Statement)|Owners check profitability monthly|Income − Expense accounts over a date range|
|Balance Sheet|Lenders, investors, tax filing require it|Asset = Liability + Equity snapshot|
|Bank Reconciliation|SMBs live and die by cash position|Match bank feed to GL entries|
|Party ledgers (AR/AP aging)|Who owes you, who you owe|Customer/Supplier ledger drill-down|
|Audit trail & submit/cancel workflow|Prevents silent tampering, satisfies auditors|Frappe's submittable doctype states (Draft/Submitted/Cancelled)|
|Multi-entity/company support|Your SaaS is multi-tenant across verticals|Company field on every voucher xero+1|

Two design principles matter more than any single feature: **never let a user create an unbalanced entry**, and **never let them edit a posted (submitted) transaction** — they must reverse or amend it, exactly as ERPNext's submit/cancel model enforces.frappe+1

## Core Screen 1 — Chart of Accounts

Present COA as a collapsible tree with inline balance, not a flat table. Since each vertical (preschool, workshop, real estate) needs a starter COA, ship industry templates that pre-populate group accounts (e.g., "Tuition Income," "Parts Inventory," "Rental Income") on top of the standard Asset/Liability/Equity/Income/Expense roots.github+2

text

`┌─ Chart of Accounts ────────────────────────── [+ New Account] [Import Template ▾] │ 🔍 Search accounts...                                    Company: [Little Stars Preschool ▾] ├──────────────────────────────────────────────────────────────────────────── │ ▾ Assets                                                        $284,500.00 │   ▾ Current Assets                                              $198,200.00 │      • Cash                                        1001         $12,400.00  [•••] │      • Bank - Operating A/c                        1002        $145,800.00  [•••] │      • Accounts Receivable                         1100         $40,000.00  [•••] │   ▸ Fixed Assets                                                 $86,300.00 │ ▸ Liabilities                                                    $52,100.00 │ ▸ Equity                                                        $180,000.00 │ ▸ Income                                                         $95,400.00 │ ▸ Expenses                                                       $61,200.00 └────────────────────────────────────────────────────────────────────────────    [ⓘ] Group accounts (▾/▸) hold no transactions; only leaf (ledger) accounts do.`

Keep "Is Group" locked once a leaf account has any GL entries — this mirrors ERPNext's rule and prevents orphaned balances.[github](https://github.com/frappe/erpnext/blob/develop/erpnext/accounts/doctype/gl_entry/gl_entry.py)

## Core Screen 2 — Journal Entry (the "manual override")

This is the highest-risk screen for user error, so the UI must do three things automatically: keep a running debit/credit balance visible at all times, disable Save until it hits zero, and offer templates for repetitive entries (depreciation, payroll, accruals).frappe+2

text

`┌─ New Journal Entry ───────────────────────────────────────── ACC-JV-2026-00042 │ Entry Type: [Journal Entry ▾]   Posting Date: [29-Aug-2026]   Company: [▾] │ Reference #: __________          Template: [Depreciation - Monthly ▾] ├──────────────────────────────────────────────────────────────────────────── │  Account                 Party        Debit        Credit      Remarks │ ┌────────────────────┬───────────┬───────────┬───────────┬──────────────┐ │ │ Depreciation Exp ▾ │           │  1,200.00 │           │ Aug dep - Van│ │ │ Accum Dep - Vehicle▾│          │           │  1,200.00 │              │ │ │ + Add Row           │           │           │           │              │ │ └────────────────────┴───────────┴───────────┴───────────┴──────────────┘ │                                    Total:      1,200.00      1,200.00 │                                    Difference:              0.00 ✓ Balanced ├──────────────────────────────────────────────────────────────────────────── │ User Remark: [Monthly depreciation - delivery van..............] │                                              [Save Draft] [Submit] [Cancel] └────────────────────────────────────────────────────────────────────────────`

UX rules to bake in: color the "Difference" badge red while nonzero and green at zero; auto-suggest the offsetting account after the first row (ERPNext's "quick entry" pattern needs only ~4 fields); and after Submit, switch the screen to read-only with a prominent "Reverse Entry" action instead of an Edit button.frappe+1

## Core Screen 3 — General Ledger Drill-Down

Every number on P&L, Balance Sheet, or Trial Balance must be clickable straight down to the GL Entry, and from there to the source voucher (Sales Invoice, Payment, JE). This traceability is what makes an SMB owner trust the system.xero+1

text

`┌─ General Ledger ─────────────────────── Account: [Bank - Operating A/c ▾] │ Date range: [01-Aug-2026] to [29-Aug-2026]   Party: [All ▾]   [Export] ├──────────┬────────────────────┬──────────────┬───────────┬───────────┬──────────┐ │ Date     │ Voucher             │ Against       │ Debit     │ Credit    │ Balance  │ ├──────────┼────────────────────┼──────────────┼───────────┼───────────┼──────────┤ │ 03-Aug   │ Sales Invoice SINV-0091│ ABC Motors │ 3,500.00  │           │148,100.00│ │ 07-Aug   │ Payment Entry PE-0044  │ Rent - Aug │           │ 2,300.00  │145,800.00│ │ 12-Aug   │ Journal Entry JV-0042  │ Depreciation│           │ 1,200.00 │144,600.00│ └──────────┴────────────────────┴──────────────┴───────────┴───────────┴──────────┘    Click any row → opens source voucher in side panel (no page reload)`

## Financial Statements: TB, P&L, Balance Sheet

Design these three as one shared "Reports" shell with a common date-range picker, comparison toggle (this period vs. last period / vs. same period last year), and a drill path down to GL — this consistency reduces training time dramatically for SMB owners who check these monthly.

text

`┌─ Trial Balance ──────────────── As of: [29-Aug-2026]  Compare: [☑ Prior Year] ├────────────────────────┬───────────┬───────────┬───────────┬───────────┐ │ Account                │ Debit     │ Credit    │ Prior Debit│Prior Credit│ ├────────────────────────┼───────────┼───────────┼───────────┼───────────┤ │ Cash                   │ 12,400.00 │           │ 9,800.00  │           │ │ Accounts Receivable    │ 40,000.00 │           │ 31,200.00 │           │ │ Accounts Payable       │           │ 18,500.00 │           │ 15,000.00 │ │ ...                                                                    │ ├────────────────────────┼───────────┼───────────┼───────────┼───────────┤ │ TOTAL                  │396,600.00 │396,600.00 │310,400.00 │310,400.00 │ └────────────────────────┴───────────┴───────────┴───────────┴───────────┘                           ✓ Balanced — Debit = Credit ┌─ Profit & Loss ── Period: [Aug 2026 ▾]     View: [Monthly ▾] [Chart 📊] │ Income                                                     Amount │   Tuition Fees Income                                      82,400.00 │   Registration Fee Income                                  13,000.00 │   Total Income                                              95,400.00 │ Expenses │   Salaries & Wages                                          38,200.00 │   Rent Expense                                               9,200.00 │   Depreciation Expense                                       1,200.00 │   Total Expenses                                             61,200.00 │ ──────────────────────────────────────────────────────────────────── │ NET PROFIT                                                    34,200.00   ▲12% └──────────────────────────────────────────────────────────────────────── ┌─ Balance Sheet ── As of: [29-Aug-2026] │ Assets                        │ Liabilities & Equity │  Current Assets   198,200.00  │  Current Liabilities   52,100.00 │  Fixed Assets      86,300.00  │  Equity               180,000.00 │  ─────────────────────────    │  Retained Earnings     52,400.00 │  TOTAL ASSETS     284,500.00  │  TOTAL LIAB+EQUITY    284,500.00   ✓ └────────────────────────────────────────────────────────────────────────`

Give every line item a small bar-chart sparkline toggle and a "%" column (of total income/assets) — SMB owners read percentages faster than raw numbers.

## Linking Vertical Apps to Accounting: Sales/Purchase Invoice

Since each vertical OS (preschool, workshop, real estate) will have its own front-end for "invoice-like" documents (tuition invoice, repair order, rent invoice), the critical architecture decision is: **your vertical UI writes to a normalized invoice doctype that ERPNext's Sales/Purchase Invoice controller consumes, so GL Entries are generated identically regardless of vertical**. Don't let each vertical app write journal entries directly — route everything through Sales Invoice / Purchase Invoice / Payment Entry so tax, AR/AP, and GL logic stay centralized and auditable.xero+1

text

`┌─ Tuition Invoice (Preschool vertical) ───────────────── looks custom to user │ Student: Emma Watson     Class: Toddler A     Month: Sep 2026 │ Items: Tuition Fee  $650   |  Lunch Program  $80 │ Total: $730                          [Save & Post to Accounting] └──────────────────────┬──────────────────────────────────────────                         ▼  (behind the scenes, maps to Sales Invoice) ┌─ Resulting Accounting Entry (auto, view-only for preschool admin) ────────── │ Dr  Accounts Receivable - Emma Watson       730.00 │    Cr  Tuition Fees Income                          650.00 │    Cr  Lunch Program Income                          80.00 └────────────────────────────────────────────────────────────────────────────`

Show this mapping as a collapsed "View Accounting Impact" link on every vertical document — it builds trust without forcing non-accountants to look at debits/credits by default.

## Interaction Patterns That Separate Good from Great

- **Role-based views**: a preschool director sees "Tuition Invoice," never "Sales Invoice"; only your Accounting module (used by the bookkeeper/accountant role) sees the raw GL terminology.
    
- **Inline validation**: block submission with a red banner, not a popup, when debit ≠ credit.
    
- **Keyboard-first entry**: journal entry grids should support Tab-to-next-field and Enter-to-add-row, since bookkeepers batch-enter dozens of lines.
    
- **Reversal, not deletion**: a "Cancel" on a submitted JE should auto-generate a reversing entry dated same-day, keeping history intact, matching ERPNext's cancel semantics.[frappe](https://docs.frappe.io/erpnext/chart-of-accounts)
    
- **Contextual reports**: every account balance shown anywhere (dashboard, invoice, COA) should be clickable to open the GL filtered to that account and date range.
    

I've compiled all the wireframes above (COA tree, Journal Entry, GL drill-down, Trial Balance, P&L, Balance Sheet, and the vertical-to-accounting mapping) into a single reference document for your dev team below.

# Accounting Module — UI Wireframes & Design Spec

## For ERPNext-based Multi-Tenant Vertical SaaS (Preschool / Auto Workshop / Real Estate)

---

## 1. Design Principles

1. Every screen traces back to a balanced double-entry ledger (Debit = Credit).
    
2. Submitted/posted transactions are immutable — reversal or amendment only, never silent edit.
    
3. Vertical apps (tuition invoice, repair order, rent invoice) never write GL entries directly — they map to Sales Invoice / Purchase Invoice / Payment Entry, which ERPNext's controller converts to GL Entries.
    
4. Non-accountant roles (preschool director, workshop manager) see business-friendly labels; the Accounting role sees the underlying ledger terminology.
    
5. Every balance figure anywhere in the product is clickable and drills down to General Ledger → source voucher.
    

---

## 2. Chart of Accounts (Tree View)

text

`┌─ Chart of Accounts ────────────────────────── [+ New Account] [Import Template ▾] │ 🔍 Search accounts...                                    Company: [Little Stars Preschool ▾] ├──────────────────────────────────────────────────────────────────────────── │ ▾ Assets                                                        $284,500.00 │   ▾ Current Assets                                              $198,200.00 │      • Cash                                        1001         $12,400.00  [•••] │      • Bank - Operating A/c                        1002        $145,800.00  [•••] │      • Accounts Receivable                         1100         $40,000.00  [•••] │   ▸ Fixed Assets                                                 $86,300.00 │ ▸ Liabilities                                                    $52,100.00 │ ▸ Equity                                                        $180,000.00 │ ▸ Income                                                         $95,400.00 │ ▸ Expenses                                                       $61,200.00 └────────────────────────────────────────────────────────────────────────────    Group accounts (▾/▸) hold no transactions; only leaf (ledger) accounts do.   "Is Group" is locked once a leaf account has any GL entries.`

Industry templates pre-seed group/ledger accounts per vertical, e.g.:

- Preschool: Tuition Income, Registration Fee Income, Lunch Program Income, Curriculum Supplies Expense
    
- Auto Workshop: Parts Inventory, Labor Income, Warranty Reserve, Shop Supplies Expense
    
- Real Estate: Rental Income, Security Deposit Liability, Property Management Fee Expense, Common Area Maintenance Income
    

---

## 3. Journal Entry (Manual Override Screen)

text

`┌─ New Journal Entry ───────────────────────────────────────── ACC-JV-2026-00042 │ Entry Type: [Journal Entry ▾]   Posting Date: [29-Aug-2026]   Company: [▾] │ Reference #: __________          Template: [Depreciation - Monthly ▾] ├──────────────────────────────────────────────────────────────────────────── │  Account                 Party        Debit        Credit      Remarks │ ┌────────────────────┬───────────┬───────────┬───────────┬──────────────┐ │ │ Depreciation Exp ▾ │           │  1,200.00 │           │ Aug dep - Van│ │ │ Accum Dep - Vehicle▾│          │           │  1,200.00 │              │ │ │ + Add Row           │           │           │           │              │ │ └────────────────────┴───────────┴───────────┴───────────┴──────────────┘ │                                    Total:      1,200.00      1,200.00 │                                    Difference:              0.00  Balanced ├──────────────────────────────────────────────────────────────────────────── │ User Remark: [Monthly depreciation - delivery van..............] │                                              [Save Draft] [Submit] [Cancel] └────────────────────────────────────────────────────────────────────────────`

Behavior rules:

- "Difference" badge is red while nonzero; turns green and label reads "Balanced" at zero.
    
- Save/Submit button disabled until difference = 0.
    
- After Submit: screen becomes read-only; "Edit" is replaced by "Reverse Entry" (auto-creates a same-date reversing JE).
    
- Templates dropdown pulls from a Journal Entry Template library (depreciation, payroll accrual, bad-debt write-off, inter-branch contra).
    
- Tab/Enter keyboard flow for fast row entry; auto-suggest offsetting account after first row.
    

---

## 4. General Ledger Drill-Down

text

`┌─ General Ledger ─────────────────────── Account: [Bank - Operating A/c ▾] │ Date range: [01-Aug-2026] to [29-Aug-2026]   Party: [All ▾]   [Export] ├──────────┬────────────────────────┬──────────────┬───────────┬───────────┬──────────┐ │ Date     │ Voucher                │ Against       │ Debit     │ Credit    │ Balance  │ ├──────────┼────────────────────────┼──────────────┼───────────┼───────────┼──────────┤ │ 03-Aug   │ Sales Invoice SINV-0091│ ABC Motors    │ 3,500.00  │           │148,100.00│ │ 07-Aug   │ Payment Entry PE-0044  │ Rent - Aug    │           │ 2,300.00  │145,800.00│ │ 12-Aug   │ Journal Entry JV-0042  │ Depreciation  │           │ 1,200.00  │144,600.00│ └──────────┴────────────────────────┴──────────────┴───────────┴───────────┴──────────┘    Clicking any row opens the source voucher in a side panel (no page reload).`

---

## 5. Trial Balance

text

`┌─ Trial Balance ──────────────── As of: [29-Aug-2026]  Compare: [x] Prior Year ├────────────────────────┬───────────┬───────────┬────────────┬────────────┐ │ Account                │ Debit     │ Credit    │ Prior Debit│Prior Credit│ ├────────────────────────┼───────────┼───────────┼────────────┼────────────┤ │ Cash                   │ 12,400.00 │           │ 9,800.00   │            │ │ Accounts Receivable    │ 40,000.00 │           │ 31,200.00  │            │ │ Accounts Payable       │           │ 18,500.00 │            │ 15,000.00  │ │ ...                                                                      │ ├────────────────────────┼───────────┼───────────┼────────────┼────────────┤ │ TOTAL                  │396,600.00 │396,600.00 │310,400.00  │310,400.00  │ └────────────────────────┴───────────┴───────────┴────────────┴────────────┘                           Balanced — Debit = Credit`

---

## 6. Profit & Loss Statement

text

`┌─ Profit & Loss ── Period: [Aug 2026 v]     View: [Monthly v] [Chart] │ Income                                                     Amount        % │   Tuition Fees Income                                      82,400.00   86% │   Registration Fee Income                                  13,000.00   14% │   Total Income                                              95,400.00  100% │ Expenses │   Salaries & Wages                                          38,200.00   62% │   Rent Expense                                                9,200.00  15% │   Depreciation Expense                                        1,200.00   2% │   Total Expenses                                              61,200.00 100% │ ────────────────────────────────────────────────────────────────────── │ NET PROFIT                                                    34,200.00   ▲12% vs last period └────────────────────────────────────────────────────────────────────────`

---

## 7. Balance Sheet

text

`┌─ Balance Sheet ── As of: [29-Aug-2026] │ Assets                          │ Liabilities & Equity │  Current Assets    198,200.00   │  Current Liabilities   52,100.00 │  Fixed Assets        86,300.00  │  Equity               180,000.00 │  ─────────────────────────      │  Retained Earnings      52,400.00 │  TOTAL ASSETS       284,500.00  │  TOTAL LIAB+EQUITY     284,500.00   [check] └─────────────────────────────────────────────────────────────────────────`

---

## 8. Vertical App → Accounting Mapping (Sales Invoice Pattern)

text

`┌─ Tuition Invoice (Preschool vertical UI) ─────────── looks fully custom to user │ Student: Emma Watson     Class: Toddler A     Month: Sep 2026 │ Items: Tuition Fee  $650   |  Lunch Program  $80 │ Total: $730                          [Save & Post to Accounting] └──────────────────────┬──────────────────────────────────────────                         v  (maps internally to Sales Invoice doctype) ┌─ Resulting Accounting Entry (auto-generated, view-only link: "View Accounting Impact") ─ │ Dr  Accounts Receivable - Emma Watson       730.00 │    Cr  Tuition Fees Income                          650.00 │    Cr  Lunch Program Income                          80.00 └────────────────────────────────────────────────────────────────────────── Same pattern applies to: - Auto Workshop Repair Order  -> Sales Invoice (Parts + Labor line items) - Real Estate Rent Invoice    -> Sales Invoice (Rent + CAM line items) - Any vertical Purchase/Bill  -> Purchase Invoice (Vendor AP + Expense/Asset accounts)`

Design rule: the vertical document is the "source of truth" for the business user; the Sales/Purchase Invoice + GL Entry underneath is the "source of truth" for the accountant. Never let both be edited independently.

---

## 9. Feature Checklist for SMB Accounting Module

- Chart of Accounts (tree, industry templates, group/ledger lock)
    
- Journal Entry (templates, live balance check, reversal workflow)
    
- General Ledger (drill-down, party filter, export)
    
- Trial Balance (period compare)
    
- Profit & Loss (monthly/quarterly/yearly, % of total, trend arrow)
    
- Balance Sheet (as-of date, side-by-side layout)
    
- AR/AP Aging & Party Ledger
    
- Bank Reconciliation
    
- Multi-company / multi-tenant company switcher on every screen
    
- Role-based terminology mapping (vertical label vs. accounting label)
    
- Audit trail (who posted, when, submit/cancel/amend history)
    
- Accounting Impact viewer embedded in every vertical transaction screen