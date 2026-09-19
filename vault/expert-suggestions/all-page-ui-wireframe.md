Your current structure covers the basics well, but it's missing the one distinction that makes an arbitrage model work — **Master Lease (what you owe the landlord) vs. Tenant Lease (what tenants owe you)** are both sitting in a single "Contracts" list with no differentiation. That's the biggest structural gap, followed by a missing Units layer and a missing Rent Roll/Payments view, both of which every property management system treats as core, separate entities.

## Structural Gaps to Fix First

**1. Master Lease vs. Tenant Lease conflation.** Your Contracts list shows only two tenant-facing leases (Rajesh, Jordan) totaling $4,000/month — but where's the master lease contract with Ooredoo Properties for Rastec 20? Without that record, you can't compute your spread (rent collected minus rent owed), which is the core metric of this business model. Split "Contracts" into two types with a filter/tab, or two separate sidebar items:[](https://www.suasarealestate.com/the-journal/master-lease-vs-sublease-types-of-lease-agreement-and-their-differences/)

text

`Leases  ├─ Tab: Tenant Leases (2)   — Rajesh $2,000, Jordan $2,000 └─ Tab: Master Leases (1)   — Ooredoo Properties, Rastec 20, $X,XXX/mo`

**2. No Units entity.** "Properties" shows "Total units: 3" as a rollup number, but there's no place to see each unit individually — its occupancy status, size, current rent, and which tenant occupies it. Right now Contracts shows "Rastec 20 - 003" as a lease label, which is really a unit code doing double duty. Add a dedicated Units directory:

text

`Units  Rastec 20  ├─ Unit 001 — Occupied — Jordan Ahmed — $2,000/mo  ├─ Unit 003 — Occupied — Rajesh Koothrapalli — $2,000/mo  └─ Unit 002 — Vacant — $0/mo — 0 days vacant`

This also fixes your Properties KPI: "Active 1 of 1" tells you the building is active, but not that 1 of 3 units is sitting vacant.

**3. No Rent Roll / Payments module.** None of your five sections show who has actually paid this month, who's overdue, or a payment history ledger — the single most-cited "essential report" in property management. This is distinct from Contracts (which is the agreement) and from your Accounting module (which is the ledger) — a business owner needs a collections-focused view:

text

`Rent Roll / Payments  Total billed this month: $4,000   Collected: $2,800   Overdue: $1,200 ┌─────────────────────┬──────────┬────────┬──────────┬────────────┐ │ Tenant               │ Unit     │ Due     │ Status   │ Days late  │ ├─────────────────────┼──────────┼────────┼──────────┼────────────┤ │ Jordan Ahmed         │ 001      │ $2,000  │ Paid     │ —          │ │ Rajesh Koothrapalli  │ 003      │ $2,000  │ Overdue  │ 3          │ └─────────────────────┴──────────┴────────┴──────────┴────────────┘`

## Fixes to Existing Screens

**Landlords — remove the "Self" hack.** Having "Own Building / Self" as a fake Landlord record to represent a building you own outright is a workaround, not a data model. Add an `Ownership Type` field on Property (Owned vs. Master-Leased) instead, and only show a Landlord record when it's genuinely a third-party master lease. This also cleans up your Landlords KPI ("With contact: 0 of 2") since a self-owned building shouldn't count against your contact-completeness metric.

**Contracts — add missing columns/KPIs.** Your current view shows tenant, unit-code, rent, status, start/end — solid, but add:

- A "Lease Type" column (Master / Tenant) once you split the data model
    
- A KPI card for "Expiring in 60 days" — lease renewal is a top risk in this model
    
- Rename "Monthly rent: $4,000 Combined" to "Total tenant rent" to avoid confusion once master leases appear in the same module
    

**Properties — surface occupancy, not just unit count.** Change "Total units: 3, Across buildings" into "Occupancy: 2 of 3 units (67%)" — a raw unit count tells the owner nothing about performance, but occupancy % is the single most important property-level KPI.[](https://www.usedatabrain.com/blog/property-management-dashboard)

**Tenants — add lease and balance context.** Right now Tenants only shows name, type, and email. Add "Current Unit," "Lease End Date," and "Balance Due" columns so this list doubles as a quick tenant-health view without navigating elsewhere.[](https://www.azibo.com/blog/rent-roll)

**Maintenance — add cost and unit linkage once populated.** The empty state is fine, but plan for columns showing Unit, Reported Date, Cost, and Days Open — maintenance cost is one of the "7 essential reports" every property owner tracks monthly, and in an arbitrage model it directly erodes your thin spread.[](https://rentingwell.com/2025/02/24/7-essential-reports-every-property-manager-should-track-monthly/)

## Suggested Final Sidebar Order

text

`Dashboard          ← the owner-facing summary you already have Properties         ← buildings you master-lease or own Units               ← NEW: individual unit inventory & occupancy Leases              ← renamed from Contracts, split Master/Tenant Tenants Rent Roll / Payments ← NEW: collections tracking Maintenance Landlords            ← only third-party master-lease landlords Accounting          ← your existing GL/JE/reports module`

This ordering follows the natural workflow an owner thinks in: what do I have (Properties/Units) → who's contracted (Leases/Tenants) → am I getting paid (Rent Roll) → what's it costing me (Maintenance) → who do I owe (Landlords) → the full books (Accounting).

Each of those items is clickable and you get more information. For instance, Properites lists Rastec 20 but clickong on that line it3em, shows all the details of that building including who is the landlord, the headlease, the units of that building etc. This same detail can be arrived by clickin gont he landlord and shows the properites listted under them and the contract they hold. Similarly for contracts and tenants. Similarly, land

This "click into any record and see its full relationship web" pattern is exactly the **360-degree view** model used by mature CRMs and property platforms — every entity gets a detail page with an overview, a KPI strip, and tabs to its related records, so you never build the same information twice. Here's how to formalize that pattern across your five entities so it stays consistent and doesn't turn into ad-hoc, one-off detail pages.

## One Template, Five Entities

Rather than designing each detail page independently, use a single reusable template: **Header → KPI Strip → Tabs**. Every entity (Property, Landlord, Contract/Lease, Tenant, Unit) gets the same skeleton, just with different tab contents — this is how HubSpot and Buildium keep dozens of record types feeling familiar.

text

`┌─ [Entity Icon] Rastec 20 ───────────────────────────── Status: Active [Edit] [•••] │  Building • Doha, Qatar                                    Landlord: Ooredoo Properties ├────────────────────────────────────────────────────────────────────────────── │  KPI STRIP (entity-specific) │  Units: 3   Occupied: 2 (67%)   Monthly spread: $2,400   Head lease renews: 4 mo ├────────────────────────────────────────────────────────────────────────────── │  [Overview] [Units (3)] [Leases (2)] [Financials] [Maintenance (0)] [Documents] └──────────────────────────────────────────────────────────────────────────────`

## Property Detail Page — Worked Example

This directly answers your Rastec 20 scenario. The Overview tab shows the head lease terms inline (so the owner doesn't have to click away just to see what they owe), while Units and Leases become tabs, not separate pages:

text

`Overview Tab: ┌────────────────────────────────────────────────────────────────────────┐ │ Address: 123 Corniche St, Doha           Building Type: Residential     │ │ Total Units: 3                            Ownership: Master-Leased      │ │                                                                          │ │ ── Head Lease (with landlord) ──────────────────────────────────────    │ │  Landlord: Ooredoo Properties [→ view landlord]                         │ │  Rent Owed: $X,XXX/month     Term: [start] – [end]     Renews in: 4 mo  │ │  [View Head Lease Contract →]                                           │ └────────────────────────────────────────────────────────────────────────┘ Units Tab (3): ┌──────┬────────────┬─────────────────────┬───────────┬──────────────┐ │ Unit │ Status     │ Tenant              │ Rent      │ Lease ends    │ ├──────┼────────────┼─────────────────────┼───────────┼──────────────┤ │ 001  │ Occupied   │ Jordan Ahmed [→]    │ $2,000    │ 2027-08-25    │ │ 003  │ Occupied   │ Rajesh K. [→]       │ $2,000    │ 2027-08-31    │ │ 002  │ Vacant     │ —                   │ $0        │ —              │ └──────┴────────────┴─────────────────────┴───────────┴──────────────┘`

Every name/link in brackets `[→]` navigates to that entity's own detail page — clicking "Ooredoo Properties" takes you to the Landlord detail page, which then shows Rastec 20 back in its own Properties tab. This bidirectional linking is the core of the 360-view pattern.

## Recommended Tabs per Entity

|Entity|Overview shows|Related-record tabs|
|---|---|---|
|Property|Address, units count, head lease summary|Units, Leases, Financials, Maintenance, Documents|
|Landlord|Contact info, landlord type|Properties (leased from them), Head Leases, Payment History (rent paid to them), Documents|
|Lease/Contract|Parties, unit, rent, term dates, renewal countdown|Linked Invoices/Payments, Amendment History, Documents|
|Tenant|Contact info, customer type|Current Lease, Unit, Payment History/Balance, Maintenance Requests filed, Documents|
|Unit|Unit number, size, status|Current Lease, Tenant, Maintenance History, Rent History|

## Design Rules to Prevent Drift

- **One source of truth per relationship.** A Property's "Landlord" field is a link, not a duplicated data entry — the Landlord detail page reads that same link in reverse. Frappe's native "Connections" feature (which auto-lists linked doctypes on a form) is built exactly for this, so lean on it instead of hand-building reverse lookups.[](https://help.vtiger.com/article/146772786-Module-Management---Module-Layouts-&-Fields)
    
- **Breadcrumbs, not dead ends.** When a user drills Landlord → Property → Unit → Tenant, keep a breadcrumb trail (`Ooredoo Properties > Rastec 20 > Unit 001 > Jordan Ahmed`) so they can back out without hitting browser-back.
    
- **KPI strip is entity-relative, not global.** A Landlord's KPI strip should show "Total rent payable across their properties," while a Tenant's KPI strip shows "Balance due" and "Lease days remaining" — same template, different math, so it never feels generic.[](https://knowledge.hubspot.com/records/work-with-records)
    
- **Tabs load counts, not empty guesses.** Label tabs "Units (3)," "Leases (2)," "Maintenance (0)" exactly as you're already doing in list views — this consistency between list-page KPIs and detail-page tab counts reinforces that it's the same live data everywhere, not a stale snapshot.
    
- **Documents tab on every entity.** Lease PDFs, tenant ID copies, head lease agreements — property management platforms consistently treat document storage as a first-class tab, not an afterthought.
    

If your message got cut off before "Similarly, land..." — I'm assuming you meant Landlords following the same drill-down pattern (Landlord → Properties → Head Leases), which the table above covers. Let me know if there's a different entity or interaction you wanted to flag.

This is how it is currently: # Details page for properties: Building Rastec 20 Edit Building Name Rastec 20 Status Active Property Manager — Landlord Ooredoo Properties Every Building needs one — buildings owned outright (no lease-in obligation) should point at the "Own Building" Landlord rather than being left blank, so "who's the landlord" is never ambiguous or accidentally skipped. See vault/custom-module/features/own-building-landlord.md. Landlord Payment Method PDC How we pay this building's landlord (Head Lease Schedule). Set at creation, editable later — see vault/decisions/0011-head-lease-landlord-payables.md. Total Units 3 Address B Ring Road, Doha, Qatar Cost Center BLD-00320 - Rastec 20 - RRE Auto-created on save. Income and expense entries tied to this building post here — see vault/payments-accounting/features/cost-center-per-building.md. Amenities Pool, Gym, Parking, Security, Central A/C Linked records Unit Units 3Generate unitsAdd unit Unit NumberStatusBedroomsBathroomsHas BalconyMonthly Rent 002 Vacant 0 0 No 2,000 003 Occupied 0 0 No 2,000 001 Occupied 0 0 No 2,000 Lease Agreement Lease Agreements 2 CustomerUnitStart DateEnd DateMonthly RentEsign Status Jordan Ahmed Rastec 20 - 001 2026-08-26 2027-08-25 2,000 Signed Rajesh Koothrapalli Rastec 20 - 003 2026-09-01 2027-08-31 2,000 Signed Head Lease Head Lease 1 LandlordHead Lease StatusMonthly RentStart DateEnd Date Ooredoo Properties Active 3,000 2026-08-27 2027-08-27 Activity No activity yet Status changes and events will appear here. # Details page of contracts: Lease Agreement CUST-2026-00006 This contract is fully signed — its terms are locked. Cancel it and create a new lease instead. Customer Jordan Ahmed Unit Rastec 20 - 001 Lease Status Active Business lifecycle of the tenancy — separate from E-Sign Status (the signature workflow). See vault/decisions/0012-lease-lifecycle-state.md. Start Date 2026-08-26 End Date 2027-08-25 Monthly Rent 2000 Not a fetch_from field on purpose — Frappe re-applies fetch_from server-side on every save (overwriting any explicit edit whenever `unit` is set), which silently clobbered rent corrections. The unit's rent is only used as a one-time default at creation (New Lease dialog, create_lease's fallback); after that this is independent per lease. Payment Frequency Monthly Security Deposit 2000 E-Sign Status Signed E-Sign Provider In-built E-Sign Envelope ID ESD-00354 E-Sign Last Error — Set when the last send attempt (invite/OTP email) failed; cleared on the next successful send. Read by the portal's Signing panel to show a retry prompt. Contract Signed Both parties have signed. Terms are locked — cancel and create a new lease to change anything. View signed contractCancel contract Accounting View Accounting Impact Every GL entry tied to this tenant — rent invoices and the payments settling them. Linked records PDC Entry Post-Dated Cheques 12 Check NumberCheck DateAmountTenant BankStatus 0010 2026-08-01 2000 QNB Cleared 0011 2026-09-01 2000 QNB Pending 0021 2027-07-01 2000 QNB Pending 0020 2027-06-01 2000 QNB Pending 0019 2027-05-01 2000 QNB Pending 0018 2027-04-01 2000 QNB Pending 0017 2027-03-01 2000 QNB Pending 0016 2027-02-01 2000 QNB Pending 0015 2027-01-01 2000 QNB Pending 0014 2026-12-01 2000 QNB Pending 0013 2026-11-01 2000 QNB Pending 0012 2026-10-01 2000 QNB Pending Security Deposit Security Deposit 1 AmountDeposit TypeCheck NumberTenant BankStatus 2000 Cheque 123 QNB Held Activity Cheque 0011 sent to bank Aug 27, 2026, 12:37 PM Signed by all parties Aug 27, 2026, 4:53 AM Signed by Imran Rafai Aug 27, 2026, 4:53 AM Signed by Jordan Ahmed Aug 27, 2026, 4:52 AM Sent for e-signature Aug 27, 2026, 4:45 AM Payment schedule generated — 12 cheques Aug 27, 2026, 4:37 AM Payment schedule generated — 12 cheques Aug 27, 2026, 4:23 AM Lease agreement created for CUST-2026-00006 Aug 27, 2026, 3:20 AM # Details of Tentnats: Customer Jordan Ahmed Edit Customer Name Jordan Ahmed Customer Type Individual Email Id jordan@4itrading.com Mobile No — Preferred Contact Method Email Emergency Contact Name — Emergency Contact Phone — ID Proof Type National ID ID Proof Number 123123213 Employer Name Ooredoo Monthly Income 10000 Lease Move this tenant into a vacant unit. New lease Portal login Let this tenant sign in to view their lease, invoices, and file maintenance requests. Enable portal login Linked records Lease Agreement Lease Agreements 1 UnitStart DateEnd DateMonthly RentEsign Status Rastec 20 - 001 2026-08-26 2027-08-25 2,000 Signed Security Deposit Security Deposits 1 LeaseAmountDeposit TypeStatus CUST-2026-00006 2000 Cheque Held Activity Rastec 20 - 001: Cheque 0011 sent to bank Aug 27, 2026, 12:37 PM Rastec 20 - 001: Signed by all parties Aug 27, 2026, 4:53 AM Rastec 20 - 001: Signed by Imran Rafai Aug 27, 2026, 4:53 AM Rastec 20 - 001: Signed by Jordan Ahmed Aug 27, 2026, 4:52 AM Rastec 20 - 001: Sent for e-signature Aug 27, 2026, 4:45 AM Rastec 20 - 001: Payment schedule generated — 12 cheques Aug 27, 2026, 4:37 AM Rastec 20 - 001: Payment schedule generated — 12 cheques Aug 27, 2026, 4:23 AM Rastec 20 - 001: Lease agreement created for CUST-2026-00006 Aug 27, 2026, 3:20 AM Tenant record created Aug 27, 2026, 2:37 AM # Details of Landlord: Landlord Ooredoo Properties Edit Landlord Name Ooredoo Properties Landlord Type Company Company Rastec Real Estate Supplier Ooredoo Properties Auto-created on save so a Head Lease can post a Purchase Invoice against this landlord — see vault/decisions/0011-head-lease-landlord-payables.md. Address — Email ooredoo@4itrading.com Phone — Bank Name QNB Picked from the same Cheque Bank list used by PDC Entry and Security Deposit — not free text. Account Number 123123123 IBAN — Linked records Building Buildings 1 Building NameStatusTotal Units Rastec 20 Active 3 Activity No activity yet Status changes and events will appear here.

This is impressive — you've already built things most commercial property platforms don't have out of the box (auto-created Cost Center per building, PDC payment scheduling, e-sign audit trail, an "Own Building" landlord to avoid null references). The gaps that remain are mostly about **surfacing what you already have** rather than building new data. Here's the audit.

## Critical Gap: The Spread Is Invisible

You have all the data to show arbitrage profitability on the Building page right now, but it's not computed or displayed anywhere. Rastec 20's Head Lease costs $3,000/month; its units are currently generating $4,000/month (2 occupied at $2,000 each) with unit 002 vacant. That's a **$1,000/month spread on paper, but really $2,000/month with the vacant unit costing you $2,000 in lost potential**. Add this as a KPI strip at the top of the Building detail page:

text

`Rastec 20                                                          Status: Active ├───────────────────────────────────────────────────────────────────────────── │  Occupancy: 2/3 (67%)   Rent Collected: $4,000   Head Lease Cost: $3,000 │  Spread: $1,000/mo  🟡   Vacant unit 002 costing ~$2,000/mo in lost rent`

This single addition turns the Building page from a records list into the profitability view you designed the dashboard around earlier.

## UI Copy: Move Developer Notes Into Tooltips

Right now every field carries a full paragraph of internal documentation directly in the body of the page — "Not a fetch_from field on purpose — Frappe re-applies fetch_from server-side...", "Auto-created on save so a Head Lease can post a Purchase Invoice...". These are excellent decision records for your dev team (and clearly well-documented, referencing your vault files), but they read as clutter to a business user filling out a lease. Convert every one of these into a small (i) icon next to the field label that reveals the note on hover/click, and keep the field itself clean. The permanent, always-visible text should be reserved for things the business user must act on — e.g., "This contract is fully signed — its terms are locked" is a legitimate warning banner and should stay visible; "Auto-created on save" is implementation trivia and should hide behind a tooltip.

## Data Bugs to Fix

**ID inconsistency on the Lease page.** The page header reads "Lease Agreement — CUST-2026-00006," but "View signed contract" links to `lease_name=LSE-2026-00328`. Two different identifiers for the same record will confuse anyone cross-referencing IDs in support tickets or accounting entries — pick one canonical ID (LSE-2026-00328 looks correct) and stop surfacing the CUST- prefix on the Lease page header; save that for the Tenant/Customer record.

**PDC list is unsorted.** Your 12 cheques display as 0010, 0011, 0021, 0020, 0019, 0018, 0017, 0016, 0015, 0014, 0013, 0012 — jumping from #0011 straight to #0021 then counting backward. Sort ascending by Check Date by default; this list is a payment schedule, and an owner scanning it needs "what's due next" to read top-to-bottom in order.

**No overdue flag on PDC entries.** With today's date at Aug 30, 2026, check #0010 (dated 2026-08-01) is correctly "Cleared," but #0011 (dated 2026-09-01) is still a month out and correctly "Pending" — good. But once a check date passes without a status update, that row needs a red "Overdue" state automatically, not just "Pending" indefinitely.

## Landlord Page Is Underpowered

Given Ooredoo Properties is the counterparty on a $3,000/month obligation, its detail page currently shows only one linked Building table and no financial visibility at all. Add:

- A **Head Leases** tab directly on the Landlord page (right now you'd have to click into the Building to find the Head Lease — that's an extra hop for information that belongs on the Landlord record itself).
    
- A **Payments Made** tab showing Purchase Invoices/payments against the auto-created Supplier record — you noted the Supplier exists specifically so Head Lease can post Purchase Invoices, but that financial trail isn't shown anywhere on this page.
    
- A KPI strip: "Total rent payable: $3,000/mo across 1 building" — this becomes "across N buildings" as you scale, which is exactly the rollup a portfolio owner needs.
    
- "No activity yet" looks wrong given there's an active Head Lease and (presumably) Purchase Invoices behind it — worth checking whether activity logging is actually wired for the Landlord doctype or just not populating.
    

One placement question worth revisiting: **Landlord Payment Method (PDC)** currently lives on the Building record, but it's really a term of the Head Lease (a landlord could have different payment terms per building or per lease renewal). Moving it onto the Head Lease record — while still showing it read-only on the Building page — would keep it accurate as head leases renew or landlords change payment terms over time.

## Tenant Page: Add a Balance Indicator

Jordan Ahmed's page shows lease and security deposit details but no current balance/arrears status — the exact "who owes me money" signal your dashboard prioritizes. Add a small status line near the top: "Current balance: $0 — up to date" or "Overdue: $2,000 (3 days)" pulled from the PDC/invoice status, so a business user checking a specific tenant doesn't have to cross-reference the PDC table manually.