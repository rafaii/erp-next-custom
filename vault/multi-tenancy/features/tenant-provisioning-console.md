---
status: in-progress
owner: developer-1
domain: multi-tenancy
created: 2026-08-23
updated: 2026-09-12
related_adr: ["0008-tenant-provisioning-control-plane", "0016-module-aware-provisioning"]
---

# Tenant Provisioning Console

## Summary

A dedicated, non-tenant-facing Platform Control Plane site + small internal app
that provisions new tenant sites (e.g. onboarding American Real Estate
alongside the live Rustic tenant) via a form instead of an SSH runbook. See
ADR-0008 for why this lives outside `real_estate_os` itself.

Supersedes the CLI-only plan in `tenant-onboarding.md` (kept for the manual
steps this automates).

**2026-08-27 (ADR-0016)**: expanded to be module-aware — the original design
hardcoded `real_estate_os` as *the* app to install. A future OS
(Automobile Workshop, etc.) means the provisioning form must let the
platform admin pick which module to install, and each module must bring
its own default-accounts setup — never a shared/global mapping (see
`PLATFORM_STRATEGY.md` §3, §4).

**2026-09-01: built, v1 scope (new-customer onboarding only)**. Imran's
own framing when asking for this: "console.nnuggets.com... is going to be
the main backend of our SaaS that sees all businesses using our product,
their subscription etc. For now, we are only building the onboarding of
new customers." Subscriptions/billing are explicitly a later, separate
build — not started, not designed.

