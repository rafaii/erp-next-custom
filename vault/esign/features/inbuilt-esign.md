---
status: done
owner: developer-1
domain: esign
created: 2026-08-16
updated: 2026-09-05
related_adr: ["0002-esign-approach"]
---

# In-Built E-Signature

## Summary

Native e-signature for lease agreements: document hash, OTP identity check,
explicit electronic consent, canvas signature capture, immutable audit log,
PDF stamping + Certificate of Completion.

## Requirements

- DocTypes: `E-Sign Document`, `E-Sign Signer`, `E-Sign Field`, `E-Sign Audit Log`.
- SHA-256 hash of finalized PDF stored before send.
- Consent screen ("I consent to sign electronically" + disclosure).
- OTP via email/SMS; record `otp_verified` + timestamp.
- signature_pad.js canvas capture (PNG/base64) + typed-name fallback.
- Append-only audit log of every event.
- Stamp PDF (pypdf), append Certificate of Completion, re-hash, lock `docstatus=1`.

## Design

- DocType JSONs under `real_estate_os/real_estate_os/doctype/`.
- Whitelisted methods in `real_estate_os/esign/`:
  `send_for_signature`, `verify_otp`, `capture_signature`, `finalize_document`.
- Print Format for the signing page + Certificate of Completion template.

## Implementation Plan

- [x] Create 4 E-Sign DocTypes + fields
- [x] Implement SHA-256 hashing (document hash on insert; PDF hash on finalize deferred)
- [x] Implement consent screen (web view) — `www/esign` page
- [x] Implement OTP verify (email gateway; SMS deferred)
- [x] Implement signature capture (jSignature canvas + typed-name fallback)
- [x] Implement PDF stamp + Certificate of Completion (pypdf merge)
- [x] Lock via docstatus=1 + audit log workflow
- [x] Invite + OTP emails sent immediately (`now=True`) so signers get the link/code instantly
- [x] Fix (2026-09-05): Imran — "I downloaded a signed file and did
      `shasum -a 256 <file>` ... the hash does not match what is there on
      the signed doc." `final_hash` was computed from the pre-stamp PDF
      plus a manually concatenated string of signer metadata — not a
      hash of any file that exists anywhere, so it could never match a
      real download. Per Imran's direction, contract and Certificate of
      Completion are now two separate files instead of one merged PDF:
      `signed_pdf` (contract + stamped fields) is hashed directly —
      `final_hash` = `sha256(signed_pdf bytes)`, so it now matches a
      signer's own `shasum`. `certificate_pdf` (new Attach field) is
      generated *after*, printing that real hash as a reference to the
      already-finalized contract file rather than attempting to hash
      itself (impossible — a document can't print its own hash and have
      that hash cover the whole document). Added `view_certificate_pdf`
      (guest) + a certificate download link on the signing page;
      `real_estate_os` gained `get_signed_contract_hash`/
      `get_signed_contract_certificate` and a hash-plus-copy-button +
      certificate download on the Contract page. Verified live via a
      disposable test E-Sign Document (`bench console`, cleaned up
      after): `final_hash` matched `sha256` of the actual downloaded
      `signed_pdf` bytes exactly, and the certificate's extracted PDF
      text genuinely contains that hash (checked via `pypdf.extract_
      text()`, not a raw byte search — PDF content streams are
      typically compressed, so a naive substring check on raw bytes
      first came back a false negative). Does **not** retroactively fix
      already-finalized E-Sign Documents — their `signed_pdf` still has
      the old merged format and their `final_hash` was computed the old,
      broken way; not something we can rewrite after the fact for an
      already-issued signed document. Deployed live on both tenant sites
      (`bench migrate` for the new field, image rebuilt, containers
      recreated).
- [x] Completion email (2026-09-05): Imran — "Once a contract is signed
      by both parties, the signed contract and certificate should be
      emailed to the customer or atleast a link to download the same if
      not as an attachment." `_finalize()` now calls
      `_send_completion_email(doc)`, sending every signer an HTML email
      (`<h2>Your signed contract</h2>`, salutation with their name,
      congratulations, a recommendation to download both files, then the
      two download links, signed off "Team {Company}") once the document
      is fully signed. Links (reusing the existing
      `view_signed_pdf`/`view_certificate_pdf` guest endpoints, each
      signer's own token), not attachments — per Imran's explicit
      fallback and to avoid transport attachment-size limits. Best-effort
      per signer (`frappe.log_error`, not raised) so a notification
      failure can't undo a signing that already completed. Verified live
      via a disposable test E-Sign Document: the queued email's decoded
      HTML contains the exact intended structure and a correctly
      URL-encoded download link; cleaned up after (no test email sent to
      a real inbox — used a `.invalid` RFC 2606 test address).
## Acceptance Criteria

- [x] End-to-end: send → OTP → consent → sign → stamped PDF + certificate
- [x] Hash breaks on any post-finalize tamper (final_hash + docstatus=1 lock)
- [x] Audit log records every event with timestamp/IP/user-agent
- [x] Emails (invite, counter-sign invite, OTP) sent immediately, not queued
- [x] `final_hash` matches `sha256` of the actual downloaded signed
      contract PDF (2026-09-05 fix, verified live)
- [x] OTP resend cooldown is scoped per signer, not shared across signers
      on the same document — confirmed by reading `request_otp`'s
      `cooldown_key` (`doc.name:signer.email`) and by live evidence: on
      the real `ESD-00467` document (2026-09-05), Rajesh's and Imran's
      OTP sends were 1m23s apart on different email addresses with no
      collision. Imran's report ("clicked send code again and got the
      cooldown error") was the same signer re-clicking within their own
      60s window, not a cross-signer block — no code bug found here, just
      a missing UI cue (fixed below).
      "Send code" now shows a live "Resend in Ns" countdown
      (`OTP_RESEND_COOLDOWN_SECONDS` supplied from the server, not
      duplicated as a JS constant) instead of surfacing the raw
      `ValidationError` on a second click.

## Related

- Domain index: `vault/esign/esign.md`
- Feature: `provider-abstraction.md`; `compliance-security/features/esign-audit-trail.md`
