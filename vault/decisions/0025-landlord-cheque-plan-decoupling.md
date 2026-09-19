# ADR-0025: Decouple Landlord Cheque Plan from Accrual Invoice and Payment

- **Status**: accepted
- **Date**: 2026-09-05
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

ADR-0011 modeled landlord payables as a strict 1:1 chain: one
`Head Lease Schedule` row (one `payment_frequency` period, e.g. one month)
= one outgoing `PDC Entry` (one cheque) = one `Purchase Invoice` (posted at
cheque clearance, `pdc.py::_post_payable_for_outgoing_pdc`). This assumes
the landlord is paid in cheques whose count and cadence exactly match the
rent-accrual schedule.

That assumption breaks for a real, current case: **some landlords require
lump-sum advance payments** — e.g. 2 cheques/year, each covering 6 months
of rent, instead of 12 monthly cheques. In that case:

- The accrual schedule should stay monthly, because margin per building
  (ADR-0011's whole reason for existing) is computed as a native GL query
  per cost center, per period — collapsing 6 months into one posting
  wrecks that.
- The cheque plan is admin-entered at Head Lease setup (check numbers,
  amount per cheque, dates) — it cannot be derived from the accrual
  schedule 1:1 once counts diverge.
- A single cheque may need to fund several accrual invoices that don't
  exist yet at the moment it clears (a cheque clearing in January for a
  Jan–Jun advance has nothing to reference — the Feb–Jun invoices haven't
  posted yet).

This was also the proximate trigger: `/overview`'s "Margin by property"
showed **Rastec 21** (`BLD-00406`) at landlord cost = $0. Root cause,
confirmed live: zero Purchase Invoices exist for that Head Lease at all —
the PDC path only ever invoices at cheque clearance
(`vault/payments-accounting/features/landlord-payables.md`), and per
Imran, real-world cheques for that lease don't map 1:1 onto the monthly
schedule the way `generate_pdc_schedule_for_head_lease` assumes, so the
existing generator was never a fit for it.

Imran's own framing of the fix (verbatim): *"PDC should not be the
purchase invoice, that should go separately whereas the payment should
also be separate. But by the time you complete the Head Lease, all these
details should be entered so it gets generated too."*

## Decision

Split what is currently one 1:1 chain into three independent records,
connected by reconciliation rather than a rigid link:

1. **`PDC Entry` (cash plan)** — what cheques exist, admin-entered as part
   of completing the Head Lease. Add a repeatable input (new child table,
   tentatively `Head Lease Cheque Plan`, or a bulk-entry action on the
   existing "Generate outgoing cheques" button) capturing per cheque:
   `check_number`, `amount`, `check_date`. Count and per-cheque amount are
   free-form — no assumption they equal `monthly_rent × payment_frequency`
   or that there's one per accrual period. Submitting this input creates
   the Outgoing `PDC Entry` rows directly, replacing
   `generate_pdc_schedule_for_head_lease`'s current "one PDC per pending
   `head_lease_schedule` row" derivation.

   **Gate, not form field**: `Head Lease.head_lease_status` is blocked
   from transitioning to `Active` until this step is complete for
   `PDC`-method buildings (validated in `head_lease.py::validate()`,
   mirroring the existing `validate_one_live_head_lease_per_building`
   pattern) — it does not need to be filled in inline on the Head Lease
   form itself, just completed before the lease can go live.

2. **`Head Lease Schedule` → `Purchase Invoice` (accrual)** — unchanged in
   shape, decoupled in behavior. `head_lease_schedule` keeps being built by
   `build_payment_schedule` at `payment_frequency` granularity (monthly by
   default) and keeps posting one `Purchase Invoice` per due row. What
   changes: **all** buildings post on the accrual due-date scheduler tick
   (`generate_due_payables`), not just Bank Transfer ones. The `pdc_entry`
   field and the `not row.pdc_entry` guard in
   `landlord_payables.py::_generate_due_for_head_lease` are retired — a
   schedule row no longer needs (or waits for) a linked cheque to invoice.
   `Building.landlord_payment_method` stops gating *when* a Purchase
   Invoice posts; it only still describes *how* the landlord is actually
   paid (cheque vs. transfer) for the settlement step below.

3. **`Payment Entry` (settlement)**, decoupled from invoice creation.
   `mark_pdc_cleared` (Outgoing) no longer creates a Purchase Invoice
   (`_post_payable_for_outgoing_pdc` is removed) — by the time a cheque
   clears, the invoice(s) it pays may or may not exist yet. Instead:
   - Resolve the landlord's outstanding Purchase Invoices (by `supplier`,
     **oldest `posting_date` first**) and allocate the cheque amount
     against them via `get_payment_entry`, same as today, up to what's
     outstanding.
   - Any remainder (the cheque is larger than everything currently
     outstanding — the advance-before-invoices-exist case) posts as an
     **on-account Payment Entry** (`unallocated_amount`, no invoice
     reference) against the Supplier — natively supported by ERPNext's
     `Payment Entry` (`unallocated_amount`/`set_unallocated_amount`), not
     something new to build.
   - A new daily scheduler step, run immediately after
     `generate_due_payables` posts each period's invoice, **automatically**
     sweeps any outstanding on-account balance for that invoice's supplier
     against it, oldest-invoice-first — using ERPNext's own
     `Payment Reconciliation` tool
     (`erpnext.accounts.doctype.payment_reconciliation.payment_reconciliation`:
     `get_unreconciled_entries` → `allocate_entries` → `reconcile`),
     called programmatically rather than reimplemented. No staff
     confirmation step for this sweep — it's a mechanical allocation of
     cash the company already holds, not a new payment (clearance itself,
     the step that recognizes a cheque as bank-honored, stays the manual,
     staff-confirmed action ADR-0010 established).