One refinement beyond the original design, raised by Imran mid-review:
the CoA choice isn't a single per-module template, it's a **per-country**
choice on the form itself ("There may be customers from Canada, US, UK,
India, Saudi etc. So not everyone uses the same CoA") — see the CoA
section below and the corresponding update to `0016-module-aware-provisioning.md`.

**2026-09-01, second refinement**: after using the built form, Imran asked
for E-Sign Mode (a single Select) to become a proper **Integrations**
section instead — "there should be a section for integrations and I can
add all integrations which at the moment is Inbuilt Esign. As we add more
integration during the setup/onboarding screen, i get an option to choose
what are enabled for this customer." Clarified that this needs to support
**multiple integrations enabled at once per tenant** (not one default
picked from a dropdown) — see Design below.

**2026-09-01, third refinement**: two more gaps found by Imran actually
reading the built form. (1) "During onboarding a new client, what is the
difference between Tenant Name and Company Name?" — there wasn't one:
`tenant_name` was never referenced anywhere in `provisioning.py`, purely
a redundant console-side label. Removed; `Company Name` (the field that
actually becomes the tenant's real ERPNext Company) now carries the
list-view column instead. (2) "The first email created will be the super
admin who can then create multiple other users in their account - right?"
— also not true as built: `admin_email` was likewise never referenced
anywhere; `bench new-site` only sets a random, undisclosed Administrator
password. Fixed by having `finalize_new_tenant` create a real `System
Manager` User for `admin_email` and hand back a password-reset link,
which `platform_console` emails to them — branded as **Aetris** (the
existing Real Estate portal brand/logo — confirmed with Imran, not
guessed), sent via Brevo's transactional API (Imran provided an API key,
not SMTP credentials) rather than Frappe's own built-in welcome email,
which mentions Frappe/ERPNext by name. See Design below for exactly how
the branding/sending responsibility is split between `real_estate_os`
(owns the User/reset-link) and `platform_console` (owns sending — a
platform onboarding action with the SaaS's own brand, not this module's
or the tenant's).

## Requirements

- New Frappe site (e.g. `console.nnuggets.com`), not installed with any
  tenant-facing OS app, reachable only by a `Platform Admin` role.
- One DocType, `Tenant Site`: tenant name, subdomain slug (validated
  `^[a-z0-9-]+$`), admin email, **`module`** (Select — "Real Estate" today,
  more options as other OS apps are built, mapping to an installable app
  name), status (Pending/Provisioning/Live/Failed), timestamps.
- Whitelisted method triggered from the form:
  1. `bench new-site <slug>.nnuggets.com --install-app <app-for-selected-module>`
     (subprocess arg list, not a shell string; app name resolved from the
     `module` field, not hardcoded).
  2. Install that module's **required modules** automatically (ADR-0017 —
     e.g. Real Estate OS requires an e-sign provider; defaults to in-built
     if the tenant hasn't chosen otherwise yet).
  3. Seed default fixtures (Signature Settings, default roles incl. the
     `Tenant` role from ADR-0009, empty Portal Field Visibility, and that
     OS's own Chart-of-Accounts naming — ADR-0016 §2).
  4. Write `project_ten/traefik/dynamic/<slug>.yml` from a template — requires
     that directory bind-mounted into the control-plane container only.
  5. Poll the new site, flip status to Live.
- `*.nnuggets.com` tenants: zero manual DNS/TLS steps (already wildcarded).
- Custom-domain tenants (v1): steps 1-3 automated, DNS/cert route surfaced as a
  manual checklist rather than automated.

## Design

- New minimal app `platform_console` ([rafaii/platform-console](https://github.com/rafaii/platform-console),
  private), installed only on the control-plane site.
- Reuses the same VPS/bench/MariaDB already running `realestate.nnuggets.com` —
  no new infrastructure, just a new site (`console.nnuggets.com`) in the
  existing `sites/` directory, and one more Traefik dynamic-route file
  pointing at the same shared `realestate-frontend:8080` (confirmed live:
  that container already sits on `shared_network`, the network the
  separate `project_ten` Traefik stack uses).
- `MODULE_APP_MAP` in `platform_console/provisioning.py`: module label ->
  `{app, finalize}` — `app` is the installable app name, `finalize` is the
  dotted path to *that module's own* `finalize_new_tenant(...)` function.
  **Not** a shared CoA-fixture lookup living in the control-plane app as
  originally planned — see the next bullet for why.
- **Each module owns its own tenant-bootstrap function** (ADR-0016: "each
  vertical app owns its own Chart-of-Accounts setup"), e.g.
  `real_estate_os.provisioning.finalize_new_tenant`. This isn't just a
  style preference: `bench --site X execute <dotted.path>` resolves
  through `frappe.get_attr`, which checks the *target* site's
  installed-app list — a function living in `platform_console` (never
  installed on a tenant site, by design) could never be reached this way.
  Found live, the hard way, on the first provisioning attempt.
- `platform_console` invokes that function via `bench --site X console`
  fed a one-line `exec(open(path).read())` on stdin, **not** `bench
  execute` — this bench version's `execute` command swallows the real
  exception from a failing target function and reports an unrelated,
  misleading `NameError` instead (confirmed live, twice, with two
  genuinely different underlying errors). `console` runs the code with no
  such wrapper, so `Tenant Site.error_log` shows the real traceback.
- CoA template choice is per-country, not per-module: `get_coa_options_for_country`
  wraps ERPNext's own `get_charts_for_country` (the same function
  Company's Desk form uses) so the Tenant Site form's dropdown shows real,
  ERPNext-shipped templates for whichever country the admin picks —
  "Standard"/"Standard with Numbers" for most, a dedicated template for
  a handful (India, UAE, ...). See `0016-module-aware-provisioning.md`'s
  2026-09-01 update.
- **Default Currency needs enabling per country, same family of gap as
  CoA**: Frappe ships every ISO currency as a `Currency` record but only
  *enables* nine by default (INR, USD, GBP, EUR, AED, AUD, JPY, CNY, CHF),
  and `frappe.desk.search` auto-filters any Link-field search to
  `enabled=1` on a doctype with an "enabled" field — so a disabled
  currency (QAR, SAR, CAD, ...) can't even be typed into the field, not
  just "not defaulted." Found live onboarding a Qatar tenant.
  `get_currency_for_country` reads `frappe.geo.country_info` (the same
  data Setup Wizard itself uses) and force-enables the match; the Tenant
  Site form's `country` handler calls it and defaults the field without
  clobbering a manual override.
- `finalize_new_tenant` creates the Company (triggering ERPNext's own
  `create_charts()`) and a Fiscal Year covering today — a headless
  `Company.insert()` skips both what ERPNext's Setup Wizard normally does,
  including seeding `Warehouse Type: Transit` (referenced by
  `Company.on_update()`'s default-warehouse creation) — found live via the
  masked-then-unmasked error above; now seeded explicitly first.
- `finalize_new_tenant` also sets Aetris branding on `Website Settings`
  (`_set_aetris_branding()`) before anything else — every new tenant's
  `/login`, `/update-password`, `/forgot-password` are otherwise stock
  ERPNext until this runs, since the React portal only covers pages
  reached after a session exists. Found via the invite email's "Set
  Password" link landing on an unbranded page — see
  `ui-portal/features/admin-portal-ui.md`'s 2026-09-01 entry for the
  full root cause and fix.
- **Every tenant site's outgoing mail (password resets, notifications)
  is also Aetris-managed-by-default / own-SMTP-if-configured** — the
  same override_email_send mechanism as the invite email, but per-tenant
  rather than platform-only. Requires two bench-wide (not per-tenant)
  prerequisites, set once directly on the VPS: `BREVO_API_KEY` in
  `backend`/`queue-short`/`scheduler`/`queue-long`'s environment (mail
  can send synchronously from a web request or via a background job, so
  every container that might send needs it), and placeholder
  `mail_server`/`mail_login`/`mail_password` keys in the bench's shared
  `common_site_config.json` — these values are never used for a real
  connection (the override hook pre-empts that), they exist purely so
  `EmailAccount.find_from_config()` returns a truthy account and Frappe's
  "an Email Account must exist" check passes for every site in this
  bench automatically. See `ui-portal/features/admin-portal-ui.md`'s
  2026-09-02 entry for the full per-tenant dispatch logic.
- `bench new-site` is called with `--mariadb-user-host-login-scope "%"` —
  without it, the new site's DB user is scoped to the backend container's
  IP *at that moment*, and breaks the moment that container is next
  recreated (routine — every deploy of any app in this bench does this).
  Found live: the first two tenant sites this code created broke exactly
  this way; `realestate.nnuggets.com`'s own DB user was already `%`-scoped,
  which is why it never showed this symptom.
- The Traefik-dynamic-dir bind mount lives on `queue-long` specifically
  (not `backend`) — `frappe.enqueue(queue="long")` is where the actual
  provisioning work runs (`bench new-site` + install-app can exceed a
  normal request timeout), and that's a separate container from the one
  serving HTTP. Also needed `group_add: ["1001"]` on that service plus a
  `chmod g+w` on the host directory — the directory is owned by the host's
  `imranrafai` user (uid 1001), not the container's `frappe` (uid 1000),
  so read-only access worked but writes didn't until this was added.
- **Integrations are a multi-enable table, not a single default Select**
  (2026-09-01 refinement). New child doctype `Tenant Site Integration`
  (`category`, `provider`, `enabled`) replaces the original `esign_mode`
  field. `MODULE_APP_MAP` gained an `integrations` list per module —
  category/provider options and each category's default — declared
  statically in `platform_console` rather than queried live from the
  target module app: there's no tenant site to query yet at onboarding
  time, and `frappe.get_hooks()` (what `real_estate_os` itself uses to
  know which providers are actually registered) is a site-bound Frappe
  runtime call this app's own process can't safely make against a site
  it isn't running on — it would read/write the wrong site's context.
  Picking **Module** auto-populates one row per category (its default,
  enabled) into an empty table only, so it never clobbers rows already
  edited; a second row for the same category can be added by hand to
  enable a second provider. Where the underlying dispatch only supports
  one active provider per category today (e-sign), the **first enabled
  row** for that category becomes `finalize_new_tenant`'s `esign_mode` —
  the other enabled rows for that category are still recorded as part of
  this tenant's integration entitlement list, just not (yet) wired to
  more than one simultaneously-active dispatch target. Verified live:
  enabling both DocuSign and In-Built for E-Sign, DocuSign listed first,
  correctly resulted in `Signature Settings.esign_mode == "DocuSign"` on
  the provisioned site, with both rows recorded as enabled on the
  Tenant Site doc.
- **Admin invite, split across the two apps by responsibility, not by
  convenience**. `real_estate_os.provisioning.finalize_new_tenant` now
  also creates a `System Manager` User for `admin_email` (idempotent —
  re-running just issues a fresh reset link) and generates a reset link
  via Frappe's own `User._reset_password(send_email=False)`, returning
  it in the finalize step's result rather than sending Frappe's own
  built-in email (which names Frappe/ERPNext). `_finalize_new_tenant`'s
  stdin-script mechanism (see the `bench execute`-bug entry above) was
  extended with a second marker line so the finalize function's return
  value round-trips back to `platform_console`'s process as JSON, not
  just a pass/fail signal.
- `platform_console.provisioning._send_branded_welcome_email` sends the
  actual invite — Aetris-branded (confirmed with Imran: the existing
  Real Estate portal's own logo/name, not the tenant's business name,
  not a separate "overall SaaS" brand). Calls Brevo's transactional
  email API directly (`requests.post` to `api.brevo.com/v3/smtp/email`),
  not `frappe.sendmail()`/a Frappe Email Account — an intermediate
  attempt used the native Email Account + SMTP relay path instead (once
  Imran provided SMTP settings), for Frappe's own mail-queue/retry
  handling, but Brevo's SMTP credentials turned out to be a genuinely
  wrong/stale secret (`535 5.7.8 Authentication failed`, confirmed
  distinct from account activation once the API key started working)
  — reverted back to the API rather than chase that down further.
  Sender: `noreply@nnuggets.com`. Logo image URL is constructed as
  `https://<tenant-host>/assets/real_estate_os/images/aetris.png` —
  served from the newly-provisioned tenant's own site, since that's
  where the `real_estate_os` app (and its public assets) actually is.
  `BREVO_API_KEY` lives in `queue-long`'s environment
  (`realestate-override.yml`, same pattern as `MARIADB_ROOT_PASSWORD`)
  — this is where `run_provisioning` (and the email send inside it)
  actually executes, a different container from the one serving HTTP.
- **Brevo blocked sending behind two separate, unrelated gates found
  live, not one**: (1) account-level activation — `403 permission_denied,
  "Your SMTP account is not yet activated"` via the API, cleared once
  Imran contacted Brevo support directly; (2) this VPS is dual-stack and
  prefers IPv6 for outbound by default, so `api.brevo.com` resolved to
  an IPv6 address Brevo never authorized (only the VPS's IPv4 was) —
  `401 unrecognised IP address`, nothing to do with the API key itself.
  Fixed by forcing IPv4 for just the Brevo call via urllib3's
  `allowed_gai_family` hook, restored immediately after (not a
  container-wide network change). The separate SMTP `535` auth failure
  persisted even after both of the above cleared, confirming it really
  is a third, distinct bad credential rather than a symptom of either.
- **A working, live tenant is never marked Failed just because the
  invite email couldn't send.** New `admin_invite_sent` (Check) and
  `admin_reset_link` (Small Text) fields on `Tenant Site` record what
  happened either way — verified live: a real end-to-end run
  (`provisioning-test6.nnuggets.com`, via the Brevo-API-key version)
  reached `status = Live` correctly even though the invite email
  failed to send; `error_log` captured the real Brevo response body,
  and `admin_reset_link` held a valid, usable link the platform admin
  could still hand over manually.

## 2026-09-11/12 fix: headless provisioning skipped four more Setup Wizard steps

Imran, on `us.nnuggets.com` (a console-provisioned tenant): P&L/Trial
Balance/Balance Sheet "all seem to be wrong" despite 3 real signed leases.
Root cause was not the reports (verified directly — TB/BS correctly reflected
whatever GL Entries existed) but that **zero rent had ever actually been
invoiced**, for four separate reasons, all the same class of gap already hit
repeatedly by this app (Territory/Customer Group/Warehouse Type/System
Settings language — see `own-building-landlord.md`,
`fix-system-settings-language-timezone`): a headless
`bench new-site --install-app` never runs ERPNext's own Setup Wizard, so a
console-provisioned tenant is missing whatever that wizard would have seeded.

Found live, in the order they blocked:
1. No Item Group or "Nos" UOM at all — `invoicing.py::_get_rent_item` threw
   "Could not find Item Group: All Item Groups, Default Unit of Measure: Nos"
   on every attempt, silently, since the tenant's first day.
2. `System Settings.enable_scheduler` was unset (disabled) on every
   console-provisioned tenant (`us`/`rastec`/`test` — only
   `realestate.nnuggets.com`, migrated by hand many times, had it on). Every
   daily job in this app (rent/payable invoicing catch-up, lease lifecycle,
   PDC auto-deposit, reminders) had simply never run, on any of them.
3. No default Price List existed — once (1) was fixed, invoice creation threw
   `MandatoryError` on `selling_price_list`/`price_list_currency` instead.
4. `_create_invoice`/`create_payable_for_head_lease` left `currency` unset,
   which silently resolved to *some* ambient default rather than reliably
   this Company's own — caught the hard way live: with `us.nnuggets.com`'s
   `Global Defaults.default_currency` itself found corrupted to `INR` (and
   `country` blank) — a **separate, fifth bug**, apparently from a past
   manual backfill session that ran with wrong/placeholder args, found and
   fixed the same way on `rastec`/`test` too (`realestate.nnuggets.com`
   checked and confirmed correct, not touched) — the first accrual invoice
   this session created posted as literally 3,000 INR (~$31.53) against a
   real $3,000 landlord cheque, requiring a cancel-and-redo once caught.

Fixed permanently in `provisioning.py::finalize_new_tenant`: calls ERPNext's
own `install_fixtures.install(country=...)` (the exact function Setup Wizard
calls — idempotent, covers Item Group/UOM/Territory/Customer
Group/Supplier Group/Warehouse Type/Mode of Payment in one call, superseding
several of this app's own narrower hand-patches for the same gap class) plus
explicit Price List creation, both gated to first-provisioning-only (never
on a re-run) because `install()` also unconditionally overwrites Selling/
Buying Settings to stock defaults with no existence check, and
`realestate.nnuggets.com` has a hand-customized `cust_master_name` a re-run
would silently revert. `enable_scheduler()` runs on every call (self-heals,
no customization risk). `invoicing.py`/`landlord_payables.py` now pin
`currency` explicitly on every invoice they create. Verified end-to-end on a
genuine fresh throwaway site (`provisioning-test-claude.nnuggets.com`,
created via `bench new-site` directly, `finalize_new_tenant` called, item/
UOM/price-list/scheduler/Global-Defaults all correct, `_get_rent_item()`
succeeded, then torn down) — not just backfilled on an already-broken site.
Backfilled `us`/`rastec`/`test.nnuggets.com` directly (fixtures + Price
Lists + scheduler + Global Defaults currency/country); `rastec`/`test` had
zero rows in either schedule table at the time, so enabling their scheduler
carried no backdated-invoice-burst risk. On `us.nnuggets.com` specifically:
re-posted the landlord's accrual Purchase Invoice correctly in USD (the
INR one cancelled), reconciled it against the existing $3,000 cheque
payment, and reconciled the tenant's two already-cleared September PDCs
against their now-existing Sales Invoices (`pdc.py::mark_pdc_cleared`
already had defense-in-depth for exactly this "cheque cleared before its
invoice existed" case — reused rather than hand-rolled). A third lease
signed live mid-session and was correctly auto-invoiced and auto-reconciled
by the now-working scheduler with no manual step, confirming the fix.

## Implementation Plan

- [x] Create `platform_console` app scaffold, install on a new `console.nnuggets.com` site — 2026-09-01
- [x] `Tenant Site` DocType (incl. `module` field) + `Platform Admin` role — 2026-09-01
- [x] Provisioning whitelisted method (bench new-site + install-app,
      parameterized on `module`; required-apps install automatically via
      `real_estate_os.hooks.required_apps`, no extra code needed) — 2026-09-01
- [x] Traefik dynamic-route file writer (template + bind mount) — 2026-09-01
- [x] Status polling + Live/Failed transitions — 2026-09-01
- [x] Multi-enable Integrations table (replacing the single E-Sign Mode
      field), extensible to future integration categories — 2026-09-01,
      per Imran's follow-up request after using the built form
- [x] Branded (Aetris) admin-invite email via Brevo, replacing the
      previously-nonfunctional Admin Email field — 2026-09-01
- [ ] Manual-checklist path for custom domains — not built; no custom-domain
      tenant exists yet to need it
- [ ] Onboard American Real Estate as the first real second tenant — separate,
      later action now that the console exists; not part of this build
- [x] Brevo account activation — Imran contacted Brevo support; confirmed
      live via a real `201` + `messageId` from the API — 2026-09-01.
      Unblocked a second, unrelated issue found in the same pass (this
      VPS defaults to IPv6 outbound, which Brevo hadn't authorized —
      fixed by forcing IPv4 for the Brevo call specifically). Sending
      itself uses Brevo's API, not SMTP — see Design above for why.
      Verified live end-to-end: rastec.nnuggets.com's invite, which had
      previously failed, was resent successfully after this fix.

## Acceptance Criteria

- [x] Filling the form for a `*.nnuggets.com` slug + "Real Estate" module
      produces a live, working tenant site with zero manual SSH steps,
      including its required e-sign module already installed — verified
      live end-to-end 2026-09-01 (`provisioning-test4.nnuggets.com`:
      created, Company/CoA/Fiscal Year/esign_mode set, Traefik route
      written, reachable at 200, then torn down as a throwaway test). Took
      four attempts to get a genuinely clean run — each surfaced a real
      bug (see Design), not a flaky test.
- [x] The control-plane site's provisioning capability is unreachable from any
      tenant site (verified: grepped every doctype/app `real_estate_os`
      ships for bench/Traefik code — none; `platform_console` is not in
      any tenant site's installed-app list, confirmed via `bench list-apps`
      on both `realestate.nnuggets.com` and the test site)
- [ ] Renaming an account on one tenant's site has zero effect on any other
      tenant's site, including one running a different module — follows
      from tenant-per-site (`0003`) by construction; not re-verified as a
      live regression check this round (no second real tenant exists yet
      to check against), carried forward as an open item

## Related

- Domain index: `vault/multi-tenancy/multi-tenancy.md`
- ADR: `0008-tenant-provisioning-control-plane`, `0016-module-aware-provisioning`
- Superseded plan: `tenant-onboarding.md`
- Depends on: `ui-portal/features/tenant-portal-ui.md` (the renter portal must
  exist before it's worth replicating to a second tenant)
