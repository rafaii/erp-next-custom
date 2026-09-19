# ADR-0023: Tenant Suspend/Cancel via Routing-Layer Enforcement

- **Status**: accepted
- **Date**: 2026-09-05
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

The console needs to Suspend and Cancel a tenant business, not just onboard one.
Confirmed with Imran: Suspend must be a soft gate — the tenant's site process and
data stay untouched, access is simply blocked, instantly reversible — and it must
work identically for **any** OS module the platform ever ships
(`real_estate_os` today, others later per `0017-platform-modularization`), not
something specific to one module.

An earlier version of this proposal put a small enforcement app on every tenant
site. **Rejected by Imran**: nothing that "controls the SaaS" should live on
tenant infrastructure at all — only `console.<saas_domain>` gets SaaS-control
capability. `real_estate_os` or any other tenant module must never carry code
whose job is enforcing platform-level lifecycle state, even a passive/minimal
one. This is the same isolation principle `0008-tenant-provisioning-control-plane`
already established for provisioning ("provisioning privilege never touches a
tenant-facing codebase") — it applies equally to de-provisioning/suspension.

`platform_console` already owns exactly the mechanism needed, with zero new
code required on any tenant site: it has host access to
`project_ten/traefik/dynamic/` (the bind-mount ADR-0008 introduced) and already
writes one route file per tenant (`_write_traefik_route` in `provisioning.py`),
which is the sole thing that makes a tenant's hostname reachable at all. Traefik
dispatches purely by Host header to whichever backend a route file names —
toggling *that* is a complete, module-independent access gate that never goes
near any tenant site's application code.

## Decision

Suspend/Reactivate/Cancel operate entirely at the routing layer, entirely within
`platform_console` — no new app, no new repo, nothing installed on any tenant site:

- **Suspend**: remove `project_ten/traefik/dynamic/<slug>.yml` so the hostname
  has no router at all. **Verified live (2026-09-05)**: an unmatched
  `*.nnuggets.com` subdomain returns Traefik's own clean 404 ("404 page not
  found") — there is no catch-all router that would make this a no-op — so
  file removal is a complete, zero-infra suspend. A friendlier "Account
  Suspended" static page is a possible cosmetic follow-up, not a blocker.
- **Reactivate**: rewrite the same file back via the existing
  `_write_traefik_route(slug, host)` template — the exact function
  `run_provisioning` already calls when a site goes live, so "undo a suspend"
  reuses code that's already proven correct.
- **Cancel**: per Imran's decision, marks the tenant Cancelled and applies the
  same suspend-style routing change (site stays fully intact, just unreachable
  via its public hostname) — no deprovisioning, no `bench drop-site`.
- **Tenant Site** gains a `subscription_status` field (Trial/Active/Past
  Due/Suspended/Cancelled) separate from the existing `status` field
  (provisioning lifecycle: Pending/Provisioning/Live/Failed) — `platform_console`
  is still the single source of truth for both, since both already live there.
- No hook, doctype, or app of any kind is added to `real_estate_os` or any
  future OS module. A brand-new OS module gets Suspend/Cancel for free the
  moment it's onboarded through the console, because the mechanism never looks
  at what module is installed — only at the Traefik route for that hostname.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. Small enforcement app installed on every tenant site (`platform_agent`, original proposal) | Enforcement happens "close to" the app, fine-grained (could vary by route/user) | Puts SaaS-control code on tenant infrastructure — exactly what Imran ruled out; every future OS module inherits a dependency it didn't ask for | rejected |
| B. Live cross-site HTTP call on every request | Always current, single source of truth | New request-time dependency + new auth surface between every tenant site and the control plane; contradicts ADR-0008's isolation intent | rejected |
| C. Traefik routing-layer suspend, entirely inside `platform_console` | Zero code/app footprint on any tenant site; module-independent by construction; reuses `_write_traefik_route`, already proven; trivially reversible | Suspension is all-or-nothing at the hostname level (can't selectively allow, e.g., an export-your-data flow, without more work later); requires the console container to still have its Traefik-dir bind mount healthy | **chosen** |

## Consequences

### Positive

- Zero code or app added to `real_estate_os` or any future OS module — Suspend/
  Cancel is a console-only capability, matching Imran's isolation requirement
  exactly.
- New OS modules get lifecycle enforcement automatically, with no per-module
  work.
- Reuses `_write_traefik_route`/the Traefik bind-mount already in production —
  no new infrastructure dependency introduced.

### Negative

- All-or-nothing at the hostname level: a suspended tenant's admin cannot, say,
  export their data before full cancellation without a separate mechanism
  (out of scope for this phase — Cancel already keeps data intact indefinitely
  per Imran's decision, so this isn't urgent).
- The Traefik-dir bind mount only exists in the `queue-long` container, not
  the web/backend container a `@frappe.whitelist()` method runs in
  (confirmed live via `docker inspect`/`docker exec`, 2026-09-05) — the
  actual route write/removal must run as a `queue="long"` job, same as
  provisioning, or it silently touches nothing. `subscription_status`
  therefore updates a moment before the route actually does; a routing
  failure surfaces into `Tenant Site.error_log` after the fact rather than
  blocking the initial request.

## Implementation

- Feature file: `vault/multi-tenancy/features/tenant-lifecycle-enforcement.md`
