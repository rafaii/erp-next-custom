# UI & Portal

## Purpose

Frappe Desk views, dashboards (occupancy map, lease status), tenant portal web
views (lease details, payments, maintenance submission), and the maintenance
staff queue.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [occupancy-dashboard](features/occupancy-dashboard.md) | planned | ui-designer-1 | 2026-08-16 |
| [tenant-portal-ui](features/tenant-portal-ui.md) | planned | ui-designer-1 | 2026-08-16 |
| [maintenance-portal](features/maintenance-portal.md) | planned | ui-designer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/ui-portal/features"
SORT updated DESC
```

## Related ADRs

- (none yet)
