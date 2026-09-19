---
status: analysis
owner: developer-1
domain: ui-portal
created: 2026-08-31
related_adr: ["0006-admin-portal-approach", "0011-head-lease-landlord-payables", "0012-lease-lifecycle-state"]
---

# All-Page UI Gap Analysis vs. Expert Wireframe

Compares `vault/expert-suggestions/all-page-ui-wireframe.md` (an expert's
back-and-forth review, including reactions to an actual pasted dump of our
live detail pages) against what's actually built in
`real_estate_os/hooks.py` (`portal_nav_items`, `portal_detail_links`) and
`ui/src/pages/Dashboard.tsx` (`ResourceListView`, `DetailView`) today.
Every finding below was verified against the live code (file:line) or an
existing vault decision record, not assumed from the wireframe's own
paraphrase of "what's missing" — several of its diagnosed causes turned
out to be slightly wrong even where the symptom was real.

## Gap table

| # | Wireframe capability | Status | Evidence |
|---|---|---|---|
| 1 | Master Lease vs. Tenant Lease are conflated in one data model | ❌ Misdiagnosed — **N/A**, data model already separate | `Head Lease` (`real_estate_os/real_estate/doctype/head_lease/head_lease.json`) has been its own DocType since ADR-0011, with its own `head_lease_status`, `landlord`, `monthly_rent`. `Lease Agreement` is exclusively the tenant side. Nothing is conflated in the data — the actual gap is #2. |
| 2 | No top-level way to see Master Leases (only Tenant Leases show as "Contracts") | ❌ Missing | `portal_nav_items` (`hooks.py:84-122`) has no "Head Leases" entry. `Head Lease` is only reachable as a linked-record table on Building (`hooks.py:170-171`) or Landlord (indirectly, via Building). `DOCTYPE_SLUGS` (`Dashboard.tsx:306`) already maps `"Head Lease": "head-lease"`, so detail-page navigation already works — this is a one-entry `portal_nav_items` addition, not new plumbing. |
| 3 | No Units entity | ❌ Misdiagnosed — **N/A**, entity already exists | `Unit` is a full DocType with `status`, `bedrooms`, `bathrooms`, `has_balcony`, `monthly_rent` (`hooks.py:167`), already shown as a Building linked-record table. `DOCTYPE_SLUGS` already maps `Unit: "units"` and `_PORTAL_DETAIL_ONLY_SLUGS` (`hooks.py:142`) already routes `/units/<name>`. |
| 4 | No top-level Units directory/list page | ❌ Missing | Same root cause as #2: `portal_nav_items` has no "Units" entry, so there's no portfolio-wide occupancy view — only per-building tables. `"units"` sits in `_PORTAL_DETAIL_ONLY_SLUGS` today (detail-view route only, no list). |
| 5 | No Rent Roll / Payments module | ❌ Missing, already planned | `vault/payments-accounting/features/rent-roll-and-arrears-report.md` — `status: planned` since 2026-08-24, unchanged as of this analysis. Zero code exists (`Implementation Plan` all unchecked). |
| 6 | "Own Building / Self" Landlord is a data-model hack to remove | ❌ Misdiagnosed — already a deliberate, documented decision | `vault/custom-module/features/own-building-landlord.md` (`status: done`, 2026-08-27): a real "Own Building" `Landlord` record (`landlord_type: "Self"`) was chosen deliberately after an incident, with `landlord` made mandatory on Building specifically so "no landlord" can never be silently blank. This is functionally equivalent to the wireframe's suggested `Ownership Type` field, just modeled as a value of an existing field (`landlord_type`) rather than a new one on Building. Don't re-litigate — see "What's already strong" below. |
| 7 | Landlords "With contact" KPI undercounts a landlord who has contact info | ✅ Real bug, root cause is different from the wireframe's guess | `Dashboard.tsx:1296-1305`: the metric checks `r.email_id \|\| r.phone \|\| r.mobile_no` — but Landlord's own email field is named `email`, not `email_id` (`landlord.json:61`, `"fieldname": "email"`), and the Landlords list doesn't even fetch it (`hooks.py:112`, `columns: ["landlord_name", "landlord_type", "phone"]`). The metric was written for Customer and reused verbatim for Landlord without adjusting the field name or the fetched columns — not a byproduct of the "Self" landlord placeholder as the wireframe assumed. |
| 8 | Properties list shows raw unit count, not occupancy % | ✅ Confirmed | `Dashboard.tsx:1269-1277`: the Building metric is literally `"Total units"` summed across rows, no occupied/vacant split anywhere in `ResourceListView`. `hooks.py:94` columns (`building_name, status, total_units, landlord.landlord_name`) carry no occupancy field either. |
| 9 | Contracts list has no "Lease Type" column | N/A until #2 ships | Once Head Leases get their own nav item (#2), Contracts stays 100% tenant-side by definition — no column needed. If Head Lease and Lease Agreement are ever shown in one merged list instead, a type column would apply then, but nothing today merges them. |
| 10 | Contracts — no "Expiring in 60 days" KPI | ❌ Missing, but the metrics mechanism to add it already exists | `ResourceListView`'s metrics builder (`Dashboard.tsx:1278-1286`) already has a per-doctype `if (item.doctype === "Lease Agreement")` branch computing "Monthly rent — Combined." Adding an expiring-count metric is an addition to an existing, working pattern, not new UI architecture. |
| 11 | Contracts — rename "Monthly rent: Combined" to "Total tenant rent" | Confirmed as literally named today | `Dashboard.tsx:1282-1284`: `label: "Monthly rent"`, `detail: "Combined"`. Trivial string change once Head Leases live in their own list (#2) removes any ambiguity. |
| 12 | Tenants list missing Current Unit / Lease End / Balance Due | ✅ Confirmed | `hooks.py:110` columns: `["customer_name", "customer_type", "email_id", "mobile_no"]` — no lease or balance fields at all. |
| 13 | Maintenance — add Unit/Reported Date/Cost/Days Open columns once populated | ⚠️ Partially met already | `hooks.py:111` columns already include `unit.unit_number`, `unit.building`; `total_cost` exists on the DocType (used by `get_building_profitability`, `get_owner_overview`) but isn't in the list columns yet. "Days Open" needs a computed field, not stored. |
| 14 | Detail pages need a Header → KPI Strip → Tabs template | ⚠️ Partial — Header exists, KPI Strip and Tabs do not | `DetailView` (`Dashboard.tsx:7231` onward) renders one long scroll: field list → linked-record tables (from `portal_detail_links`, config-driven, `hooks.py:149-215`) → Activity feed (`getActivity`). No KPI strip concept exists anywhere in `DetailView`, and no tab mechanism exists — every linked-record group renders as its own always-visible section (`Dashboard.tsx:553`, `g.doctype` section headers), not a tab switcher. |
| 15 | Every entity needs bidirectional link navigation (click Landlord → see its Buildings; click Building → see its Landlord) | ✅ Substantially met already | `portal_detail_links` already gives Building → Units/Lease Agreements/Head Lease, and Landlord → Buildings (`hooks.py:181-187`). The Building → Landlord direction is a plain field value shown on the Building form itself (not yet a clickable `[→]` link — see #16). |
| 16 | Linked names should be clickable `[→]` links to the related entity's own detail page | ⚠️ Partial | Linked-record *table rows* are clickable (standard `DetailView` row navigation). Scalar Link *fields* shown in the field list (e.g. Building's own "Landlord: Ooredoo Properties" field) render as plain text/`value_label`, not a navigable link, per `get_doc_detail`'s field serialization (`api.py:1046-1057`) — no `onClick`/`navigate` wired for scalar Link field values specifically. |
| 17 | Documents tab on every entity | ❌ Missing | No document/attachment concept surfaced anywhere in `DetailView`, `get_doc_detail`, or `portal_detail_links`. Frappe's native File attachments exist on every doctype at the framework level but aren't exposed in the portal UI. |
| 18 | Building detail page — the arbitrage spread is invisible | ✅ Confirmed, but it's a wiring gap, not new computation | `get_building_profitability()` (`api.py:431-491`) already computes exactly this (income, landlord cost, maintenance cost, net margin) per building — it's called only from `OverviewView` (`Dashboard.tsx:901`, `1121`, inside `BuildingProfitabilityPanel`). Zero references to it exist inside `DetailView`. Occupancy % itself isn't part of that function's return either — would need a small addition (unit status counts), same pattern already used in `get_owner_overview`. |
| 19 | Dev/decision notes as permanent body text instead of tooltips | ✅ Confirmed | `description` on every field (`api.py:1053`, `field.description = df.description or None`) is rendered wherever the frontend shows field help — this is exactly the vault-cross-referencing prose visible in the wireframe's pasted dump ("Not a fetch_from field on purpose...", "Auto-created on save so a Head Lease can post..."). No tooltip/hover component wraps it today; it renders inline. |
| 20 | Lease Agreement page shows two different IDs for the same record (header vs. "View signed contract" link) | ✅ Confirmed, root cause found | `Lease Agreement.json:193`: `"title_field": "customer"`. `get_doc_detail`'s title logic (`api.py:1072`): `title = doc.get(meta.title_field) if ... else name` — for Lease Agreement this evaluates to the raw `customer` **Link value** (e.g. `CUST-2026-00006`), not the customer's display name and not the lease's own `name`. The page H1 (`Dashboard.tsx:171`) renders `doc.data?.title` first, so it shows the Customer ID. Meanwhile "View signed contract" (`Dashboard.tsx:6239`) correctly uses the `name` prop (the real `LSE-YYYY-#####` id) since it never goes through `title` at all — two different fields, both technically "correct" for their own purpose, producing two different IDs on one page. |
| 21 | PDC list is unsorted (0010, 0011, 0021, 0020...) | ✅ Confirmed, root cause found | `get_linked_records`'s `"field"` branch (`api.py:1198-1203`, used for Lease Agreement → PDC Entry) calls `frappe.get_all(link_doctype, filters={link["field"]: name}, fields=...)` with **no `order_by`** — Frappe's default order is `modified desc`, i.e. edit order, not `check_date`. Every other linked-record config in `portal_detail_links` has this same gap; PDC is just the one where it's visible because rows get touched out of date order (PDC #0011 was marked "sent to bank" recently, bumping its `modified` timestamp above later-dated but untouched rows). |
| 22 | No overdue flag on PDC entries once check_date passes | ✅ Confirmed, computation already exists elsewhere | `PDC Entry.status` options are literally `Pending\nDeposited\nCleared\nBounced\nCancelled` (`pdc_entry.json:115`) — there is no stored "Overdue" state. But `get_accounts_data`'s `pdc_past_due` query (`api.py:313-318`, `status="Pending" AND check_date <= today`) already computes exactly this derived flag for the Actions page. The Lease Agreement detail page's PDC table is pure `portal_detail_links` config with no computed-column mechanism, so it can't reuse that query as-is without either a `method`-based link (like `get_building_leases`) or a frontend-side derivation on the already-fetched rows. |
| 23 | Landlord page needs a Head Leases tab | ❌ Missing, trivially cheap | `Head Lease.landlord` is a direct Link field (`head_lease.json:28-33`). Adding `{"label": "Head Leases", "doctype": "Head Lease", "field": "landlord", "columns": [...]}` to Landlord's `portal_detail_links` entry (`hooks.py:181-187`) is a config-only addition — the exact same mechanism Building already uses for its own Head Lease link, no new backend code. |
| 24 | Landlord page needs a Payments Made tab (Purchase Invoices against the Supplier) | ❌ Missing, needs one small new function | `Landlord.supplier` is a direct read-only Link field (`landlord.json:45-49`), and `landlord_payables.py` already creates Purchase Invoices against it (`create_payable_for_head_lease`, `landlord_payables.py:61-100`). But Purchase Invoice has no field pointing at the *Landlord's own name* — only at its Supplier — so this can't use the plain `"field"` link mechanism. Needs a `method`-based link (`get_landlord_payments_made(landlord_name)`, resolving `Landlord.supplier` then querying Purchase Invoice by it), the same idiom already used twice (`get_building_leases`, `get_customer_security_deposits`). |
| 25 | Landlord page needs a KPI strip ("Total rent payable: $X/mo across N buildings") | ❌ Missing | No KPI strip mechanism exists in `DetailView` at all (see #14) — this is blocked on that broader gap, not landlord-specific. |
| 26 | "No activity yet" looks wrong for an active Landlord | ✅ Confirmed as expected, not a bug | `doc_events` (`hooks.py:28`) has no entry for `"Landlord"` at all — activity was never wired for this doctype (no `add_activity` call anywhere references it). The empty state is accurate: nothing has ever been logged, not a fetch failure. |
| 27 | Landlord Payment Method (PDC/Bank Transfer) belongs on Head Lease, not Building | Design question, not yet evaluated | `Building.landlord_payment_method` is the current field (per the wireframe's own pasted dump). Not independently re-derived here — flagging as a legitimate open design question for Imran, since moving it changes an established field's meaning across `landlord_payables.py`'s existing consumers; out of scope for this analysis pass. |
| 28 | Tenant page needs a balance/arrears indicator | ❌ Missing, cheap to add | No per-Customer balance query exists yet in `api.py`. `get_balance_on(party_type="Customer", party=customer)` (already imported and used elsewhere for cash-on-hand, `api.py:560`) or a direct `Sales Invoice.outstanding_amount` sum (the same idiom `get_owner_overview`'s `overdue` list already uses, `api.py:647-675`) would both work — this is "reuse an established pattern," not new capability. |

## What's already strong (don't re-litigate these)

- **Head Lease as its own DocType (#1)** — the wireframe's single biggest
  flagged risk ("you can't compute your spread without a master lease
  record") is already fully solved at the data-model level, has been
  since ADR-0011. The only real gap is navigational (#2), not structural.
- **Unit as its own DocType (#3)** — same pattern: the wireframe reacted
  to the *absence of a sidebar item*, not an absence of the underlying
  entity, which already carries real occupancy/rent/bedroom data.
- **"Own Building" Landlord (#6)** — this is a considered, documented
  design decision from an actual incident
  ([[own-building-landlord]]), not an unexamined hack. The wireframe's
  proposed `Ownership Type` field would be functionally redundant with
  the existing `landlord_type: "Self"` value.
- **`portal_detail_links` config mechanism (#15, #23)** — the wireframe
  asks for "one template, five entities" bidirectional linking as if it
  needs to be built; the config-driven linked-records system already
  *is* that template, just needs entries added, not new architecture, for
  most of the requested tabs (Head Leases on Landlord is a one-line add).
- **`get_building_profitability` (#18)** — the "spread" computation the
  wireframe calls the single most critical missing number already
  exists, tested, and live on the Overview page. This is a placement gap,
  not a modeling gap.

## Phased implementation plan

Phased by cost-to-value, matching the accounting gap analysis's own
convention. Each phase is independently shippable.

### Phase 1 — Sidebar completeness (small, high value)

- [x] **Revised 2026-08-31 (Imran): no new sidebar items** — "Not sure if
      we need to add new sidebar items. Lesser the better." Units stays
      exactly as-is (Building's own linked-records table only, no
      portfolio-wide list). Head Lease got a tab inside the existing
      Contracts page instead of its own nav item — `ContractsView`
      (`Dashboard.tsx`), a client-side-only tab switch between "Tenant
      Leases" (Lease Agreement, unchanged) and "Landlord Leases" (Head
      Lease, browse-only — it has no standalone create flow, always
      created from a Building's "Set up Head Lease" dialog). No new
      `portal_nav_items` entry, no new backend endpoint — reuses
      `ResourceListView`'s existing generic doctype+columns rendering.
      Shipped real-estate PR #58, deployed 2026-08-31.
- [ ] Add a Head Leases tab to Landlord's `portal_detail_links` (#23) —
      one config entry, zero backend code. Still open — this is the
      Landlord *detail page* itself (Ooredoo Properties → its Head
      Leases), a different surface from the Contracts list tabs above.
- [x] Rename "Monthly rent — Combined" metric label to "Total tenant
      rent" on the Contracts list (#11) — done as "Tenant rent" (next to
      a new "Landlord rent" metric on the Landlord Leases tab) as part of
      PR #58, since both now sit on the same page.

### Phase 2 — Data-bug fixes (small, high trust value) — ✅ done 2026-08-31

- [x] Fixed Lease Agreement's page title (#20): dropped
      `title_field: "customer"` entirely so `get_doc_detail` falls back
      to `name` (the `LSE-` id), matching every other doctype. Verified
      post-`bench migrate` that `frappe.get_meta(...).title_field`
      actually cleared to `None` — a removed JSON key doesn't always
      clear the synced DB column, so this was checked live, not assumed.
- [x] Added an optional per-link `order_by` to `get_linked_records`'s
      `"field"`/`"filters"` branches (#21), set to `"check_date asc"` on
      both PDC-driven links (Lease Agreement's Post-Dated Cheques, Head
      Lease's Outgoing Cheques) — the two tables that actually carry a
      meaningful date to sort by. Other linked tables left on Frappe's
      default order for now (not touched, no reported problem there).
- [x] Fixed the Landlords "With contact" metric (#7): checks `email` (not
      just `email_id`, which is Customer's field name) and `email` is
      now fetched in the Landlords list `columns`.
- [x] Surfaced a computed "Overdue" status for PDC rows on the same
      linked tables (#22) — server-side in `get_linked_records`
      (display value only), scoped exactly to PDC Entry rows rather than
      a generic status-field heuristic, using the server's `today()` so
      it can never disagree with `get_accounts_data`'s `pdc_past_due`
      definition. An earlier frontend-only draft was caught by advisor
      review for both of those risks before it shipped.

Shipped real-estate PR #59, deployed 2026-08-31 (required `bench migrate`
for the title_field removal, plus a cache clear for the hooks.py
changes — both run and verified live in a fresh process post-deploy).

### Phase 3 — Rent Roll / Payments — ✅ done 2026-08-31

- [x] Built `vault/payments-accounting/features/rent-roll-and-arrears-
      report.md` — shipped as a new "Rent Roll" tab on the existing
      Accounts page (not `portal_nav_items` — the doc's own original
      "add it to the nav" plan predated the tab-bar pattern established
      by Phase 1/PR #58; no new sidebar item, matching "lesser the
      better"). `get_rent_roll()` + `RentRollPanel`, real-estate PR #60.

### Phase 4 — Detail-page template (larger, foundational)

- [ ] Add a KPI-strip slot to `DetailView`, doctype-specific like
      `ResourceListView`'s existing metrics builder (#10) — start with
      Building (occupancy %, spread, wiring in the already-built
      `get_building_profitability`, #18) and Tenant (balance due, #28)
      since both reuse existing backend logic.
- [ ] Evaluate whether tabs are actually needed given every entity here
      has ≤4 linked-record groups today (Building has the most: Units,
      Lease Agreements, Head Lease) — the wireframe's "Header → KPI Strip
      → Tabs" template may be over-engineered for the current data
      volume; a KPI strip alone might close most of the perceived gap
      without a tab-navigation rewrite. Worth a direct conversation with
      Imran before building tab infrastructure nothing needs yet.
- [ ] Add `Payments Made` tab to Landlord (#24) — one new `method`-based
      link function, `get_landlord_payments_made`.
- [ ] Make scalar Link field values in the field list clickable (#16) —
      touches `get_doc_detail`'s field serialization and `DetailView`'s
      field renderer together.
- [ ] Convert field `description` text into hover/click tooltips instead
      of permanent inline text (#19) — pure frontend, no backend change,
      but touches every field row in `DetailView`.
- [ ] Documents tab (#17) — genuinely new capability; no attachment UI
      exists in the portal today, would need a new Frappe File-list
      endpoint plus upload/download UI, unlike everything else in this
      list which reuses existing data.

### Not scoped here — flagged for a decision, not a build item

- [ ] Whether `Building.landlord_payment_method` should move to `Head
      Lease` (#27) — a real design tradeoff (per-building vs.
      per-head-lease-term), not evaluated in this pass.

## Related

- Source: `vault/expert-suggestions/all-page-ui-wireframe.md`
- `vault/custom-module/features/own-building-landlord.md`
- `vault/custom-module/features/head-lease-doctype.md`
- `vault/custom-module/features/landlord-doctype.md`
- `vault/payments-accounting/features/rent-roll-and-arrears-report.md`
- `vault/payments-accounting/features/cost-center-per-building.md`
- `vault/decisions/0012-lease-lifecycle-state.md`
- `vault/ui-portal/features/admin-portal-ui.md`
- Same-pattern precedent: `vault/payments-accounting/accounting-ui-gap-analysis.md`
