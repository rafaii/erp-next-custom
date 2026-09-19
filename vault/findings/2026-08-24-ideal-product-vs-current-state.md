---
status: done
owner: developer-1
domain: cross-cutting
created: 2026-08-24
updated: 2026-08-24
related_adr: ["0003-multi-tenancy", "0005-counter-signature-pdc", "0010-pdc-schedule-and-bank-reconciliation"]
---

# Finding — Ideal Product vs. Current Implementation

## Method

Independent analysis, not a rewrite of the reference gap-analysis Imran pasted.
Every "current state" claim below is **code-verified** against
`apps/real_estate_os` @ `010a72e` (DocType JSON field lists, `api.py`,
`payments/invoicing.py`, `payments/pdc.py`, `hooks.py`, fixtures, role grants),
not taken from `HANDOFF.md` or feature-file `status:` frontmatter — several of
those are stale.

---

## 1. First, the framing the reference doc gets wrong

The pasted gap analysis describes a **property-management ERP**: an agent who
manages buildings *on behalf of owners*, earns a commission, and whose main job
is collecting rent and reporting to the owner. Its entire "Accounting Team View"
is about rent in, landlord payouts as a pass-through, and GAAP statements.

That is not this business. Per `MASTERPLAN.md`, Real Estate OS serves a
**real-estate arbitrage operator**:

> lease whole buildings *from* landlords (head lease — a fixed, committed
> monthly obligation), then sublet individual units *to* renters at a markup.

The consequences are structural, and they change what "ideal" means:

| Property manager | Arbitrage operator (us) |
| --- | --- |
| Revenue = commission % | Revenue = **spread** (sublease income − head-lease cost) |
| Vacancy hurts the *owner* | Vacancy hurts **us** — the head-lease rent is owed regardless |
| Landlord is the customer | Landlord is a **supplier/creditor** |
| Key KPI: collection rate | Key KPI: **margin per unit + break-even occupancy per building** |
| Cost side is someone else's | Cost side (rent-out, utilities, maintenance) **is the P&L** |

So the single most important thing an ideal product does here — model **both
sides of the ledger per building** and expose the spread — is the thing the
reference doc never mentions, and (see §3) the thing we have not built.

---

## 2. What the ideal product looks like

Nine pillars. "Ideal" = what a mature operator running 20+ buildings needs, not
a wish list.

### P1 — Portfolio & head-lease register (the cost side)
Buildings and units as assets, each with the **lease-in terms**: landlord, term
start/end, committed monthly rent, escalation schedule, deposit paid, renewal
option, notice period. Every building knows its monthly burn before a single
renter signs. Head-lease expiry is a bigger risk event than any sublease expiry
— losing the head lease evicts every renter in the building.

### P2 — Leasing lifecycle (the revenue side)
Unit availability → applicant/KYC → lease drafting → e-sign (both parties) →
activation → renewal / termination / eviction. A **real lease state machine**
(Draft / Sent / Active / Expiring / Expired / Renewed / Terminated), separate
from signature status, so "active lease" is a fact, not an inference.

### P3 — Money in
Rent schedule per lease, invoices generated on schedule, PDC (post-dated
cheque) register with deposit/clear/bounce lifecycle, other collections (utility
recharges, late fees, key deposits), security deposits held as a **liability**
and released or forfeited at move-out.

### P4 — Money out
Landlord payout schedule generated from the head lease, issued as real payables;
vendor/contractor bills from maintenance; utilities; municipal fees. Money out
is roughly 70-80% of gross rent in this model and is where the margin actually
gets decided.

### P5 — Financial core
ERPNext GL doing the heavy lifting, plus the mapping that makes it meaningful:
**cost center per building** (every rent invoice, landlord payment, maintenance
expense tagged), deposits to a refundable-liability account, tax/VAT handling,
and revenue recognised across the lease term rather than at invoice date.

### P6 — Operations
Maintenance intake (tenant portal) → triage → assignment → parts and labour →
cost posted to the building's cost center → tenant notified → SLA measured.
Recurring/preventive maintenance, not just reactive tickets. Vendor register.

### P7 — Intelligence
The reports an operator actually runs weekly:
**rent roll**, **arrears aging**, **margin per unit / per building**
(rent-in minus allocated rent-out minus maintenance), **break-even occupancy**,
**lease expiry ladder** (both head leases and subleases), **cash-flow forecast**
driven by the PDC calendar and landlord payment calendar, **collection rate**,
**maintenance cost per unit**.

