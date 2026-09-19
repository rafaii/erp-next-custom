---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-26
updated: 2026-08-26
related_adr: []
---

# Contracts Page Fixes: Cancel Draft Contracts + Readable Columns

## Summary

Two bugs Imran hit looking at Contracts: (1) no way to cancel a Draft or
Sent contract — the "Cancel contract" button only existed for Signed
leases; (2) the Contracts table showed raw Customer/Unit IDs
(`CUST-2026-00006`, `UNT-00062`) instead of names.

## Requirements

- A Draft, Sent, or manually-signed-but-not-yet-finalized contract needs
  the same "Cancel contract" action Signed contracts already have — same
  confirmation, same `cancel_lease` call.
- The Contracts list shows the customer's name and the unit's number, not
  their internal IDs.

## Design

- `ui/src/pages/Dashboard.tsx::LeaseSigningPanel`: added the existing
  Cancel button (unchanged `handleCancel`/`cancelLease` call) to the two
  branches that didn't have it — the manual-signing branch (Print/Upload/
  Finalize) and the not-yet-signed branch (Send for e-signature). Only
  the Signed branch had it before.
- `hooks.py::portal_nav_items`'s "Contracts" entry: columns changed from
  `["customer", "unit", ...]` to `["customer.customer_name",
  "unit.unit_number", ...]` — Frappe's REST list endpoint resolves a
  `link.fieldname` column via a join and keys the response by the target
  fieldname (`customer_name`, not `customer.customer_name`).
  `ResourceListView`'s column derivation now strips the `link.` prefix to
  get the actual lookup key, so the display column reads the resolved
  name instead of the raw Link ID.
- Scoped to the Contracts page only (what was reported) — the same
  `link.fieldname` mechanism would also fix the same raw-ID display on
  other list pages (Buildings' Landlord column, etc.) if asked for later.

## Implementation Plan

- [x] Cancel button added to both previously-missing `LeaseSigningPanel` branches
- [x] Contracts columns switched to dotted link-display fields
- [x] `ResourceListView` column-key derivation strips the `link.` prefix

## Acceptance Criteria

- [x] `tsc -b && vite build` clean
- [x] Deployed and verified live on 2026-08-26: a real (throwaway) Draft
      lease was cancelled end-to-end via `cancel_lease` (Draft →
      Cancelled — the same call the new button makes); a live query
      mirroring the Contracts list's actual field request
      (`customer.customer_name`, `unit.unit_number`) confirmed real names
      resolve correctly instead of raw IDs.

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `admin-portal-ui.md`
