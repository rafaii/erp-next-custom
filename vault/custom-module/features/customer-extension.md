---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Customer Extension (KYC Fields)

## Summary

Extend the built-in `Customer` DocType with lease-approval and KYC fields via
Custom Field fixtures (no core code edits).

## Requirements

- New fields: `preferred_contact_method`, `emergency_contact_name`,
  `emergency_contact_phone`, `id_proof_type`, `id_proof_number`,
  `employer_name`, `monthly_income`.
- Persisted as fixtures so a fresh install reproduces them.

## Design

- Custom Fields exported to `fixtures` in `hooks.py`:
  `{"dt": "Custom Field", "filters": [["dt", "=", "Customer"]]}`.
- No changes to core Customer DocType.

## Implementation Plan

- [x] Add 7 custom fields via Custom Field fixtures
- [x] Export to fixtures; commit JSON
- [x] Verify on fresh install

## Acceptance Criteria

- [ ] Fresh `install-app` shows fields on Customer form

## Related

- Domain index: `vault/custom-module/custom-module.md`
