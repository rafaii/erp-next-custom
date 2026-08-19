# ADR-0003: Multi-Tenancy — Tenant-per-Site vs Single-DB

- **Status**: accepted
- **Date**: 2026-08-16
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

The product is a multi-tenant SaaS (100+ real-estate companies). We must choose
how tenants are isolated. ERPNext/Frappe does not support cross-database queries
within a single site, and true SaaS compliance (one tenant's P&L must never be
queryable by another) demands strict isolation.

## Decision

**Tenant-per-Site (separate database per tenant).**

- Each real-estate company gets its own Frappe Site with its own DB + file store.
- Single codebase (`real_estate_os`) deployed across all sites via Bench.
- Onboarding: `bench new-site <tenant>.<domain>` → `install-app real_estate_os`.
- Billing tracked externally (Stripe) against site subscriptions.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Single-DB multi-company (ERPNext built-in) | Cheaper, simpler ops | Weak isolation, no cross-DB query, compliance risk for true SaaS | rejected |
| Tenant-per-Site (separate DB) | Strong isolation, per-tenant backup/restore/audit, standard Frappe | More DBs to manage; needs automation (Bench/Frappe Operator) | **chosen** |
| Schema-per-tenant in one DB | Middle ground | Not idiomatic to Frappe; high migration risk | rejected |

## Consequences

### Positive

- Legal/compliance-grade isolation (GDPR-ready).
- Per-tenant backup/restore/audit is trivial.
- Matches Frappe's deployment model (Bench / Frappe Operator).

### Negative

- More infrastructure; requires onboarding automation + centralized upgrades
  (canary/staged rollouts).

## Implementation

- Feature file: `vault/multi-tenancy/features/tenant-onboarding.md`
