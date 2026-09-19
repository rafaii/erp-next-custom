---
status: planned
owner: developer-1
domain: custom-module
created: 2026-08-27
updated: 2026-08-27
related_adr: ["0013-lead-crm-and-tenant-field-curation"]
---

# Lead Management (pre-tenant pipeline)

## Summary

Part B of ADR-0013. These real-estate businesses run ad campaigns to find
tenants — before this ADR, there was no concept of "someone interested but
not yet a tenant" anywhere in the app; `Customer` was the only record type,
and it means "an actual or former tenant." Approved direction: adopt
ERPNext's existing `Lead` doctype (source, campaign_name, lead_owner,
status, already installed and unused) for the pre-tenant pipeline, convert
to `Customer` once someone is actually signing a lease. **Not started** —
scoped and constrained by ADR-0013, but deliberately deferred as its own
follow-up rather than built alongside the field-curation fix.

## Requirements (v1 scope, per ADR-0013)

- Staff can manually create a Lead: name, contact info, source (which ad
  channel/platform), notes. No public-facing capture form in v1.
- A status pipeline (e.g. New / Contacted / Qualified / Lost — exact
  states TBD at implementation time).
- A "Convert to Tenant" action that creates the `Customer` record (reusing
  `api.create_tenant`'s shape) and links back to the originating Lead
  (`Customer.lead_name`, a field ERPNext's Customer doctype already has).
- **No source/campaign attribution reporting in v1** — that's an explicit
  later addition, not part of this scope.
- **The frontend must never surface "ERPNext," "Lead" as an ERPNext
  concept, or any other ERPNext branding** — this app's portal already
  never names Customer/Supplier/Cost Center as ERPNext concepts to the
  user; the Lead-backed pipeline must follow the exact same convention.
  This also means the Lead-pipeline UI can't reuse ERPNext Desk's own
  Lead kanban/views — it needs its own React portal implementation, same
  effort level as every other doctype this portal wraps.

## Design (not yet started — sketch only, refine at implementation time)

- New portal nav item ("Leads"), doctype `Lead`, likely reusing
  `ResourceListView`'s sort/filter/search/pagination (already generic)
  the same way every other list page does.
- Light field curation on `Lead` itself (same `Portal Field Visibility`
  mechanism from `tenant-field-curation.md`) — stock `Lead` has 67 fields,
  also skewed B2B (`no_of_employees`, `market_segment`, `company_name`).
- New whitelisted method, e.g. `convert_lead_to_tenant(lead_name, ...)`,
  building on `api.create_tenant`'s existing shape.
- RBAC note: a "Leasing Agent" role (named in `os/REALESTATE_MASTERPLAN.md` §2, not yet
  built — see the 2026-08-24 findings doc, gap G8) would naturally own
  this pipeline. Not a blocker for building Lead management itself, but
  worth sequencing awareness of.

## Implementation Plan

- [ ] Not started — pick up as its own task when prioritized.

## Acceptance Criteria

- [ ] Not started.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0013-lead-crm-and-tenant-field-curation.md`
- Feature: `tenant-field-curation.md`, `customer-extension.md`
- Finding: `vault/findings/2026-08-24-ideal-product-vs-current-state.md` (G8, RBAC/Leasing Agent role)
