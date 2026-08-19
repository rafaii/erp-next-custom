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
| [cost-center-per-building](features/cost-center-per-building.md) | planned | developer-1 | 2026-08-16 |
| [building-profitability-report](features/building-profitability-report.md) | planned | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/payments-accounting/features"
SORT updated DESC
```

## Related ADRs

- `0004-recurring-invoicing` — invoice generation mechanism
