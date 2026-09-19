# ADR-0008: Tenant Site Provisioning — Control-Plane UI

- **Status**: accepted
- **Date**: 2026-08-23
- **Deciders**: Imran
- **Supersedes**: refines the onboarding mechanism in `0003-multi-tenancy` (which specified a `bench new-site` CLI script)
- **Superseded by**: -

## Context

`0003-multi-tenancy` already decided tenant isolation: **one Frappe site (own DB)
per real-estate company**, single `real_estate_os` codebase deployed across all
sites. That ADR's implementation was scoped as a CLI onboarding script.

We now have a second real client to onboard (American Real Estate, alongside the
live Rustic tenant on `realestate.nnuggets.com`), and the ask has changed: a
**simple UI-based way** to create a new tenant site, not an SSH/CLI runbook.

Confirmed infra facts (checked live on `vps-claude` before writing this):

- Deployment is `frappe_docker`, one bench, one MariaDB, one shared
  `realestate-frontend` container. Frappe's own frontend already resolves the
  correct site **by Host header** across every site in `sites/` — adding a site
  needs no new container and no per-site nginx/app config.
- Public routing is Traefik (a separate `project_ten` stack), with one **dynamic
  route file per hostname** (`project_ten/traefik/dynamic/<slug>.yml`) pointing
  at the same shared `realestate-frontend:8080` service.
- `*.nnuggets.com` is already wildcard-DNS'd to the VPS, and Traefik already has
  a `wildcard-resolver` cert resolver in play. So any `<slug>.nnuggets.com`
  tenant needs **zero new DNS or TLS work** — only a new Frappe site + a new
  Traefik route file. A tenant on a fully custom domain (their own domain, not a
  nnuggets.com subdomain) still needs manual DNS + its own cert route — out of
  scope for v1, called out below.
- Creating the site itself (`bench new-site`, `install-app`) is a bench
  operation runnable from inside the `realestate-backend-1` container — it does
  **not** need Docker/host access. Writing the Traefik route file **does** need
  host filesystem access, which the tenant-facing app containers don't have.

This last point is the crux of the decision: the actual "create a new tenant"
operation spans two different privilege domains (in-container bench operations,
and host-level Traefik/DNS), and touches infrastructure that must never be
reachable from a *tenant's own* site — doing so would undercut the isolation
guarantee `0003-multi-tenancy` was written to establish (no tenant's app should
be able to affect another tenant's data or provisioning).

## Decision

**A separate, non-tenant-facing Platform Control Plane** — its own dedicated
Frappe site on the same bench (e.g. `console.nnuggets.com`), running a small
internal app — is the only thing capable of creating/modifying tenant sites.
`real_estate_os` itself (the app every tenant runs) never carries this
capability.

Mechanics (v1 scope):

- New minimal app (e.g. `platform_console`), installed only on the control-plane
  site, gated to a `Platform Admin` role (just you, initially).
- One DocType, `Tenant Site` (tenant name, subdomain slug, admin email, status:
  Pending/Provisioning/Live/Failed, created/provisioned timestamps) — this *is*
  the UI: a Frappe list/form, no custom frontend needed for v1.
- A whitelisted method, triggered from the form, that:
  1. Runs `bench new-site <slug>.nnuggets.com --install-app real_estate_os`
     (subprocess, argument list — not a shell string — with the slug validated
     against a strict `^[a-z0-9-]+$` pattern before use).
  2. Seeds default fixtures: `Signature Settings`, default roles, empty
     Portal Field Visibility.
  3. Writes `project_ten/traefik/dynamic/<slug>.yml` from a template (needs that
     directory bind-mounted into the control-plane container — the *only*
     container that gets this mount).
  4. Polls/verifies the new site responds, flips `Tenant Site.status` to Live.
- Custom-domain tenants (not a nnuggets.com subdomain): v1 handles steps 1-2
  automatically and surfaces a manual checklist (DNS record + Traefik cert
  route to add) rather than automating it — low volume, safer to keep a human
  in the loop until there's a real DNS-provider API integration.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A. CLI script only (original `0003` plan) | Simplest to build | Not what's being asked for now — needs SSH access per onboarding | superseded |
| B. Provisioning API/UI built into `real_estate_os` itself | Reuses the existing portal shell, no new site | Every tenant site would carry code capable of creating/deleting *other* tenants' databases, gated only by an app-level role check — one RBAC bug or app compromise on any single tenant site becomes a platform-wide breach. Directly contradicts the isolation goal `0003` was written for | rejected |
| C. Separate Platform Control Plane site/app | Keeps provisioning privilege fully outside any tenant's blast radius; matches how Frappe Cloud itself is architected; tenant sites stay exactly as isolated as `0003` intended | One more site to run (small: one DocType + one method) | **chosen** |

## Consequences

### Positive

- New tenant = fill a form on the control-plane site, not an SSH session.
- Tenant isolation guarantee from `0003` stays intact — provisioning privilege
  never touches a tenant-facing codebase.
- Zero new DNS/TLS work for `*.nnuggets.com` tenants (wildcard already covers it).

### Negative

- One more Frappe site to maintain (`console.nnuggets.com`), access to it must
  be tightly restricted (it can create databases and write host files).
- Custom-domain tenants still need a manual DNS/cert step in v1.
- The control-plane container needs a host bind-mount into
  `project_ten/traefik/dynamic/` — a new cross-stack dependency to document.

## Implementation

- Feature file: `vault/multi-tenancy/features/tenant-provisioning-console.md` (to be created on approval)
- Not started — awaiting approval before an implementation plan is written.
