---
status: done
owner: developer-1
domain: esign
created: 2026-08-16
updated: 2026-08-17
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

## Acceptance Criteria

- [x] End-to-end: send → OTP → consent → sign → stamped PDF + certificate
- [x] Hash breaks on any post-finalize tamper (final_hash + docstatus=1 lock)
- [x] Audit log records every event with timestamp/IP/user-agent
- [x] Emails (invite, counter-sign invite, OTP) sent immediately, not queued

## Related

- Domain index: `vault/esign/esign.md`
- Feature: `provider-abstraction.md`; `compliance-security/features/esign-audit-trail.md`
