---
status: in-progress
owner: developer-1
domain: email-system
created: 2026-09-04
updated: 2026-09-05
related_adr: ["0021-email-system-outgoing-mail-branding"]
---

# Outgoing Mail Branding

## Summary

New Frappe app `email_system` replaces "Sent via ERPNext" (from ERPNext's
`default_mail_footer` hook) on every outgoing email with the sending
business's own logo/address plus a "Powered by Aetris" credit, rendered
identically regardless of whether the tenant sends via their own SMTP or
Aetris-managed (Brevo) delivery. Also fixes the Settings page's "Portal
branding" logo control, which was returning a hardcoded Aetris asset path
instead of a real, editable per-tenant value.

## Requirements

- No email built via `frappe.sendmail()` shows ERPNext/Frappe branding,
  on any tenant site, regardless of which OS app sent it.
- Every such email shows: the sending business's own logo (if set) at
  top, their postal address in the footer (if set), and an always-present
  "Powered by Aetris" credit + Aetris logo in the footer.
- The rendered template is byte-for-byte the same whether delivery goes
  through the tenant's own configured SMTP Email Account or through
  Aetris-managed Brevo sending — no divergence by transport.
- A business has exactly one Company on this platform (a second Company
  means a second account/tenant site) — branding always resolves to
  `Global Defaults.default_company`, no multi-company case to handle.
- Settings page's "Portal branding" logo becomes real: read from
  `Company.company_logo`, uploadable by a user with Company write
  permission, uploaded as a **public** file (required for email clients
  and unauthenticated portal views to load it).
- `email_system` ships its own copy of the Aetris logo asset — it must
  not depend on `real_estate_os`'s `/assets/` path, so any future OS can
  depend on it standalone.

## Design

**New app**: `apps/email_system` (own repo `rafaii/email-system`, private).

- `email_system/hooks.py`:
  - `required_apps = ["frappe"]`
  - `override_email_send = "email_system.email_system.dispatch.dispatch_outgoing_mail"`
  - `after_install` / `after_migrate` = `email_system.email_system.setup.after_install`
- `email_system/email_system/setup.py` — `after_install()` calls
  `frappe.db.set_default("disable_standard_email_footer", 1)`.
- `email_system/email_system/dispatch.py` — replaces
  `real_estate_os/brevo_mail.py` (deleted). `dispatch_outgoing_mail`:
  1. Parses the already-rendered MIME `message` (Frappe has finished
     building header/body/footer by the time this hook fires).
  2. Applies one branding transform to the html/text parts in place —
     top logo from `Company.company_logo` (via `Global Defaults.
     default_company`, only if the file is not private), footer address
     from System Settings `email_footer_address`, and an unconditional
     "Powered by Aetris" line + shipped logo
     (`email_system/public/images/aetris.png`).
  3. *Then* branches transport exactly as `brevo_mail.py` did: tenant's
     own configured Email Account (real SMTP session) vs. Aetris-managed
     Brevo API — both now send the identically-branded message.
- `email_system/email_system/public/images/aetris.png` +
  `aetris.svg` — copied from `real_estate_os`'s existing assets.

**real_estate_os changes**:
- `hooks.py`: `required_apps` gains `"email_system"`; `override_email_send`
  line removed (now owned by `email_system`).
- `brevo_mail.py` deleted (superseded).
- `api.py::get_settings_data()`: `portal.logo` now reads
  `Company.company_logo` for `Global Defaults.default_company` (falls back
  to `None`, not an Aetris asset path); adds `portal.can_manage` from
  `frappe.has_permission("Company", "write")`.
- `ui/src/lib/api.ts::uploadFile()`: gains an `isPrivate` parameter
  (default `true`, preserving existing callers) so the logo upload can
  pass `false`.
- `ui/src/pages/Dashboard.tsx` Settings page "Portal branding" card: adds
  a file-upload control (gated on `portal.can_manage`) calling
  `uploadFile(file, "Company", company_name, "company_logo", false)`.

## Implementation Plan

