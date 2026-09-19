---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-16
updated: 2026-08-24
related_adr: ["0009-renter-self-service-portal"]
---

# Tenant Portal UI

## Summary

Renter (Customer) self-service login into the same React admin portal shell
(ADR-0006/0007), scoped to their own data by Frappe's `User Permission`
mechanism, with a reduced role-gated nav: **My Lease**, **My Invoices**, **My
Maintenance** (see `maintenance-portal.md`). Supersedes the original
Jinja/Bootstrap web-view design in favor of reusing the existing portal shell —
see ADR-0009 for why.

## Requirements

- Tenant login via a Website User account (`Tenant` role, no Desk access).
- Server-side data scoping via `User Permission` (User → Customer) — a UI bug
  must never be able to leak another tenant's data; the permission engine is
  the actual gate.
- Views: My Lease (read-only, reuses the existing Lease Agreement detail view
  + "View signed contract" download), My Invoices (Sales Invoice list scoped
  to their Customer, PDF download), My Maintenance (see `maintenance-portal.md`).
- Staff action: "Enable portal login" on the Customer (tenant) detail page —
  creates the User + User Permission, does not touch existing tenants unless
  triggered.

## Design

- Same `ui/` React SPA, no new frontend stack. `portal_nav_items` becomes
  role-aware in `www/portal/index.py` / `api.get_portal_nav()`: a session
  whose only role is `Tenant` (no staff role) gets the reduced nav instead of
  the 9-section staff nav.
- New nav-driven doctype exposure: `Sales Invoice` (read-only list/detail,
  `DOCTYPE_SLUGS["Sales Invoice"] = "invoices"` on the frontend).
- Backend: `Customer` + `Sales Invoice` (core doctypes) get a `Tenant`
  DocPerm row via `frappe.permissions.add_permission` (patch, since core JSON
  can't be edited directly); `Lease Agreement` gets a `Tenant` row directly in
  its own doctype JSON. All read-only — scoping comes from the `User
  Permission` row on `Customer`, which Frappe auto-propagates to every
  doctype with a `customer` Link field.
- `real_estate_os.api.get_my_customer()` — resolves the logged-in tenant's
  Customer from their `User Permission` row.
- `real_estate_os.api.create_tenant_portal_login(customer, email)` —
  staff-triggered, idempotent creation of the User + `Tenant` role + `User
  Permission`, sends the Frappe welcome/set-password email.

## Implementation Plan

- [x] `Tenant` role (patch, desk_access=0)
- [x] `Customer` / `Sales Invoice` read-only Tenant DocPerm (patch, core doctypes)
- [x] `Lease Agreement` Tenant DocPerm (doctype JSON)
- [x] `get_my_customer()` + `create_tenant_portal_login()` API
- [x] "Enable portal login" action on the Customer detail page
- [x] Role-aware `get_portal_nav()` / portal bootstrap (My Lease / My Invoices / My Maintenance)
- [x] `Sales Invoice` wired into `DOCTYPE_SLUGS` + `portal_nav_items` as a read-only nav item
- [ ] Payment-history view beyond the raw invoice list (aggregated paid/overdue summary) — deferred, not blocking v1
- [x] Deploy + verify live with a real tenant login

## Acceptance Criteria

- [x] A user with only the `Tenant` role sees the reduced nav, not the 9-section staff nav
- [x] A tenant's `/contracts`, `/invoices`, `/maintenance` list only their own records (server-side enforced) — verified live via `frappe.get_list` as `khalid@4itrading.com` and `anvar@4itrading.com`: each sees exactly one customer's rows, `frappe.has_permission("Customer", "read", doc=<other customer>)` is `False`
- [x] Verified live: `create_tenant_portal_login` run for Khalid Al Mansoori (CUST-2026-00001) and Anwar P (CUST-2026-00006) — both can now sign in with their `4itrading.com` email

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- ADR: `0009-renter-self-service-portal`
- Feature: `maintenance-portal.md`
- Feature: `compliance-security/features/role-based-access.md`