### P8 — Access, trust, continuity
Real RBAC per `MASTERPLAN.md` §2 (Admin, Leasing Agent, Accountant, Maintenance
Staff, Renter), immutable audit trail on financial and signature events, tenant
data isolation, tested backup/restore, and a documented DR path.

### P9 — SaaS platform
Provisioning a new tenant site without SSH, centralised upgrades across sites,
per-tenant configuration, subscription/billing, and per-tenant observability.

---

## 3. Where we actually are

Legend: **Built** = verified in code · **Partial** = exists but materially
incomplete · **Absent** = no code.

### P1 — Portfolio & head-lease register — **Partial (weakest pillar)**

| Item | State | Evidence |
| --- | --- | --- |
| Building / Unit DocTypes | Built | `real_estate/doctype/building`, `unit` |
| Bulk unit generator | Built | `api.generate_units` (capped at 1000) |
| Landlord record | **Reference data only** | `landlord.json` = name, type, address, email, phone, bank name/account/IBAN. Nothing else. |
| Head-lease terms (rent paid, term, escalation, notice) | **Absent** | `building.json` has a `landlord` Link and no financial fields at all |
| Head-lease expiry tracking | **Absent** | — |

**This is the core gap.** The system knows who the landlord is and where to wire
money, but not *how much*, *when*, or *until when*. Nothing in the codebase
mentions Purchase Invoice or a landlord obligation (`grep -rn "Purchase Invoice"`
→ no hits). The cost side of an arbitrage business does not exist yet.

### P2 — Leasing lifecycle — **Built, with one structural flaw**

Built: Lease Agreement DocType, rent schedule child table, in-built e-sign
(consent + OTP with a 5-min session gate + signature capture + SHA-256 hashing +
PDF stamping + Certificate of Completion + immutable audit log), two-party
counter-signature, term-locking after signature, cancel-contract path, unit
status lifecycle (Vacant → Reserved → Occupied → Vacant), Jinja contract print
format, manual (non-e-sign) lease finalisation with countersigned upload.

This is genuinely strong — better than most commercial products at the signing
step, and it costs nothing per envelope.

**Flaw:** there is no lease status field. `esign_status == "Signed"` is used as a
proxy for "active" (`api.py:257`, `payments/invoicing.py`). An expired or
terminated lease still counts as active on the dashboard, and always will, since
nothing ever transitions it. Renewal has no representation at all.

### P3 — Money in — **Built**

Rent schedule generation, daily scheduler for due invoices + 3-day reminders,
PDC Entry with tenant-managed Cheque Bank, `generate_pdc_schedule` (blocked once
any cheque leaves Pending), daily Pending→Deposited, manual Cleared → real
submitted Payment Entry reconciling the linked Sales Invoice, Bounced with
activity logging, Security Deposit DocType linked to lease + customer.

Gaps: bounce has no late-fee or re-presentation workflow; no arrears aging; no
online payment rail (Stripe/Interac remains Phase 5); deposits are tracked but
never posted to the GL (see P5).

### P4 — Money out — **Absent**

No landlord payment schedule, no Purchase Invoice/Payment Entry on the payable
side, no vendor register, no utility bills, no expense capture beyond a
`total_cost` field on Maintenance Request that posts nowhere.

### P5 — Financial core — **Partial**

Built: Sales Invoices post to the company's default income + receivable accounts
(`invoicing.py:235-256`), Payment Entries reconcile properly on PDC clear.

Absent: **cost center per building** (the `Building.after_insert` hook is still
commented out in `hooks.py`); no cost-center tagging on any invoice, payment, or
maintenance cost; security deposits never hit a liability account; no tax
template on rent invoices; no revenue recognition across term. `_get_company()`
picks the first Company on the site — acceptable under tenant-per-site
(ADR-0003), worth an explicit assertion rather than a silent `[0]`.

### P6 — Operations — **Partial**

Built: Maintenance Request DocType with priority/status/assignment/spares child
table/labour hours/cost fields, permlevel-1 protection on internal fields so
renters can't see or set them, server-side unit resolution from the caller's own
lease, tenant-facing creation endpoint.

Absent: cost never posts to a cost center or the GL, no vendor/contractor
records, no SLA measurement, no preventive/recurring maintenance, no photo
upload, no auto-assignment.

### P7 — Intelligence — **Absent (near-total)**

**Zero report definitions exist in the app** (`find -path "*report*"` → nothing).
What exists is two dashboard endpoints: `get_dashboard_data` (building count,
unit occupied/vacant, "active" leases, leases expiring in 30d, open maintenance,
tenant count) and `get_accounts_data` (outstanding invoices, collected total,
PDC pending/deposited, 6-month revenue by month).