- [x] ADR-0021 drafted, revised per feedback, approved.
- [x] Vault domain + feature file created.
- [x] Scaffold `apps/email_system` (hooks, setup, dispatch, assets,
      pyproject/README/LICENSE). `dispatch.py`'s MIME parse/modify/
      re-serialize round trip and the header/footer HTML injection were
      sanity-checked against Python's stdlib `email` module directly
      (bs4 itself unavailable in this sandbox — it's a stock Frappe
      dependency, `beautifulsoup4~=4.15.0`, confirmed in frappe's own
      `pyproject.toml`, so not a new dependency).
- [x] Create GitHub repo `rafaii/email-system` (private), push initial
      commit (`8c7c5bf`).
- [x] `real_estate_os`: add `email_system` to `required_apps`, remove
      `override_email_send`, delete `brevo_mail.py`. Branch
      `agent/developer-1/email-system-branding`, merged to `main`
      (`14ed2f5`), pushed.
- [x] `real_estate_os/api.py`: fix `get_settings_data()` logo + add
      `can_manage`.
- [x] `real_estate_os/ui`: logo upload control + `uploadFile` public-upload
      support; `bun run build` (tsc + vite both clean), regenerated
      `public/portal/` bundle committed alongside.
- [x] Deploy: cloned `email_system` into the VPS's `apps/` dir, added its
      `COPY`/`pip install -e`/`public/` symlink lines to `apps/Dockerfile`
      (untracked, edited via SSH per `project_vps_dockerfile_untracked`
      memory), pulled `real_estate_os` to `14ed2f5`, rebuilt
      `realestate-custom:v15.119.3`, recreated containers (full `-f` list,
      `-p realestate`, no `--remove-orphans` — db/redis untouched),
      installed `email_system` on all three sites (`rastec.nnuggets.com`,
      `realestate.nnuggets.com`, `console.nnuggets.com`), cleared cache on
      each (hook moved apps).
- [x] Verify live: on `rastec.nnuggets.com`, confirmed via `bench console`
      that `frappe.get_hooks("override_email_send")` resolves to
      `email_system.email_system.dispatch.dispatch_outgoing_mail` (no
      stale duplicate) and `disable_standard_email_footer` reads `1` on
      all three sites. Queued a real `frappe.sendmail()` through the full
      `QueueBuilder` path (not delivered — cleaned up before the scheduler
      could pick it up) and confirmed its rendered HTML has no "Sent via"
      text and an empty `default_mail_footer` slot. Ran `dispatch.
      _brand_message()` directly against a hand-built MIME message using
      this tenant's real data: correctly fell back to no logo (this
      tenant hasn't set `Company.company_logo` yet) and rendered the
      "Powered by Aetris" footer with a working image URL. Confirmed
      `https://rastec.nnuggets.com/assets/email_system/images/aetris.png`
      returns 200 (public, loadable by an external mail client). One
      cosmetic gap found and accepted, not fixed: `frappe.utils.get_url()`
      returns `http://` (not `https://`) when called with no active
      request context (true of `bench console`, not of a real signing
      invite triggered from a web request) — Traefik redirects http→https
      at the entrypoint (same known behavior already documented in
      `platform_console/provisioning.py`), so the image still loads; not
      re-verified from an actual live web-request-triggered send.
- [ ] Verify live: upload a business logo via the Settings page for a real
      tenant, confirm it renders in the portal and shows up in the next
      outgoing email with the correct (not-yet-confirmed-https) URL.
- [x] Fix (2026-09-05): Imran sent a real e-sign invite from
      `realestate.nnuggets.com` (tenant "4iTrading") and it rendered
      full-width, no rounded container, unlike the Welcome-to-Aetris
      template — screenshots in `ui-screenshot/`. Root cause: Frappe's
      `templates/emails/standard.html` only applies its 600px contained/
      rounded-card CSS when the sender passes `with_container=True`, which
      none of our senders do (and turning it on would pull Frappe's own
      masthead logo from Website Settings, which this platform points at
      Aetris for pre-auth pages — the wrong logo source for a tenant's
      customer-facing mail). Old `_inject_into_html` only prepended/
      appended header/footer divs around Frappe's own full-width,
      uncontained output. Replaced with `_extract_message_content` +
      `_rebuild_html`: pulls just the sender's message out of Frappe's
      `td.email-body-cell` (CSS already inlined by that point) and
      rebuilds the whole email from scratch — centered 480px column, logo
      centered above, message in a `#F9FAFB` rounded card, centered
      "Powered by Aetris" credit below — matching
      `platform_console`'s Welcome-to-Aetris template's exact CSS values.
      Verified live against the real "4iTrading" tenant/logo via
      `dispatch._brand_message()` in `bench console` on
      `realestate.nnuggets.com`: correct structure, correct logo URL.
      `rafaii/email-system` `1dc9ba3`, rebuilt + redeployed
      (`realestate-custom:v15.119.3`, no `clear-cache` needed — no hook
      registration changed, only the function body).
