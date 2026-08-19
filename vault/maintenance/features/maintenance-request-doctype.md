---
status: planned
owner: developer-1
domain: maintenance
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Maintenance Request DocType

## Summary

Custom DocType `Maintenance Request` + workflow: tenant submission → assignment →
in-progress → resolved → closed, with spares, labor, and cost tracking.

## Requirements

- Fields: `request_id` (auto `MNT-{YYYY}-{#####}`), `customer`, `unit`
  (auto from active lease), `issue_type` (Plumbing / Electrical / HVAC /
  Appliance / Other), `description`, `priority` (Low / Medium / High /
  Emergency), `status` (Open / Assigned / In Progress / Resolved / Closed),
  `assigned_to` (Link User), `work_order` (optional), `spares_used` (child
  table: Item, Quantity, Rate), `labor_hours`, `total_cost` (auto),
  `cost_center` (auto from unit's building).
- Completion email + tenant feedback/rating.

## Design

- DocType JSON at `real_estate_os/real_estate_os/doctype/maintenance_request/`.
- Child table `Maintenance Spare` for spares.
- `unit` auto-resolved from customer's active lease.

## Implementation Plan

- [ ] Create Maintenance Request DocType + child table + fields
- [ ] Auto-fill unit from active lease
- [ ] `total_cost` auto-calc from spares + labor
- [ ] Status workflow + completion email + feedback

## Acceptance Criteria

- [ ] Request auto-fills unit from lease
- [ ] total_cost computes correctly
- [ ] Resolve → completion email sent

## Related

- Domain index: `vault/maintenance/maintenance.md`
- Feature: `maintenance-cost-allocation.md`; `ui-portal/features/maintenance-portal.md`
