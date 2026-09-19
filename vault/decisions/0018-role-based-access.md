# ADR-0018: Role-Based Access Control (Leasing Agent, Accountant, Maintenance Staff)

- **Status**: accepted
- **Date**: 2026-08-28
- **Deciders**: Imran
- **Supersedes**: refines the original (pre-ADR, 2026-08-16) `role-based-access.md` 4-role sketch
- **Superseded by**: -

## Context

`compliance-security/features/role-based-access.md` is the other P0 item in
`IMPLEMENTATION-PLAN.md`, but it predates the ADR workflow (written
2026-08-16, `related_adr: []`) and conflicts with the 2026-08-24 findings
doc (`2026-08-24-ideal-product-vs-current-state.md`, P8), which names 5
roles (Admin, Leasing Agent, Accountant, Maintenance Staff, Renter) instead
of the original 4 (no Accountant). This ADR reconciles that and fixes the
design before any code — permissions touch every DocType in the app, and
staff currently run as System Manager, so this is on AGENTS.md's
stop-and-ask list.

**Verified live before drafting** (`realestate.nnuggets.com`, 2026-08-28):
the site has exactly 3 User records — `Administrator`, `Guest`, and
`imran@rafais.com` (System Manager). Zero other staff accounts and zero live
tenant portal logins currently exist — every tenant test login mentioned in
earlier CHANGELOG entries was created for a live test and deleted afterward.
This is a build-ahead-of-need decision, not a live-incident patch — there's
room to get the design right.

**The landmine this design has to account for**, from this session's own
evidence (`landlord-payables.md`'s PDC Entry IDOR, PR #11): Frappe's `User
Permission` scoping does not restrict a document whose restricted link
field is empty, and `apply_strict_user_permissions` is off on this site
(confirmed again live today). "Maintenance Staff scoped to assigned
buildings" has exactly that shape.

## Decision

Adopt the 5-role model: **Admin** (System Manager, unchanged), **Leasing
Agent**, **Accountant**, **Maintenance Staff**, **Tenant** (ADR-0009,
unchanged).

### Permission matrix

| DocType | Leasing Agent | Accountant | Maintenance Staff |
| --- | --- | --- | --- |
| Building, Unit, Landlord, Head Lease | Read/Write/Create | Read only | No access |
| Lease Agreement, Customer | Read/Write/Create | Read only | No access |
| Sales Invoice, Purchase Invoice, Journal Entry, GL Entry, PDC Entry, Security Deposit | Read only | Read/Write/Create | No access |
| Maintenance Request | Read/Write/Create (all) | No access | Read/Write, scoped to assigned buildings only |
| Cheque Bank | Read only | Read/Write/Create | No access |

Confirmed with Imran: Leasing Agent gets full (not read-only) access to
Landlord/Head Lease data — managing a Building day-to-day requires seeing
what's owed to its landlord (e.g. checking head-lease terms before a
sublease renewal). Accountant is the only role with write access to the
books, which is what makes gating the Journal Entry create/list UI
(ADR-0015) behind a real role coherent rather than arbitrary.

### Maintenance Staff building-scoping

Not `User Permission` on `Unit`/`Building` directly — Maintenance Request
has no direct `building` field, only `unit`, and Frappe's User Permission
auto-propagation doesn't cross that second hop.

Instead: `User Permission` on **Cost Center** (Maintenance Request already
has a direct `cost_center` Link, and Cost Center is 1:1 with Building —
`cost-center-per-building.md`). A Maintenance Staff user gets one User
Permission row per Building they're assigned to, scoped via `cost_center`.
Hardened with a `permission_query_conditions` + `has_permission` pair on
Maintenance Request (same pattern as the PDC Entry fix) that explicitly
**denies** — not silently allows — any row whose `cost_center` is unset,
for anyone without System Manager or Leasing Agent.

### Sequencing & verification

All 5 roles' DocPerm rows ship in one PR (one fixture, not independent
slices). Verified with throwaway test Users (`frappe.set_user()`, one per
role, both `frappe.get_list` **and** direct `frappe.get_doc` on a row that
role shouldn't see — `get_all` bypasses permissions and would give a false
pass), then deleted — mirroring the PDC Entry IDOR verification, since no
real non-Admin staff account exists yet to test against.

### Out of scope

- Journal Entry create/list UI itself (`accounting-reports.md`) — this ADR
  only unblocks it; the UI is tracked separately.
- Any change to the existing Tenant role or ADR-0009's mechanism.
- Role Profiles — plain Roles + DocPerm rows are sufficient for 5 roles.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Keep the stale 4-role model (no Accountant) | Less work | Leasing Agent would get accounting write-access, undermining the point of separating property management from the books | rejected |
| Maintenance Staff scoping via `User Permission` on `Unit` directly | Simpler | Far more rows per staff member (per-unit vs per-building) and no hardening against the empty-field IDOR shape | rejected |
| Do nothing until a real hire happens | Zero speculative work | RBAC is explicitly P0; building it calmly ahead of a hire beats building it under pressure the day someone new needs an account | rejected |
| Adopt the 5-role model, Leasing Agent full access to Landlord/Head Lease | Matches how a Leasing Agent actually needs to work a Building day-to-day | None material | **chosen** |

## Consequences

### Positive

- Imran can onboard real staff with least-privilege accounts on day one
  instead of everyone defaulting to System Manager.
- Closes gap G8/P8 from the 2026-08-24 findings doc.
- Gives Accountant a real reason to exist, making the Journal Entry
  RBAC-gate in ADR-0015 coherent.

### Negative

- One more fixture to keep in sync as new DocTypes are added.
- Verified only against throwaway test Users, not a real Leasing
  Agent/Accountant/Maintenance Staff account, until Imran's first real hire.

## Implementation

- Feature file: `vault/compliance-security/features/role-based-access.md`
