# ESignProvider Contract

Category-level interface every e-sign provider module implements, so
`real_estate_os` can dispatch to whichever one `Signature Settings.
esign_mode` names without ever importing a specific vendor's code. Written
against the in-built implementation as it actually exists today — since
2026-08-29, its own app,
[rafaii/inbuilt-esign](https://github.com/rafaii/inbuilt-esign)
(`inbuilt_esign/inbuilt_esign/workflow.py`), extracted from
`real_estate_os` per ADR-0017 — this describes working code, it isn't a
spec for a rewrite. See `vault/decisions/0017-platform-modularization.md`
and `features/provider-abstraction.md`.

## Registration

A provider registers itself in **its own app's** `hooks.py` — not the
consuming OS app's:

```python
# inbuilt_esign/hooks.py
esign_provider = {
    "In-Built": "inbuilt_esign.inbuilt_esign.workflow.send_lease_for_signature",
}
```

`frappe.get_hooks("esign_provider")` merges this across every installed
app (Frappe's own dict-hook merge — confirmed by reading `frappe.
append_hook`'s source rather than assumed: a dict-valued hook becomes a
dict of *lists*, one list per key, even for a single value declared by one
app). Dispatch (`real_estate_os/e_sign/dispatch.py`, which stays in the OS
app along with the `Signature Settings` Business Config singleton — see
ADR-0017 §4) reads `Signature Settings.esign_mode`, looks up that key, and
calls the **last** registered path for it — a provider can override an
earlier one by declaring the same key, matching how Frappe resolves other
dict hooks (e.g. `doc_events`).

The dotted path is a plain function, not a method on some required base
class — Python doesn't enforce interface naming across a hook-registered
callable. What's required is the *call signature and behavior* described
below, not a literal function name every provider must share.

## Functions

### `send_for_signature(lease_name) -> str`

Starts the signing process for a `Lease Agreement` already carrying its
final Rent Schedule (see `send_lease_for_signature`'s own guard against
sending without one). Returns some provider-meaningful reference — the
in-built implementation returns the created `E-Sign Document` name.

What "starts" means is provider-specific:
- **In-built**: creates an `E-Sign Document` + `Signer` rows from the
  lease's Landlord/Tenant, emails the first pending signer an OTP-gated
  link, sets `Lease Agreement.esign_status = "Sent"`.
- **Third-party (DocuSign/Zoho, not yet implemented)**: would call the
  vendor's API to create an envelope from the rendered lease PDF, store
  the vendor's envelope ID somewhere on the Lease Agreement or a linked
  record, and set the same `esign_status = "Sent"` convention.

Every provider is expected to converge on the same `Lease Agreement.
esign_status` values (`Draft`/`Sent`/`Signed`/`Declined`/`Cancelled` — see
`lease-lifecycle-state.md`) so the rest of the app (portal panels,
`finalize_manual_lease`, invoicing gating) never needs to know which
provider is active.

### `resend(lease_name) -> str` (optional)

Re-notifies whichever signer is currently pending — the Contract page's
"Resend invite" button, for when a customer says the original email never
arrived. Registered as a **separate** dict hook, `esign_provider_resend`
(not overloading `esign_provider`'s single path), since a provider may
have no concept of this (a third-party vendor's own hosted page might
offer its own resend action, with nothing for this app to call). Dispatch
(`real_estate_os/e_sign/dispatch.py::resend_lease_invite`) throws a clear
"not registered for mode X" error if a provider hasn't implemented it,
rather than silently doing nothing.

**In-built**: `inbuilt_esign.inbuilt_esign.workflow.resend_lease_invite` —
re-emails the same signing link (same `signing_token`, no PDF/hash
regeneration) to whichever signer `_next_pending_signer` returns.
Deliberately does **not** call `send_for_signature` again — that resets
every signer back to `Pending` and clears `otp_verified`/`consented`,
which would silently undo an already-completed counter-signature step if
one party had already signed. A 60-second cooldown (mirroring the OTP
resend cooldown) guards against repeated clicks.

### `get_status(reference) -> str`

Not yet called by any dispatch path — the in-built flow doesn't need
polling since every step (OTP verify, consent, signature capture) already
writes `esign_status` directly as it happens. Exists in the contract
because a third-party provider's completion is asynchronous and a
business owner or a scheduled job may need to actively check rather than
wait on a webhook. **Not implemented for In-Built** (would just be
`frappe.db.get_value("Lease Agreement", lease_name, "esign_status")` — no
real polling to do), and not yet implemented for any third-party provider
either, since none exists yet.

### Webhook handler

A whitelisted, `allow_guest=True` endpoint a third-party provider's own
servers call back on completion/status-change. **Not applicable to
In-Built** — its entire flow runs through this app's own guest-accessible
pages (`request_otp`/`verify_otp`/`give_consent`/`capture_signature`/
`finalize_document` in `workflow.py`), which are themselves the in-built
provider's "webhook" in the sense that they're the async-completion entry
points, just invoked by a human clicking a link rather than a vendor's
server calling an API. A real third-party provider registers its own
route (e.g. `zoho_esign/www/webhook.py`) and updates `esign_status` the
same way `_finalize()` does today.

## Reacting to completion: `doc.run_method("on_esign_completed")`

A provider's `E-Sign Document` (or equivalent) is generic — it has no idea
what a `Lease Agreement` is, or what a consuming OS app wants to happen
once every signer has signed. `inbuilt_esign`'s `_finalize()` fires
`frappe.get_doc(reference_doctype, reference_name).run_method
("on_esign_completed")` on the referenced document itself once it's fully
signed, rather than hardcoding OS-specific side effects inline (which is
exactly what the original in-built implementation did before the
extraction — `if doc.reference_doctype == "Lease Agreement": mark_unit_
occupied(...); trigger_recurring_invoicing(...)`, hardcoded in the e-sign
code, the coupling this contract exists to prevent).

`Document.run_method` dispatches through `doc_events` hooks for *any*
method name, not just Frappe's built-in lifecycle events (confirmed by
reading `Document.hook`'s source, not assumed) — so a consuming app
reacts by registering its own `doc_events` entry:

```python
# real_estate_os/hooks.py
doc_events = {
    "Lease Agreement": {
        "on_esign_completed": "real_estate_os.real_estate.doctype.lease_agreement.lease_agreement.on_esign_completed",
    },
}
```

Any provider implementing this contract should fire the same event on the
reference document at the same point (fully signed, before returning from
`send_for_signature`'s async continuation) — it costs a provider nothing
to fire on a doctype with no handler registered (a no-op), but a consuming
app that expects it and doesn't get it will silently miss its own
post-signature logic.

## What's deliberately NOT part of this contract

Everything else in `workflow.py` — OTP session validity, consent capture,
signature-image handling, PDF stamping, certificate-of-completion
generation, the audit log (`_record_audit`) — is **in-built-specific**
implementation detail, not a category interface. A DocuSign or Zoho
provider doesn't reimplement OTP verification; the vendor's own hosted
signing experience replaces all of it. Only `esign_status` convergence and
the three functions above are the actual contract surface.

## Related

- `vault/esign/features/provider-abstraction.md`
- `vault/decisions/0017-platform-modularization.md`
- `vault/findings/2026-08-27-platform-modularization.md`