- [x] Fix (2026-09-05, same session): Imran sent another screenshot —
      the card looked narrow and its text (the signing URL, no spaces in
      its query string) overflowed past the card's right edge instead of
      wrapping inside it. Root cause: `<div style="max-width:600px">` is
      well known to be ignored by Outlook desktop's Word rendering
      engine (sizes divs to their content, not their CSS) — the card was
      never actually width-constrained by anything but luck of short
      content. Replaced with the standard email "fluid-hybrid" pattern:
      a `<table width="600">` (which old Outlook does respect) plus
      matching CSS `max-width`/`width:100%`, the same technique Frappe's
      own `templates/emails/standard.html` already uses. Also added
      `word-break:break-word;overflow-wrap:anywhere;` on the card cell so
      a long unbroken token wraps inside the column regardless. Widened
      480px → 600px (Frappe's own convention) since this template wraps
      arbitrary sender content, not two short welcome lines — the
      Welcome-to-Aetris email is untouched, still its own narrower width;
      only the visual language (colors/radius/padding/font) is shared,
      not a literal width. Verified live via `dispatch._brand_message()`
      against the same real 4iTrading lease/logo/URL from the screenshot.
      `rafaii/email-system` `0cbe99c`, rebuilt and redeployed (no
      `clear-cache` needed — function body only).
- [x] Configurable marketing link (2026-09-05): Imran — the footer's
      "Powered by [logo] Aetris" had a redundant text label (the logo
      image already has the wordmark baked in), and the destination
      should be the platform's own domain, configurable rather than
      hardcoded. Removed the text span; the logo is now the click target,
      wrapped in `<a href="{marketing_url}">`. `marketing_url` reads
      `frappe.conf.get("saas_marketing_url")` (falls back to
      `https://nnuggets.com`, today's test domain, if unset) — set via a
      new "Platform Settings" Single DocType in `platform_console`
      (`console.nnuggets.com`, Platform Admin/System Manager only), whose
      `on_update` runs `bench set-config -g` against the shared bench so
      every site's `frappe.conf` picks it up with zero per-tenant
      propagation (a platform-wide constant, not a per-tenant one — same
      reasoning as `BREVO_SENDER`). Verified live: saved
      `https://nnuggets.com` via `bench console` on `console.nnuggets.com`,
      confirmed it landed in the shared `common_site_config.json`, and
      confirmed `realestate.nnuggets.com` (a different site, same bench)
      reads it back correctly with no restart. `rafaii/email-system`
      `caf95f7` + `rafaii/platform-console` `88b8b24`, deployed
      (`bench migrate` on `console.nnuggets.com` for the new DocType).

## Acceptance Criteria

- [x] An e-sign OTP or signing-request email sent from a tenant site shows
      no "Sent via ERPNext" text/link anywhere. Verified via a real
      queued `frappe.sendmail()` on `rastec.nnuggets.com` — no "Sent via"
      text, empty `default_mail_footer` slot.
- [x] The same email shows the tenant's own logo (if set) and a "Powered
      by Aetris" line with the Aetris logo in the footer. Verified via
      `dispatch._brand_message()` against real site data (no logo shown —
      none set yet on this tenant — footer credit present and correct).
- [ ] Settings page "Portal branding" card lets a user with Company write
      permission upload a new logo, and it's immediately reflected both in
      the portal and in the next outgoing email.
- [ ] A tenant with their own SMTP Email Account configured sees the exact
      same branded template as a tenant on Aetris-managed sending.

## Related

- Domain index: `vault/email-system/email-system.md`
- ADR: `vault/decisions/0021-email-system-outgoing-mail-branding.md`
- Superseded code: `real_estate_os/brevo_mail.py`
