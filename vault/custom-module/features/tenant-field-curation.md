---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-27
updated: 2026-08-27
related_adr: ["0013-lead-crm-and-tenant-field-curation"]
---

# Tenant (Customer) Field Curation

## Summary

Part A of ADR-0013. `Customer` carries 78 stock ERPNext sales-order fields
plus 7 KYC fields this app added — 85 total, all shown in the portal's
generic edit form. A `Portal Field Visibility` mechanism already existed
to curate this, but only the read-only detail view honored it; edit mode
ignored it and always rendered every field. Fixed both the bug and seeded
a sensible curated default for Customer specifically.

## Requirements

- Edit mode respects the same field visibility/order config the
  read-only view already does.
- Customer gets a sensible default curated field list out of the box,
  still fully editable via the existing "View settings" dialog.

## Design

- `ui/src/pages/Dashboard.tsx`: the Detail page's edit-mode grid now maps
  over `orderedFields` (field_order-filtered) instead of the raw,
  unfiltered `doc.data.fields`. `FieldVisibilityDialog`'s description
  updated to say it applies to viewing *and* editing.
- `real_estate_os/patches/v0_0/seed_customer_field_visibility.py` (new,
  `post_model_sync`): idempotently seeds one `Portal Field Visibility` row
  for `"Customer"` with `visible_fields` = `["customer_name",
  "customer_type", "email_id", "mobile_no", "preferred_contact_method",
  "emergency_contact_name", "emergency_contact_phone", "id_proof_type",
  "id_proof_number", "employer_name", "monthly_income"]` — the exact field
  set `api.create_tenant` already populates at creation, matching
  `os/REALESTATE_MASTERPLAN.md` §3.1's original (never-implemented) "keep fields" intent.

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed + `bench migrate` clean (new patch ran with no errors)
- [x] Verified live — with a real, unexpected finding: the seed patch's
      idempotent "only create if missing" check correctly found a
      **pre-existing** `Portal Field Visibility` row for Customer
      (created 2026-08-23, predating this work), so it skipped seeding
      as designed. That existing row was itself broken — missing
      `customer_name`/`customer_type`/the emergency-contact fields
      entirely, and containing `"contact_and_address_tab"` (a Tab Break
      layout field, not real data) because `get_doc_detail`'s
      `skip_fieldtypes` never excluded Tab Break the way it excludes
      Section Break/Column Break — so a stale, incomplete customization
      from early UI-building work was silently still in effect. Fixed
      both: added `Tab Break` to `skip_fieldtypes` (PR #26, prevents this
      leaking into "View settings" again for any doctype), and manually
      corrected the live row to the intended 11-field list. Confirmed
      via `api.get_field_order("Customer")` after the fix.
- [ ] Not click-tested in an actual browser — no browser tool available
      in this environment.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0013-lead-crm-and-tenant-field-curation.md`
- Feature: `customer-extension.md`, `lead-management.md`
