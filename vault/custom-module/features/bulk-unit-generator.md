---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-17
related_adr: []
---

# Bulk Unit Generator

## Summary

One-click generation of N units (e.g. 20) for a Building, with sequential
numbering and defaults, via a Custom Button + whitelisted method. Guarded
against runaway generation (server cap + client confirmation).

## Requirements

- "Generate Units" button on Building form.
- Prompt for count + defaults (unit type, sq-ft, rent).
- Create units `001`..`020` as Vacant.
- Server-side cap: reject > 1000 units per call (`MAX_BULK_UNITS`).
- Client confirmation dialog showing the summary + warning for large counts.

## Design

- Whitelisted method `real_estate_os.building.utils.generate_units(building, count)`.
- Custom Button (Server Script / client script fixture) calling the method.
- Mirror the masterplan §10.1 snippet.

## Implementation Plan

- [x] Implement `generate_units()` whitelisted method
- [x] Add Custom Button fixture on Building (Client Script)
- [x] Emit success msgprint with count
- [x] Guard: reject > 1000 units per call (`MAX_BULK_UNITS`), client confirmation dialog with summary + large-count warning

## Acceptance Criteria

- [x] Click Generate Units → N Vacant units created with sequential numbering
- [x] Idempotent-ish: no duplicate unit_number within a building
- [x] > 1000 units rejected server-side; client shows confirmation before generating

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Feature: `building-doctype.md`, `unit-doctype.md`
