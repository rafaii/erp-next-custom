---
status: done
owner: developer-1
domain: multi-tenancy
created: 2026-08-27
updated: 2026-08-27
related_adr: []
---

# Finding — Platform Modularization (Domain OS Apps, Reusable Modules, Per-Customer Extensions)

## Method

Investigated what already exists before proposing anything new: read
`MASTERPLAN.md` in full, `0002-esign-approach.md` (the closest existing
precedent for a "pluggable provider"), and checked the actual live e-sign
code to see whether that abstraction was ever really implemented, not
just designed.

## The one-line answer

**Three of Imran's four pillars already have real, working precedent in
this codebase — but one of them (plug-and-play module dispatch) was
designed once and never actually finished.** The gap isn't "we need to
invent a modularization scheme" — it's "finish the one abstraction we
already started, then apply the same pattern deliberately instead of by
accident."

## What already exists, verified

**Domain OS apps (pillar 1)** — already the working pattern, not a new
idea: `real_estate_os` is exactly this. Every future vertical (Kindergarten
OS, Workshop OS, Retail OS) would be a new, separate Frappe app the same
shape, on the same bench (per `draft-adr-module-aware-provisioning.md`
from earlier today).

**One concrete asset worth naming explicitly**: a large part of the
portal's own frontend (`ui/src/pages/Dashboard.tsx`'s `ResourceListView`,
`DetailView`, `EditableField`, `FieldVisibilityDialog`, sort/filter/
search/pagination) is **already vertical-agnostic** — it renders whatever
`portal_nav_items`/`get_doc_detail` describe generically and knows
nothing about "Building" or "Lease" specifically. This is a real,
already-built reusable asset for pillar 1, not something to rebuild per
OS — it just isn't packaged as one today (it lives inside `real_estate_os`
because there's only ever been one OS to build it for).

**Per-customer extension mechanism (pillar 4)** — already proven, multiple
times, in this exact app:
- Custom Field fixtures (`customer-extension.md`'s 7 KYC fields on
  Customer) — the standard Frappe way to add fields without touching core
  DocType JSON.
- `Portal Field Visibility` (built this session,
  `tenant-field-curation.md`) — a bespoke but exactly-fitting example of
  per-tenant UI customization with zero code change, since each site's
  rows are independent.
- Frappe also natively offers Property Setter (per-site field tweaks:
  mandatory, label, hidden) and Client/Server Script (per-site logic)
  without a code deploy — neither has been *needed* yet in this app, but
  both are standard, proven Frappe mechanisms, not something to build.
- For the rarer case where actual custom DocTypes/logic are needed for
  one specific client: Frappe's multi-app model supports installing a
  small, bespoke overlay app on just that one client's site, on top of
  the OS app — the standard way Frappe handles "one client needs more
  than config."

**Standardized integration architecture (pillar 3)** — the mechanism
already used throughout this app for *internal* extension points
(`portal_nav_items`, `portal_detail_links`, `scheduler_events` are all
Frappe hooks) is the same mechanism that would carry *external* module
dispatch: a vertical OS declares it wants "a payment provider" via a
well-known hook name; whichever module app is installed provides the
concrete implementation, resolved at runtime via `frappe.get_hooks(...)`.
Nothing new to invent here either — just a new *use* of a pattern already
load-bearing in this app.

## The one real gap, verified in code, not assumed

**Reusable modules with actual plug-and-play dispatch (pillar 2) — designed
once, never finished.** `0002-esign-approach.md` explicitly decided a
"provider abstraction" for e-signing: a `Signature Settings.esign_mode`
Select field (`In-Built | DocuSign | ZohoSign`) so a tenant could pick
their provider. Checked the actual code: **`esign_mode` is never read
anywhere** (`grep -rn "esign_mode" --include="*.py"` returns nothing
outside the DocType JSON itself). The e-sign workflow is 100% hardcoded to
the in-built implementation regardless of what the field is set to — the
config option exists, the dispatch mechanism to act on it does not.

`MASTERPLAN.md` §10.2 makes the same anti-pattern explicit as sample code:
a `docusign_utils.py` example with a `send_for_signature(lease_id)`
function that calls DocuSign's API directly, hardcoded, with no interface
boundary — exactly the "integration baked into the OS instead of a
standalone module" problem Imran is now asking to avoid going forward.

## Recommendation

Approved as `vault/decisions/0017-platform-modularization.md`. Headline:
finish e-sign's provider abstraction for real — extract it into its own
module app, implement the missing dispatch, wire ZohoSign as the actual
second provider — and let that be the *first* real, working example of
the general pattern, rather than designing the general pattern in the
abstract and hoping e-sign gets retrofitted to it later.

## Resolved: MASTERPLAN.md's scope

The question below was decided the same day it was raised: `MASTERPLAN.md`
is renamed to `vault/PLATFORM_STRATEGY.md` (platform-wide: multi-OS
strategy, modularization, provisioning) and its real-estate-specific
content moved to a new `vault/os/REALESTATE_MASTERPLAN.md`. Original text
of this section, kept for the record:

> `MASTERPLAN.md` is titled and scoped as "Real Estate Leasing SaaS on
> ERPNext" — a single vertical's PRD. Imran's ask to "include
> modularization in our master plan" doesn't fit cleanly inside a document
> scoped to one vertical; modularization is explicitly cross-cutting (it
> exists *because* more verticals are coming).

## Related

- ADR: `vault/decisions/0017-platform-modularization.md`
- ADR: `0002-esign-approach.md` (the precedent this generalizes)
- ADR: `vault/decisions/0016-module-aware-provisioning.md`
  (the related decision — module *selection* at provisioning; this
  finding is about module *architecture* once selected)
- PRD: `vault/PLATFORM_STRATEGY.md`, `vault/os/REALESTATE_MASTERPLAN.md`
