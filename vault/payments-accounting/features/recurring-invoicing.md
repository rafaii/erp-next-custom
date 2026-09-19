---
status: done
owner: developer-1
domain: payments-accounting
created: 2026-08-16
updated: 2026-08-27
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

## 2026-08-27 fix: don't invoice a period before it's due

"On signature, auto-generate... the first Sales Invoice" (above) never
checked whether that first period had actually arrived — only the daily
scheduler's `_generate_due_for_lease` gated on `due_date <= today`.
Surfaced live: a lease signed 2026-08-27 for a 2026-09-01 start (first
rent due 2026-09-10) was invoiced same-day, and since the invoice posts
with `today()`, that September rent counted as August revenue on
`get_accounts_data`'s posting-date-grouped chart — while the new
due-date-grouped Cash Flow Projection (`cash-flow-forecast.md`) correctly
placed it in September, producing a visible discrepancy between the two
(Imran caught this by comparing them side by side). Not just a display
bug: a tenant could be billed for next month's rent nearly two weeks
early. Fixed: `create_invoice_for_lease` now applies the same
`due_date <= today` gate the scheduler already used. The common case
(lease starts around now) is unchanged.

## Related

- Domain index: `vault/payments-accounting/payments-accounting.md`
- ADR: `0004-recurring-invoicing`
- Feature: `cash-flow-forecast.md`
