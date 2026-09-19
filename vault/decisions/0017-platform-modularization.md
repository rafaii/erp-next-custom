# ADR-0017: Platform Modularization — Domain OS Apps, Reusable Modules, Extension Tiers

- **Status**: accepted
- **Date**: 2026-08-27
- **Deciders**: Imran
- **Supersedes**: refines `0002-esign-approach.md` (its provider
  abstraction was decided but never actually wired up — this finishes it
  as the first real example of the general pattern; the underlying
  in-built-default e-sign decision is unchanged)
- **Superseded by**: -

## Context

Imran's plan: domain-specific "operating system" apps (Real Estate today,
Kindergarten/Workshop/Retail later), each built on reusable, **standalone**
modules (Stripe, Zoho e-sign, etc.) selected at setup rather than
hardcoded per OS, plus a consistent way to make small, per-customer
tweaks (the "5%") without forking the core OS app.

See `vault/findings/2026-08-27-platform-modularization.md`. Short version:
three of the four pillars already have real, working precedent in this
app (OS-per-vertical, per-customer extension via Custom Field/Portal
Field Visibility, hooks as the dispatch mechanism). The fourth —
standalone modules with actual plug-and-play dispatch — was designed once
for e-sign (`0002`'s `esign_mode` field) and never finished: verified live
that `esign_mode` is never read anywhere in the code, so the "provider
abstraction" was a config field with no effect.

Imran's concrete refinement for e-sign specifically, folded into this
decision: e-sign should be extracted into its own external, reusable
module app — installed **automatically as a required dependency** whenever
an OS that needs it (Real Estate today, others later) is provisioned, not
something the client separately opts into. The *choice* of which e-sign
implementation actually runs (in-built vs. Zoho, later DocuSign) is a
**business-config-page decision** made by the tenant after their site
exists, not a provisioning-time decision — the module dependency and the
active-provider choice are two different things.

## Decision

**1. Domain OS apps** (codifying current practice, no change): each
vertical is its own Frappe app, its own repo, its own DocTypes and
`portal_nav_items`. The already-vertical-agnostic parts of the current
portal frontend (`ResourceListView`, `DetailView`, `EditableField`,
`FieldVisibilityDialog`, sort/filter/search/pagination) get extracted into
a shared package **once a second OS is actually being built**, not
before — no second consumer exists yet to validate the extraction
boundary against.

**2. Reusable modules are separate Frappe apps, with a "required by"
relationship to OS apps**:
- Each integration (Stripe, e-sign, a future DocuSign) is its own Frappe
  app — never code inside a vertical OS app.
- Each module ships its own settings DocType (e.g. `Signature Settings`,
  a future `Stripe Settings`) holding credentials/config, and implements
  a documented **interface contract** for its category (below).
- An OS app declares which module categories it **requires** (e.g. Real
  Estate requires an `ESignProvider`, doesn't require a `PaymentProvider`
  yet). Provisioning (per `0016-module-aware-provisioning`) installs an
  OS's required modules automatically alongside it — a client never ends
  up with a Real Estate site missing its e-sign capability.
- Multiple modules can implement the same category (in-built e-sign app,
  a future ZohoSign app both implement `ESignProvider`). All of a
  category's installed implementations are available to the tenant; which
  one is *active* is a per-tenant runtime choice, not a provisioning-time
  or code choice — see point 4.

**3. Standardized integration architecture** — dispatch via Frappe hooks,
the mechanism already load-bearing in this app (`portal_nav_items`,
`portal_detail_links`, `scheduler_events`):
- One interface per module **category**, not per vendor (`ESignProvider`,
  `PaymentProvider`, ...) — a short list of function names + expected
  args/return shape, documented once.
- An OS app resolves the *active* provider generically
  (`frappe.get_hooks("esign_provider")` returns whichever installed
  module(s) declare that hook; the tenant's chosen one, from `Signature
  Settings.esign_mode`, is the one actually called) — never
  `import zoho_esign` directly in `real_estate_os`.
- **First real implementation, not a new example**: finish `0002`'s e-sign
  abstraction for real. Extract the existing in-built e-sign code out of
  `real_estate_os` into its own module app implementing `ESignProvider`;
  build one real second provider (ZohoSign) as a second standalone app
  implementing the same contract; make `esign_mode` actually dispatch
  between them. This validates the whole pattern against a real,
  already-built integration instead of designing it in the abstract.

**4. The tenant-facing choice lives on a Business Config page** (already
being built — the portal's Settings area): once modules are installed,
the business owner picks which *active* implementation to use per
category (e.g. "E-Signature: In-Built" vs "E-Signature: Zoho Sign") —
this is exactly `Signature Settings.esign_mode`, made to actually mean
something. Enabling a module in the first place (has Zoho Sign even been
installed on this site) is a provisioning/platform-admin concern per
`0016`; which enabled module is *active* is the tenant's own choice from
their config page.

**5. Per-customer extension — a tiered policy**, using mechanisms Frappe
already provides:

| Tier | Mechanism | When |
| --- | --- | --- |
| 1 | Custom Field (fixtures or per-site) | Need one more field on an existing DocType |
| 2 | Property Setter | Change a field's label/mandatory/hidden state per site |
| 3 | Portal Field Visibility (already built) | Curate which fields a business sees/edits |
| 4 | Client Script / Server Script | Per-site logic without a code deploy |
| 5 | A small per-customer overlay app | Real custom DocTypes/logic only one client needs — kept out of the OS app entirely, installed only on that client's site |

Tiers 1-4 need no new engineering — standard Frappe. Tier 5 needs a
documented convention (naming, where it lives, how it depends on the OS
app) so it isn't reinvented differently per client.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **Hooks-based interface contracts, modules as required-by-OS separate apps** (chosen) | Reuses a mechanism already proven here; OS apps never hard-depend on a module's code; matches how Imran described the enrollment flow (module attaches automatically, provider choice is a later tenant decision) | Requires documenting each category's contract once, and discipline to call through the hook | **Chosen** |
| Each OS app imports specific module apps directly | Simpler for exactly one module | Every new module requires editing every OS app that wants it; reintroduces the coupling `0002` tried to avoid | Rejected |
| Provider choice made at provisioning time (platform admin picks it, not the tenant) | Simpler data model | Contradicts Imran's described flow explicitly — "business owner then can decide if they use the built in esign or Zoho Sign" from their own config page | Rejected |
| Generic plugin registry/marketplace DocType now | Looks more "platform-y" upfront | Same premature-abstraction risk as extracting the frontend shell early — no second module exists yet to validate the shape against | Deferred until ZohoSign is actually built |

## Consequences

### Positive

- Finishes a decision (`0002`) already made and paid for in design effort.
- Matches Imran's described enrollment/config flow exactly: module
  attachment is automatic (required-by-OS), active-provider choice is a
  tenant-facing config decision made later.
- Future OS apps inherit working modules for free.

### Negative

- Real near-term work: extracting in-built e-sign into its own app,
  building ZohoSign as a second real app, and wiring actual dispatch is
  substantial, not just documentation.
- The extension-tier policy needs discipline (reach for tier 1-4 before
  tier 5) that isn't enforced by tooling.

## Implementation

- Feature file: `vault/esign/features/provider-abstraction.md` (updated to
  reflect module-app extraction instead of in-app adapter classes)
- Priority and sequencing: `vault/IMPLEMENTATION-PLAN.md`
