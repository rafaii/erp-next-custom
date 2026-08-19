# Implementation Plan

Phased roadmap for Real Estate OS. Master PRD: [`MASTERPLAN.md`](MASTERPLAN.md).
Each feature file (`vault/<domain>/features/<feature>.md`) carries its own
step-by-step plan + acceptance criteria.

## Phase 1 — Core Modules (Weeks 1-6)

Goal: app scaffold + core DocTypes + CRUD.

| Feature | Domain | Status |
| --- | --- | --- |
| [app-scaffold](custom-module/features/app-scaffold.md) | custom-module | done |
| [building-doctype](custom-module/features/building-doctype.md) | custom-module | in-progress |
| [unit-doctype](custom-module/features/unit-doctype.md) | custom-module | done |
| [bulk-unit-generator](custom-module/features/bulk-unit-generator.md) | custom-module | done |
| [lease-agreement-doctype](custom-module/features/lease-agreement-doctype.md) | custom-module | done |
| [customer-extension](custom-module/features/customer-extension.md) | custom-module | done |
| [occupancy-dashboard](ui-portal/features/occupancy-dashboard.md) | ui-portal | planned |

## Phase 2 — E-Sign & Recurring Invoicing (Weeks 7-10)

| Feature | Domain | Status |
| --- | --- | --- |
| [inbuilt-esign](esign/features/inbuilt-esign.md) | esign | done |
| [counter-signature](esign/features/counter-signature.md) | esign | done |
| [provider-abstraction](esign/features/provider-abstraction.md) | esign | planned |
| [esign-audit-trail](compliance-security/features/esign-audit-trail.md) | compliance-security | done |
| [recurring-invoicing](payments-accounting/features/recurring-invoicing.md) | payments-accounting | done |
| [tenant-portal-ui](ui-portal/features/tenant-portal-ui.md) | ui-portal | planned |

## Phase 3 — PDC & Accounting (Weeks 11-14)

| Feature | Domain | Status |
| --- | --- | --- |
| [pdc-processing](payments-accounting/features/pdc-processing.md) | payments-accounting | done |
| [cost-center-per-building](payments-accounting/features/cost-center-per-building.md) | payments-accounting | planned |
| [building-profitability-report](payments-accounting/features/building-profitability-report.md) | payments-accounting | planned |

## Phase 4 — Maintenance & Reporting (Weeks 15-18)

| Feature | Domain | Status |
| --- | --- | --- |
| [maintenance-request-doctype](maintenance/features/maintenance-request-doctype.md) | maintenance | planned |
| [maintenance-cost-allocation](maintenance/features/maintenance-cost-allocation.md) | maintenance | planned |
| [maintenance-portal](ui-portal/features/maintenance-portal.md) | ui-portal | planned |

## Phase 5 — Future

E-transfers (Stripe/Interac), voice-AI intake, marketing video generation.

## Cross-cutting

| Feature | Domain | Notes |
| --- | --- | --- |
| [tenant-onboarding](multi-tenancy/features/tenant-onboarding.md) | multi-tenancy | applies from Phase 1 (deploy model) |
| [role-based-access](compliance-security/features/role-based-access.md) | compliance-security | applies from Phase 1 |

## Gate

A phase is "done" when all its features are `done` and their acceptance
criteria pass on a fresh `install-app`.
