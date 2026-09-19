# ADR-0021: Platform-Wide Outgoing Mail Branding (Replace ERPNext Branding)

- **Status**: accepted
- **Date**: 2026-09-04
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Every email built via plain `frappe.sendmail()` (no `template`/`header`/
`with_container`) gets a "Sent via ERPNext" link in its footer. This is
`erpnext/hooks.py`'s `default_mail_footer` hook (upstream, not ours to
edit), rendered by `frappe/email/email_body.py:get_footer()` unless System
Settings → "Disable Standard Email Footer" (`disable_standard_email_footer`)
is checked. It defaults to unchecked.

Confirmed call sites building emails this bare way today:
- `apps/inbuilt_esign/inbuilt_esign/workflow.py` — OTP + signing-request
  invites (`_send_otp`, `_send_invite`) — Imran's original complaint.
- `apps/real_estate_os/real_estate_os/payments/invoicing.py:402` — invoice
  email.

Same code path also means these emails carry **no logo at all** — the
masthead only renders when the caller passes `with_container=True` or
`header=`, which none of the above do.

**This is not a Real Estate OS problem — first draft of this ADR wrongly
scoped the fix to `real_estate_os`.** Per `PLATFORM_STRATEGY.md` Pillar 2,
"integrations (e-sign, payments, email/SMS, etc.) are never code inside an
OS app" — each is its own reusable Frappe app that any OS declares as a
`required_apps` dependency (the same pattern `inbuilt_esign` and
`accounts_portal` already follow). Outgoing-mail branding is exactly that
category and needs its own app, not a fix living inside `real_estate_os`.

Today's delivery already has one central per-tenant choke point:
`real_estate_os/brevo_mail.py`, registered as the `override_email_send`
hook, branching each send between "tenant's own SMTP" (raw pass-through)
and "Aetris-managed" (Brevo API) — but it lives in the wrong app for the
reason above, and (per Imran) the *rendered template* must be identical on
both paths, not something only the Brevo path gets dressed up for.

Business branding data already exists in shared ERP core, unused for this:
- `Company.company_logo` (stock ERPNext `Attach Image` field) — the
  business's own logo. Zero new schema needed.
- System Settings `email_footer_address` (stock Frappe field, already
  per-tenant-site) — the business's own postal address for the footer.

