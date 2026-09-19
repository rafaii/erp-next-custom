# ADR-0012: Lease Lifecycle State

- **Status**: accepted
- **Date**: 2026-08-24
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

`Lease Agreement` has no `status` field. Every consumer that needs to know
whether a lease is "active" — `api.get_dashboard_data` (with an explicit
code comment acknowledging the gap) and `payments/invoicing.py`'s scheduler
filters — reuses `esign_status`, a field describing the *signature
workflow* (Draft → Sent → Signed), not the *business lifecycle* of the
tenancy. A lease whose `end_date` has long passed still reads
`esign_status == "Signed"` forever; nothing ever transitions it.

## Decision

Add a stored `lease_status` Select field to `Lease Agreement`, decoupled
from `esign_status`, with **three distinct terminal states** rather than
one catch-all "inactive":

`Draft → Pending Signature → Active → Expiring Soon → Expired`, plus two
side branches: `Renewed` and `Terminated` (tenant breaks the lease early,
e.g. non-payment eviction) — kept separate from `Cancelled` (voided before
ever going live, the existing `cancel_lease` behavior). The three-way split
is deliberate: deposit-forfeiture rules and churn-vs-turnover reporting
differ across all three, so collapsing them would just recreate the
`esign_status`-as-proxy problem one level up.

- The e-sign `_finalize` hook sets `lease_status = "Active"`.
- `cancel_lease` sets `lease_status = "Cancelled"` explicitly.
- A new "Terminate Lease" action (client script + whitelisted method,
  mirroring `cancel_lease`'s shape) sets `lease_status = "Terminated"` for
  an early exit on an already-active lease — distinct code path from
  `cancel_lease`, which only ever applies pre-signature.
- Daily scheduler job (`lease_lifecycle.py::transition_lease_statuses`)
  moves `Active → Expiring Soon` inside 30 days of `end_date`, and
  `Active`/`Expiring Soon → Expired` once `end_date` has passed.
- `api.get_dashboard_data` and `invoicing.py`'s due-invoice filter switch
  from `esign_status == "Signed"` to
  `lease_status in ("Active", "Expiring Soon")`.
- Migration patch backfills `lease_status` on existing leases from current
  `esign_status` + `end_date` at migration time.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Computed status (not stored) | No backfill; always consistent | Can't be filtered/sorted in desk list views or reports | Rejected |
| Stored `lease_status`, single "Inactive" catch-all | Simpler enum | Loses the Terminated/Cancelled/Expired distinction Imran explicitly wants — same information loss as the current `esign_status` proxy, one level up | Rejected |
| Stored `lease_status`, three distinct terminal states + scheduler | Filterable/reportable; matches `Unit.status`'s pattern; deposit and churn-reporting logic can key off the right state | Needs a backfill patch and a new scheduler job | **Chosen** |
| Frappe native `Workflow` DocType | Built-in approval UI/audit trail | Overkill — these transitions are time/system-driven, not human-approval steps | Rejected |

## Consequences

### Positive

- Dashboard "active leases" / occupancy numbers stay correct after a
  lease's term ends, instead of permanently over-counting.
- Unblocks a real renewal workflow and an "expiring in N days" report.
- Terminated vs. Cancelled vs. Expired gives deposit handling and
  churn/turnover reporting the distinction they actually need.

### Negative

- Backfill patch required across existing leases at deploy time.
- New daily scheduler job to monitor alongside `generate_due_invoices`.
- Every current reader of the `esign_status == "Signed"` proxy must move
  to `lease_status` in the same change, or the system is split-brain.

## Implementation

- Feature file: `vault/custom-module/features/lease-lifecycle-state.md`
