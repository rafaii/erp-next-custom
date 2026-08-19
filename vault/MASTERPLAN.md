# SYSTEM DESIGN

## Product Requirements Document: Real Estate Leasing SaaS on ERPNext

### 1. Overview

Requirements for RealEstateOS — multi-tenant SaaS on ERPNext (Frappe Framework) for real-estate arbitrage. Property managers lease whole buildings + sublet units to tenants. Full automation of leasing, billing, accounting, maintenance.

Extends ERPNext core modules (Customer, Sales Invoice, Accounting) + custom DocTypes (buildings, units, leases, maintenance). Custom tenant portal. Integrates e-sign providers (DocuSign, ZohoSign) for contracts.

### 2. User Roles
Role	Permissions & Responsibilities
Admin (Super User)	Full access. Manage tenants (SaaS), buildings/units, all leases/contracts/payments/maintenance. Accounting reports + cost-center analytics.
Leasing Agent	Create/manage customers, units, leases. Generate contracts, send for e-sign, track signing. View payment schedules; cannot edit accounting entries.
Maintenance Staff	Receive/resolve requests. Log work, spare parts, update status. Limited to maintenance module + assigned buildings.
Customer (Tenant)	Portal only. View lease, payments, invoices. Submit/track maintenance requests. Download signed contracts.

### 3. Modules

3.1 Customer (Extended)

Base: ERPNext built-in Customer DocType.

Customizations:

    Keep fields: customer_name, email_id, phone, job_title, company (B2B lease).

    Add custom fields via Customize Form (exported as fixtures):

        preferred_contact_method (Select: Email, Phone, SMS)

        emergency_contact_name, emergency_contact_phone

        id_proof_type, id_proof_number (KYC)

        employer_name, monthly_income (lease approval)

Implementation: use hooks.py fixtures.


hooks.py

```
fixtures = [
    {"dt": "Custom Field", "filters": [["dt", "=", "Customer"]]}
]
```

3.2 Building & Unit Management

