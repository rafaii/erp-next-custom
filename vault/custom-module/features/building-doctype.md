---
status: in-progress
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-17
related_adr: []
---

# Building DocType

## Summary

Custom DocType `Building` representing a leased apartment building, with a
linked cost center and one-to-many Units.

## Requirements

- Fields: `building_id` (auto `BLD-{#####}`), `building_name`, `address`,
  `total_units`, `amenities`, `property_manager` (Link User), `status`
  (Active / Under Maintenance / Vacant), `cost_center` (Link, auto-created).
- On save: auto-create a Cost Center for the building (or link existing).
- `status` drives the occupancy dashboard.

## Design

- DocType JSON at `real_estate_os/real_estate_os/doctype/building/`.
- Naming series `BLD-.#####`.
- `doc_events` hook: `after_insert` → create/link cost center (delegated to
  `payments-accounting/features/cost-center-per-building.md`).
- One Building → Many Units via `unit.building` link.

## Implementation Plan

- [x] Create Building DocType JSON + fields
- [x] Add naming series `BLD-.#####`
- [ ] Wire `after_insert` cost-center hook (Phase 3)
- [x] Add role permissions (System Manager; Leasing Agent role in Phase 4)

## Acceptance Criteria

- [ ] Create a Building → auto cost center linked
- [ ] Building list shows status; units linkable

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Feature: `unit-doctype.md`, `cost-center-per-building.md`
