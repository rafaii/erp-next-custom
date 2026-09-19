---
status: in-progress
owner: developer-1
domain: multi-tenancy
created: 2026-09-05
updated: 2026-09-05
related_adr: ["0023-tenant-lifecycle-enforcement"]
---

# Tenant Lifecycle Enforcement (Suspend/Cancel)

## Summary

Adds Suspend/Reactivate/Cancel to `platform_console`, enforced entirely at the
Traefik routing layer that `platform_console` already owns — no code or app is
added to `real_estate_os` or any future OS module, so every module gets this
for free.

## Requirements

- Suspend: blocks public access to the tenant's hostname; site process and
  data are untouched; instantly reversible via Reactivate.
- Cancel: marks the tenant Cancelled and applies the same access block; no
  deprovisioning, no data deletion, ever (per Imran's decision).
- Must work identically regardless of which OS module (`real_estate_os` or a
  future module) is installed on the tenant site.
- Zero new network/auth surface between tenant sites and the control plane.

## Design

- `Tenant Site` gains `subscription_status` (Select: Trial/Active/Past
  Due/Suspended/Cancelled) and `subscription_changed_on`, kept separate from
  the existing `status` field (provisioning lifecycle:
  Pending/Provisioning/Live/Failed) — two independent state machines on the
  same doc.
- New whitelisted methods in `platform_console/provisioning.py`, guarded by
  the existing `_require_platform_admin()` pattern: `suspend_tenant_site(name)`,
  `reactivate_tenant_site(name)`, `cancel_tenant_site(name)` (all funnel
  through `_set_subscription_status`), and `set_plan(name, plan)` (catalog
  reference only, no routing side-effect).
- **Route toggling** (`provisioning.py`), single source of truth:
  - `_sync_route_for_subscription_status(doc)` — the only function that ever
    writes/removes a Live tenant's route file. Trial/Active/Past Due → route
    present (`_write_traefik_route`, atomic temp-file + `os.replace` write);
    Suspended/Cancelled → route file removed. **Verified live (2026-09-05)**:
    an unmatched `*.nnuggets.com` host gets Traefik's own clean 404 (no
    catch-all router), so removal is a complete, zero-infra suspend — no
    static "Account Suspended" responder was needed.
  - **Container placement matters and was verified live**: `TRAEFIK_DYNAMIC_DIR`
    is bind-mounted into the `queue-long` container only (ADR-0008) —
    `docker inspect`/`docker exec` on `realestate-backend-1` (2026-09-05)
    confirmed no mount and no such path there. A `@frappe.whitelist()`
    method runs in the web/backend container, so it can never touch the real
    directory directly. `_require_traefik_dir()` asserts the directory
    exists (raises rather than the earlier, wrong `os.makedirs(...,
    exist_ok=True)`, which silently created a throwaway local dir in
    whichever container called it and made suspend a no-op that still
    reported success). `_set_subscription_status(name, status)` therefore
    updates `subscription_status` synchronously (no filesystem access needed)
    and enqueues `_apply_subscription_status` onto `queue="long"` to do the
    actual route write/removal — the same execution model
    `provision_tenant_site`/`run_provisioning` already use for the same
    reason.
  - On a routing failure inside that queued job, `_apply_subscription_status`
    reloads the doc and writes the failure into `error_log` (mirroring
    `run_provisioning`) — `subscription_status` can therefore be briefly
    "ahead of" the real route state until the job runs; the console UI
    refetches once immediately and once ~2.5s later to surface that gap
    without a manual refresh.
  - Guard: all three actions `frappe.throw` unless `status == "Live"` — a
    route file only exists once provisioning succeeded, so acting on a
    Pending/Provisioning/Failed tenant would be a no-op at best.
  - `set_plan(name, plan)` never touches routing directly, but **does**
    convert Trial → Active when a plan is assigned to a Live, still-Trial
    tenant — the natural "this is now a paying subscription" signal (raised
    by Imran testing live: picking a plan for a Trial business should not
    leave it stuck in Trial). No route change is needed for that
    transition (Trial and Active are both in `_ROUTED_SUBSCRIPTION_STATUSES`).
    Deliberately does **not** do the same for Suspended/Cancelled → Active —
    reactivating from either of those stays behind the explicit
    `reactivate_tenant_site` confirm dialog, since it also has to restore
    public access, not just flip a status field.
- No hook, doctype, or dependency is added to `real_estate_os` or any other
  module app. Nothing about this feature touches `MODULE_APP_MAP`.
- **Known gap**: custom-domain tenants (ADR-0008's manual-checklist path)
  aren't necessarily named `<slug>.yml` — none exist today, so this isn't
  built for; flagged here rather than guessed at.

## Implementation Plan

- [x] Add `subscription_plan`, `subscription_status`, `subscription_changed_on`
      fields to `Tenant Site` (`.json`, `before_insert` default `Trial`).
- [x] New `Platform Plan` doctype (internal catalog only, no gateway fields).
- [x] Verify live: unmatched `*.nnuggets.com` host → clean Traefik 404, no
      catch-all router — confirms route-file removal is a complete suspend.
- [x] `_sync_route_for_subscription_status`, `_set_subscription_status`,
      `suspend_tenant_site`, `reactivate_tenant_site`, `cancel_tenant_site`,
      `set_plan` in `provisioning.py`; `_write_traefik_route` made atomic
      (temp file + `os.replace`).
- [x] Fixed after live verification: the route write/removal now runs via
      `frappe.enqueue(queue="long")`, not inline in the whitelisted method —
      the Traefik dir is only mounted in `queue-long`, and the first version
      of this code would have silently no-op'd on suspend.
- [ ] Wire the Businesses detail page's subscription panel (from
      `console-admin-ui.md`) to these methods, with confirm dialogs.
- [ ] Retrofit: none needed — mechanism only touches the route file, which
      already exists for the one live tenant (`realestate.nnuggets.com`); no
      action required there until it's actually suspended.
- [ ] Manual end-to-end test: provision a throwaway tenant through the console
      itself (never test against a real customer site), then Suspend →
      confirm the hostname 404s → Reactivate → confirm access restored →
      Cancel → confirm status changes and the same block applies.

## Acceptance Criteria

- [ ] Suspending a tenant blocks its public hostname (Traefik 404 — no
      friendlier page in this phase); the site's container/database is
      untouched (verified: `bench --site <host> console` still works directly).
- [ ] Reactivating restores the original Traefik route and access within
      seconds, no manual intervention needed.
- [ ] Cancelling a tenant applies the same block and leaves data fully intact
      indefinitely (no scheduled deletion of any kind).
- [ ] `real_estate_os` (and any future OS module) requires zero code changes
      for Suspend/Cancel to work — verified by grepping `real_estate_os` for
      any new dependency on this feature (there should be none).
- [ ] A failed route write surfaces clearly in `Tenant Site.error_log` rather
      than reporting success.

## Related

- Domain index: `vault/multi-tenancy/multi-tenancy.md`
- ADR: `vault/decisions/0023-tenant-lifecycle-enforcement.md`
- Companion feature: `vault/multi-tenancy/features/console-admin-ui.md`
- Existing mechanism reused: `_write_traefik_route` in
  `apps/platform_console/platform_console/provisioning.py`