**Cutover — no retroactive backfill.** This model applies automatically to
every Head Lease created from this point forward; there is no per-Building
toggle and no migration of Head Leases already built under the old 1:1
model. Periods already elapsed under the old PDC-path model (e.g. Rastec
21's entire history — zero invoices ever posted) are a one-time manual
data-entry pass, not an automated catch-up run. `ACC-PINV-2026-00003`
(Rastec 20's manually-created invoice from earlier verification testing,
never reconciled against a real cheque) is left untouched by this ADR —
handled separately as its own manual cleanup task.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Keep strict 1:1, force cheque count to match accrual periods | No new fields, no reconciliation-sweep step | Doesn't match the actual landlord term (lump-sum advances); the business requirement this ADR exists to satisfy is exactly what it can't express | Rejected |
| Prorate accrual synthetically for reporting only (no monthly Purchase Invoice; post one invoice per cheque, matching cash) | No advance/on-account reconciliation step needed at all | Margin per building stops being a native GL query per period (ADR-0011's core positive) — becomes a report-time approximation instead | Rejected — fallback only if the on-account sweep proves unworkable in practice |
| Auto-reconcile via bank feed instead of staff-confirmed clearance | Removes the manual "Mark Cleared" step entirely | Out of scope — reopens ADR-0010's explicit "no auto-clear, no bank feed" decision, which Imran confirmed staying manual for this round | Rejected |
| One Payment Entry per invoice only, no on-account/advance support | Simpler `mark_pdc_cleared` — no sweep step | Cannot represent a cheque that clears before its invoices exist — exactly the case this ADR is written for | Rejected |
| Staff manually confirms/adjusts the advance-sweep allocation before it posts | More control over edge cases (partial periods, disputes) | Imran confirmed oldest-outstanding-first automatic sweep is good enough; extra manual step not warranted | Rejected |
| Per-Building configurable cutover date | Lets specific existing buildings opt into the new model without a full manual re-entry | Imran confirmed "all new implementation should automatically have this" — a single forward-only cutover, no per-building knob, is sufficient | Rejected |

## Consequences

### Positive

- Landlord payment terms that don't map 1:1 onto the accrual cadence
  (advance lump sums, irregular cheque counts) become representable.
- Margin per building stays a native, monthly GL query — accrual posting
  no longer depends on cheque cadence or clearance timing.
- Reuses ERPNext's own `Payment Reconciliation` tool for the advance-sweep
  step rather than hand-rolling multi-invoice allocation.
- Fixes the root cause behind Rastec 21 showing $0 landlord cost: accrual
  invoices post on schedule regardless of the cheque plan's shape.

### Negative

- More moving parts: a new cheque-plan input surface, a retired
  `pdc_entry` gating field, and a new scheduler step for the advance
  sweep.
- Temporary optics: an accrual invoice can sit `Outstanding` for weeks
  after its cash was already received via an earlier advance cheque,
  until the sweep step reconciles them. Needs on-account balance surfaced
  somewhere (e.g. Building Profitability or the Head Lease page) so this
  doesn't read as a real receivable-side risk.
- `head_lease_schedule.pdc_entry` and `Head Lease Schedule` row status
  semantics (`Pending → Invoiced`) need to be re-verified once they're no
  longer gated by a cheque — low risk (the field/status already exist,
  only the gating logic changes) but touches code covered by
  `landlord-payables.md`'s existing acceptance criteria.
- No historical backfill — Rastec 21 (and any other PDC-path building with
  a similar gap) needs a manual one-time data-entry pass for
  already-elapsed periods, not something this ADR automates. Same for
  `ACC-PINV-2026-00003`.

## Implementation

- Feature file: `vault/payments-accounting/features/landlord-cheque-plan-decoupling.md`
  (new — implementation plan for all three pieces above). Also updates
  `vault/payments-accounting/features/landlord-payables.md` and
  `vault/payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`,
  whose "done" status this ADR partially supersedes.
