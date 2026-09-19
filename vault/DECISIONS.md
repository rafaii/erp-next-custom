# Decisions Index (ADRs)

Approved architectural decisions, sequential numbering.

## Approved

| ADR | Title | Status |
| --- | --- | --- |
| [0001-app-repo-structure](0001-app-repo-structure.md) | App & Git Repository Structure | accepted |
| [0002-esign-approach](0002-esign-approach.md) | E-Signature — In-Built vs Third-Party | accepted |
| [0003-multi-tenancy](0003-multi-tenancy.md) | Multi-Tenancy — Tenant-per-Site vs Single-DB | accepted |
| [0004-recurring-invoicing](0004-recurring-invoicing.md) | Recurring Invoicing Mechanism | accepted |
| [0005-counter-signature-pdc](0005-counter-signature-pdc.md) | Two-Party Counter-Signature & PDC-Driven Invoicing | accepted |
| [0006-admin-portal-approach](0006-admin-portal-approach.md) | Admin Portal UI — Headless-Hybrid Custom SPA (Config-Driven) | accepted |
| [0007-admin-portal-react-frontend](0007-admin-portal-react-frontend.md) | Admin Portal Frontend — React SPA (replaces Vue default) | accepted |
| [0008-tenant-provisioning-control-plane](0008-tenant-provisioning-control-plane.md) | Tenant Site Provisioning — Control-Plane UI | accepted |
| [0009-renter-self-service-portal](0009-renter-self-service-portal.md) | Renter Self-Service Portal (Tenant Login) | accepted |
| [0010-pdc-schedule-and-bank-reconciliation](0010-pdc-schedule-and-bank-reconciliation.md) | PDC Schedule Generation & Bank Reconciliation | accepted |
| [0011-head-lease-landlord-payables](0011-head-lease-landlord-payables.md) | Head Lease & Landlord Payables (outgoing PDC, configurable per Building) | accepted |
| [0012-lease-lifecycle-state](0012-lease-lifecycle-state.md) | Lease Lifecycle State (lease_status, distinct Terminated/Cancelled/Expired) | accepted |
| [0013-lead-crm-and-tenant-field-curation](0013-lead-crm-and-tenant-field-curation.md) | Lead/CRM Tracking & Tenant Field Curation (ERPNext Lead, not a Tenant doctype split) | accepted |
| [0014-manual-bank-deposit-workflow](0014-manual-bank-deposit-workflow.md) | Manual Bank-Deposit Workflow for PDC Entries (replaces automatic Pending -> Deposited scheduler sweep) | accepted |
| [0015-portal-native-accounting-reports](0015-portal-native-accounting-reports.md) | Portal-Native Accounting Reports (GL, Trial Balance, P&L, Balance Sheet, Journal Entry via ERPNext's own report engines) | accepted |
| [0016-module-aware-provisioning](0016-module-aware-provisioning.md) | Module-Aware Tenant Provisioning & Per-Module Chart of Accounts | accepted |
| [0017-platform-modularization](0017-platform-modularization.md) | Platform Modularization — Domain OS Apps, Reusable Modules, Extension Tiers | accepted |
| [0018-role-based-access](0018-role-based-access.md) | Role-Based Access Control (Leasing Agent, Accountant, Maintenance Staff) | accepted |
| [0019-esign-template-designer](0019-esign-template-designer.md) | E-Sign Template & Field-Placement Designer (business-owner-controlled contract templates + signing-field placement) | accepted |
| [0020-esign-pdf-upload-and-guided-signing](0020-esign-pdf-upload-and-guided-signing.md) | PDF Template Upload & Guided Click-to-Sign (upload a blank PDF instead of authoring HTML; signers tap placed fields with a generated cursive signature) | accepted |
| [0021-email-system-outgoing-mail-branding](0021-email-system-outgoing-mail-branding.md) | Platform-Wide Outgoing Mail Branding — new `email_system` module app replaces ERPNext's "Sent via ERPNext" footer with business logo + "Powered by Aetris" branding, identical regardless of SMTP vs Aetris-managed delivery | accepted |
| [0022-console-admin-ui](0022-console-admin-ui.md) | Platform Console Admin UI — Custom React Frontend (extends ADR-0008, replaces plain Frappe desk forms) | accepted |
| [0023-tenant-lifecycle-enforcement](0023-tenant-lifecycle-enforcement.md) | Tenant Suspend/Cancel via Routing-Layer Enforcement — zero code on any tenant site, module-independent | accepted |
| [0024-tenant-permanent-deletion](0024-tenant-permanent-deletion.md) | Permanent Tenant Deletion — Cancelled-first, typed-subdomain confirm, pre-drop backup, record kept as Deleted | accepted |
| [0025-landlord-cheque-plan-decoupling](0025-landlord-cheque-plan-decoupling.md) | Decouple Landlord Cheque Plan from Accrual Invoice and Payment — admin-entered cheque plan, accrual posts on schedule regardless of cheque cadence, on-account advance auto-swept oldest-invoice-first | accepted |
| [0026-outgoing-pdc-auto-deposit](0026-outgoing-pdc-auto-deposit.md) | Auto-Deposit Outgoing (Landlord) Cheques on Their Own check_date — partially supersedes 0014 (Outgoing only; Incoming stays manual) | accepted |
| [0027-selectable-admin-ui](0027-selectable-admin-ui.md) | Selectable Admin UI — optional Vue 3 "Classic" skin (vue-element-admin visual family) alongside the React "Modern" default, per-tenant toggle, phased rollout (4 generic sections first, bespoke sections gated) | accepted |

## Drafts

_(none pending)_

## Numbering rules

- Sequential, zero-padded, chronological (`0001`, `0002`, ...). Never reuse or renumber.
- New ADR flow: create draft in `decisions/drafts/draft-adr-<slug>.md` → get approval → number + move to `decisions/000N-<slug>.md` → delete draft → add row above.
