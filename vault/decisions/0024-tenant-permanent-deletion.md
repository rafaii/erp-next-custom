# ADR-0024: Permanent Tenant Deletion

- **Status**: accepted
- **Date**: 2026-09-05
- **Deciders**: Imran
- **Supersedes**: extends `0023-tenant-lifecycle-enforcement` (does not relax it — Cancel still means what it meant; Delete is a distinct, later, explicit step)
- **Superseded by**: -

## Context

`0023-tenant-lifecycle-enforcement` deliberately made Cancel non-destructive:
"no deprovisioning, no `bench drop-site`" — the site and its data stay intact
indefinitely. Imran now needs an actual permanent-delete capability from the
console (e.g. to clean up test tenants, or a real business that's truly
leaving and shouldn't keep consuming a site/database indefinitely). This is
a new, genuinely irreversible capability, not a relaxation of ADR-0023 —
Cancel still means what it meant; Delete is a distinct, later, explicit step.

Confirmed with Imran:
- Delete is only enabled once a tenant is already Cancelled (two-step,
  never a direct Live/Suspended → gone jump).
- Confirmation requires typing the exact subdomain, not a plain Yes/No.
- A `bench backup` runs before `bench drop-site`, as a last-resort safety net.
- The `Tenant Site` record is kept afterward, marked `Deleted`, as an audit
  trail — never removed from the console entirely.

## Decision

- **New `platform_console` whitelisted method**: `delete_tenant_site(name,
  confirm_subdomain)`. Guards: `_require_platform_admin()`,
  `subscription_status == "Cancelled"`, and `confirm_subdomain ==
  doc.subdomain` (defense in depth — the console UI also gates on this, but
  the backend never trusts the frontend alone for a destructive action).
- **Execution**: same model as provisioning — sets `status = "Deleting"`
  synchronously, then `frappe.enqueue(queue="long")` (the container with
  `bench`/Traefik-dir access) runs the actual work:
  1. `bench --site <host> backup` (data safety net).
  2. Remove the Traefik route file if it still exists (defensive — it
     should already be gone from Cancel, via `_sync_route_for_subscription_status`).
  3. `bench drop-site <host> --force --db-root-password <...>` (same
     `MARIADB_ROOT_PASSWORD` env var `_create_site` already uses).
  4. On success: `status = "Deleted"`. On failure: `status = "Delete
     Failed"`, full traceback into `error_log` — mirrors the existing
     Provisioning/Failed pattern, never silently reports success.
- **`Tenant Site.status`** gains three new terminal/transitional values:
  `Deleting`, `Deleted`, `Delete Failed` (alongside the existing
  Pending/Provisioning/Live/Failed). The record is never removed from the
  doctype — it's the audit trail of what existed and when it was deleted.
- **Console UI**: a "Delete Permanently" action on the Business Detail page,
  visible only when `subscription_status == "Cancelled"`. Opens a dialog
  requiring the admin to type the tenant's exact subdomain before the
  destructive button enables — same pattern as GitHub's repo-deletion
  confirmation.
- **Known gap, not solved here**: `Tenant Site.subdomain` is `autoname:
  field:subdomain` with `unique=1` — a deleted tenant's record keeps that
  name permanently, so the same subdomain can never be reused for a new
  tenant later without a manual rename of the old (Deleted) record.
  Low-probability edge case (test data today); flagged rather than
  engineered around preemptively.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Allow Delete directly from Live/Suspended, single confirmation | Fewer clicks | One dialog away from destroying a live paying customer's data — no deliberate two-step | rejected |
| B. Require Cancelled first + typed-subdomain confirm + pre-drop backup | Matches the escalation already established by Suspend → Cancel; typed confirmation matches industry practice (GitHub) for irreversible actions; backup gives a real (if manual) undo path | More clicks, and a backup step adds a little time before the drop | **chosen** |
| C. Remove the `Tenant Site` record too once the site is dropped | Cleanest-looking list | Destroys the only record of what existed and when — no audit trail for "why did this business disappear" | rejected |

## Consequences

### Positive

- A genuine, safe path to actually remove test/departed tenants, closing
  the gap ADR-0023 deliberately left open.
- The Cancelled-first requirement means Delete can never be the very first
  action taken against a live business — Suspend/Cancel's own confirm
  dialogs are already a checkpoint before Delete is even reachable.
- A pre-drop backup means "I clicked delete by mistake" has a real recovery
  path (manual restore from the backup file on the VPS), even though the
  console itself doesn't offer one-click restore.

### Negative

- `bench backup` + `bench drop-site` both need `queue="long"` execution and
  real time (seconds to a couple of minutes depending on data size) — the
  console UI must show a "Deleting" in-progress state, not imply
  instant completion.
- Backup files accumulate on the VPS with no automated cleanup in this
  phase — a manual/future concern, not blocking.
- The subdomain-reuse gap above is real, if low-probability today.

## Amendment (2026-09-05): Pending/Failed tenants are also deletable without Cancel

Found live (Imran): `provisioning-test`/`-test2`/`-test3` (all `status:
Failed`) had no Cancel button and therefore no path to Delete at all —
Cancel itself requires `status == "Live"` (`_set_subscription_status`), so
a provisioning attempt that never went Live could never reach `Cancelled`,
the precondition this ADR originally required unconditionally.

Cancel is a subscription-lifecycle concept that only means something for a
business that was actually running; a `Pending`/`Failed`/`Delete Failed`
record was never in service, so it carries none of the risk the
Cancelled-first rule exists to prevent (impulsively destroying a live
paying customer's data). `delete_tenant_site` now also allows deletion
when `status` is `Pending`, `Failed`, or `Delete Failed` (the last so a
failed delete attempt on one of these can be retried), independent of
`subscription_status`. The Cancelled-first requirement is unchanged for
any tenant that actually reached `Live`.

## Implementation

- Feature file: `vault/multi-tenancy/features/tenant-permanent-deletion.md`
