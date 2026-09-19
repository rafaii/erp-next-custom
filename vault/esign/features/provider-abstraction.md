---
status: in-progress
owner: developer-1
domain: esign
created: 2026-08-16
updated: 2026-09-05
related_adr: ["0002-esign-approach", "0017-platform-modularization"]
---

# E-Sign Provider Abstraction

## Summary

A `Signature Settings` singleton lets each tenant choose the e-sign mode:
In-Built, DocuSign, or ZohoSign — with the same downstream contract (send,
status, callback).

**2026-08-27 redesign (ADR-0017)**: verified live that `esign_mode` is
never actually read anywhere in the code — the field has existed since
`0002` with zero effect. This feature closes that gap for real, and
changes *where* the code lives: per ADR-0017, e-sign becomes its own
external, reusable module (installed as a required dependency of Real
Estate OS, and any future OS that needs e-sign), not adapter classes
living inside `real_estate_os`. The original "Design" below (adapters
under `real_estate_os/esign/providers/`) is superseded by the module-app
approach; kept struck through for history rather than deleted.

## Requirements

- `Signature Settings` DocType: `esign_mode` (In-Built / DocuSign / ZohoSign)
  + provider credentials — this is the **Business Config page** control
  (ADR-0017 §4): the tenant's own choice of *active* provider among
  whichever e-sign module(s) are installed on their site.
- Common `ESignProvider` interface (category-level, not per-vendor):
  `send_for_signature`, `get_status`, `webhook handler` — documented once,
  implemented by each concrete provider app.
- Webhook endpoint (whitelisted) to receive third-party completion callbacks.
- Dispatch: the OS app resolves the active provider via
  `frappe.get_hooks("esign_provider")` (returns installed provider apps'
  dotted paths) filtered to the one `Signature Settings.esign_mode` names
  — never a direct `import zoho_esign` in `real_estate_os`.

## Design

- ~~Python adapter classes under `real_estate_os/esign/providers/`~~
  (superseded by ADR-0017 — see below).
- **New**: extract the existing in-built e-sign implementation
  (`E-Sign Document`/`Signer`/`Field`/`Audit Log`, the signing workflow,
  PDF stamping) out of `real_estate_os` into its own module app (e.g.
  `inbuilt_esign`), implementing the `ESignProvider` contract. This is a
  real extraction of already-working, `done` code (`inbuilt-esign.md`,
  `counter-signature.md`), not a rewrite.
- **New**: a second module app, `zoho_esign` (or similar), implementing
  the same `ESignProvider` contract against Zoho Sign's real API — the
  first genuinely new provider, proving the contract works for more than
  one implementation.
