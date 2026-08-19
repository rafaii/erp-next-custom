---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-17
related_adr: ["0005-counter-signature-pdc"]
---

# Post-Dated Cheque (PDC) Processing

## Summary

`PDC Entry` standalone DocType records post-dated cheques against a lease.
The `Rent Schedule` is derived from PDC `check_date`s (amount = PDC amount)
when PDCs exist, falling back to the monthly/quarterly/annual cadence when they
don't. Schedule creation is idempotent (built once, dedup at send + finalize).

## Requirements

- `PDC Entry` fields: `pdc_id` (auto), `lease`, `customer` (fetched),
  `check_number`, `check_date`, `amount`, `bank_account`, `status`
  (Pending/Deposited/Cleared/Bounced/Cancelled), `deposit_date`.
- `build_payment_schedule` uses PDC dates when PDCs exist; monthly fallback.
- Idempotent schedule: build only when the schedule is empty.
- Schedule created at send time; finalize creates it only if still missing.
- First invoice at finalize, dedup-by-status (already exists).

## Design

- `PDC Entry` DocType at `real_estate/doctype/pdc_entry/`.
- `payments/invoicing.py::build_payment_schedule` reads PDC entries, sorts by
  `check_date`, emits one schedule row per PDC; skips if schedule non-empty.
- `send_lease_for_signature` calls `build_payment_schedule` after sending.
- `cancel_lease` (contract cancellation) marks pending PDCs as Cancelled.
- Deposit/bounce scheduler (`process_due_pdc`) remains a later slice.

## Implementation Plan

- [x] Add `PDC Entry` DocType (pdc_id, lease, customer, check_number, check_date, amount, bank_account, status, deposit_date)
- [x] `build_payment_schedule`: idempotent guard (skip if non-empty); PDC-driven rows with monthly fallback
- [x] `send_lease_for_signature`: build schedule at send time (dedup via idempotent guard)
- [x] `cancel_lease` marks pending PDCs as Cancelled (contract cancellation path)

## Acceptance Criteria

- [x] PDC Entry created and linked to lease
- [x] Schedule rows derived from PDC check_dates when PDCs exist
- [x] No PDCs → monthly/quarterly/annual fallback
- [x] Schedule built exactly once (no duplicates across send + finalize)
- [x] Cancelling a contract cancels pending PDCs and stops future schedule rows

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `vault/decisions/0005-counter-signature-pdc.md`
- Feature: `recurring-invoicing.md`
