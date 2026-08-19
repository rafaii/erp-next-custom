---
status: planned
owner: developer-1
domain: maintenance
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Maintenance Cost Allocation

## Summary

On resolution, post spares + labor cost to the building's Cost Center via an
Expense Claim (or Stock Entry for inventory spares).

## Requirements

- Spares (Inventory) → Expense Claim / Stock Entry → GL Entry.
- Cost posts to the building's Cost Center.
- Security deposit refunds remain a liability (not expensed).

## Design

- `maintenance_request.py::on_resolve` → create Expense Claim / Stock Entry
  using `spares_used` + `labor_hours` + `cost_center`.

## Implementation Plan

- [ ] Implement `on_resolve` auto-posting
- [ ] Route spares via Stock Entry (inventory) or Expense Claim
- [ ] Confirm cost center propagation

## Acceptance Criteria

- [ ] Resolved request posts cost to correct building Cost Center

## Related

- Domain index: `vault/maintenance/maintenance.md`
- Feature: `maintenance-request-doctype.md`; `payments-accounting/features/cost-center-per-building.md`
