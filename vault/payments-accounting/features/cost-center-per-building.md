---
status: planned
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Cost Center Per Building

## Summary

Auto-create a Cost Center on Building creation and route rent income, PDC
deposits, and maintenance expenses to it.

## Requirements

- On Building save: create Cost Center (name = building) if absent.
- Sales Invoices, Payment Entries, Expense Claims post to the building's Cost Center.
- Security deposits post to a refundable liability account.

## Design

- `building.py::after_insert` → create Cost Center (frappe Cost Center DocType).
- Set `cost_center` on invoice/payment/expense docs from the linked unit/building.

## Implementation Plan

- [ ] Cost center auto-create on Building save
- [ ] Propagate cost center to rent invoices + payment entries
- [ ] Propagate to maintenance expense claims

## Acceptance Criteria

- [ ] Building creation produces a linked Cost Center
- [ ] Rent/maintenance entries carry correct Cost Center

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- Feature: `building-doctype.md`, `building-profitability-report.md`
