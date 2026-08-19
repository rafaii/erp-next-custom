# ADR-0004: Recurring Invoicing Mechanism

- **Status**: accepted
- **Date**: 2026-08-16
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Signed leases must auto-generate recurring rent invoices (monthly/quarterly/
annual) with a payment schedule, plus due-date reminders. ERPNext offers several
mechanisms; we must pick one that is self-contained (no external app) and
reliable for variable lease schedules.

## Decision

**Sales Invoice + Payment Schedule child table + custom daily scheduler job.**

- On lease signature: create the first Sales Invoice with a Payment Schedule
  (due dates + amounts) derived from `payment_frequency`.
- A custom scheduler job (registered in `hooks.py`) generates the next invoice
  on the schedule and fires due-date reminders (3 days before).
- No `auto_repeat`; no `Subscription` DocType (avoids the `simple_subscription` app dependency).

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| `auto_repeat` on Sales Invoice | Built-in | Limited control over variable schedules; awkward for payment schedules | rejected |
| `Subscription` DocType (simple_subscription app) | Purpose-built | External app dependency, not in base ERPNext | rejected |
| Payment Schedule + custom scheduler | Full control, self-contained, uses native Invoice+Payment Schedule | We own the scheduler logic | **chosen** |

## Consequences

### Positive

- Self-contained (no extra app); uses native ERPNext accounting objects.
- Precise control over frequency, reminders, and schedule regeneration.

### Negative

- Scheduler + schedule-generation logic is ours to build and test.

## Implementation

- Feature file: `vault/payments-accounting/features/recurring-invoicing.md`
