---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-16
updated: 2026-08-17
related_adr: ["0002-esign-approach"]
---

# Lease Agreement DocType

## Summary

Custom DocType `Lease Agreement` linking Customer + Unit, generating a PDF
contract, and driving the e-sign workflow and recurring invoicing.

## Requirements

- Fields: `lease_id` (auto `LSE-{YYYY}-{#####}`), `customer`, `unit`,
  `start_date`, `end_date`, `monthly_rent` (auto-filled from Unit),
  `payment_frequency` (Monthly / Quarterly / Annual), `security_deposit`,
  `contract_pdf`, `esign_status` (Draft / Sent / Signed / Declined),
  `esign_provider`, `esign_envelope_id`.
- Validate unit is Vacant (blocks double-lease).
- On save: generate PDF via Print Format.
- On submit/signed: trigger e-sign + recurring invoice creation.
- Unit status lifecycle: Vacant → Reserved on lease create → Occupied on
  full signature → Vacant on lease delete (release).
- Activity feed: lease + unit timeline events for created / sent / signed.
- Cancel contract: easy action that cancels pending PDCs, stops future
  invoices, voids the open e-sign document, and releases the unit.

## Design

- DocType JSON at `real_estate_os/real_estate_os/doctype/lease_agreement/`.
- Print Format `Lease Agreement` (Jinja) for contract PDF.
- `monthly_rent` auto-fetch from `unit.monthly_rent`.
- `esign_status` transitions drive downstream actions (see `esign` domain).
- Unit field carries `link_filters` `[["Unit","status","=","Vacant"]]` so only
  vacant units appear in the selector (defense-in-depth with `validate_unit_vacant`).
- `after_insert`/`on_update`/`on_trash` on the controller manage the unit
  status; `real_estate_os/utils.add_activity` writes timeline entries.

## Implementation Plan

- [x] Create Lease Agreement DocType JSON + fields
- [x] Add naming series `LSE-.YYYY.-.#####`
- [x] Add Print Format template (Phase 2) — dedicated contract template (Jinja) at `real_estate/print_format/lease_agreement/`, set as `default_print_format`
- [x] `validate()`: unit Vacant, dates consistent
- [x] Hook e-sign send (Client Script "Send for E-Signature" → `send_lease_for_signature`)
- [x] Hook invoice creation on signature (Phase 3, `recurring-invoicing.md`)
- [x] Unit status lifecycle: `after_insert` → Reserved, e-sign `_finalize` → Occupied, `on_trash` → release to Vacant
- [x] Activity feed: lease + unit timeline entries for created / sent-for-esign / signed (via `real_estate_os.utils.add_activity`)
- [x] Unit field `link_filters` so only Vacant units are selectable (blocks double-lease)
- [x] Cancel contract action: `cancel_lease` marks lease Cancelled, cancels pending PDCs + rent-schedule rows, voids open e-sign doc, releases unit, writes activity

## Acceptance Criteria

- [x] Lease auto-fills rent from unit; rejects occupied unit
- [x] PDF generated from print format
- [x] Signed lease creates invoice (see `recurring-invoicing.md`)
- [x] Unit becomes Reserved on lease create, Occupied on full signature, Vacant on delete
- [x] Lease + Unit activity shows created / sent / signed events
- [x] Second lease for a leased unit is blocked
- [x] Cancel contract cancels PDCs, stops future invoices, releases unit, blocks re-sign

## Related

- Domain index: `vault/custom-module/custom-module.md`
- Feature: `unit-doctype.md`; domains `esign`, `payments-accounting`
