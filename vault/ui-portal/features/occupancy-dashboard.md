---
status: planned
owner: ui-designer-1
domain: ui-portal
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Occupancy Dashboard

## Summary

Visual building/occupancy map with color-coded unit status (Vacant = green,
Occupied = red, Reserved = amber, Under Maintenance = grey).

## Requirements

- Dashboard page showing buildings and their units as a color-coded grid.
- Filters by building; lease status summary.

## Design

- Frappe Dashboard (Vue) + workspace page in `real_estate_os/public/`.
- Data from Unit.status; responsive.

## Implementation Plan

- [ ] Build occupancy grid component (color by status)
- [ ] Add building filter + lease summary cards
- [ ] Responsive (mobile-friendly)

## Acceptance Criteria

- [ ] Units render color-coded by status
- [ ] Filters + summary correct

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
