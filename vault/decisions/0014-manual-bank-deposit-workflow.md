# ADR-0014: Manual Bank-Deposit Workflow for PDC Entries

- **Status**: accepted
- **Date**: 2026-08-27
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: 0026-outgoing-pdc-auto-deposit.md (Outgoing direction only — the Incoming/tenant-cheque half of this ADR is still in effect)

## Context

Imran asked how a cheque moves from Pending → Deposited (auto or manual?)
and proposed a different design: a daily page listing all due cheques
where an admin explicitly chooses which to send to the bank; a separate
reconciliation step then marks a sent cheque Cleared, auto-syncing the
linked invoice.

**Current mechanism, verified in code:** `payments/pdc.py::mark_pdc_deposited`
is a daily scheduler job with no human step — it flips *every* `PDC Entry`
(both directions) with `status=Pending` and `check_date <= today` straight
to `Deposited`, unconditionally. Its own docstring: "a cheque past its
date has been sent to the bank — this is a fact the system owns, not
something that needs confirmation." No record of *who* decided to deposit
it, or whether it physically happened that day.

**What already exists, so isn't new scope:** the reconciliation half
(`mark_pdc_cleared`/`mark_pdc_bounced`, auto-creating and reconciling a
Payment Entry) already works — verified live this session. Only the
*deposit* step needed to change.

## Decision

Replace the automatic scheduler transition with a manual one:

- `mark_pdc_sent_to_bank(pdc_names)` (new, whitelisted) — staff select
  which Pending, due (`check_date <= today`) cheques to mark Deposited.
  Same status transition as before, now explicit and attributable instead
  of a nightly sweep.
- `mark_pdc_deposited` (the automatic sweep) removed from
  `hooks.py::scheduler_events.daily` — no automatic fallback (Option B,
  not the grace-period hybrid).
- Surfaced on the two existing homes for each direction rather than a new
  unified page (smaller lift, matches how Clear/Bounce already works):
  - Accounts page (Incoming/tenant cheques): a new "Due for deposit"
    panel (select + "Send to bank"), and the top stat row's "PDC pending"
    card replaced with **"Cheques past due"** (Pending + `check_date <=
    today` — actionable, unlike the old raw pending count, which included
    cheques not due for weeks).
  - Head Lease detail page (Outgoing/landlord cheques): the same
    mechanism added to `HeadLeasePanel`, alongside its existing
    Clear/Bounce confirmation section.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A — Keep fully automatic | Zero staff effort | Doesn't reflect physical cheque handling; no accountability trail; "Deposited" stops meaning "this happened" | Rejected |
| **B — Manual deposit selection** | Matches real bank-run operations; real audit trail; reconciliation half already built | Adds a daily task; needs a past-due indicator so a missed day isn't silent | **Accepted** |
| C — Manual with automatic grace-period fallback | Best of both | More states to build/explain for an arbitrary grace period | Rejected — not needed |
| Overview Dashboard widget for past-due cheques | Central visibility | Accounts page's top stat row already exists and is the natural home; "PDC pending" there was low-value and could just be replaced instead of adding new surface area | Rejected in favor of replacing the existing Accounts stat card |

## Consequences

### Positive

- Trustworthy status field — "Deposited" means a person confirmed it.
- Accountability trail for cash handling that had none.
- The Accounts page's top row now shows something actionable ("N cheques
  past due, QAR X") instead of a raw pending count that included cheques
  not due for weeks.

### Negative

- Real behavior change: cheques no longer advance on their own. If
  forgotten, they sit Pending past due — the "past due" stat card is the
  mitigation, not a guarantee someone looks at it.
- One more manual daily habit on whoever runs Accounts/Head Lease pages.

## Implementation

- Feature file: `vault/payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`
