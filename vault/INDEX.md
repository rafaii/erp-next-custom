# Vault Index

Single source of truth for Real Estate OS (real-estate arbitrage SaaS on ERPNext).
Master plan: [`MASTERPLAN.md`](MASTERPLAN.md). Roadmap: [`IMPLEMENTATION-PLAN.md`](IMPLEMENTATION-PLAN.md). Resume point: [`HANDOFF.md`](HANDOFF.md).

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

## Decisions

- Approved ADRs: [`DECISIONS.md`](DECISIONS.md) (0001-0004 accepted)

## Feature Status (auto)

```dataview
TABLE status, owner, updated, domain
FROM "vault"
WHERE contains(file.path, "/features/")
SORT domain ASC, updated DESC
```

## Roadmap

See [`IMPLEMENTATION-PLAN.md`](IMPLEMENTATION-PLAN.md). Phases:

1. **Phase 1 — Core Modules** (Weeks 1-6): app scaffold, Building/Unit, bulk generator, Lease Agreement, Customer extension.
2. **Phase 2 — E-Sign & Recurring Invoicing** (Weeks 7-10): in-built e-sign, provider abstraction, invoice schedules.
3. **Phase 3 — PDC & Accounting** (Weeks 11-14): PDC processing, cost centers, profitability report.
4. **Phase 4 — Maintenance & Reporting** (Weeks 15-18): maintenance portal, cost allocation, dashboards.
5. **Phase 5 — Future**: e-transfers, AI intake, marketing video.