Not present: rent roll, arrears aging, **margin per unit or per building**,
break-even occupancy, cash-flow forecast, collection rate, maintenance cost per
unit, P&L by building. Margin/P&L are not just unbuilt — they're *unbuildable*
until P1 and P4 exist, since there is no cost side to subtract.

### P8 — Access, trust, continuity — **Partial**

Built: Tenant role (`desk_access=0`) scoped by a User Permission that Frappe
auto-propagates; permlevel separation on Maintenance Request; e-sign audit log is
append-only; a real IDOR in `get_linked_records` was found and fixed (2026-08-24).

Absent: **only two roles exist in the entire app — System Manager and Tenant**
(`grep '"role"'` across all DocType JSON: 13 × System Manager, 5 × Tenant). The
four-role model in `MASTERPLAN.md` §2 — Leasing Agent, Maintenance Staff,
Accountant, and a read-only Owner/CEO view — is unimplemented; today every
staff user is effectively a super-user. No backup/restore drill, no documented
DR, no data-retention or PIPEDA/GDPR handling.

### P9 — SaaS platform — **Partial**

One live tenant, deployed via Docker on the VPS. ADR-0008 (control-plane
provisioning) is approved but the console is `planned` — provisioning is still
manual SSH + `bench new-site`. No subscription billing, no per-site upgrade
orchestration, no cross-site monitoring.

---

## 4. Gap register (ranked)

Rank = business impact × how much else it blocks. Effort: S ≤ 3d, M ≤ 2w, L > 2w.

| # | Gap | Pillar | Impact | Effort | Blocks |
| --- | --- | --- | --- | --- | --- |
| G1 | **Head lease (lease-in) not modelled** — no landlord rent, term, or escalation | P1 | Critical | M | G2, G4, G5 |
| G2 | **Landlord payables absent** — no payout schedule, no Purchase Invoice | P4 | Critical | M | G4 |
| G3 | **Cost center per building not wired** — hook still commented out | P5 | Critical | S | G4 |
| G4 | **No margin / P&L per building or unit** — the operator's primary KPI | P7 | Critical | M | — |
| G5 | **No cash-flow forecast** (PDC calendar in, landlord calendar out) | P7 | High | M | needs G1/G2 |
| G6 | **Lease has no lifecycle state** — `esign_status` proxies for "active"; expired leases count as active forever | P2 | High | S | accurate occupancy, renewals |
| G7 | **Zero reports** — no rent roll, no arrears aging, no collection rate | P7 | High | M | — |
| G8 | **RBAC is 2 roles** — no Leasing Agent / Accountant / Maintenance / Owner | P8 | High | M | safe delegation, CEO view |
| G9 | **Maintenance cost posts nowhere** — no GL, no cost center, no vendor | P6 | High | M | needs G3 |
| G10 | **Security deposits not in the GL** as a refundable liability | P5 | Medium | S | audit-clean balance sheet |
| G11 | **No renewal / expiry workflow** — expiry count exists, nothing acts on it | P2 | Medium | S | needs G6 |
| G12 | **PDC bounce has no late-fee / re-present path** | P3 | Medium | S | — |
| G13 | **No tax/VAT on rent invoices** | P5 | Medium | S | jurisdiction-dependent |
| G14 | **Provisioning console unbuilt** — manual SSH per tenant | P9 | Medium | M | 2nd+ client onboarding |
| G15 | **No online payment rail** (Stripe/Interac) — cheque-only | P3 | Low-Med | M | — |
| G16 | **No backup/restore drill or DR doc** | P8 | Medium | S | SaaS credibility |

---

## 5. Structural risks found while reading the code

These are not product gaps; they are risks to the project itself.

**R1 — The admin portal's source code is in no git repository.** *(Highest.)*
The React SPA lives at `~/Development/erpnext/ui/` — untracked in the meta repo
(which per `AGENTS.md` is never committed or pushed) and not in the app repo.
What *is* committed to `real_estate_os` is the **compiled bundle**
(`real_estate_os/public/portal/assets/Dashboard.js`, `index.js`, `index.css`) —
commit `010a72e`, the "New Lease dialog" fix, changed exactly one file: a minified
bundle. Meanwhile the app repo still tracks the superseded **Vue** portal at
`portal/src/*.vue`, which ADR-0007 records as removed.

Consequences: the entire admin UI exists on one laptop, unbacked. And the
`AGENTS.md` acceptance test — clone the repo onto a fresh ERPNext and get the
product — yields un-modifiable minified JS plus a dead Vue app. **This should be
fixed before any further feature work.**

