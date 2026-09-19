# PLATFORM STRATEGY

## High-Level Plan: A Multi-Industry SaaS Platform on ERPNext

**2026-08-27**: this file replaces `MASTERPLAN.md`, which was scoped
entirely to one vertical (real estate). That content now lives in
`vault/os/REALESTATE_MASTERPLAN.md`. This file is the platform-level
plan — what's shared across every future domain-specific "operating
system" — and does not itself describe real estate, kindergarten,
workshop, or retail workflows in detail; each gets its own
`vault/os/<DOMAIN>_MASTERPLAN.md`.

## 1. Vision

A back-end platform that lets us stand up a fully-functioning, branded
business system for a new client in any of several industries, quickly —
without rebuilding accounting, financial reporting, or platform
infrastructure per industry. Each industry gets a dedicated **"operating
system"** (Real Estate OS today; Kindergarten OS, Workshop OS, Retail OS,
etc. as the business grows) that defines that industry's own workflows
and terminology. Every OS runs on the same underlying ERP core.

**The underlying platform is ERPNext (Frappe Framework) — this is
deliberately never shown to a client.** A business owner using Workshop
OS should have no more reason to know it's "ERPNext" than a Shopify
merchant needs to know Shopify's own internal stack. Every OS's frontend
is a custom, branded portal (see `0006`/`0007`); Frappe Desk is never
tenant-facing (see `0013`'s explicit "never reference ERPNext in the
frontend," which this generalizes into a platform-wide rule, not a
Lead/CRM-specific one).

## 2. The Four Pillars (ADR-0017)

1. **Domain-specific OS apps** — each industry is its own Frappe app, own
   repo, own DocTypes, own portal navigation. `real_estate_os` is the
   first and, so far, only one. A second OS reuses the generic,
   already-vertical-agnostic parts of the portal frontend (the
   list/detail/field-curation shell) rather than rebuilding them —
   extracted into a shared package once a second OS actually needs it,
   not before (see ADR-0017 for why not before).
2. **Independent, reusable modules** — integrations (e-sign, payments,
   email/SMS, etc.) are never code inside an OS app. Each is its own
   Frappe app implementing a documented interface contract for its
   category (`ESignProvider`, `PaymentProvider`, ...). An OS declares
   which module categories it *requires*; provisioning (§4) installs
   them automatically. Which installed implementation is *active* is a
   business-config-page decision the tenant makes themselves (e.g.
   "E-Signature: In-Built" vs "Zoho Sign"), not a provisioning-time or
   code-level choice.
3. **Standardized integration architecture** — dispatch via Frappe hooks
   (`frappe.get_hooks(...)`), the same mechanism already used internally
   for `portal_nav_items`/`portal_detail_links`/`scheduler_events`. An OS
   app never hard-imports a specific module's code.
4. **Per-customer extension, tiered** — most clients run ~95% of the
   standard OS unmodified. The remaining 5% is handled by reaching for
   the *lightest* mechanism that fits, in order: Custom Field → Property
   Setter → Portal Field Visibility → Client/Server Script → a small
   per-customer overlay app (only for real custom DocTypes/logic one
   client needs). See ADR-0017 for the full table.

## 3. Shared ERP Core

Accounting, financial reporting, and the general ledger are **not**
something any OS app owns or reimplements — they are the `erpnext`
framework app itself (Company, Chart of Accounts, GL Entry, Journal
Entry, and ERPNext's own P&L/Balance Sheet/Trial Balance/General Ledger
report engines), a prerequisite on every tenant site regardless of which
OS is installed. Every OS only ever *extends* this core the same way
`real_estate_os` extends Customer/Supplier/Cost Center — never forks or
replaces it. See ADR-0015 for how this core gets a portal-native UI
(rather than requiring Frappe Desk access) for the first OS.

**Account naming is per-OS, never shared.** Each OS ships its own
Chart-of-Accounts fixture, applied automatically at install time, because
each tenant site has its own independent Company and Account tree
(tenant-per-site, §4/`0003` — there is no cross-tenant or cross-OS Account
table to keep in sync). Real Estate OS renaming "Cost of Goods Sold" to
"Head Lease Rent Expense" has zero bearing on what a future Workshop OS's
accounts are named — verified live, not assumed (`0016`).

