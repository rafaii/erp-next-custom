---
status: planned
owner: developer-1
domain: multi-tenancy
created: 2026-09-05
updated: 2026-09-05
related_adr: ["0024-tenant-permanent-deletion"]
---

# Permanent Tenant Deletion

## Summary

Adds a genuine, irreversible delete capability to the console for a
Cancelled tenant: backs up, drops the site, and marks the `Tenant Site`
record `Deleted` (kept as an audit trail, never removed). A distinct,
later, more consequential step than Cancel (ADR-0023), which stays
non-destructive.

## Requirements

- Delete reachable once `subscription_status == "Cancelled"` — **or**,
  amended 2026-09-05, when the tenant never successfully went Live at all
  (`status` is `Pending`, `Failed`, or `Delete Failed`), since Cancel
  itself requires `status == "Live"` and is otherwise unreachable for a
  broken/never-started provisioning attempt.
- Console requires typing the exact subdomain before the destructive action
  enables.
- A `bench backup` runs before `bench drop-site`.
- The `Tenant Site` record persists afterward with `status = "Deleted"`.
- Backend independently re-validates both the Cancelled precondition and the
  typed confirmation — never trusts the frontend gate alone.

## Design

- `Tenant Site.status` gains `Deleting`, `Deleted`, `Delete Failed`
  (alongside Pending/Provisioning/Live/Failed).
- `platform_console/provisioning.py`:
  - `delete_tenant_site(name, confirm_subdomain)` (whitelisted) — guards,
    then sets `status = "Deleting"` synchronously and enqueues
    `_run_delete_tenant_site` on `queue="long"` (same container as
    provisioning, for `bench`/Traefik-dir access).
  - `_run_delete_tenant_site(tenant_site_name)` — `bench --site <host>
    backup`, remove the Traefik route file if still present (defensive),
    `bench drop-site <host> --force --db-root-password <MARIADB_ROOT_PASSWORD>`.
    On success: `status = "Deleted"`. On failure: `status = "Delete Failed"`,
    full traceback into `error_log` (mirrors `run_provisioning`'s pattern).
- Console UI (`Businesses/Detail.tsx`): "Delete Permanently" action, visible
  only when `subscription_status === "Cancelled"`; a dialog requiring the
  admin to type the subdomain before the button enables.
- **Businesses list default filter** (raised by Imran while reviewing):
  Deleted tenants must not clutter the default view — `Businesses/List.tsx`
  excludes `status === "Deleted"` unless the admin explicitly filters for it
  via the existing provisioning-status dropdown.

## Implementation Plan

- [x] Add `Deleting`/`Deleted`/`Delete Failed` to `Tenant Site.status` Select options.
- [x] `delete_tenant_site` + `run_delete_tenant_site` + `_backup_site`/`_drop_site`
      in `provisioning.py` (`bench backup --with-files` then `bench drop-site
      --force --no-backup --db-root-password`).
- [x] `Businesses/Detail.tsx` — Delete Permanently action + typed-confirmation
      dialog, plus a polling refetch every 4s while `status === "Deleting"`
      (backup+drop can take longer than the lighter suspend/cancel actions).
- [x] `Businesses/List.tsx` and `Dashboard.tsx` — default-exclude
      `status === "Deleted"`; still selectable via the provisioning-status filter.
- [x] `StatusBadge`/`PROVISIONING_TONE` — add the three new status values.
- [x] Amendment verified live: `provisioning-test`/`-test2`/`-test3` (all
      `status: Failed`, never had a Cancel button) deleted cleanly through
      the real `delete_tenant_site` entry point after the precondition fix
      — all three had no real site directory, resolved straight to
      `Deleted` with no errors.
- [x] Manual end-to-end test (Imran, live): deleting `provisioning-test5`
      failed with `bench backup`'s generic "Database or site_config.json
      may be corrupted" error. Root cause: that `Tenant Site` record said
      `status: Live` but had no site directory (and no Traefik route) on
      disk at all — a record left over from manual cleanup done before
      this Delete feature existed, genuinely out of sync with reality, not
      real data corruption. `run_delete_tenant_site` now checks
      `_site_dir_exists(host)` first; if the directory genuinely doesn't
      exist, it skips backup/drop-site entirely (nothing to protect) and
      finalizes straight to `Deleted`, logging an informational note via
      `frappe.log_error` rather than treating it as a failure. A real
      backup/drop-site failure on an existing site still surfaces normally
      as `Delete Failed` with the full traceback.

## Acceptance Criteria

- [ ] Delete is not reachable/enabled for a Live or Suspended tenant.
- [ ] Typing the wrong subdomain keeps the destructive button disabled.
- [ ] A real backup file exists on the VPS before the site is dropped.
- [ ] After deletion, the tenant's hostname is fully unreachable (site gone,
      not just routed away) and the `Tenant Site` record shows `Deleted`.
- [ ] The Businesses list hides `Deleted` tenants by default; filtering for
      "Deleted" (provisioning status) reveals them.
- [ ] A failed backup or drop-site surfaces in `Tenant Site.error_log` as
      `Delete Failed`, never silently reports success.

## Related

- Domain index: `vault/multi-tenancy/multi-tenancy.md`
- ADR: `vault/decisions/0024-tenant-permanent-deletion.md`
- Extends: `vault/multi-tenancy/features/tenant-lifecycle-enforcement.md`
- Companion: `vault/multi-tenancy/features/console-admin-ui.md`
