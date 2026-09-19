# Vault Index

Single source of truth for the platform (multi-industry SaaS on ERPNext) and
its first operating system, Real Estate OS (real-estate arbitrage SaaS).
Platform strategy: [`PLATFORM_STRATEGY.md`](PLATFORM_STRATEGY.md) (multi-OS, modularization, shared ERP core). First OS's own PRD: [`os/REALESTATE_MASTERPLAN.md`](os/REALESTATE_MASTERPLAN.md). Live task list: [`IMPLEMENTATION-PLAN.md`](IMPLEMENTATION-PLAN.md). Resume point: [`HANDOFF.md`](HANDOFF.md).

## Domains

| Domain | Index | Purpose |
| --- | --- | --- |
| Custom Module | [custom-module/custom-module.md](custom-module/custom-module.md) | App scaffold + core DocTypes (Building, Unit, Lease, Customer) |
| E-Sign | [esign/esign.md](esign/esign.md) | In-built e-signature + provider abstraction |
| Payments & Accounting | [payments-accounting/payments-accounting.md](payments-accounting/payments-accounting.md) | Recurring invoicing, PDC, cost centers, reports |
| Maintenance | [maintenance/maintenance.md](maintenance/maintenance.md) | Maintenance requests + cost allocation |
| UI & Portal | [ui-portal/ui-portal.md](ui-portal/ui-portal.md) | Dashboards, occupancy map, tenant/maintenance portals |
| Multi-Tenancy | [multi-tenancy/multi-tenancy.md](multi-tenancy/multi-tenancy.md) | Tenant-per-site SaaS deployment |
| Compliance & Security | [compliance-security/compliance-security.md](compliance-security/compliance-security.md) | E-sign audit trail, RBAC, data isolation |
| Email System | [email-system/email-system.md](email-system/email-system.md) | Platform-wide outgoing mail branding (business + Aetris, no ERPNext branding) |

## Decisions

- Approved ADRs: [`DECISIONS.md`](DECISIONS.md)

## Feature Status (auto)

```dataview
TABLE status, owner, updated, domain
FROM "vault"
WHERE contains(file.path, "/features/")
SORT domain ASC, updated DESC
```

## Live Task List

See [`IMPLEMENTATION-PLAN.md`](IMPLEMENTATION-PLAN.md) — a live, prioritized
list of every pending/in-progress feature across all domains (not phases;
completed work moves to that file's Completed section for reference).
