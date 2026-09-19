---
status: done
owner: developer-1
domain: compliance-security
created: 2026-08-16
updated: 2026-08-28
related_adr: ["0018-role-based-access"]
---

# Role-Based Access Control

## Summary

Frappe RBAC for 5 roles (Admin, Leasing Agent, Accountant, Maintenance
Staff, Tenant) with least-privilege permissions and data isolation. Rewrites
the original 2026-08-16 sketch (4 roles, no ADR) per ADR-0018, which
reconciled it against the 2026-08-24 findings doc's 5-role model and fixed
the Maintenance Staff building-scoping design ahead of any code.

## Requirements

- **Admin** (System Manager, unchanged): full access.
- **Leasing Agent**: full Read/Write/Create on Building, Unit, Landlord,
  Head Lease, Lease Agreement, Customer, Maintenance Request (all); read
  only on Sales/Purchase Invoice, Journal Entry, GL Entry, PDC Entry,
  Security Deposit, Cheque Bank.
- **Accountant**: full Read/Write/Create on Sales/Purchase Invoice, Journal
  Entry, GL Entry, PDC Entry, Security Deposit, Cheque Bank; read only on
  Building, Unit, Landlord, Head Lease, Lease Agreement, Customer; no
  access to Maintenance Request.
- **Maintenance Staff**: Read/Write on Maintenance Request only, scoped to
  buildings they're assigned to; no access to any other DocType above.
- **Tenant**: unchanged (ADR-0009) — portal only, own lease/invoices/
  maintenance requests.

## Design

- Plain Roles + `DocPerm` fixture rows (not Role Profiles — 5 roles doesn't
  need that layer of indirection).
- Maintenance Staff scoping: `User Permission` on **Cost Center**, not
  `Unit`/`Building` directly — Maintenance Request's only building-adjacent
  field is `unit` (a second hop to Building), which Frappe's User
  Permission auto-propagation doesn't cross. `cost_center` is a direct Link
  field already on Maintenance Request and is 1:1 with Building
  (`cost-center-per-building.md`), so a User Permission row per assigned
  Building's Cost Center is the correct scoping mechanism.