Custom DocTypes:
Building

    building_id (Auto-name: BLD-{#####})

    building_name (Data)

    address (Text)

    total_units (Int)

    amenities (Text: pool, gym, parking, etc.)

    property_manager (Link: User)

    status (Select: Active, Under Maintenance, Vacant)

Unit

    unit_id (Auto-name: UNT-{#####})

    building (Link: Building)

    unit_number (Data: e.g., "001", "002A")

    unit_type (Select: 1-Bedroom, 2-Bedroom, Studio, Penthouse)

    square_footage (Float)

    has_balcony (Check)

    floor_number (Int)

    monthly_rent (Currency)

    status (Select: Vacant, Occupied, Reserved, Under Maintenance)

    amenities (Text: in-unit washer, AC, etc.)

Relationship: One Building → Many Units (via building link).

UI Features:

    Bulk unit creation: auto-generate 20 units (001–020) with one click (Server Script / Custom Button).

    Tabbed form views: building details, unit list, occupancy status.

3.3 Lease & Contract Management

Custom DocType: Lease Agreement
Field	Type	Description
lease_id	Auto-name	LSE-{YYYY}-{#####}
customer	Link: Customer	Tenant
unit	Link: Unit	Rented unit
start_date	Date	Lease commencement
end_date	Date	Lease expiry
monthly_rent	Currency	Auto-filled from Unit
payment_frequency	Select	Monthly, Quarterly, Annual
security_deposit	Currency	
contract_pdf	Attach	Generated PDF
esign_status	Select	Draft, Sent, Signed, Declined
esign_provider	Select	DocuSign, ZohoSign, In-built
esign_envelope_id	Data	External provider reference

Workflow:

    Agent creates Lease Agreement: pick Customer + Unit.

    Validate unit is Vacant.

    On Save: generate contract PDF via Print Format (custom HTML/Jinja).

    On Submit: trigger e-sign:

        Call DocuSign/ZohoSign API via Server Script or Webhook.

        Email contract to customer with signing link.

        Set esign_status "Sent".

    On signature done (webhook callback): set esign_status "Signed", attach signed PDF, auto-create:

        Sales Invoice (recurring)

        Payment Schedule (child table or Subscription DocType)

Print Format: custom Jinja template, dynamic fields (customer, unit, rent, dates, terms).
3.4 Payment Processing
Recurring Invoices

    Use ERPNext Sales Invoice + Payment Schedule child table.

    Alternative: Subscription DocType (simple_subscription app or custom) to auto-generate invoices.

    Configure auto_repeat in hooks; invoices on 1st each month.

Post-Dated Checks (PDC)

Custom DocType: PDC Entry

    pdc_id (Auto-name)

    lease (Link: Lease Agreement)

    customer (Link: Customer)

    check_number (Data)

    check_date (Date)

    amount (Currency)

    bank_account (Link: Bank Account)

    status (Select: Pending, Deposited, Cleared, Bounced)

    deposit_date (Date)

Automation:

    Daily job (hooks.py scheduler): find PDCs with check_date = today, status "Pending".

    Auto-create Payment Entry against Sales Invoice.

    Set PDC "Deposited".

    Email admin if PDC bounces (bank webhook needed).

Future Phase: e-transfer APIs (Interac, Stripe) for automated collections.
3.5 Accounting Integration

Out-of-box: ERPNext Accounting Module (GL, P&L, Balance Sheet).

Customizations:

    Cost Center per Building: auto-create on Building creation (Server Script).

    Auto-posting: Sales Invoices (rent), Payment Entries (PDC), Expense Claims (spares) → correct Cost Center.

    Custom Report: "Building Profitability" — P&L grouped by Cost Center (Building).

Integration Points:

    Maintenance spares (Inventory) → Expense Claim → GL Entry.

    Lease income → Sales Invoice → GL Entry.

    Security deposits → Liability Account (refundable).

3.6 Maintenance Request Portal & Queue

Custom DocType: Maintenance Request
Field	Type	Description
request_id	Auto-name	MNT-{YYYY}-{#####}
customer	Link: Customer	Tenant submitting
unit	Link: Unit	Auto-filled from Customer's active lease
issue_type	Select	Plumbing, Electrical, HVAC, Appliance, Other
description	Text	Detailed issue
priority	Select	Low, Medium, High, Emergency
status	Select	Open, Assigned, In Progress, Resolved, Closed
assigned_to	Link: User	Maintenance staff
work_order	Link: Work Order (optional)	For complex jobs
spares_used	Table	Child table: Item, Quantity, Rate
labor_hours	Float	
total_cost	Currency	Auto-calculated
cost_center	Link: Cost Center	Auto-filled from Unit's Building

Workflow:

    Customer submits via Portal (web form / Frappe Web View).

    Create Maintenance Request, status "Open".

    Admin assigns staff (assigned_to).

    Staff sets "In Progress", logs spares_used + labor_hours.

    On "Resolved":

        Auto-create Expense Claim / Stock Entry for spares.

        Post cost to Building Cost Center.

        Email customer completion.

    Customer rates (optional).

Portal: custom Frappe Web View, auth via API.

### 4. Key Features
4.1 Easy Building/Unit Creation

    Bulk Unit Generator: click "Generate Units" to auto-create N units (e.g., 20), sequential numbering.

    Customizable: unit type, sq-ft, balcony, amenities per unit.

    Occupancy Map: dashboard, building layout, color-coded status (Vacant = Green, Occupied = Red).

4.2 Automated Contract Generation & E-Sign

    Print Format: Jinja-based PDF template, dynamic lease terms.

    E-Sign Integration: DocuSign/ZohoSign API calls via whitelisted Python methods.

    Webhook Listener: endpoint for signature-completion callbacks → update Lease status.

4.3 Recurring Invoicing & Payment Schedule

    Auto-Repeat: invoices monthly/quarterly per lease terms.

    Payment Schedule: child table in Sales Invoice, due dates + amounts.

    Email Reminders: email 3 days before due (scheduler).

4.4 Post-Dated Check Processing

    PDC Dashboard: upcoming checks by date/building/customer.

    Auto-Deposit Job: daily cron → Payment Entries for due PDCs.

    Bounce Handling: flag bounced, notify admin, late-fee invoice.

4.5 Maintenance Request Routing & Cost Allocation

    Tenant Portal: simple issue form, photo upload.

    Auto-Assignment: round-robin / skill routing to staff.

    Cost Tracking: spares + labor auto-post to Building Cost Center.

### 5. Non-Functional Requirements
Requirement	Specification
Responsive UI	Frappe Desk + portal mobile-friendly (Bootstrap).
Security	Role-based perms (Frappe). API endpoints whitelisted + authenticated.
Audit Trails	Frappe versioning + docstatus on all DocTypes.
Performance	< 2s portal load. Background jobs for heavy tasks (invoices, PDC).
Scalability	100+ tenants (SaaS), separate DB per tenant.
Compliance	Data isolation per tenant (GDPR). Encrypted PDC + customer data at rest.
### 6. Multi-Tenancy Architecture (SaaS)

Approach: Tenant-per-Site (separate database).
Aspect	Implementation
Isolation	Each customer = separate Frappe Site, own DB + file store.
Codebase	Single codebase (custom app) across all sites via Bench.
Deployment	Frappe Bench or Frappe Operator (Kubernetes).
Onboarding	Automated script: bench new-site tenant1.com → install-app real_estate_os.
Billing	External SaaS billing (e.g., Stripe) tracks site subscriptions.
Upgrades	Centralized: bench update → push to all sites.

Why not single-DB multi-tenancy?

    ERPNext/Frappe: no cross-DB queries within one site.

    Legal: strict data isolation (tenant P&L not queryable by another).

    Easier backup/restore/audit per tenant.

Alternative (small scale): single DB, Company-based scoping (ERPNext multi-company). Not for true SaaS — weaker isolation.

### 7. Milestones & Phases
Phase 1: Core Modules (Weeks 1–6)

    Custom app scaffold (bench new-app real_estate_os).

    DocTypes: Building, Unit, Lease Agreement.

    Customer customization (fixtures).

    Basic CRUD forms.

    Unit bulk-creation script.

    Basic dashboard (occupancy, lease status).

Phase 2: E-Sign & Recurring Invoicing (Weeks 7–10)

    Print Format for lease contracts (PDF).

    DocuSign/ZohoSign integration (API + webhook).

    Lease → Signed → Auto-create Sales Invoice + Payment Schedule.

    Email notifications (contract sent, invoice due).

Phase 3: PDC Automation & Accounting Sync (Weeks 11–14)

    PDC Entry DocType.

    Daily background job for auto-deposit.

    Cost Center per Building (auto-create on Building save).

    Custom P&L report by Building (Cost Center).

    Security deposit handling (Liability Account).

Phase 4: Maintenance Portal & Reporting (Weeks 15–18)

    Maintenance Request DocType + Portal (Web View).

    Spares tracking (Inventory integration).

    Auto-post maintenance costs to Cost Center.

    Tenant feedback/rating system.

    Analytics dashboard (maintenance SLA, cost per building).

Phase 5 (Future): E-Transfers & Advanced Features

    Stripe/Interac integration for automated rent collection.

    Voice AI for maintenance request intake.

    Video generation for property marketing (AI-based).

### 8. Technical Stack
Layer	Technology
Backend	Python (Frappe Framework), MariaDB/PostgreSQL
Frontend	Frappe Desk (Vue.js), Custom Portal (Jinja + JS)
API	REST (auto-generated for all DocTypes), Whitelisted RPC methods
E-Sign	DocuSign API / ZohoSign API (webhooks for callbacks)
Deployment	Frappe Bench (single server) or Frappe Operator (Kubernetes)
SaaS Billing	Stripe (external) + custom Site subscription tracking
### 9. Risks & Mitigations
Risk	Mitigation
E-sign API rate limits	Queue signing requests; webhooks for async completion.
PDC bounce handling	Manual review workflow; late-fee automation.
Multi-tenant upgrade complexity	Staged rollouts (canary sites); automated backup before update.
Portal performance	Cache hot data (unit status, lease details).
### 10. Appendix: Sample Code Snippets
10.1 Bulk Unit Creation (Server Script)


#### Real Estate OS > Server Script > Generate Units
```
import frappe

@frappe.whitelist()
def generate_units(building_name, num_units):
    building = frappe.get_doc("Building", building_name)
    for i in range(1, num_units + 1):
        unit = frappe.get_doc({
            "doctype": "Unit",
            "unit_number": f"{i:03d}",
            "building": building.name,
            "unit_type": "1-Bedroom",
            "square_footage": 750,
            "has_balcony": 1,
            "monthly_rent": 2000,
            "status": "Vacant"
        })
        unit.insert()
    frappe.msgprint(f"Generated {num_units} units for {building_name}")
```

10.2 DocuSign Integration (Whitelisted Method)

#### Real Estate OS > API > docusign_utils.py
```
import frappe
import requests

@frappe.whitelist()
def send_for_signature(lease_id):
    lease = frappe.get_doc("Lease Agreement", lease_id)
    # Generate PDF
    pdf_url = frappe.get_print("Lease Agreement", lease.name)
    # Call DocuSign API
    response = requests.post(
        "https://api.docusign.net/v2.1/accounts/XXX/envelopes",
        headers={"Authorization": f"Bearer {access_token}"},
        json={"envelope_definition": {...}}
    )
    lease.esign_envelope_id = response.json()["envelopeId"]
    lease.esign_status = "Sent"
    lease.save()
    return {"envelope_id": lease.esign_envelope_id}
```

10.3 Daily PDC Processing (Scheduler Hook)


#### hooks.py
```
scheduler_events = {
    "daily": [
        "real_estate_os.real_estate_os.utils.process_pdc"
    ]
}
```

#### utils.py
```
def process_pdc():
    from datetime import date
    today = date.today()
    pdc_list = frappe.get_all("PDC Entry",
        filters={"check_date": today, "status": "Pending"},
        fields=["name", "lease", "amount", "bank_account"]
    )
    for pdc in pdc_list:
        # Create Payment Entry
        pe = frappe.get_doc({
            "doctype": "Payment Entry",
            "payment_type": "Receive",
            "party_type": "Customer",
            "party": frappe.db.get_value("Lease Agreement", pdc.lease, "customer"),
            "paid_amount": pdc.amount,
            "reference_no": pdc.name,
            "reference_date": today
        })
        pe.insert()
        pe.submit()
        # Update PDC status
        frappe.db.set_value("PDC Entry", pdc.name, "status", "Deposited")
    frappe.db.commit()
```

Build native e-sign module in custom Frappe app, skip DocuSign/ZohoSign. Preferable for most SaaS operators — no per-envelope fees, no third-party data residency, full UX control. Design for legal defensibility + technical soundness.

Legal Baseline You Must Hit

Calgary, serving US/Canada: e-sign must satisfy ESIGN Act/UETA (US) + PIPEDA-adjacent provincial laws (Canada). Both need four criteria: intent to sign, consent to electronic business, association of signature with document, reliable record retention. Courts don't require PKI "digital signature" — simple e-sign + strong audit trail is legally sufficient for most commercial leases. Skip PKI certs unless you want Advanced Electronic Signature assurance. Audit trail = main legal shield.
Module Design: In-Built E-Sign
New DocTypes
DocType	Purpose	Key Fields
E-Sign Document	Wraps the Lease Agreement PDF for signing	reference_doctype, reference_name, document_hash (SHA-256), status (Draft/Sent/Viewed/Signed/Declined/Expired), signed_pdf
E-Sign Signer	One row per party who must sign	esign_document, signer_name, email, phone, signing_order, otp_verified, status
E-Sign Audit Log	Immutable, append-only event trail	esign_document, event_type, timestamp, ip_address, user_agent, details
E-Sign Field	Placement of signature/initial/date fields on the PDF	esign_document, signer, field_type, page_no, x_coord, y_coord
Signing Workflow

    Generate document hash — on PDF finalize, compute SHA-256, store before sending. Any tamper breaks hash — tamper-evidence.

    Consent screen — explicit "I consent to sign electronically" checkbox + disclosure (paper alternative, right to withdraw). Satisfies consent requirement; key for B2C tenants.

    Identity verification — OTP via email/SMS before signing page. Record otp_verified = 1 + timestamp. Replaces DocuSign identity-proofing.

    Signature capture — JS signature-pad (canvas, mouse/finger) in portal. Store PNG/base64 + typed-name fallback.

    Log every event to E-Sign Audit Log: sent, viewed (timestamp/IP), consent, OTP verified, each field signed, completed — mirrors DocuSign Certificate of Completion.

    Stamp PDF — server-side, overlay signature image at page/coords (reportlab or pypdf), re-hash final PDF, store both hashes.

    Append Certificate of Completion page — final page with signer name, email, IP, timestamps, document hash. Standard DocuSign/eIDAS practice; same evidentiary structure.

    Lock document — set docstatus = 1 (submitted) in Frappe → immutable via built-in versioning/audit trail.

Optional: Cryptographic Upgrade (Advanced Electronic Signature)

Optional AdES assurance (higher-value commercial leases): use PyHanko (PAdES-compliant PDF signing) for real cryptographic signature + timestamp authority, not just image overlay. Not required for residential legality; strengthens enforceability.
Multi-Tenant Consideration

Per-tenant e-sign provider: Signature Settings singleton DocType, esign_mode (Select: In-Built, DocuSign, ZohoSign). Upsell in-built as cost-saver; keep third-party for tenants with compliance needs.
Revised Module Stack
Component	Tool
PDF generation	Frappe Print Format (Jinja) + reportlab/pypdf for overlay
Signature capture	signature_pad.js (canvas)
Hashing	Python hashlib (SHA-256)
OTP delivery	Existing email/SMS gateway
Audit trail	Custom immutable DocType (no edit permission after insert)
Advanced signing (optional)	PyHanko for PAdES/cryptographic signatures

All inside Frappe app — no external API dependency, no per-signature cost, full data residency, meets legal bar for enforceable e-signatures.
