# ADR-0013: Lead/CRM Tracking & Tenant Field Curation

- **Status**: accepted
- **Date**: 2026-08-27
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Imran asked two related questions:

1. Editing a Tenant in the portal shows "a bunch of fields that really
   don't make sense for regular users" — should Tenant and Customer be
   split apart (Customer = not-yet-a-tenant lead, Tenant = has a lease)?
2. These real-estate businesses will run ad campaigns to find tenants, so
   some form of CRM (lead capture, source tracking, nurture) matters —
   is that part of the plan?

**Checked against the plan before answering, not assumed:** `MASTERPLAN.md`,
`IMPLEMENTATION-PLAN.md` (all 6 phases), and `HANDOFF.md` have zero mention
of leads, CRM, campaigns, prospects, or applicants. The only "marketing"
item anywhere is "AI marketing video generation" (Phase 5/future) — content
generation, not lead tracking. Confirmed this is genuinely new scope, not
something already scheduled that got missed.

**Root cause of complaint #1, verified in code:** `Customer` is ERPNext's
standard CRM/sales-order doctype — **78 stock fields** (customer_group,
territory, tax_id, tax_category, default price list, credit_limit,
sales_team, loyalty_program_tier, market_segment, industry, "is internal
customer", website, etc.), most aimed at a B2B sales-order workflow this
app doesn't use at all. `customer-extension.md` (2026-08-16) added 7 more
KYC fields (preferred_contact_method, emergency_contact_name/phone,
id_proof_type/number, employer_name, monthly_income) on top — **85 fields
total**. The admin portal's generic `get_doc_detail`/`EditableField`
rendered every one of them, in both the read-only and edit views.
`MASTERPLAN.md` §3.1 actually said to curate this down to
`customer_name, email_id, phone, job_title, company` — that intent was
never implemented. A `Portal Field Visibility` mechanism already exists
for exactly this (per-doctype field order/visibility, user-editable via
each detail page's "View settings"), but only the read-only detail view
honored it — edit mode had a bug where it always rendered every field
regardless of that config.

**What already exists that's relevant, checked on the live bench:** the
`erpnext` app (already installed, not a new dependency) ships `Lead`
(67 fields, including `source`, `campaign_name`, `lead_owner`, `status`,
and a `customer` link for conversion), `Opportunity`, and `Campaign`
doctypes in its CRM module — completely unused by this app today.
`Customer` itself already has a `lead_name` field (a Link to Lead) for
conversion lineage. This changes the CRM question from "build lead
tracking from scratch" to "wrap what ERPNext already ships," the same
pattern already used for `Customer`, `Supplier`, and `Cost Center`.

## Decision

Two separable problems, two different fixes, not one rearchitect:

**(A) Field curation — decided and implemented in this same ADR (small,
no schema change, no risk to anything depending on `Customer`):**
`EditableField`'s edit-mode grid now honors the same `Portal Field
Visibility` config the read-only view already did (previously it always
rendered all 85 fields regardless). A curated default of 11 fields is
seeded for `Customer` — `customer_name, customer_type, email_id,
mobile_no, preferred_contact_method, emergency_contact_name,
emergency_contact_phone, id_proof_type, id_proof_number, employer_name,
monthly_income` — via `patches/v0_0/seed_customer_field_visibility.py`,
fully editable afterward from the existing "View settings" dialog.

**(B) Lead/CRM tracking — approved in direction, scoped for v1, but
implementation is its own separate follow-up (not built as part of this
ADR):**
Adopt ERPNext's existing `Lead` doctype for pre-tenant prospects (ad
campaign source, status pipeline, lead owner) rather than building a
parallel concept or splitting `Customer` into a new `Tenant` doctype.
Convert a Lead to `Customer` only once they're actually signing a lease.
`Customer` keeps its single meaning: an actual or former tenant.

Decided scope for v1, from Imran directly:

- **Frontend must never reference "ERPNext" anywhere a user can see** —
  the underlying doctype being ERPNext's `Lead` is an implementation
  detail; all portal labels, dialogs, and empty states use this app's
  own terminology ("Lead", "Leads pipeline") exactly like every other
  ERPNext-doctype-backed feature in this app already does (Customer,
  Supplier, Cost Center are never named as such to the user either).
- **Depth**: capture + a status pipeline + a convert-to-Customer action.
  No source/campaign attribution reporting in v1 — that's a later
  addition once the pipeline itself is in use.
- **Capture method**: manual entry by staff only in v1. No public-facing
  lead-capture web form for ad landing pages yet.
- **Priority**: ship (A) now (this ADR); (B) gets its own implementation
  plan and is picked up as a separate, later piece of work — not started
  as part of landing this ADR.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **A — Field curation only** | Small, fast, zero blast radius | Doesn't address the CRM/ad-campaign ask | **Chosen — do regardless of B, it was a real bug** |
| **B — Adopt ERPNext's native Lead/Opportunity/Campaign**, convert to `Customer` on lease intent | Reuses built, tested ERPNext CRM instead of building one; `Customer` keeps one stable meaning; same "extend, don't replace" pattern already used elsewhere | New portal UI to build (Leads list, conversion action); Lead's own fields skew B2B too, need light curation; frontend must fully hide the ERPNext origin | **Chosen for the CRM ask — implementation deferred to its own follow-up** |
| **C — New custom `Tenant` doctype**, `Customer` becomes internal-only | Perfectly curated fields, zero ERPNext CRM baggage | Biggest blast radius by far — `Lease Agreement.customer`, `Maintenance Request.customer`, `Sales Invoice`/`PDC Entry`/`Security Deposit.customer`, tenant-portal-login (`User Permission` scoped to Customer) all reference `Customer` directly today; weeks of work for a problem (A) already solves | **Rejected** — Imran confirmed Option B over this |
| **D — Bespoke CRM module from scratch** | Full control over fields/workflow | Reinvents what ERPNext already ships for free | **Rejected** |

## Consequences

### Positive

- (A) ships immediately with no risk — the edit form now matches
  whatever a business has curated its record types down to, for any
  doctype, not just Customer.
- (B)'s direction avoids a costly Customer/Tenant doctype split while
  still giving these businesses a real lead pipeline later, built on
  proven ERPNext CRM machinery rather than bespoke code.
- `Customer` keeps one stable meaning across the whole app — nothing
  that currently depends on it (Lease Agreement, Maintenance, PDC,
  Security Deposit, tenant portal login) needs to change.

### Negative

- (B) is deferred scope — Imran's original CRM question isn't answered
  with working software yet, only a direction and constraints. Needs its
  own implementation plan before any Lead-pipeline code is written.
- The "hide all ERPNext references from the frontend" constraint means
  the Lead-pipeline UI can't reuse ERPNext Desk's own Lead views/kanban
  at all — it has to be built fresh in the custom React portal, same
  effort level as every other doctype this portal already wraps.

## Implementation

- Feature file (done, this ADR): `vault/custom-module/features/tenant-field-curation.md`
- Feature file (planned, separate follow-up): `vault/custom-module/features/lead-management.md`
