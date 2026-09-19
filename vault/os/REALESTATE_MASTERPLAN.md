# Real Estate OS — Master Plan

## Product Requirements Document: Real Estate Leasing SaaS

**2026-08-27**: this file is the Real Estate OS-specific content
extracted from the former `vault/MASTERPLAN.md`, which is now
`vault/PLATFORM_STRATEGY.md` (platform-wide concerns: multi-tenancy,
modularization, technical stack, non-functional requirements — all now
shared across every OS, not just this one). Anything real-estate-specific
stays here.

### 1. Overview

Requirements for Real Estate OS — a real-estate arbitrage SaaS. Property
managers lease whole buildings (head lease) and sublet units to tenants.
Full automation of leasing, billing, accounting, and maintenance.

Extends the shared ERP core (Customer, Sales Invoice, Accounting — see
`PLATFORM_STRATEGY.md` §3) with real-estate-specific custom DocTypes
(Building, Unit, Lease Agreement, Head Lease, Maintenance Request). Uses
e-sign as a required module (see `PLATFORM_STRATEGY.md` §2.2 and
`vault/esign/`) for contract signing.

### 2. User Roles

| Role | Permissions & Responsibilities |
| --- | --- |
| Admin (Super User) | Full access. Manage buildings/units, all leases/contracts/payments/maintenance. Accounting reports + cost-center analytics. |
| Leasing Agent | Create/manage customers, units, leases. Generate contracts, send for e-sign, track signing. View payment schedules; cannot edit accounting entries. |
| Maintenance Staff | Receive/resolve requests. Log work, spare parts, update status. Limited to maintenance module + assigned buildings. |
| Customer (Tenant) | Portal only. View lease, payments, invoices. Submit/track maintenance requests. Download signed contracts. |

Status: only Admin (System Manager) and Tenant actually exist today —
Leasing Agent and Maintenance Staff are still gap G8 in
`vault/findings/2026-08-24-ideal-product-vs-current-state.md`, tracked in
`vault/IMPLEMENTATION-PLAN.md`.

### 3. Modules

#### 3.1 Customer (Extended)

Base: the shared ERP core's `Customer` doctype.

- Kept fields: `customer_name`, `email_id`, `mobile_no`.
- Added custom fields (fixtures): `preferred_contact_method`,
  `emergency_contact_name`, `emergency_contact_phone`, `id_proof_type`,
  `id_proof_number` (KYC), `employer_name`, `monthly_income` (lease
  approval).
