# Implementation Plan

Live, prioritized task list for the platform — not a phased roadmap. This
file is a running index of every pending/in-progress feature plus a
reference list of completed ones. It is updated **every time** a feature
implementation plan is created or completed (see `AGENTS.md` §4.3) — never
let it drift out of sync with the feature files it links to.

Platform-level plan: [`PLATFORM_STRATEGY.md`](PLATFORM_STRATEGY.md).
Real Estate OS's own PRD: [`os/REALESTATE_MASTERPLAN.md`](os/REALESTATE_MASTERPLAN.md).
Each feature file (`vault/<domain>/features/<feature>.md`) carries its own
step-by-step implementation plan + acceptance criteria — this file only
tracks what exists and how urgent it is.

**Priority scale**: P0 (do next — critical/blocking) > P1 (high — near-term
real value) > P2 (medium) > P3 (low / speculative future).

A feature moves from Pending to Completed once its `status` frontmatter is
`done` **and** its acceptance criteria pass (verified live, not just
code-complete).

## Pending

### P0 — Critical

_(none currently — landlord-cheque-plan-decoupling shipped 2026-09-06)_

### P1 — High

| Feature | Domain | Status | Why P1 |
| --- | --- | --- | --- |
| [outgoing-mail-branding](email-system/features/outgoing-mail-branding.md) | email-system | in-progress | ADR-0021. Every OTP/signing/invoice email showed "Sent via ERPNext" and no logo — direct customer-facing complaint from Imran. New `email_system` module app owns this platform-wide (not `real_estate_os`-specific), replacing it with business logo + "Powered by Aetris", identical on own-SMTP vs Brevo-managed sending. Built, merged, and deployed live on all three sites (verified: hook resolves correctly, no ERPNext branding, correct Aetris footer). Remaining: verify the Settings-page logo upload live end-to-end. |
| [provider-abstraction](esign/features/provider-abstraction.md) | esign | in-progress | ADR-0017. `esign_mode` dispatch works (PR #47), in-built e-sign is its own app (PR #48), and the Business Config settings-page UI to switch `esign_mode` is now live (PR #64, 2026-08-31) — all verified live. Remaining: Zoho Sign provider (blocked on Imran providing API credentials) and its webhook route/status sync. |
| [tenant-provisioning-console](multi-tenancy/features/tenant-provisioning-console.md) | multi-tenancy | in-progress | ADR-0008/0016. `console.nnuggets.com` is live with `platform_console` (new private repo) — verified live end-to-end 2026-09-01: a Tenant Site form creates a real site, installs the module, sets up its Company with a per-country Chart of Accounts template, and adds its Traefik route, no SSH. Remaining: custom-domain manual-checklist path (untested, no custom-domain tenant exists) and actually onboarding American Real Estate as the first real second tenant. |
| [console-admin-ui](multi-tenancy/features/console-admin-ui.md) | multi-tenancy | in-progress | ADR-0022. Replaces `platform_console`'s plain Frappe desk form with a React admin UI (same stack as `real_estate_os/ui`) — business list/detail, onboarding wizard, subscription tracking, platform settings. Deployed live to `console.nnuggets.com` 2026-09-05 (two live bugs found and fixed along the way — missing Dockerfile asset symlink, and `www`/`public` nested one level too deep). Remaining: Imran's live create/delete tenant test. |
| [tenant-lifecycle-enforcement](multi-tenancy/features/tenant-lifecycle-enforcement.md) | multi-tenancy | in-progress | ADR-0023. Suspend/Reactivate/Cancel via Traefik routing inside `platform_console` — zero code on `real_estate_os` or any future OS module, so every module gets lifecycle enforcement for free. Deployed live 2026-09-05; the route-file write/removal was moved onto `queue="long"` after finding the whitelisted-method container has no Traefik-dir mount. Companion to `console-admin-ui`. |
| [tenant-permanent-deletion](multi-tenancy/features/tenant-permanent-deletion.md) | multi-tenancy | planned | ADR-0024. Genuine, irreversible delete for a Cancelled tenant — backup, `bench drop-site`, record kept marked Deleted. Imran needs this to actually clean up test tenants rather than accumulate Cancelled-but-still-running sites forever. |
| [esign-template-management](esign/features/esign-template-management.md) | esign | in-progress | ADR-0019, Phase 1. Business-owner-editable contract templates + rendering-pipeline swap (with fallback), replacing the single hardcoded Print Format. Merged and deployed live on both tenant sites 2026-09-03 (migration verified clean, old data intact) — remaining gap is a human click-through of the new template-authoring UI itself. Prerequisite for field-placement-designer below. |
| [esign-field-placement-designer](esign/features/esign-field-placement-designer.md) | esign | in-progress | ADR-0019, Phase 2. Visual signature/field placement (PDF.js) + real PDF stamping at finalize time. Merged and deployed live on both tenant sites 2026-09-03 (migration verified clean, stamping mechanism verified via direct round-trip). Remaining gap: a human click-through of the designer UI and a full real test signing through the guest flow. |
| [esign-pdf-template-upload](esign/features/esign-pdf-template-upload.md) | esign | in-progress | ADR-0020, Phase 3. Upload a blank PDF instead of authoring HTML; place data merge fields visually via the existing Phase 2 designer. Merged and deployed live on both tenant sites 2026-09-03 (migration verified clean, stamping mechanism verified via direct round-trip) — remaining gap is a human click-through. |
| [esign-guided-click-to-sign](esign/features/esign-guided-click-to-sign.md) | esign | in-progress | ADR-0020, Phase 4. Guest signing page becomes field-position-aware (PDF.js, tap-to-place) with a generated cursive-font signature for typed names. Merged and deployed live 2026-09-03 (verified against the real signing pipeline — zero regression for existing documents). Remaining gap: a human click-through against a document with real field placements. |

### P2 — Medium

| Feature | Domain | Status | Why P2 |
| --- | --- | --- | --- |
| [maintenance-cost-allocation](maintenance/features/maintenance-cost-allocation.md) | maintenance | in-progress | Cost-center routing has worked since ADR-0018; the real gap closed 2026-09-01 was `total_cost` never pricing `labor_hours` at all. GL posting (Expense Claim/Stock Entry, the original plan) was deliberately deferred per Imran — `get_building_profitability`/`get_owner_overview` already consume the simpler non-GL `total_cost` figure. Also fixed a data leak found along the way: `get_doc_detail` wasn't filtering by DocField permlevel, exposing Maintenance Request's internal cost fields to Tenants. |
| [lead-management](custom-module/features/lead-management.md) | custom-module | planned | Pre-tenant CRM pipeline (ADR-0013 scope) — valuable for growth, not blocking current tenant operations. |
| [occupancy-dashboard](ui-portal/features/occupancy-dashboard.md) | ui-portal | planned | Visualization only — the underlying occupancy data is already fully queryable through the existing portal; this is a convenience view. |
| [all-page-ui-gap-analysis](ui-portal/all-page-ui-gap-analysis.md) | ui-portal | in-progress | Phase 1-3 done 2026-08-31 (Contracts split into Tenant/Landlord Lease tabs, Landlord Head Leases tab, 4 data-bug fixes, Rent Roll report — PRs #58-60). Remaining: Phase 4 detail-page template — KPI strips, tooltips for the dev-note field text, a Landlord "Payments Made" tab, clickable Link field values, a Documents tab — flagged in the doc itself as needing a scoping conversation first (may be over-engineered given how few linked-record groups each entity currently has). |
| [selectable-admin-ui](ui-portal/features/selectable-admin-ui.md) | ui-portal | in-progress | ADR-0027. Optional Vue 3 "Classic" admin UI (vue-element-admin visual family) alongside the React "Modern" default, toggled per-tenant from Settings. Phase 1 (Properties/Tenants/Maintenance/Landlords) + Overview (added 2026-09-18 as an explicit per-section Phase 2 item) both merged and deployed live. Imran confirmed in-browser on test.nnuggets.com: fallback panel and Overview both render correctly. Actions/Accounts/Settings/Contracts remain gated behind a fresh go/no-go each. |

### P3 — Low / Future

| Feature | Domain | Status | Why P3 |
| --- | --- | --- | --- |
| [tenant-onboarding](multi-tenancy/features/tenant-onboarding.md) | multi-tenancy | planned | Superseded by `tenant-provisioning-console` for the UI path; kept only as the manual CLI fallback runbook. |
| Stripe/Interac e-transfers, voice-AI intake, marketing video generation | — | speculative | No ADR, no design, no committed need yet — parking lot ideas only. |

### Known follow-up gaps (not yet separate feature files)

Small, explicitly-flagged open items inside otherwise-`done` features. Kept
here for traceability so they aren't lost, without the overhead of a
dedicated feature file until one of them is actually prioritized:

- **Landlord-payables escalation** — `escalation_percent`/`escalation_frequency`
  are captured on `Head Lease` but not applied to the generated payment
  schedule (flat-rate only today). See `payments-accounting/features/landlord-payables.md`. — P2
- **Outgoing-cheque source bank account** — closed 2026-08-29, PR #44. See
  [bank-account-setup](payments-accounting/features/bank-account-setup.md)
  step 3 and `landlord-payables.md`.
- **Security Deposit refund/forfeit action** — still only editable via the
  generic edit form (no dedicated Clear/Bounce-style action, and
  deliberately excluded from the signed-lease field lock for this reason).
  See `payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`. — P2
- **Per-role portal nav filtering** — Leasing Agent/Accountant/Maintenance
  Staff all get the same full staff nav as System Manager; DocPerm is the
  real security boundary but a role sees nav items that render empty for
  it. See `compliance-security/features/role-based-access.md`. — P3
- **Maintenance Staff can't log completed work** — permlevel-1 fields on
  Maintenance Request (spares/labor/cost) are read-only for this role,
  since the scoping field (`cost_center`) shares that permlevel; needs a
  dedicated action (mirrors PDC Entry's Clear/Bounce pattern). See same
  file. — P2

## Completed

| Feature | Domain | Updated |
| --- | --- | --- |
| [landlord-cheque-plan-decoupling](payments-accounting/features/landlord-cheque-plan-decoupling.md) | payments-accounting | 2026-09-06 |
| [role-based-access](compliance-security/features/role-based-access.md) | compliance-security | 2026-08-28 |
| [app-scaffold](custom-module/features/app-scaffold.md) | custom-module | 2026-08-16 |
| [unit-doctype](custom-module/features/unit-doctype.md) | custom-module | 2026-08-17 |
| [bulk-unit-generator](custom-module/features/bulk-unit-generator.md) | custom-module | 2026-08-17 |
| [lease-agreement-doctype](custom-module/features/lease-agreement-doctype.md) | custom-module | 2026-08-17 |
| [customer-extension](custom-module/features/customer-extension.md) | custom-module | 2026-08-16 |
| [landlord-doctype](custom-module/features/landlord-doctype.md) | custom-module | 2026-08-20 |
| [building-doctype](custom-module/features/building-doctype.md) | custom-module | 2026-08-27 |
| [head-lease-doctype](custom-module/features/head-lease-doctype.md) | custom-module | 2026-08-27 |
| [building-amenity-options](custom-module/features/building-amenity-options.md) | custom-module | 2026-08-25 |
| [building-setup-checklist](custom-module/features/building-setup-checklist.md) | custom-module | 2026-08-25 |
| [lease-lifecycle-state](custom-module/features/lease-lifecycle-state.md) | custom-module | 2026-08-26 |
| [own-building-landlord](custom-module/features/own-building-landlord.md) | custom-module | 2026-08-27 |
| [tenant-field-curation](custom-module/features/tenant-field-curation.md) | custom-module | 2026-08-27 |
| [inbuilt-esign](esign/features/inbuilt-esign.md) | esign | 2026-08-17 |
| [counter-signature](esign/features/counter-signature.md) | esign | 2026-08-17 |
| [esign-audit-trail](compliance-security/features/esign-audit-trail.md) | compliance-security | 2026-08-17 |
| [maintenance-request-doctype](maintenance/features/maintenance-request-doctype.md) | maintenance | 2026-08-20 |
| [recurring-invoicing](payments-accounting/features/recurring-invoicing.md) | payments-accounting | 2026-08-27 |
| [pdc-processing](payments-accounting/features/pdc-processing.md) | payments-accounting | 2026-08-17 |
| [pdc-schedule-generation-and-reconciliation](payments-accounting/features/pdc-schedule-generation-and-reconciliation.md) | payments-accounting | 2026-08-27 |
| [cost-center-per-building](payments-accounting/features/cost-center-per-building.md) | payments-accounting | 2026-08-27 |
| [landlord-payables](payments-accounting/features/landlord-payables.md) | payments-accounting | 2026-08-27 |
| [building-profitability-report](payments-accounting/features/building-profitability-report.md) | payments-accounting | 2026-08-26 |
| [accounts-income-vs-expense](payments-accounting/features/accounts-income-vs-expense.md) | payments-accounting | 2026-08-26 |
| [bank-account-setup](payments-accounting/features/bank-account-setup.md) | payments-accounting | 2026-08-30 |
| [accounting-reports](payments-accounting/features/accounting-reports.md) | payments-accounting | 2026-08-30 |
| [rent-roll-and-arrears-report](payments-accounting/features/rent-roll-and-arrears-report.md) | payments-accounting | 2026-08-31 |
| [cash-flow-forecast](payments-accounting/features/cash-flow-forecast.md) | payments-accounting | 2026-08-31 |
| [admin-portal-ui](ui-portal/features/admin-portal-ui.md) | ui-portal | 2026-08-25 |
| [tenant-portal-ui](ui-portal/features/tenant-portal-ui.md) | ui-portal | 2026-08-24 |
| [maintenance-portal](ui-portal/features/maintenance-portal.md) | ui-portal | 2026-08-24 |
| [portal-source-consolidation](ui-portal/features/portal-source-consolidation.md) | ui-portal | 2026-08-27 |
| [contracts-page-fixes](ui-portal/features/contracts-page-fixes.md) | ui-portal | 2026-08-26 |
| [portal-list-column-fixes](ui-portal/features/portal-list-column-fixes.md) | ui-portal | 2026-08-26 |
| [portal-list-table-controls](ui-portal/features/portal-list-table-controls.md) | ui-portal | 2026-08-26 |
| [contract-detail-page-cleanup](ui-portal/features/contract-detail-page-cleanup.md) | ui-portal | 2026-08-27 |

## Approved ADRs awaiting no further tracking

`0001`-`0017` — see [`DECISIONS.md`](DECISIONS.md) for the full list. Each
approved ADR's implementation is tracked via the feature file(s) linked from
its own Implementation section, not duplicated here.