- Hardened against the PDC Entry IDOR class of bug (PR #11): new
  `permission_query_conditions` + `has_permission` pair on Maintenance
  Request that explicitly denies (not silently allows) any row with an
  unset `cost_center`, for anyone without System Manager or Leasing Agent
  — `apply_strict_user_permissions` is off on this site, so an empty Link
  field is visible to everyone by Frappe's default behavior unless this is
  added.
- Fixture: new `Custom DocPerm`/`DocPerm` rows added the same way the
  existing Tenant-role permissions were added (`create_tenant_role_and_
  permissions.py` patch pattern) — new roles + permission rows via a patch,
  not manual Desk clicks, so a fresh `install-app` gets them too.

## Implementation Plan

- [x] ADR approved — `0018-role-based-access`
- [x] Create `Leasing Agent`, `Accountant`, `Maintenance Staff` roles —
      PR #34, `create_staff_roles_and_permissions.py` patch. Used a custom
      `_grant` helper instead of `frappe.permissions.add_permission`:
      `add_permission` only sets one ptype flag per call and silently
      no-ops on a second call for the same (doctype, role, permlevel) —
      confirmed by reading its source — which would have made
      read+write+create grants impossible via repeated calls.
- [x] DocPerm rows per the permission matrix, for all 3 new roles — PR #34.
      Verified live via `Custom DocPerm` query: every sampled (doctype,
      role) pair matches the intended matrix exactly.
- [x] `permission_query_conditions`/`has_permission` hooks on Maintenance
      Request (Cost Center-based scoping, deny-on-unset hardening) — PR #34.
- [x] Found and fixed during implementation: `Maintenance Request.
      cost_center` had NO auto-population at all before this (only
      `demo_data.py`'s seeding ever set it — the ADR's assumption that it
      mirrored the existing `unit` auto-resolution was wrong). Added
      resolution in `validate()` (Unit -> Building -> Cost Center). No
      backfill needed — zero live Maintenance Requests existed at deploy
      time.
- [x] Verified each role with throwaway test Users — 2026-08-28, live on
      production, all cleaned up afterward:
      - Maintenance Staff (assigned to one real Building's Cost Center):
        `get_list` returned only that building's request; direct access
        via the real portal API path (`api.get_doc_detail`, not a bare
        `frappe.get_doc` — which never enforces permissions by itself)
        correctly raised `PermissionError` for a different building's
        request AND for a request with an unset `cost_center`.
      - Leasing Agent: `get_list` saw all 3 test requests (unscoped, per
        the matrix); `has_permission("Building", "write")` True,
        `has_permission("Sales Invoice", "write")` False.
      - Accountant: `has_permission("Sales Invoice", "write")` True,
        `has_permission("Building", "write")` False, zero permission on
        Maintenance Request.
      - **Regression check that mattered most**: a real Tenant login
        (customer-scoped, ADR-0009) still saw all 3 of their own test
        requests regardless of `cost_center` — confirms the new
        Maintenance-Staff-only scoping doesn't interfere with Tenant's
        existing, unrelated visibility mechanism on the same doctype.

## Acceptance Criteria

- [x] Each role limited to exactly its permission-matrix scope — verified
      live per role via throwaway test Users, not just by reading the
      DocPerm fixture (see above)
- [x] A Maintenance Staff user sees only Maintenance Requests for their
      assigned building(s)'s Cost Center, both via list view and the real
      document-access API path
- [x] A Maintenance Request with an unset `cost_center` is NOT visible to
      any Maintenance Staff user — verified live (regression check for the
      PDC Entry IDOR class of bug)
- [x] Existing Tenant role behavior (ADR-0009) unaffected — verified live
      with a real (throwaway) tenant login

## Known follow-up gaps (flagged, not fixed here)

- No per-role portal nav filtering yet — Leasing Agent/Accountant/
  Maintenance Staff all get the same full 9-section nav as System Manager
  (`STAFF_ROLES` in `api.py`); DocPerm is the actual security boundary, but
  a Maintenance Staff session would see nav items (Properties, Landlords,
  Accounts) that render empty for them. Tracked in `IMPLEMENTATION-PLAN.md`
  as a P2/P3 polish item, not blocking this feature's completion.
- Maintenance Staff has permlevel-1 **read-only** on Maintenance Request
  (not write) — they can see `assigned_to`/`spares_used`/`labor_hours`/
  `total_cost`/`cost_center`/`work_order` but not edit them via the raw
  form, since `cost_center` sits at that same permlevel and is also the
  scoping field itself (write access there would let a Maintenance Staff
  user self-escalate to another building). Logging completed work
  (spares/labor/cost) needs a dedicated action, mirroring PDC Entry's
  Clear/Bounce action pattern, rather than raw field-edit access — not
  built in this pass.
- Verified only against throwaway test Users — no real non-Admin staff
  account exists yet (0 staff besides the owner as of 2026-08-28); re-verify
  against Imran's first real hire once one exists.
- **Resolved 2026-09-02**: the "Admin (System Manager): full access" line
  above turned out to be false in practice — found live when rastec.admin
  (a real tenant's first user, System Manager only) hit a raw permission
  error adding a bank account. Bank Account/Sales Invoice/Purchase
  Invoice/Journal Entry/GL Entry only ever had Custom DocPerm rows for
  Accountant, never System Manager. Fixed via
  `grant_system_manager_core_accounts_access` (real-estate PR #76) — see
  `payments-accounting/features/bank-account-setup.md` and the
  2026-09-02T06:00:00Z CHANGELOG entry for detail. Any *future* core
  ERPNext doctype this app wires into the portal should grant System
  Manager the same as Accountant/Leasing Agent up front, not just the
  narrower staff role — this class of gap is easy to reintroduce
  one doctype at a time.

## Related

- Domain index: `vault/compliance-security/compliance-security.md`
- ADR: `vault/decisions/0018-role-based-access.md`
- Finding: `vault/findings/2026-08-24-ideal-product-vs-current-state.md` (P8/G8)
- Feature: `ui-portal/features/tenant-portal-ui.md`,
  `payments-accounting/features/cost-center-per-building.md`,
  `payments-accounting/features/landlord-payables.md` (PDC Entry IDOR precedent)
