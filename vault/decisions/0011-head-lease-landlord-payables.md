# ADR-0011: Head Lease & Landlord Payables

- **Status**: accepted
- **Date**: 2026-08-24
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

The product's revenue model is arbitrage: we take on a head lease with a
landlord for a whole building, then sublet units to renters at a markup
(`vault/findings/2026-08-24-ideal-product-vs-current-state.md`, §1). Margin
per building — the only number the business runs on — is the spread between
tenant rent collected and landlord rent owed, minus operating cost.

Today the codebase only models the tenant side. `Landlord` holds contact and
bank fields only — no lease term, no rent amount, no escalation, no end
date. `Building` links a `landlord` field but carries no financial terms.
`grep -rn "Purchase Invoice" --include="*.py"` across the whole app returns
zero hits: there is no code path that ever creates a payable to a landlord.

## Decision

Introduce a **`Head Lease`** DocType (module `real_estate`), **one per
`Building`** (1:1 — a building re-leased after a gap gets a new Head Lease
record, not a mutated old one), linked to a `Landlord`:

- Header fields: `building`, `landlord`, `start_date`, `end_date`,
  `monthly_rent`, `payment_frequency`, `security_deposit_paid`,
  `escalation_percent`, `escalation_frequency`, `head_lease_status`
  (same enum shape as `lease_status`, ADR-0012).
- Child table `Head Lease Schedule` (same shape as the tenant-side `Rent
  Schedule`): `due_date`, `amount`, `status`.

**Landlord payment method is PDC — outgoing cheques we issue to the
landlord — mirroring the existing `Cheque Bank`/`PDC Entry` pattern already
used for cheques received from tenants, extended with a `direction` field
(`Incoming` / `Outgoing`) so both flows share one DocType and one
reconciliation code path instead of forking it.**

This is **configurable per Building**, not hardcoded: `Building` gets a
`landlord_payment_method` Select field (`PDC` / `Bank Transfer`, default
`PDC`), set at Building creation and editable afterward. `Head Lease
Schedule` generation branches on it:
- `PDC` → schedule rows require a linked outgoing `PDC Entry`
  (`Cheque Bank`-backed, `direction = "Outgoing"`) before they can post;
  reconciliation follows the same Deposited → Cleared/Bounced lifecycle
  ADR-0010 already built for incoming PDCs.
- `Bank Transfer` → schedule rows post straight to a `Purchase Invoice`
  against `default_payable_account` on their due date (the
  `generate_due_payables` scheduler job from the draft ADR), no PDC
  involved.

Every `Purchase Invoice` (either path ends in one — outgoing PDCs clear
into a Purchase Invoice + Payment Entry the same way incoming PDCs clear
into a Sales Invoice + Payment Entry today) carries the Building's Cost
Center (ADR/feature `cost-center-per-building.md`), making margin per
building a native GL query.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Landlord-rent fields directly on `Building` | No new DocType | Conflates a building's physical attributes with a time-bound financial contract; no history across renewals | Rejected |
| `Head Lease` DocType, symmetric to `Lease Agreement` | Matches an understood pattern; preserves renewal history; plugs into the cost-center model | New DocType + schedule + scheduler to build | **Chosen** |
| Payment Entry only, no outgoing PDC | Simpler — no new `direction` field, no outgoing reconciliation | Doesn't match how landlord rent is actually paid in this business (PDCs), and forecloses the per-building configurability Imran asked for | Rejected |
| Separate `Landlord PDC Entry` DocType instead of extending `PDC Entry` | No risk of the two flows interfering | Duplicates the entire Deposited/Cleared/Bounced lifecycle and its reconciliation code instead of reusing ADR-0010's work | Rejected |

## Consequences

### Positive

- Margin per building becomes computable: Sales Invoice (tenant) minus
  Purchase Invoice (landlord) on the same Cost Center is a native GL query.
- Landlord obligations become visible and auditable.
- Outgoing-PDC reconciliation reuses ADR-0010's Deposited/Cleared/Bounced
  machinery instead of duplicating it — one `PDC Entry` DocType, one
  `direction` field.
- Per-building configurability means a building whose landlord actually
  wants bank transfer (not everyone takes post-dated cheques) doesn't
  force a workaround.

### Negative

- Extending `PDC Entry` with `direction` touches shared code
  (`payments/pdc.py`) that the live tenant-PDC flow depends on — needs
  care that `direction = "Incoming"` (the default, backfilled onto every
  existing `PDC Entry` row) doesn't change behavior for anything already
  in production.
- Real implementation effort: new DocType, child table, payables module,
  scheduler job, `direction`-aware reconciliation, cost-center backfill.
- No historical head-lease data exists for the live Rustic building —
  needs a real data-entry pass, not just a migration script.

## Implementation

- Feature files: `vault/custom-module/features/head-lease-doctype.md`,
  `vault/payments-accounting/features/landlord-payables.md`

## Amendment (2026-08-25)

Discovered mid-implementation and confirmed with Imran before writing any
code: ERPNext `Purchase Invoice` requires a `supplier`, and `Landlord` (a
custom DocType — `landlord_name/type/company/address/email/phone/
bank_name/account_number/iban`) has no accounting identity at all. This
wasn't caught drafting the ADR because the tenant side never needed the
analogous decision — it posts against ERPNext's own `Customer` directly,
with no parallel "Tenant" record.

**Decided**: `Landlord` gets a `supplier` field, auto-created on insert —
same pattern already used for the "Rent" Item (`invoicing.py::
_get_rent_item`) and the per-Building Cost Center. No manual setup step
for staff creating a Landlord. The "Rent" Item also becomes
`is_purchase_item: 1` in addition to its existing `is_sales_item: 1` (one
Item, both directions, rather than a second item for the expense side).
Landlord-rent expense posts to `Company.default_expense_account`
(the payables-side analogue of `default_income_account`, which the
tenant-side Sales Invoice line already uses).

**Also confirmed**: `Head Lease` gets **no Tenant permission row** at
all — unlike `Lease Agreement`, which has one for renter self-service
(ADR-0009). A renter has no legitimate reason to see what's owed to a
landlord, and it's the direct margin-leak vector the cost-center work
exists to make computable only for staff.
