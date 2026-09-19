---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-24
updated: 2026-08-27
related_adr: ["0011-head-lease-landlord-payables"]
---

# Head Lease DocType

## Summary

Models the landlord side of the arbitrage relationship: the head lease we
hold on a `Building` from a `Landlord`, including its rent, term, escalation,
and its own payment schedule — the mirror image of `Lease Agreement` on the
tenant side. Without this, the cost side of the business (what we owe) has
no data model at all. See `vault/findings/2026-08-24-ideal-product-vs-
current-state.md` gaps G1/G2/G4 and `vault/decisions/0011-head-lease-
landlord-payables.md`.

## Requirements

- One `Head Lease` per `Building` (1:1 — ADR-0011 §Decision).
- Captures: `landlord`, `building`, `start_date`, `end_date`, `monthly_rent`,
  `payment_frequency`, `escalation_percent`, `escalation_frequency`,
  `security_deposit_paid`, `head_lease_status`.
- Child table `Head Lease Schedule`: `due_date`, `amount`, `status`
  (Pending / Invoiced / Paid / Overdue) — same shape as the existing
  `Rent Schedule` child table on `Lease Agreement`. For Buildings whose
  `landlord_payment_method` is `PDC`, each row additionally needs a linked
  `PDC Entry` before it can post (see `landlord-payables.md`).
- `head_lease_status` uses the same enum shape as the tenant-side
  `lease_status` (ADR-0012): Draft / Active / Expiring Soon / Expired /
  Renewed / Terminated.
- `Building` gets `landlord_payment_method` (Select: `PDC` / `Bank
  Transfer`, default `PDC`) — set at Building creation, editable later.
  This is a Building-level field (not Head Lease-level) precisely so it's
  configurable independent of any one lease's term, per Imran's requirement.
- **No Tenant permission row on Head Lease** (ADR-0011 amendment,
  confirmed with Imran) — unlike `Lease Agreement`, which has one for
  renter self-service (ADR-0009). Renters have no legitimate reason to
  see what's owed to a landlord.
- `Landlord` gets a `supplier` field (Link to `Supplier`, read-only),
  auto-created on insert — same pattern as the auto-created "Rent" Item
  and per-Building Cost Center. Required because `Purchase Invoice`
  needs a `supplier`; `Landlord` had no accounting identity at all
  before this (ADR-0011 amendment).

## Design

- New DocType `real_estate_os/real_estate/doctype/head_lease/head_lease.json`
  + child table `head_lease_schedule` (mirrors `Rent Schedule` including a
  `pdc_entry` link field, so PDC-path schedule rows can tie back to the
  cheque that satisfies them).
- `landlord_payment_method` field added to `building.json`.
- `Landlord.supplier` + `real_estate_os/real_estate/doctype/landlord/
  landlord.py::after_insert` — creates a matching `Supplier` record.
- `build_payment_schedule(head_lease)` in a new
  `real_estate_os/payments/landlord_payables.py`, mirroring
  `payments/invoicing.py::build_payment_schedule`.
- Depends on `cost-center-per-building.md` (done, PR #2, live) — every
  `Purchase Invoice` generated from a `Head Lease Schedule` row must
  carry the Building's `cost_center` or margin reporting breaks.
- Depends on the `PDC Entry` `direction` field from ADR-0011 (extends
  `pdc-schedule-generation-and-reconciliation.md`'s existing DocType) —
  built as a separate PR/feature, sequenced after this one so Head Lease
  exists before PDC Entry needs to link to it.

## Implementation Plan

- [x] ADR approved — `0011-head-lease-landlord-payables`
- [x] ADR amendment confirmed with Imran — Landlord.supplier
      auto-creation, no Tenant permission on Head Lease
- [x] `Landlord.supplier` field + auto-create-Supplier on insert — PR #6
- [x] `Head Lease` DocType + `Head Lease Schedule` child table — PR #6
- [x] `landlord_payment_method` field on `Building` — PR #6
- [x] `build_payment_schedule` — generate schedule rows automatically on
      `Head Lease.after_insert` (not a submit — Head Lease isn't
      submittable) — PR #9, see
      `payments-accounting/features/landlord-payables.md`
- [x] Link from `Building` to its active `Head Lease` — PR #6
- [ ] Backfill: data-entry the live Rustic building's real head lease terms

## Acceptance Criteria

- [x] A created Head Lease produces a full payment schedule (Head Lease
      isn't submittable — this happens automatically on insert instead,
      see `payments-accounting/features/landlord-payables.md`)
- [x] Building detail view shows the linked Head Lease and its terms
      (PR #12); `landlord_payment_method` was already editable via the
      portal's generic field-edit mechanism, no extra work needed
- [x] No orphaned Buildings without a landlord — resolved by
      `own-building-landlord.md` (2026-08-27): `Building.landlord` is now
      mandatory, so a self-owned Building points at the "Own Building"
      placeholder instead of being left blank. A landlorded Building
      still needs its own real Head Lease before Units can be added
      (`Unit.validate_head_lease_set_up`), which was always enforced.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0011-head-lease-landlord-payables.md`
- Feature: `payments-accounting/features/landlord-payables.md`,
  `payments-accounting/features/cost-center-per-building.md`,
  `payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`
