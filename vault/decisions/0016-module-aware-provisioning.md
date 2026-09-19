# ADR-0016: Module-Aware Tenant Provisioning & Per-Module Chart of Accounts

- **Status**: accepted
- **Date**: 2026-08-27
- **Deciders**: Imran
- **Supersedes**: refines `0008-tenant-provisioning-control-plane` (the
  provisioning mechanism itself is unchanged; this parameterizes which app
  it installs and adds a per-module Chart-of-Accounts step it didn't have)
- **Superseded by**: -

## Context

Imran's long-term plan: a back-end control plane where he selects an
industry **module** (Real Estate today, e.g. Automobile Workshop later)
and enters a new client's details; the system provisions that client's
own site, running that module's own domain-specific UI and DocTypes, but
sharing the same underlying ERP/accounting engine (GL, P&L, Balance
Sheet, Journal Entry) across every module. He asked specifically: if
`real_estate_os`'s default expense account is renamed away from "Cost of
Goods Sold" (per the accounting finding from earlier today), does that
renaming risk leaking into or conflicting with a future, differently-
named module?

**This is not new architecture — it's an expansion of two already-approved
decisions that were never built:**

- `0003-multi-tenancy`: tenant-per-site, separate database per client.
  Each client's `Company` (and therefore its entire Chart of Accounts) is
  already fully independent — a different site, a different database, no
  shared Account table. This alone answers the COGS question: renaming an
  account on a real-estate client's site cannot affect any other site,
  full stop, because there is no cross-site data access at all.
- `0008-tenant-provisioning-control-plane`: a separate, non-tenant-facing
  control-plane site provisions new tenant sites via `bench new-site
  <slug> --install-app real_estate_os` + fixture seeding. Approved, but
  **never built** (`tenant-provisioning-console.md` is still `planned`).
  As written, it hardcodes `real_estate_os` as *the* app — it has no
  concept of choosing between modules.

**Also true, and worth stating plainly so the multi-module plan isn't
mistaken for new risk:** the accounting engine Imran wants shared across
modules (Company, Account, GL Entry, Journal Entry, the P&L/Balance
Sheet/GL report engines) is not something `real_estate_os` owns or could
accidentally entangle with another module — it's the `erpnext` framework
app itself, a prerequisite on every site regardless of which vertical app
is installed alongside it. `real_estate_os` only ever *extends* it
(Customer, Cost Center, Supplier), the same pattern a future
`auto_workshop_os` would follow. This was true before this conversation;
it's why the multi-module plan is straightforward rather than a
rearchitecture.

## Decision

1. **`Tenant Site` gains a `module` field** (Select, e.g. "Real Estate" |
   "Automobile Workshop" — options grow as modules are built) mapping to
   an installable app name (`real_estate_os` | `auto_workshop_os`, ...).
   The provisioning method's `bench new-site ... --install-app` becomes
   parameterized on this instead of hardcoded.
2. **Each vertical app owns its own Chart-of-Accounts setup**, applied
   automatically when that module is installed on a new site — never a
   shared or global mapping.

   **Resolved 2026-09-01, built as part of the Platform Console
   (`tenant-provisioning-console.md`)**: option (a) below, but sourced
   from ERPNext's own per-country template library rather than a single
   `real_estate_os`-authored template — Imran's actual need, raised while
   reviewing this plan, was "customers from Canada, US, UK, India, Saudi
   etc... not everyone uses the same CoA," which a single opinionated
   real-estate template wouldn't have served. Confirmed live:
   `erpnext...chart_of_accounts.get_charts_for_country(country)` (the same
   function Company's own Desk form uses) already returns real,
   ERPNext-shipped templates per country — India and UAE have dedicated
   ones, most others (US, UK, Canada, Saudi Arabia among them, in this
   ERPNext version) fall back to "Standard" or "Standard with Numbers".
   `real_estate_os.provisioning.finalize_new_tenant` creates the Company
   with `create_chart_of_accounts_based_on="Standard Template"` +
   whichever template the platform admin picked on the Tenant Site form —
   this triggers ERPNext's own `create_charts()` automatically, no custom
   account-building code. `rename_default_income_expense_accounts`
   (option (b), already shipping since `0015`) still runs afterward
   regardless of template chosen, giving every tenant the real-estate-
   appropriate income/expense names on top of whatever base CoA their
   country uses.

   Two things below this line were the *original* two candidates
   considered before that clarification, kept for history:
   - **(a) Custom Chart-of-Accounts template at Company-creation time** —
     ERPNext supports supplying a named CoA template (not just "Standard")
     when a Company is created.
   - **(b) Seed with the Standard template, then rename via a patch** —
     the same pattern already used five times this session
     (`seed_amenity_options`, `seed_own_building_landlord`,
     `seed_customer_field_visibility`, `seed_pdc_entry_field_visibility`,
     `seed_pdc_entry_field_visibility`'s sibling) — an idempotent
     `patches/v0_0/` script run on install/migrate.
3. **A future `auto_workshop_os` is a separate Frappe app** (own repo,
   likely, mirroring `real_estate_os`), installed alongside `real_estate_os`
   on the same bench — a bench can host many apps; each *site* chooses
   which to install. Building it is out of scope here (Imran said "in the
   future I may add" it) — this ADR only makes sure today's provisioning
   design doesn't have to be reworked when that day comes.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **Parameterize `Tenant Site`'s module + per-app CoA ownership** (proposed) | Matches how Frappe multi-app-per-bench already works natively; each module's accounts are self-contained, no shared config to keep in sync; reuses the exact fixture-seeding pattern already proven 5 times this session | `tenant-provisioning-console.md`'s implementation plan needs updating before it's built (it isn't yet, so no rework of shipped code) | **Recommended** |
| One shared, configurable CoA mapping table across all modules | Looks reusable | Solves a problem that doesn't exist — modules don't share a database, so there's nothing to keep in sync; adds a config surface for zero benefit | Rejected |
| Hardcode a second `install-app` branch per module as modules are added | Fastest to hack in later | Doesn't scale past 2-3 modules; the `module` field + lookup is barely more work and scales cleanly | Rejected |

## Consequences

### Positive

- Confirms, rather than changes, the isolation guarantee `0003` was
  written for — multi-module is "just" multi-app-per-bench, which Frappe
  already supports natively.
- Answers Imran's actual question with certainty: no cross-module GL
  naming risk exists, by construction of tenant-per-site.
- `tenant-provisioning-console.md` (still unbuilt) gets the right shape
  from the start instead of needing rework once a second module exists.

### Negative

- Slightly more upfront design on a feature that's still `planned`/
  unbuilt — worth it now specifically because Imran raised the multi-
  module question before any of it was built, not after.
- Whichever CoA approach (custom template vs. seed-then-rename) is chosen
  needs a real implementation-time check against this ERPNext version —
  flagged above as unverified, not assumed.

## Implementation

- Feature file: `vault/multi-tenancy/features/tenant-provisioning-console.md`
  (update its Implementation Plan to include the `module` field + per-app
  CoA seeding once this ADR is approved)
- Related: `vault/findings/2026-08-27-full-accounting-system.md`,
  `vault/decisions/0015-portal-native-accounting-reports.md`
  (the account-renaming question this ADR answers),
  `vault/decisions/0017-platform-modularization.md` (the module/interface
  architecture this provisioning design installs)
