---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-16
related_adr: ["0004-recurring-invoicing"]
---

# Recurring Invoicing & Payment Schedule

## Summary

On lease signature, auto-generate a `Rent Schedule` (due dates + amounts) and the
first Sales Invoice. A daily scheduler job generates subsequent due invoices and
sends due-date reminders.

## Requirements

- Sales Invoice + Payment Schedule child table (due dates + amounts).
- Frequency: Monthly / Quarterly / Annual.
- Reminders 3 days before due date (email).
- Security deposit handled as refundable liability (see lease + accounting).

## Design

- Whitelisted + scheduler method `real_estate_os/payments/invoicing.py`:
  `create_invoice_for_lease(lease)`, `build_payment_schedule(lease)`,
  `generate_due_invoices()`, `send_reminders()`.
- `hooks.py` `scheduler_events.daily` runs `generate_due_invoices` +
  `send_reminders`.
- Payment Schedule derives from `start_date`, `end_date`, `payment_frequency`.
- Child table `Rent Schedule` (istable) on `Lease Agreement` — named to avoid a
  collision with ERPNext's native `Payment Schedule` child of Sales Invoice.
- Each generated Sales Invoice gets its native `payment_schedule` for free.

## Implementation Plan

- [x] Implement schedule generation from lease terms
- [x] Create Sales Invoice with Payment Schedule on signature
- [x] Scheduler job for next-invoice generation
- [x] Email reminder job (3 days before due)

## Acceptance Criteria

- [x] Signed lease produces correct schedule + first invoice
- [x] Daily job generates the next due invoice exactly once
- [x] Reminder sent 3 days pre-due

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `0004-recurring-invoicing`
