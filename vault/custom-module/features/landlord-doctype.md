---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-19
updated: 2026-08-20
related_adr: ["0006-admin-portal-approach"]
---

# Landlord DocType

## Summary

New `real_estate_os`-app DocType `Landlord` (app-specific; NOT part of the portal
shell/framework) representing the **building owner** from whom the company leases
a building. The company may own a building itself, in which case the Landlord
references the operating Company (self-owned).

## Requirements

- Fields: `landlord_name`, contact/address, `landlord_type` (Individual /
  Company / Self), `company` (Link Company, set when self-owned), bank/payment
  details for lease payouts.
- `Building` links to a `landlord` (Link Landlord).
- One Landlord → many Buildings.

## Design

- DocType JSON at `real_estate_os/real_estate_os/real_estate/doctype/landlord/`.
- `landlord_type == "Self"` → links the operating Company (internal ownership;
  no external rent-payable flow).
- Add `landlord` Link field on the Building DocType.

## Implementation Plan

- [x] Create Landlord DocType JSON + fields
- [x] Add `landlord` Link field on Building
- [x] Naming series + role permissions
- [x] Verify on fresh install

## Acceptance Criteria

- [x] Create Landlord (Individual / Company / Self) — appears on Building form
- [x] Self-owned building → Landlord linked to Company

## Follow-up (2026-08-27)

Two things found and fixed while this was actually put to use for the first
time — see `own-building-landlord.md` for the first, this file's own note
for the second:

- `landlord_type == "Self"` was defined here in the original design (and in
  `demo_data.py`'s seed data) but the app's actual logic never checked it
  anywhere — a "Self" Landlord was treated exactly like any other real
  landlord (still required a Head Lease, still got a Supplier). Wired up
  for real in `own-building-landlord.md` (PR #21), which also made
  `Building.landlord` mandatory and seeded a shared "Own Building" record.
- The `/landlords` list page's "Add" button did nothing at all — no
  `NewLandlordDialog` had ever been built, and `canCreate` didn't include
  `"Landlord"`, so the button silently rendered disabled. Fixed in PR #22.
- The "Self → links the operating Company" design note above was never
  actually implemented (the `company` field is a plain optional Link,
  nothing auto-sets it) — the acceptance checkbox below was stale. Left
  as-is rather than building that now; nothing currently depends on it,
  and `own-building-landlord.md`'s "no Head Lease, no Supplier" treatment
  doesn't need it.
- `bank_name` was a plain Data field (free text) — every other bank
  reference in the app (PDC Entry, Security Deposit) is a Link to
  Cheque Bank. Changed to match (PR #23), so New Landlord's Bank Name
  field uses the existing `BankSelect` picker (configured banks +
  inline "+ Add bank") instead of a text box, and the generic edit form
  gets a proper Link picker for it too. `demo_data.py`'s 3 seeded
  Landlords updated to resolve against real Cheque Bank records
  (get-or-create) instead of plain strings, which would otherwise fail
  Link validation.
- Imran then asked why the generic Landlord *edit* form's Bank Name
  field didn't get the same inline "+ Add bank" treatment, and what the
  read-only Supplier field even is (PR #24). The edit form's generic
  `EditableField` renderer dispatched every Link field to a plain
  search-and-pick widget with no inline create — fixed generically by
  making it use `BankSelect` for any Link targeting Cheque Bank, so this
  isn't Landlord-specific (Security Deposit/PDC Entry's `tenant_bank`
  get it too, if ever edited through this same generic form). Supplier's
  answer: it already had a `description` in the DocType JSON explaining
  it's auto-created so a Head Lease can post a Purchase Invoice against
  this landlord — `get_doc_detail` just never returned it and nothing
  rendered it, so there was no way to find out except reading the
  source. Now `description` (and `read_only`) are returned for every
  field and shown as a caption; found and fixed a related bug while
  doing this — read-only fields (Supplier, Building.cost_center)
  rendered as normal editable pickers in edit mode, now render as plain
  read-only text.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `0006-admin-portal-approach`
- Feature: `building-doctype.md`, `own-building-landlord.md`
