# ADR-0026: Auto-Deposit Outgoing (Landlord) Cheques on Their Own check_date

- **Status**: accepted
- **Date**: 2026-09-08
- **Deciders**: Imran
- **Supersedes**: partially supersedes 0014-manual-bank-deposit-workflow (Outgoing direction only — Incoming is untouched)
- **Superseded by**: -

## Context

ADR-0014 made the Pending → Deposited transition a deliberate staff
action for *both* PDC directions, replacing a fully-automatic nightly
sweep, on the reasoning that "Deposited" should mean a person confirmed
it physically happened, with an accountability trail for cash handling.

Imran, live: for Outgoing (landlord) cheques specifically, that
reasoning doesn't hold. The business hands every landlord cheque over
to the landlord up front, at Head Lease cheque-plan creation — not on
some later day the business chooses to "send it to the bank." The
landlord is the one who actually deposits it, on their own schedule,
outside this system entirely. There is no discrete "we sent this today"
moment for staff here to record — the cheque's own `check_date` arriving
already is the fact.

Separately, while testing the reconciliation half (`mark_pdc_cleared`)
on a cheque already manually marked Deposited, found live: `Could not
find Mode of Payment: Cheque`. `payments/pdc.py` hardcodes
`payment_entry.mode_of_payment = "Cheque"` on both the invoice-
reconciliation and pure-advance settlement branches — another instance
of this session's recurring class of bug (Territory, Customer Group,
Supplier Group, ...): ERPNext's Setup Wizard seeds a default `Mode of
Payment` list via `install_fixtures.py`, never run by a headless `bench
new-site --install-app`. Worth noting for future-proofing: that fixture
itself names this record **"Check"** (not "Cheque") when the Company's
country is "United States" — the real_estate_os code has no such
branch, so a proper per-country seed would have hit this exact bug for
US tenants regardless of the provisioning gap.

Fixing that surfaced a *third* instance of the same class of bug on the
same clear attempt: `mark_pdc_cleared` on the real cheque (PDC-00046,
us.nnuggets.com) then failed with a raw `TypeError: 'NoneType' object
is not subscriptable` inside `num2words`, called via Payment Entry's
own `set_total_in_words` → `frappe.utils.money_in_words`/`in_words`,
which reads `frappe.local.lang`. Traced to `System Settings.language`
and `.time_zone` — both mandatory (`"reqd": 1`) fields with no schema
default, normally filled in by ERPNext's Setup Wizard — being blank on
every tenant site here. Not PDC-specific: this breaks
`money_in_words()`/`in_words()` (and therefore any Payment
Entry/Sales/Purchase Invoice "Amount in words") the moment nothing else
in a request happens to pin a language first. Same gotcha as this
session's own Global Defaults fix: `SystemSettings.on_update()` is what
actually propagates `language` into `frappe.db.set_default("lang",
...)` — a raw `frappe.db.set_single_value` would have silently not
fixed this, same as it silently didn't fix `default_company` earlier.

## Decision

**Outgoing only** — Incoming (tenant) cheques keep ADR-0014's manual
"Due for deposit" workflow unchanged; that accountability trail is
still correct for cash a tenant physically hands this business.

- New scheduler job `payments.pdc.auto_deposit_outgoing_pdc` (daily):
  finds every `PDC Entry` with `direction=Outgoing`, `status=Pending`,
  `check_date <= today`, and calls the existing `mark_pdc_sent_to_bank`
  on them — same `deposit_date` + Head Lease activity-log entry an
  explicit deposit would have produced, just system-initiated.
- Removed the now-dead "Due for deposit" UI for Outgoing cheques: the
  `DueForDepositOutgoingPdcPanel` on the global Actions page, and the
  equivalent section in the Head Lease detail page's `HeadLeasePanel`.
  An Outgoing cheque will no longer sit Pending past its own date for
  staff to act on. The "confirm Cleared/Bounced" panels (both places)
  are untouched — that reconciliation step stays a deliberate action.
- Seeded `Mode of Payment` "Cheque" the same way as the other missing
  ERPNext Setup Wizard fixtures found this session (Territory, Customer
  Group, Supplier Group): a new patch
  (`seed_default_mode_of_payment`), picked up automatically for every
  future tenant by `provisioning._run_own_patches`, and backfilled onto
  all 4 existing tenant sites. Seeded the literal name the code already
  references ("Cheque") rather than mirroring ERPNext's own
  country-dependent naming — sidesteps the US "Check" mismatch entirely
  rather than reproducing it.
- Set `System Settings.language`/`.time_zone` to `"en"`/`"UTC"` in
  `provisioning.finalize_new_tenant` (only if unset, saved through the
  Document so `on_update()` actually propagates them), and backfilled
  the same on all 4 existing tenant sites.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **Auto-deposit Outgoing only, keep Incoming manual** (chosen) | Matches the actual physical workflow for each direction; no wasted staff clicks on a queue that can never mean anything for Outgoing | Two different behaviors for what looks like the same doctype/status field | **Accepted** |
| Auto-deposit both directions (revert ADR-0014 entirely) | Simpler, one rule | Reintroduces exactly the accountability problem ADR-0014 fixed for Incoming, which Imran did not ask to revisit | Rejected |
| Keep Outgoing manual too, just fix the Mode of Payment bug | Smallest change | Leaves a genuinely pointless daily manual task for something the business never chooses the timing of | Rejected |

## Consequences

### Positive

- No more manual daily habit for a queue that never reflected a real
  staff decision in the first place.
- The Mode of Payment gap is fixed for every tenant, present and
  future, not just patched around for one cheque.

### Negative

- An Outgoing cheque now advances to Deposited without a human
  double-check that the landlord actually banked it that day — accepted
  as correct given the landlord, not this business, controls that step
  anyway; Clear/Bounce confirmation against the real bank statement is
  still the actual verification point.

## Implementation

- Feature file: `vault/payments-accounting/features/pdc-schedule-generation-and-reconciliation.md`
- Related: `0014-manual-bank-deposit-workflow.md` (Incoming half unchanged)
