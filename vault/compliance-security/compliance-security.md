# Compliance & Security

## Purpose

Legal defensibility of e-signatures (ESIGN/UETA/PIPEDA), immutable audit trails,
role-based access control (RBAC), data isolation, and encryption at rest.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [esign-audit-trail](features/esign-audit-trail.md) | done | developer-1 | 2026-08-17 |
| [role-based-access](features/role-based-access.md) | planned | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/compliance-security/features"
SORT updated DESC
```

## Related ADRs

- `0002-esign-approach` — legal baseline for e-signature