- Curated to just these fields in the portal (`Portal Field Visibility` —
  Customer carries 78 stock fields from the shared core plus these 7; the
  edit/view form shows only what's relevant here, not all 85).

#### 3.2 Building & Unit Management

**Building**: `building_name`, `address`, `total_units`, `amenities`,
`property_manager` (Link User), `status` (Active / Under Maintenance /
Vacant), `landlord` (Link Landlord, mandatory — a self-owned building
links to a placeholder "Own Building" Landlord, never left blank),
`cost_center` (Link, auto-created).

**Unit**: `building` (Link), `unit_number`, `unit_type`, `square_footage`,
`has_balcony`, `floor_number`, `monthly_rent`, `status` (Vacant /
Occupied / Reserved / Under Maintenance), `amenities`.

One Building → Many Units. Bulk unit creation (generate N units,
sequential numbering) is built and used throughout.

#### 3.3 Lease & Contract Management

`Lease Agreement`: `customer`, `unit`, `start_date`, `end_date`,
`monthly_rent`, `payment_frequency`, `security_deposit`, `contract_pdf`,
`esign_status` (signing-workflow state), `lease_status` (the real tenancy
lifecycle — Draft/Pending Signature/Active/Expiring Soon/Expired/Renewed/
Terminated/Cancelled, distinct from `esign_status`, which freezes at
"Signed" forever), `esign_provider`, `esign_envelope_id`.

Workflow: agent creates the lease (Customer + vacant Unit only) → contract
PDF generated → sent for e-sign (via whichever provider is active, see
`PLATFORM_STRATEGY.md` §2.2) → on full signature: `lease_status` →
Active, unit → Occupied, first rent invoice generated (if actually due —
not invoiced early, regardless of how far in advance the lease was
signed).

#### 3.4 Payment Processing

**Recurring invoicing**: `Rent Schedule` child table on Lease Agreement
(due dates + amounts, derived from term + frequency); a daily job
generates the Sales Invoice once a period is actually due.

**Post-dated cheques (PDC)**: `PDC Entry` (Incoming for tenants, Outgoing
for landlords — one doctype, ADR-0011), `check_number`, `check_date`,
`amount`, `tenant_bank` (Link Cheque Bank), `status` (Pending / Deposited
/ Cleared / Bounced / Cancelled). Deposit is a **manual** staff action
(ADR-0014 — not automatic; a business handling literal paper cheques
decides which to physically take to the bank). Clearing auto-creates and
reconciles a Payment Entry against the linked invoice.

#### 3.5 Accounting Integration

The shared ERP core (Company, Chart of Accounts, GL, P&L, Balance Sheet —
`PLATFORM_STRATEGY.md` §3) does the heavy lifting. Real-estate-specific
on top of it:

- **Cost Center per Building** — auto-created on Building save.
- **Head Lease & landlord payables** (ADR-0011) — the cost side of the
  arbitrage model: committed monthly rent owed to a landlord, its own
  payment schedule, posting as Purchase Invoices.
- **Building Profitability report** — rent income minus landlord cost
  minus maintenance cost, grouped by Cost Center (building).
- Portal-native GL/Trial Balance/P&L/Balance Sheet/Journal Entry viewers
  (ADR-0015) — real-estate account naming applied via this OS's own
  Chart-of-Accounts fixture (never a platform-shared mapping —
  `PLATFORM_STRATEGY.md` §3).
- Security deposits as a refundable liability — not yet posted to the GL
  (gap G10, tracked in `IMPLEMENTATION-PLAN.md`).

#### 3.6 Maintenance Request Portal & Queue

`Maintenance Request`: `customer`, `unit` (auto-filled from the tenant's
active lease), `issue_type`, `description`, `priority`, `status`,
`assigned_to`, `spares_used` (child table), `labor_hours`, `total_cost`,
`cost_center` (from the unit's building). Cost currently computed but not
yet posted to the GL (gap G9, tracked in `IMPLEMENTATION-PLAN.md`).

### 4. Key Features

- **Bulk Unit Generator** — generate N sequentially-numbered units per
  building in one action.
- **Automated Contract Generation & E-Sign** — Jinja print format +
  whichever e-sign module is active (`PLATFORM_STRATEGY.md` §2.2).
- **Recurring Invoicing & Payment Schedule** — auto-repeat per lease
  terms; 3-day-before reminders.
- **Post-Dated Cheque Processing** — manual deposit selection (ADR-0014),
  clearance auto-reconciles the invoice.
- **Maintenance Request Routing & Cost Allocation** — tenant portal
  intake, staff assignment, cost tracked to the building's cost center.

### Appendix: Representative Code Patterns

**Bulk unit creation** (the actual shape, simplified — see
`real_estate_os/api.py::generate_units` for the real, capped, validated
version):

```python
import frappe

@frappe.whitelist()
def generate_units(building_name, num_units):
    building = frappe.get_doc("Building", building_name)
    for i in range(1, num_units + 1):
        frappe.get_doc({
            "doctype": "Unit",
            "unit_number": f"{i:03d}",
            "building": building.name,
            "status": "Vacant",
        }).insert()
```

**Daily PDC-related jobs** (scheduler hook shape — see `hooks.py` for the
real, current job list, which does *not* include an automatic deposit
sweep per ADR-0014):

```python
scheduler_events = {
    "daily": [
        "real_estate_os.payments.lease_lifecycle.transition_lease_statuses",
        "real_estate_os.payments.invoicing.generate_due_invoices",
        "real_estate_os.payments.invoicing.send_reminders",
        "real_estate_os.payments.landlord_payables.generate_due_payables",
    ]
}
```

Note: an earlier version of this document included a `docusign_utils.py`
sample that called DocuSign's API directly, hardcoded into this OS's own
codebase. That pattern is now explicitly what `PLATFORM_STRATEGY.md` §2.2
says not to do — e-sign (and any future integration) is a separate module
app, dispatched via hooks, never imported directly here. See
`vault/decisions/0017-platform-modularization.md`.

## Related

- Platform-wide concerns (multi-tenancy, modularization, technical
  stack): `vault/PLATFORM_STRATEGY.md`
- Live task list: `vault/IMPLEMENTATION-PLAN.md`
- Domain indexes: `vault/custom-module/`, `vault/payments-accounting/`,
  `vault/maintenance/`, `vault/ui-portal/`, `vault/esign/`,
  `vault/compliance-security/`, `vault/multi-tenancy/`