The Settings page's "Portal Branding" section (`real_estate_os/api.py:
get_settings_data`) currently returns a **hardcoded**
`"/assets/real_estate_os/images/aetris.svg"` for `portal.logo` — not read
from anywhere per-tenant, which is why the business user can't change it
today (Imran's observation). Separately, `provisioning.py:_set_aetris_branding()`
deliberately sets Website Settings' `app_logo`/`app_name` to Aetris on
every tenant site, on purpose, so that unauthenticated Frappe pages
(`/login`, `/update-password`) show Aetris, not stock ERPNext — meaning
Website Settings is already claimed for *platform* branding and is not
available to repurpose as "the business's own logo."

## Decision

**New module app, named `email_system`** ("Email System"), OS-agnostic,
`required_apps` dependency of every OS app (added to `real_estate_os`'s
list alongside `inbuilt_esign`/`accounts_portal`; any future OS declares
it too). Its own repo, `rafaii/email-system`, private, mirroring
`inbuilt_esign`/`accounts_portal`'s extraction pattern. Owns:

1. **The `override_email_send` hook**, moved out of
   `real_estate_os/brevo_mail.py` into this app. Same per-tenant
   own-SMTP-vs-Aetris-managed transport choice as today — that choice
   isn't OS-specific either.
2. **One content-branding step, applied before the transport branch**
   (not only inside the Brevo path) — so the exact same rendered HTML goes
   out whether the tenant uses their own SMTP or Aetris-managed sending:
   - Parse the already-rendered `html_body` (Frappe has finished building
     it by the time this hook fires).
   - Strip/neutralize any leftover ERPNext footer html (belt-and-suspenders
     — Layer 1 below should already prevent it from being generated).
   - Inject a top logo block using `Company.company_logo` for this
     tenant's one Company — every business on this platform has exactly
     one Company (a business wanting a second Company creates a second
     account/tenant site instead), resolved via `Global Defaults.
     default_company`, no ambiguity to handle. Only rendered if set — no
     placeholder image when unset.
   - Inject a footer block: business address (`email_footer_address`, if
     set) + an always-present, separate "Powered by Aetris" line with the
     Aetris logo — unconditional, regardless of transport or whether the
     business set their own logo.
3. **Layer 1**: `frappe.db.set_default("disable_standard_email_footer", 1)`
   in this app's `after_install`/`after_migrate` — kills "Sent via
   ERPNext" at the source, for every current and future
   `frappe.sendmail()` caller, on every site.
4. **Ships its own copy of the Aetris logo** (`email_system/public/images/
   aetris.png`) rather than reaching into `real_estate_os`'s `/assets/`
   path — keeps the module deployable standalone for any future OS.

**Settings page fix**: `get_settings_data()`'s `portal.logo` becomes a real
read from `Company.company_logo` for the tenant's default Company (falling
back to nothing, not the Aetris asset), and the Settings UI gets an actual
upload control writing back to that field — this is what makes "business
can add their own logo" true instead of a hardcoded string.

No changes needed in `inbuilt_esign`, `accounts_portal`, or
`real_estate_os/payments/invoicing.py` beyond adding the new
`required_apps` entry to whichever OS app(s) need it — they keep calling
plain `frappe.sendmail()`; branding is added once, centrally, at send time.
`inbuilt_esign` stays free of any Aetris-specific knowledge, preserving its
documented no-OS-dependency design.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Keep the fix inside `real_estate_os` (original draft) | Least code movement | Violates Pillar 2 — a second OS gets none of this; contradicts Imran's explicit correction | Rejected |
| Brand only the Aetris-managed (Brevo) path, leave own-SMTP pass-through untouched (original draft) | Simpler; "you chose your own SMTP, your own branding" | Imran wants the *same* template on both paths — a tenant's customer emails should look identical regardless of which transport delivered them | Rejected |
| Patch each sender to build its own branded HTML | Full control per email type | N-times the work; every new app/email type has to remember it again | Rejected |
| Register our own `default_mail_footer` hook instead of a content-injection step | Looks like the "intended" Frappe extension point | `get_hooks()` concatenates across apps — erpnext's "Sent via ERPNext" would still render *alongside* ours; the disable flag is all-or-nothing | Rejected |
| New OS-agnostic module app, single content-branding step before transport branch (proposed) | Matches Pillar 2's module architecture; identical template on both transports; reuses stock `Company.company_logo`/`email_footer_address` — no new schema; makes Settings page's logo control real | More code movement (new app, new repo, hooks.py surgery in `real_estate_os`) | **Chosen** |

## Consequences

### Positive

- Fixes "Sent via ERPNext" everywhere, on every OS, immediately.
- Every OS's tenant emails get that tenant's own logo (if they've set one)
  + address, with a consistent "Powered by Aetris" credit — identical
  regardless of SMTP vs Brevo.
- Settings page's branding section becomes truthful/editable instead of a
  hardcoded string.
- No new dependency added to `inbuilt_esign`; future OS apps get this for
  free by declaring the one `required_apps` entry.

### Negative

- New app + repo to stand up and maintain (mirrors `inbuilt_esign`/
  `accounts_portal`'s existing extraction pattern, so not a new kind of
  overhead).
- HTML injection into an already-fully-rendered MIME message is string/DOM
  surgery — coupled to Frappe's current `standard.html` wrapper shape; a
  future Frappe upgrade changing that markup could need a matching tweak.
- Every existing tenant site needs `bench get-app`/`install-app` for the
  new module plus a `bench --site <site> clear-cache` after the
  `override_email_send` hook moves apps (stale resolved hook otherwise
  keeps pointing at the old `real_estate_os.brevo_mail` path).

## Implementation

- Feature file: `vault/email-system/features/outgoing-mail-branding.md`
