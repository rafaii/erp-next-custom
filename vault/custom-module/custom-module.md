# Custom Module

## Purpose

The core `real_estate_os` Frappe app: scaffold, custom DocTypes (Building, Unit,
Lease Agreement, PDC Entry, Maintenance Request), Customer extension via Custom
Field fixtures, and the hooks that wire fixtures/scheduler/whitelisted methods.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [app-scaffold](features/app-scaffold.md) | done | developer-1 | 2026-08-16 |
| [building-doctype](features/building-doctype.md) | in-progress | developer-1 | 2026-08-17 |
| [unit-doctype](features/unit-doctype.md) | done | developer-1 | 2026-08-16 |
| [bulk-unit-generator](features/bulk-unit-generator.md) | done | developer-1 | 2026-08-16 |
| [lease-agreement-doctype](features/lease-agreement-doctype.md) | done | developer-1 | 2026-08-17 |
| [customer-extension](features/customer-extension.md) | done | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/custom-module/features"
SORT updated DESC
```

## Related ADRs

- `0001-app-repo-structure` — app/git layout
