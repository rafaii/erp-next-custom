---
status: planned
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Building Profitability Report

## Summary

Custom report "Building Profitability": P&L grouped by building Cost Center.

## Requirements

- Income (rent), expenses (maintenance, spares), net per building.
- Drill-down to GL entries.

## Design

- Frappe Report (Query or Script Report) grouped by Cost Center.
- Uses standard GL; Cost Center = building.

## Implementation Plan

- [ ] Implement Script Report grouped by Cost Center
- [ ] Add columns: income, expenses, net
- [ ] Add drill-down

## Acceptance Criteria

- [ ] Report shows per-building P&L with correct net

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `cost-center-per-building.md`