- `real_estate_os` declares both as required-by-OS modules (ADR-0016:
  provisioning installs an OS's required modules automatically).
  `Lease Agreement.esign_provider` continues to default from
  `Signature Settings.esign_mode`, now actually meaningful.
- Third-party credentials stored in site config / encrypted, never in fixtures.

## Implementation Plan

- [x] Document the `ESignProvider` interface contract (function names +
      signatures) once, in the vault — `esign-provider-contract.md`, PR #47.
      Derived from what `e_sign/workflow.py` actually implements (only
      `send_for_signature` has a real call site today; `get_status` and
      the webhook handler are specified for when a third-party provider
      exists to need them), not an invented spec.
- [x] Wire actual `esign_mode` dispatch in `real_estate_os` — PR #47,
      2026-08-29. New `e_sign/dispatch.py` resolves `esign_mode` via
      `frappe.get_hooks("esign_provider")`; both prior direct-call sites
      (the portal's send action, and `Lease Agreement.
      resend_if_terms_changed_while_sent`'s auto-resend-on-edit) now go
      through it instead of importing `e_sign.workflow` directly. The
      in-built implementation registers itself in `hooks.py`, unchanged
      in place — this closes the "missing piece" from
      `2026-08-27-platform-modularization.md` without doing the riskier
      physical extraction below. Verified live: `frappe.get_hooks
      ("esign_provider")` correctly merges into
      `{"In-Built": ["real_estate_os.e_sign.workflow.
      send_lease_for_signature"]}`, and switching to `esign_mode =
      "ZohoSign"` (nothing registered) produces a clean `ValidationError`
      naming the mode, not a crash. **Not verified**: an actual send
      through the dispatch path — no lease was in a safe state to trigger
      a real signature-invite email without disturbing live customer
      data; the underlying `workflow.send_lease_for_signature` function
      body is unchanged (only its `esign_provider` field write moved to
      the dispatch layer), so this is a low-risk gap, not an open
      question.
- [x] Extract in-built e-sign into its own module app — done 2026-08-29,
      as its own isolated deploy with a full DB backup taken first (per
      Imran's explicit go-ahead: "Every module's audit trail sits in the
      respective sites" — confirmed correct live, see verification below).
      New repo: [rafaii/inbuilt-esign](https://github.com/rafaii/inbuilt-esign),
      cloned onto the VPS as a sibling to `real_estate_os` (not a
      submodule of the vault meta-repo — that repo isn't checked out on
      the VPS at all, only individual app repos are, so a plain subfolder
      there would never have been deployable via the existing `git pull`
      step). `real-estate` PR #48 moved `E-Sign Document`/`Signer`/
      `Field`/`Audit Log`, `workflow.py`, and the guest `/esign` pages;
      `Signature Settings` and `e_sign/dispatch.py` stayed in
      `real_estate_os` (OS-level per ADR-0017 §4).

      Two real coupling points were found and fixed mid-extraction, not
      anticipated in the original plan:
      - `add_activity` (a generic Comment-posting helper `real_estate_os`
        owns) — now a small local copy inside `inbuilt_esign`.
      - `_finalize()` hardcoded Lease Agreement-specific side effects
        (mark a unit occupied, trigger recurring invoicing) inline behind
        `if doc.reference_doctype == "Lease Agreement"` — replaced with
        `doc.run_method("on_esign_completed")`, a generic Frappe
        `doc_events` dispatch that works for any method name, not just
        built-in lifecycle events (confirmed by reading `Document.hook`'s
        source before relying on it). `real_estate_os` registers the
        Lease Agreement-specific reaction as a new `doc_events` hook;
        `inbuilt_esign` has no idea what a Lease Agreement is.
      - Also caught a third call site neither PR #47 nor the original
        plan accounted for: a Desk-side Client Script fixture (the "Send
        for E-Signature" button on the Lease Agreement form) called
        `e_sign.workflow` directly in a `frappe.call()` string — same
        bug PR #47 fixed for the portal, missed because it's JS embedded
        in a JSON fixture, not a Python import a `grep` for imports would
        catch.
      - New app's Frappe module named `Inbuilt Esign`, not `E-Sign` —
        deliberately avoids relying on Frappe's cross-app Module Def
        reassignment (untested, no rehearsal environment) in favor of a
        clean additive module; the stale `E-Sign` Module Def row (still
        owned by `real_estate_os` after its `modules.txt` dropped it) was
        deleted post-migrate once confirmed nothing referenced it.
      - Root-caused and fixed one deploy-time bug: `bench install-app`
        failed with `TypeError: expected str, bytes or os.PathLike
        object, not NoneType` — the new app's Frappe-module folder
        (`inbuilt_esign/inbuilt_esign/inbuilt_esign/`) was missing its
        own `__init__.py`, making it an implicit Python namespace package
        with no single `__file__` for `frappe.get_module()` to resolve.
        Fixed, redeployed, installed cleanly on retry.

      **Verified live, 2026-08-29** (in order): `install-app` +
      `migrate` completed with no errors; all four moved doctypes report
      `module: "Inbuilt Esign"`, `Signature Settings` reports `module:
      "Real Estate"`; the one real existing `E-Sign Document`
      (`ESD-00354`, tied to `LSE-2026-00328`) still loads with both
      signers and its audit log intact — proving the module reassignment
      didn't touch data, exactly as Imran said it wouldn't;
      `frappe.get_hooks("esign_provider")` resolves to
      `inbuilt_esign.inbuilt_esign.workflow.send_lease_for_signature` as
      a real, importable function; picking an unregistered mode still
      throws cleanly; `/esign` still returns 200; the main site and
      `/reports` still serve normally.
- [ ] Build the Zoho Sign module app (envelope creation + webhook) as the
      second real provider — blocked on Imran providing real Zoho Sign
      API credentials; not started.
- [x] Business Config page: `esign_mode` selector, wired to the dispatch
      — PR #64, 2026-08-31. `get_settings_data()` returns `esign.mode`
      and `esign.modes` (each flagged `active` via the same
      has-a-registered-provider check `get_integrations()` already does
      against `esign_provider`); new `set_esign_mode(mode)` validates
      against the DocType's real Select options and saves to the
      `Signature Settings` singleton, mirroring `set_business_signatory`.
      Settings page gained a matching selector with a warning banner
      when the saved mode has no registered provider.

      Permission gap caught before deploy (advisor review): `Signature
      Settings` write is System Manager-only (no `Custom DocPerm` rows)
      but the Settings page renders for every role — both this new
      selector and the pre-existing signatory selector would let a
      non-admin fill the form and hit a raw `PermissionError` on Save.
      Fixed by adding `esign.can_manage` (`frappe.has_permission`, no
      throw) and disabling both controls in the UI when false, with an
      explanatory note. Verified live: `can_manage` reads `True` for
      Administrator, `False` for Guest.

      Verified live via `bench console` before and after deploy:
      `esign.mode`/`modes`/`can_manage` round-trip correctly,
      `set_esign_mode("Nonsense")` rejects with `ValidationError`, a
      full save/restore cycle persists correctly. **Not click-tested in
      a browser** — a dropdown-plus-two-Save-buttons UI, the same shape
      of change that produced two browser-only bugs earlier in this
      session (the Accounts `?tab=` sync issues), so worth a
      click-through before relying on it.
- [ ] Add webhook route + status sync for the third-party path
- [x] Resend invite (2026-09-05): Imran — "there should be a way to
      resend [if] the customer has not received the email." Contract
      page's "Send for e-signature" button disappeared entirely once
      `esign_status` became "Sent", leaving no action at all if the
      original invite was lost/undelivered. Added a new, separate dict
      hook `esign_provider_resend` (parallel to `esign_provider`, not
      overloading it — a provider may have no concept of "resend") +
      dispatch's `resend_lease_invite`. In-built implementation
      (`workflow.resend_invite`) deliberately does **not** call
      `send_for_signature` again — that resets every signer to `Pending`
      and would silently undo an already-completed counter-signature
      step; instead re-emails the same link (same `signing_token`, no
      PDF/hash regeneration) to whichever signer `_next_pending_signer`
      returns, with a 60s cooldown mirroring the OTP resend cooldown.
      Contract page shows a "Resend invite" button whenever
      `esign_status === "Sent"` and there's no send error. `bun run
      build` clean (tsc + vite).

      **Local `apps/inbuilt_esign` was stale before this landed** —
      caught before pushing: `git push` rejected the fast-forward because
      origin/main had ~1300 lines of merged PR history (PDF templates,
      guided signing, merge-field preview, envelope watermark, etc.) this
      local checkout never pulled. Reset to `origin/main` and re-applied
      the change against the real current `workflow.py` rather than
      force-pushing the stale-context version. Worth remembering: this
      non-worktree `apps/inbuilt_esign` checkout drifts behind the
      worktree-based agent branches that actually merge PRs — `git fetch`
      + compare before editing, not just before pushing.

      Deployed live: `inbuilt-esign` `f22c417`, `real-estate` `ec512ab`,
      image rebuilt, containers recreated, cache cleared on both tenant
      sites (hooks.py changed — new `esign_provider_resend` entry).
      Verified live via `bench console` on both tenant sites: both hooks
      resolve correctly; a negative-path call
      (`resend_lease_invite("NONEXISTENT-LEASE-NAME")`) throws the
      expected `ValidationError`. Found a real live candidate matching
      Imran's exact complaint: `LSE-2026-00464` / `ESD-00467` (the
      4iTrading contract from the email-branding screenshots) is still
      `Sent`, both signers `Pending` — structurally confirmed
      `resend_lease_invite` would correctly resend to `rajesh@4itrading.com`
      (signing_order 1), but didn't actually trigger it (that sends a
      real email to a real customer) — offered to Imran to trigger for
      real on request.

## Acceptance Criteria

- [x] Switching `esign_mode` from the Business Config page changes the
      active provider without a code change or redeploy — PR #64. The
      Settings page's "E-sign provider" selector calls `set_esign_mode`,
      which the dispatch (`e_sign/dispatch.py`) already reads
      dynamically; verified live pre- and post-deploy via `bench
      console`. Gated to System Manager (`esign.can_manage`) since
      `Signature Settings` write isn't open to other roles today.
- [ ] Zoho Sign's callback updates `esign_status` correctly, matching the
      in-built flow's semantics — blocked on the Zoho module existing
- [x] `real_estate_os` contains no direct import of the Zoho module —
      dispatch only via hooks. Trivially true today (no Zoho module
      exists yet), but structurally guaranteed going forward: the two
      former direct-import call sites now resolve through
      `frappe.get_hooks("esign_provider")` instead.

## Related

- Domain index: `vault/esign/esign.md`
- Feature: `inbuilt-esign.md`
- Contract: `vault/esign/esign-provider-contract.md`
- Repo: [rafaii/inbuilt-esign](https://github.com/rafaii/inbuilt-esign) —
  the extracted app
- ADR: `vault/decisions/0017-platform-modularization.md`,
  `vault/decisions/0002-esign-approach.md`
- Finding: `vault/findings/2026-08-27-platform-modularization.md`