## 4. Multi-Tenancy & Provisioning

**Tenant-per-site** (`0003-multi-tenancy`, unchanged by this document):
each client gets its own Frappe site with its own database — strong
isolation, standard Frappe deployment, one tenant's data never queryable
by another.

**Enrollment flow** (`0008` + `0016-module-aware-provisioning`): a
non-tenant-facing Platform Control Plane (its own site, reachable only by
a Platform Admin) is where a new client gets onboarded:

1. Platform admin selects the industry **module** (OS) — Real Estate
   today, others as they're built — and enters the client's details
   (company name, address, admin email).
2. The control plane provisions a new site: `bench new-site <slug> --
   install-app <os-app-for-selected-module>`, plus that OS's required
   modules (e.g. Real Estate OS requires an e-sign provider — see §2.2 —
   so one gets installed automatically, defaulting to in-built).
3. The client logs into their own new site and configures branding,
   email/IMAP, and any optional modules/integrations from their own
   Business Config page, then starts using the OS (adding buildings and
   tenants, for Real Estate; whatever the equivalent onboarding is for a
   future OS).

Custom domains (not a platform subdomain) remain a manual DNS/cert
checklist item in v1 (`0008`).

## 5. Non-Functional Requirements (platform-wide)

| Requirement | Specification |
| --- | --- |
| Responsive UI | Custom portal, mobile-friendly, per OS |
| Security | Frappe RBAC; whitelisted + authenticated API endpoints only |
| Audit Trails | Frappe versioning + `docstatus` on all DocTypes |
| Performance | < 2s portal load; background jobs for heavy tasks |
| Scalability | 100+ tenants across all OS's combined, separate DB per tenant |
| Compliance | Data isolation per tenant (GDPR-class); encrypted sensitive data at rest |

## 6. Technical Stack (platform-wide)

| Layer | Technology |
| --- | --- |
| Backend | Python (Frappe Framework), MariaDB |
| Frontend | Custom React SPA per OS (`0007`), never Frappe Desk for tenants |
| API | Whitelisted RPC methods; REST auto-generated per DocType |
| Modules | Independent Frappe apps per integration category (`0017`) |
| Deployment | Frappe Bench (Docker Compose today), one bench hosting all OS apps |
| SaaS Billing | External (Stripe), tracked against site subscriptions |

## 7. Risks & Mitigations (platform-wide)

| Risk | Mitigation |
| --- | --- |
| Third-party module API rate limits/outages | Queue requests; webhooks for async completion; degrade to in-built where a module has one |
| Multi-tenant, multi-OS upgrade complexity | Staged rollouts (canary sites); automated backup before update |
| Premature abstraction (extracting shared frontend/module patterns before a second real consumer exists) | Deliberately deferred per ADR-0017 until a second OS/module is actually being built |
| Portal performance across many tenants | Cache hot data; background jobs for heavy work |

## 8. Where the actual work is tracked

This document does not carry a phase roadmap or task list — that's
`vault/IMPLEMENTATION-PLAN.md`, a live, prioritized list across every
domain, not a fixed set of phases. Architectural decisions live in
`vault/decisions/` (see `DECISIONS.md` for the index). Each OS's own
domain-specific requirements live in its own `vault/os/<NAME>_MASTERPLAN.md`
(currently: `REALESTATE_MASTERPLAN.md`).

## Related

- `vault/os/REALESTATE_MASTERPLAN.md` — the first OS's own PRD
- `vault/decisions/0003-multi-tenancy.md`,
  `0006-admin-portal-approach.md`, `0007-admin-portal-react-frontend.md`,
  `0008-tenant-provisioning-control-plane.md`,
  `0013-lead-crm-and-tenant-field-curation.md` (the "never show ERPNext"
  precedent this generalizes),
  `0015-portal-native-accounting-reports.md`,
  `0016-module-aware-provisioning.md`, `0017-platform-modularization.md`
- `vault/findings/2026-08-27-full-accounting-system.md`,
  `2026-08-27-platform-modularization.md`
