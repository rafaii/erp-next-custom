---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-16
updated: 2026-08-24
related_adr: ["0009-renter-self-service-portal"]
---

# Maintenance Portal

## Summary

Tenant-facing "My Maintenance" section (part of `tenant-portal-ui.md`): renters
see and file their own `Maintenance Request`s from the portal; staff keep the
existing full view (assignment, spares, cost) at `/maintenance`. Supersedes
the original Jinja web-form design — reuses the React portal shell per
ADR-0009.

## Requirements

- Tenant: list their own requests (status, issue type, priority), submit a new
  one (issue type, priority, description, unit — auto-resolved from their
  active lease when they have exactly one).
- Tenant must never be able to set or see internal fields: `assigned_to`,
  `spares_used`, `labor_hours`, `total_cost`, `work_order`, `cost_center`.
- Tenant cannot edit a request after submitting it (no `write` permission) —
  status changes are staff-only.
- Staff: unchanged existing `/maintenance` view (full CRUD, assignment).

## Design

- `Maintenance Request` internal fields moved to `permlevel: 1`; a `Tenant`
  DocPerm row grants `permlevel 0` `create` + `read` only (no `write`) — this
  alone blocks a tenant from ever setting or editing the internal fields, at
  creation or after. A `System Manager` `permlevel: 1` row is added alongside
  the existing `permlevel: 0` row so staff keep full field access (Frappe
  requires an explicit permlevel-1 grant once any field uses it — it does not
  cascade from permlevel 0 automatically).
- `MaintenanceRequest.validate()` gets server-side auto-resolution of `unit`
  from the customer's active (Signed) lease when not supplied — previously
  only implemented as Desk client JS, which a portal-created request would
  have skipped entirely (same class of bug as the `fetch_from` issues found
  earlier in the Lease Agreement work).
- `real_estate_os.api.create_maintenance_request(unit, issue_type, priority,
  description)` — whitelisted, resolves `customer` from the caller's own
  `User Permission` (via `get_my_customer()`), never trusts a client-supplied
  customer — prevents a tenant from filing a request under another tenant's name.
- Frontend: reuses the existing `Maintenance Request` nav item (relabeled "My
  Maintenance" for tenant sessions) + a new lightweight "New Request" dialog
  (issue type / priority / description, unit preselected when unambiguous).

## Implementation Plan

- [x] `Maintenance Request` permlevel split (internal fields → 1) + Tenant/System Manager DocPerm rows
- [x] Server-side `unit` auto-resolution in `validate()`
- [x] `create_maintenance_request()` API (customer resolved server-side)
- [x] "New Request" dialog on the portal, wired for Tenant sessions
- [ ] Photo/attachment upload on the request — deferred, not blocking v1
- [x] Deploy + verify: tenant can file a request and see its status; cannot see/set internal fields

## Acceptance Criteria

- [x] Tenant submits a request without providing `unit` and it resolves correctly from their active lease — verified live: Anwar P (CUST-2026-00006, signed lease LSE-2026-00063 on UNT-00062) filed a request with no `unit` given, it resolved to `UNT-00062`
- [x] Tenant's create/list calls never expose `assigned_to`/`spares_used`/`total_cost`/`cost_center`/`work_order` — permlevel 1, Tenant role has no permlevel-1 grant
- [x] A tenant cannot edit a request after submission — verified live: `frappe.has_permission("Maintenance Request", "write", doc=<own request>)` is `False` for the filing tenant, `"read"` is `True`

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- ADR: `0009-renter-self-service-portal`
- Feature: `tenant-portal-ui.md`
- Feature: `maintenance/features/maintenance-request-doctype.md`
