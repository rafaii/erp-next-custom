# Payments & Accounting

## Purpose

Recurring rent invoicing with payment schedules, post-dated cheque (PDC)
automation, building-level cost centers, and building profitability reporting.
Security deposits tracked as refundable liability.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [recurring-invoicing](features/recurring-invoicing.md) | done | developer-1 | 2026-08-16 |
| [pdc-processing](features/pdc-processing.md) | done | developer-1 | 2026-08-17 |
| [pdc-schedule-generation-and-reconciliation](features/pdc-schedule-generation-and-reconciliation.md) | done | developer-1 | 2026-08-27 |
| [cost-center-per-building](features/cost-center-per-building.md) | done | developer-1 | 2026-08-27 |
| [building-profitability-report](features/building-profitability-report.md) | done | developer-1 | 2026-08-26 |
| [landlord-payables](features/landlord-payables.md) | done | developer-1 | 2026-08-27 |
| [accounts-income-vs-expense](features/accounts-income-vs-expense.md) | done | developer-1 | 2026-08-26 |
| [rent-roll-and-arrears-report](features/rent-roll-and-arrears-report.md) | planned | developer-1 | 2026-08-24 |
| [cash-flow-forecast](features/cash-flow-forecast.md) | in-progress | developer-1 | 2026-08-27 |
| [accounting-reports](features/accounting-reports.md) | in-progress | developer-1 | 2026-08-28 |
| [accounting-ui-gap-analysis](accounting-ui-gap-analysis.md) | in-progress | developer-1 | 2026-08-30 |
| [landlord-cheque-plan-decoupling](features/landlord-cheque-plan-decoupling.md) | done | developer-1 | 2026-09-06 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/payments-accounting/features"
SORT updated DESC
```

## Related ADRs

- `0004-recurring-invoicing` — invoice generation mechanism
- `0010-pdc-schedule-and-bank-reconciliation` — PDC schedule generation, Cheque Bank, Security Deposit, Payment Entry reconciliation
- `0011-head-lease-landlord-payables` — Head Lease DocType + outgoing-PDC landlord payables, configurable per Building
- `0014-manual-bank-deposit-workflow` — staff-driven cheque deposit, not automatic
- `0015-portal-native-accounting-reports` — GL/Trial Balance/P&L/Balance Sheet/Journal Entry via ERPNext's own report engines
- `0025-landlord-cheque-plan-decoupling` — cheque plan / accrual invoice / payment settlement split into independent, reconciled records; admin-entered cheque plan gates Head Lease going Active; on-account advance auto-swept oldest-invoice-first
