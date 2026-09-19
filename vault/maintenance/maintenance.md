# Maintenance

## Purpose

Maintenance Request DocType and workflow: tenant submission, assignment, spares
tracking, labor logging, and cost allocation to the building's cost center.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [maintenance-request-doctype](features/maintenance-request-doctype.md) | done | developer-1 | 2026-08-20 |
| [maintenance-cost-allocation](features/maintenance-cost-allocation.md) | planned | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/maintenance/features"
SORT updated DESC
```

## Related ADRs

- (none — see `payments-accounting` for cost-center ADR)
