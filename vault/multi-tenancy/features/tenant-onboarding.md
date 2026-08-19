---
status: planned
owner: developer-1
domain: multi-tenancy
created: 2026-08-16
updated: 2026-08-16
related_adr: ["0003-multi-tenancy"]
---

# Tenant Onboarding

## Summary

Automated provisioning of a new tenant site: `bench new-site` → `install-app
real_estate_os` → default roles/settings.

## Requirements

- Script/playbook to create a site and install the app.
- Default role profiles + `Signature Settings` per tenant.
- External billing (Stripe) hooks for subscription tracking.

## Design

- Onboarding script + docs (`deploy/onboard.sh`).
- Frappe Operator (Kubernetes) as the long-term deployment target.

## Implementation Plan

- [ ] Write `bench new-site` + `install-app` onboarding script
- [ ] Default role/settings fixtures per tenant
- [ ] Document staging (canary) upgrade flow

## Acceptance Criteria

- [ ] One command provisions a working tenant site

## Related

- Domain index: `vault/multi-tenancy/multi-tenancy.md`
- ADR: `0003-multi-tenancy`
