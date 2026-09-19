# Custom Module

## Purpose

The core `real_estate_os` Frappe app: scaffold, custom DocTypes (Building, Unit,
Lease Agreement, PDC Entry, Maintenance Request, Landlord), Customer extension
via Custom Field fixtures, and the hooks that wire fixtures/scheduler/whitelisted
methods.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [app-scaffold](features/app-scaffold.md) | done | developer-1 | 2026-08-16 |
| [building-doctype](features/building-doctype.md) | done | developer-1 | 2026-08-27 |
| [unit-doctype](features/unit-doctype.md) | done | developer-1 | 2026-08-16 |
| [bulk-unit-generator](features/bulk-unit-generator.md) | done | developer-1 | 2026-08-16 |
| [lease-agreement-doctype](features/lease-agreement-doctype.md) | done | developer-1 | 2026-08-17 |
| [customer-extension](features/customer-extension.md) | done | developer-1 | 2026-08-16 |
| [landlord-doctype](features/landlord-doctype.md) | done | developer-1 | 2026-08-20 |
| [head-lease-doctype](features/head-lease-doctype.md) | done | developer-1 | 2026-08-27 |
| [building-amenity-options](features/building-amenity-options.md) | done | developer-1 | 2026-08-25 |
| [building-setup-checklist](features/building-setup-checklist.md) | done | developer-1 | 2026-08-25 |
| [lease-lifecycle-state](features/lease-lifecycle-state.md) | done | developer-1 | 2026-08-26 |
| [own-building-landlord](features/own-building-landlord.md) | done | developer-1 | 2026-08-27 |
| [tenant-field-curation](features/tenant-field-curation.md) | done | developer-1 | 2026-08-27 |
| [lead-management](features/lead-management.md) | planned | developer-1 | 2026-08-27 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/custom-module/features"
SORT updated DESC
```

## Related ADRs

- `0001-app-repo-structure` — app/git layout
- `0011-head-lease-landlord-payables` — Head Lease DocType
- `0012-lease-lifecycle-state` — lease_status field, distinct Terminated/Cancelled/Expired
- `0013-lead-crm-and-tenant-field-curation` — ERPNext Lead for pre-tenant pipeline (not a Tenant doctype split), Customer field curation
