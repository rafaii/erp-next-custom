# UI & Portal

## Purpose

Frappe Desk views, dashboards (occupancy map, lease status), tenant portal web
views (lease details, payments, maintenance submission), and the maintenance
staff queue.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [occupancy-dashboard](features/occupancy-dashboard.md) | planned | ui-designer-1 | 2026-08-16 |
| [tenant-portal-ui](features/tenant-portal-ui.md) | done | developer-1 | 2026-08-24 |
| [maintenance-portal](features/maintenance-portal.md) | done | developer-1 | 2026-08-24 |
| [admin-portal-ui](features/admin-portal-ui.md) | done | ui-designer-1 | 2026-08-25 |
| [portal-source-consolidation](features/portal-source-consolidation.md) | done | developer-1 | 2026-08-27 |
| [contracts-page-fixes](features/contracts-page-fixes.md) | done | developer-1 | 2026-08-26 |
| [portal-list-column-fixes](features/portal-list-column-fixes.md) | done | developer-1 | 2026-08-26 |
| [portal-list-table-controls](features/portal-list-table-controls.md) | done | developer-1 | 2026-08-26 |
| [contract-detail-page-cleanup](features/contract-detail-page-cleanup.md) | done | developer-1 | 2026-08-27 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/ui-portal/features"
SORT updated DESC
```

## Related ADRs

- `0006-admin-portal-approach` — headless-hybrid Admin Portal UI (config-driven sidebar)
- `0009-renter-self-service-portal` — renter login scoped by User Permission, reusing the same shell
