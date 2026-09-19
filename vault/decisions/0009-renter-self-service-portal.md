# ADR-0009: Renter Self-Service Portal (Tenant Login)

- **Status**: accepted
- **Date**: 2026-08-23
- **Deciders**: Imran
- **Supersedes**: fills in the design left open by the `tenant-portal-ui` /
  `maintenance-portal` feature stubs (both `planned`, no ADR yet)
- **Superseded by**: -

## Context

Every tenant (the renter — a `Customer` record in this app) currently has **no
login at all**. Their only touchpoint with the system is the one-time e-sign
magic link (`/esign?doc=...&signer=...&token=...`) sent during contract
signing. There's no way for them to log back in afterward to see their lease,
download the signed contract, check invoices, or report a maintenance issue —
an admin/staff member has to do all of that on their behalf today.

The request is to give each renter a real account, scoped strictly to their own
data, with two capabilities to start: **view invoices** and **create/track
maintenance requests**. This sits on top of the per-tenant-site architecture in
`0003-multi-tenancy` and the config-driven portal shell in `0006`/`0007` — it's
a per-*renter* concern inside a single tenant's site, distinct from the
Platform Control Plane draft ADR (which provisions whole tenant *companies*).

Checked before writing this: `Maintenance Request` (the DocType already built,
see `maintenance-request-doctype.md`) currently has exactly one permission row
— `System Manager` — so renters can't read or create their own requests today;
that has to change as part of this.

## Decision

**Website User login for renters, scoped by `User Permission`, served from the
same React portal shell with a role-gated nav set.**

Mechanics:

- Each `Customer` gets an associated Frappe **User** (Website User type, no
  System Manager/Desk access) — created at tenant-creation time (mirrors the
  existing `create_tenant` flow) or backfilled for existing tenants, with a
  "Tenant" role.
- A `User Permission` row (`User` → `Customer`, applied to `Customer`, `Lease
  Agreement`, `Sales Invoice`, `Maintenance Request`) restricts every query a
  logged-in tenant makes to rows linked to *their own* Customer — enforced by
  Frappe's permission engine server-side, not just hidden in the UI.
- `Maintenance Request` gets a new `Tenant` permission row: create + read,
  `if_owner`-style scoping via the same User Permission mechanism (no delete,
  no access to `spares_used`/`total_cost`/`assigned_to` internals — those stay
  staff-only fields, hidden from the tenant-facing form).
- Same React SPA (`ui/`), same login page and session-cookie auth already
  built. `portal_nav_items` (the hook from `0006`) becomes **role-aware**:
  staff roles get the existing 9-section nav; a user whose only role is
  `Tenant` gets a reduced set — My Lease, My Invoices, My Maintenance — reusing
  the existing `DetailView`/`ResourceListView` components, not a new frontend.
- New sections:
  - **My Lease** — read-only Lease Agreement detail (reuses the existing detail
    view + the "View signed contract" download already built), no edit access.
  - **My Invoices** — list of `Sales Invoice` linked to their Customer, status
    (paid/unpaid/overdue), PDF download.
  - **My Maintenance** — list of their own `Maintenance Request`s + a "New
    Request" form (issue type, priority, description — the same fields staff
    see minus the internal cost/assignment fields).

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Separate tenant-only mini frontend (new app/build) | Fully independent surface, can't leak staff nav by accident | New toolchain to maintain; duplicates components already built for the admin portal | rejected |
| B. Reuse the existing portal shell with a role-gated nav | No new frontend; `0006` already designed the shell to be config-driven; reuses existing detail/list components and API layer | Nav-gating logic must be airtight — a bug here is a data-exposure bug, not just a UX one; permission enforcement must live server-side (User Permission), the nav hiding is only a convenience layer | **chosen** |
| C. Give renters access to the existing Desk (`/app`) with restricted roles | Zero new UI work | Confusing/unusable ERPNext chrome for a renter; also cuts against `0006`'s whole reason for building a custom portal | rejected |

## Consequences

### Positive

- Renters get real self-service (lease, invoices, maintenance) without staff
  relaying everything manually.
- Server-side `User Permission` scoping means a UI bug can't leak another
  tenant's invoice or maintenance data — the permission engine is the actual
  gate, not the nav.
- No new frontend stack; extends the same shell, components, and API pattern
  already proven for the admin portal.

### Negative

- `Maintenance Request` permissions need real design (own-record-only create/
  read, internal fields hidden) — not just "add a role."
- User/Customer lifecycle gets more surface: creating a tenant now also means
  creating (or deliberately deferring) a login for them; deactivation on lease
  end needs a policy.
- Two audiences now share one shell (staff vs renter) — nav-gating regressions
  need test coverage, since a mistake here is a security bug.

## Implementation

- Feature file: `vault/ui-portal/features/tenant-portal-ui.md` (exists as a stub, to be filled in on approval)
- Feature file: `vault/maintenance/features/maintenance-portal.md` (exists as a stub, to be filled in on approval)
- Not started — awaiting approval before an implementation plan is written.
