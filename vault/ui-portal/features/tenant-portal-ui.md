---
status: planned
owner: ui-designer-1
domain: ui-portal
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Tenant Portal UI

## Summary

Customer-facing portal web view: lease details, payment history, invoices,
signed contract download.

## Requirements

- Authenticated (tenant login via API / website user).
- Views: lease details, payment history, invoices, signed contract download.
- Mobile-friendly (Bootstrap).

## Design

- Frappe Web Views / website pages in `real_estate_os/www/` or portal pages.
- Jinja + JS; role `Customer (Tenant)` permissions.

## Implementation Plan

- [ ] Build lease-details page
- [ ] Build payment-history + invoice list
- [ ] Signed contract download link
- [ ] Auth + role gating

## Acceptance Criteria

- [ ] Tenant sees only their own data
- [ ] Contract PDF downloadable

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `compliance-security/features/role-based-access.md`
