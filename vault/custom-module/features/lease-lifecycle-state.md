---
status: done
owner: developer-1
domain: custom-module
created: 2026-08-24
updated: 2026-08-26
related_adr: ["0012-lease-lifecycle-state"]
---

# Lease Lifecycle State

## Summary

Adds a stored `lease_status` field to `Lease Agreement`, decoupled from
`esign_status`, with three distinct terminal states — Expired, Terminated,
Cancelled — rather than one catch-all. Today `esign_status == "Signed"` is
used everywhere as a proxy for "active lease" — including in
`api.get_dashboard_data` — with no transition ever firing, so an expired
lease reads as active forever. See gap G6 in `vault/findings/2026-08-24-
ideal-product-vs-current-state.md` and `vault/decisions/0012-lease-
lifecycle-state.md`.

## Requirements

- `lease_status` Select: `Draft / Pending Signature / Active / Expiring
  Soon / Expired / Renewed / Terminated / Cancelled`.
- `Expired` (ran full term), `Terminated` (tenant breaks lease early —
  e.g. non-payment eviction), and `Cancelled` (voided pre-signature) are
  **three distinct states**, not merged — they imply different
  deposit-forfeiture handling and different churn/turnover reporting.
- E-sign `_finalize` hook sets `lease_status = "Active"` (in addition to
  its existing unit → Occupied transition).
- `cancel_lease` (`real_estate_os/api.py`) sets `lease_status =
  "Cancelled"` explicitly — unchanged behavior, now a named status.
- New whitelisted `terminate_lease` (mirrors `cancel_lease`'s shape but
  only valid from `Active`/`Expiring Soon`) sets `lease_status =
  "Terminated"` for an early exit on a live lease.
- Daily scheduler transitions `Active → Expiring Soon` within 30 days of
  `end_date`, and `Active`/`Expiring Soon → Expired` once `end_date` has
  passed.
- `api.get_dashboard_data` and `payments/invoicing.py`'s due-invoice filter
  switch from `esign_status == "Signed"` to
  `lease_status in ("Active", "Expiring Soon")`.
- Migration patch backfills `lease_status` on existing leases from current
  `esign_status` + `end_date`.

## Design

- New field on `real_estate_os/real_estate/doctype/lease_agreement/
  lease_agreement.json`.
- New `real_estate_os/payments/lease_lifecycle.py`:
  `transition_lease_statuses` (scheduler job), `terminate_lease`
  (whitelisted method, alongside `cancel_lease` in `api.py` or co-located
  here — mirrors `cancel_lease`'s PDC-cancellation/unit-release shape but
  does NOT cancel already-cleared PDCs/invoices, since a Terminated lease
  has real billing history a Cancelled one never had).
- Wired into `hooks.py::scheduler_events.daily`.
- Patch file under `real_estate_os/patches/v0_0/` + `patches.txt` entry
  for the backfill.

## Implementation Plan

- [x] ADR approved — `0012-lease-lifecycle-state`
- [x] `lease_status` field + migration patch backfill
- [x] `_finalize` hook + `finalize_manual_lease` set Active;
      `send_lease_for_signature` sets Pending Signature
- [x] `cancel_lease` sets Cancelled
- [x] `terminate_lease` — new action, Terminated status, wired to a
      "Terminate Lease" Desk button (client script)
- [x] `transition_lease_statuses` scheduler job (Expiring Soon / Expired,
      30-day window), wired into `hooks.py` daily, ordered first
- [x] Updated `get_dashboard_data`, `invoicing.py` (`_active_leases`,
      renamed from `_signed_leases`), and `maintenance_request.py`'s
      active-lease-to-unit resolution to read `lease_status`
- [x] Fixed `demo_data.py`'s seeded leases (would've read as inactive
      with no `lease_status` set) — caught in review before the PR

Code complete and deployed via `agent/developer-1/lease-lifecycle-state`
([PR #3](https://github.com/rafaii/real-estate/pull/3)) — this file's
`status` had drifted stale (still said `planned` after the PR merged and
deployed; caught and corrected 2026-08-26 while working the findings-doc
follow-up, see R3 in the 2026-08-24 findings doc).

## Acceptance Criteria

- [x] Cancelled leases show `Cancelled`, active leases show `Active` — not
      `Signed`/active for both. Confirmed live 2026-08-26 (PR #19 work):
      all 9 real Lease Agreements have `lease_status` populated, 6
      Cancelled / 3 Active, and the Contracts portal page filters/displays
      on this field, not `esign_status`.
- [x] Dashboard/invoicing read `lease_status`, not `esign_status` — code
      confirmed (`api.get_dashboard_data`, `payments/invoicing.py`).
- [ ] `Expired`/`Terminated`/`Renewed` transitions specifically (as opposed
      to `Cancelled`/`Active`, already confirmed above) not yet observed
      against a real lease that's actually reached end-of-term — no live
      lease has hit that point yet.

## Related

- Domain index: `vault/custom-module/custom-module.md`
- ADR: `vault/decisions/0012-lease-lifecycle-state.md`
- Feature: `head-lease-doctype.md` (shares the same status enum shape)
