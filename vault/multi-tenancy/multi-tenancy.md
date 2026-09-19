# Multi-Tenancy

## Purpose

SaaS deployment model: tenant-per-site (separate database per real-estate
company), automated onboarding, centralized upgrades, and external billing.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [tenant-onboarding](features/tenant-onboarding.md) | planned | developer-1 | 2026-08-16 |
| [tenant-provisioning-console](features/tenant-provisioning-console.md) | planned | developer-1 | 2026-08-23 |
| [console-admin-ui](features/console-admin-ui.md) | in-progress | developer-1 | 2026-09-05 |
| [tenant-lifecycle-enforcement](features/tenant-lifecycle-enforcement.md) | in-progress | developer-1 | 2026-09-05 |
| [tenant-permanent-deletion](features/tenant-permanent-deletion.md) | planned | developer-1 | 2026-09-05 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/multi-tenancy/features"
SORT updated DESC
```

## Related ADRs

- `0003-multi-tenancy` — tenant-per-site vs single-DB
- `0008-tenant-provisioning-control-plane` — UI-based provisioning via a separate control-plane site
- `0022-console-admin-ui` — custom React frontend for the console (replaces plain Frappe desk forms)
- `0023-tenant-lifecycle-enforcement` — Suspend/Cancel via Traefik routing, zero code on any tenant site
- `0024-tenant-permanent-deletion` — Cancelled-first, typed-subdomain confirm, pre-drop backup
