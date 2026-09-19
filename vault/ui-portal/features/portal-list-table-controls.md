---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-26
updated: 2026-08-26
related_adr: []
---

# Portal List Tables: Sort, Filter, Search, Pagination

## Summary

Imran asked for two things together: (1) Contracts lists every lease
including Cancelled with no way to filter, and its Status column shows
`esign_status` (Draft/Sent/Signed/Declined/Cancelled) rather than the
lease's actual lifecycle; (2) every list page (Properties, Contracts,
Tenants, Maintenance, Landlords) should have the same structure —
sortable columns, filters, working search, pagination.

All five are rendered by one shared component (`ResourceListView`), so
this was one generalized change, not five separate ones.

## Requirements

- Contracts' Status column shows the contract's real status (Draft,
  Active, Expired, Cancelled, etc.), not the e-sign workflow state.
- A filter control for status-like columns, so Cancelled/Expired
  contracts (or any other closed-set value) can be filtered out of view.
- Every column header is clickable to sort asc/desc.
- Search actually filters the currently-viewed table.
- Pagination with a selectable rows-per-page.
- All of the above applies uniformly to every `ResourceListView`-backed
  list page, not just Contracts.

## Design

- `hooks.py`: Contracts' `columns` fetches `lease_status` instead of
  `esign_status` — `lease_status` (Draft/Pending Signature/Active/
  Expiring Soon/Expired/Renewed/Terminated/Cancelled, ADR-0012) is the
  field that actually tracks a lease's lifecycle; `esign_status` only
  tracks the signing workflow and freezes at "Signed" forever once a
  lease is active, which is why "Cancelled" contracts were showing as
  whatever their e-sign state happened to be instead of their real state.
- `ResourceListView` (`ui/src/pages/Dashboard.tsx`), generalized once for
  every doctype it renders:
  - **Filters**: any displayed column whose key is in a small
    closed-set list (`status`, `lease_status`, `priority`, `issue_type`,
    `customer_type`, `landlord_type`) gets a dropdown, options derived
    from the loaded data. Contracts gets one (Status/`lease_status`);
    Maintenance gets three (status, issue_type, priority); Properties
    and Tenants/Landlords get one each. No column in that set → no
    filter shown for that page, rather than forcing an irrelevant one.
  - **Sort**: click a column header to cycle asc → desc → none, with a
    chevron indicator. Generic `compareValues` comparator: numeric if
    both values parse as numbers, date if both parse as dates
    (`Date.parse`), else case-insensitive string compare — no
    per-doctype/per-column sort configuration needed.
  - **Search**: the table toolbar previously had a decorative
    "Search {label}" chip with no input in it at all — search only
    worked through the global top-bar box. Replaced it with a real
    controlled input, seeded from the top-bar query but independently
    editable per page.
  - **Pagination**: rows-per-page selector (10/25/50/100, default 25)
    plus prev/next; the "Showing X–Y of Z" line and page controls sit
    below the table. Filter/search state resets the page back to 1.
  - Fixed a related bug found while doing this: previously, when a
    filter (only client-side search existed before) yielded zero rows,
    the *entire* toolbar disappeared along with the table — with no way
    to see or clear whatever caused the empty result. Filters/search now
    always render; only the table body swaps for an empty-state message.
  - `StatusBadge`: added a red/negative style for
    `cancelled`/`terminated`/`expired`/`declined`, and matched
    `expiring soon` (previously only the bare word `expiring` matched).

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; `py_compile` clean
- [x] Deployed and verified live on 2026-08-26: all 9 real Lease
      Agreements have `lease_status` populated (6 Cancelled, 3 Active —
      confirms the field this now displays/filters on isn't a partial
      migration), and the exact field list the Contracts page requests
      (`customer.customer_name`, `unit.unit_number`, `unit.building`,
      `monthly_rent`, `lease_status`, `start_date`, `end_date`) resolves
      correctly against real data.
- [ ] Not click-tested in an actual browser — no browser tool available
      in this environment. Sort/filter/pagination interactions haven't
      been clicked through live.

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `portal-list-column-fixes.md`, `contracts-page-fixes.md`
