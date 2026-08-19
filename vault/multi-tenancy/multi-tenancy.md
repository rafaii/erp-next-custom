# Multi-Tenancy

## Purpose

SaaS deployment model: tenant-per-site (separate database per real-estate
company), automated onboarding, centralized upgrades, and external billing.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [tenant-onboarding](features/tenant-onboarding.md) | planned | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/multi-tenancy/features"
SORT updated DESC
```

## Related ADRs

- `0003-multi-tenancy` — tenant-per-site vs single-DB
