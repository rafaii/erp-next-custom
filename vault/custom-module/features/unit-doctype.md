---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-17
related_adr: []
---

# Unit DocType

## Summary

Custom DocType `Unit` representing a rentable unit within a Building.

## Requirements

- Fields: `unit_id` (auto `UNT-{#####}`), `building` (Link Building),
  `unit_number`, `unit_type` (1-Bedroom / 2-Bedroom / Studio / Penthouse),
  `square_footage`, `has_balcony`, `floor_number`, `monthly_rent` (Currency),
  `status` (Vacant / Occupied / Reserved / Under Maintenance), `amenities`.

## Design

- DocType JSON at `real_estate_os/real_estate_os/doctype/unit/`.
- Naming series `UNT-.#####`.
- Lease Agreement links to `unit` and validates it is Vacant before save.

## Implementation Plan

- [x] Create Unit DocType JSON + fields
- [x] Add naming series `UNT-.#####`
- [x] `validate()` guard: unique unit_number per building
- [x] Add role permissions (System Manager)

## Acceptance Criteria

- [x] Unit links to Building; list view filters by building/status
- [x] Lease rejects non-Vacant unit

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Feature: `building-doctype.md`, `lease-agreement-doctype.md`, `bulk-unit-generator.md`
