---
status: planned
owner: developer-1
domain: compliance-security
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Role-Based Access Control

## Summary

Frappe RBAC for the four roles (Admin, Leasing Agent, Maintenance Staff, Tenant)
with least-privilege permissions and data isolation.

## Requirements

- Admin: full access.
- Leasing Agent: create/manage customers, units, leases, e-sign; view (not edit) accounting.
- Maintenance Staff: maintenance module only, scoped to assigned buildings.
- Tenant: portal only — own lease, payments, invoices, maintenance requests.

## Design

- Role Profiles / fixtures in the app; per-DocType permission matrix.
- Website user for tenants; no Desk access.

## Implementation Plan

- [ ] Define 4 roles + permission matrix
- [ ] Add Role Profiles to fixtures
- [ ] Verify tenant sees only own data

## Acceptance Criteria

- [ ] Each role limited to its responsibilities
- [ ] Tenant data isolation verified

## Related

- Domain index: `vault/compliance-security/compliance-security.md`
- Feature: `ui-portal/features/tenant-portal-ui.md`