**R2 — `HANDOFF.md` is materially stale** (last updated 2026-08-17). It lists
maintenance, portals, and PDC processing as "not built yet"; all three shipped
since. Anyone resuming from it starts with a wrong map.

**R3 — Feature-file `status:` frontmatter has drifted from reality.**
`maintenance-request-doctype` is marked `done` (the DocType exists, but cost
allocation — the point of it — does not); `IMPLEMENTATION-PLAN.md` Phase 1 shows
`building-doctype` as `in-progress` and `occupancy-dashboard` as `planned` though
a dashboard ships in the portal. The plan file no longer describes the build.

**R4 — No automated tests anywhere in the app.** Every verification in the
changelog is a manual `bench console` session against live data, several against
*production* records (leases reset, invoices cancelled, test data cleaned up by
hand). That has worked so far and will not keep working.

---

## 6. Recommendation — what to do, in order

**Wave 1 — Stop the bleeding (days).**
1. R1: move `ui/` into the app repo, commit the source, delete the stale Vue
   `portal/`, make the build reproducible from a clean clone.
2. G3: uncomment and implement `Building.after_insert` → Cost Center; backfill
   existing buildings; tag rent invoices with it.
3. G6: add a real `lease_status` field + transitions; fix "active lease" counts.
4. R2/R3: refresh `HANDOFF.md` and reconcile `IMPLEMENTATION-PLAN.md` statuses.

**Wave 2 — Build the cost side (the actual product gap, ~3-4 weeks).**
5. G1: `Head Lease` DocType (landlord, building, term, monthly rent, escalation,
   deposit, notice period, renewal option) — needs an ADR.
6. G2: landlord payout schedule → Purchase Invoice / Payment Entry, tagged to the
   building's cost center — same ADR.
7. G9: maintenance cost → GL against the cost center; vendor records.
8. G10: security deposits → refundable liability account.

**Wave 3 — Make it legible (~2-3 weeks).**
9. G4: Building Profitability / margin-per-unit report — now computable.
10. G7: rent roll, arrears aging, collection rate.
11. G5: cash-flow forecast from the PDC calendar minus the landlord calendar.
12. G8: the four-role RBAC model, including a read-only Owner/CEO dashboard.

**Then:** G11-G16 (renewals, bounce handling, tax, provisioning console, payment
rail, DR), and testing as a standing requirement (R4).

The one-line version: **the leasing and collections half of this product is
strong and close to complete; the cost half — head leases, landlord payables,
cost-center accounting — does not exist, and until it does, the operator cannot
see the only number the business runs on.**

---

## 7. Proposed vault follow-ups (need Imran's approval per `AGENTS.md` §4)

- **New ADR** — *Head Lease & Landlord Payables* (covers G1 + G2; defines whether
  head leases are a new DocType or an extension of Building, and whether payouts
  are ERPNext Purchase Invoices or a custom payable). Requires approval before
  numbering.
- **New ADR** — *Lease Lifecycle State* (G6): where lease status lives and how it
  relates to `esign_status`. Small but it changes a field every report will read.
- **New feature files**: `payments-accounting/features/landlord-payables.md`,
  `payments-accounting/features/rent-roll-and-arrears.md`,
  `payments-accounting/features/cash-flow-forecast.md`,
  `custom-module/features/lease-lifecycle-state.md`,
  `compliance-security/features/role-model.md`.
- **`INDEX.md` link to this findings folder** — not added; `INDEX.md` is on the
  §4 stop-and-ask list.

## Update 2026-08-26

Re-verified against the live app before continuing implementation. Since
this finding was written: **G1** (Head Lease), **G2** (landlord payables /
Purchase Invoice), **G3** (cost center per building), and **G6** (lease
status field) are all built and confirmed live (PRs #9-#14, #19). **G4**
(margin per building) closed this session — see
`payments-accounting/features/building-profitability-report.md` (PR #20).
Still open: G5 (cash-flow forecast), G7 (rent roll/arrears, other reports),
G8 (RBAC roles), G9 (maintenance cost to GL), G10-G16.

The building-profitability work also surfaced a concrete instance of the
"unattributed cost" risk implicit in G3/G4: Cost Center is assigned lazily,
so pre-existing invoices can't be retroactively attributed once submitted.
Worth keeping in mind for G7/G5 too, since they'll read off the same GL
data.

## Related

- PRD: `vault/MASTERPLAN.md`
- Roadmap: `vault/IMPLEMENTATION-PLAN.md`
- ADRs: `0003-multi-tenancy`, `0005-counter-signature-pdc`, `0010-pdc-schedule-and-bank-reconciliation`
