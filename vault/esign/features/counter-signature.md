---
status: done
owner: developer-1
domain: esign
created: 2026-08-16
updated: 2026-08-17
related_adr: ["0005-counter-signature-pdc"]
---

# Two-Party Counter-Signature

## Summary

Lease agreements are signed by two parties in sequence: the customer
(`signing_order=1`) then the business signatory (`signing_order=2`, configured
in a `Signature Settings` singleton). The document finalizes only when all
signers have signed. The audit log gains `Created` and `Counter Signed` events.

## Requirements

- `Signature Settings` singleton: `esign_mode` + `business_signatory` (User).
- `send_lease_for_signature` adds customer + business signatory (order 2).
- Sending requires `business_signatory`; clear error if unset / no email.
- Customer signs first; business signatory is emailed to counter-sign next.
- Counter-signature email states who has already signed and asks to complete.
- Finalize only when all signers have signed.
- Audit events: `Created`, `Sent`, `Viewed`, `OTP Verified`, `Consent Given`,
  `Signed` (customer), `Counter Signed` (business), `Completed`.

## Design

- `Signature Settings` (is_single) at `e_sign/doctype/signature_settings/`.
- `send_for_signature` emails the first (order-1) signer; `capture_signature`
  notifies the next pending signer after each signature.
- `capture_signature` records `Counter Signed` when `signing_order > 1`.
- `send_lease_for_signature` records a `Created` event on ESD insert.
- `_get_signer` matches email+token across ALL signers (fixes two signers
  sharing one email — e.g. owner testing with the customer's address).

## Implementation Plan

- [x] Add `Signature Settings` singleton DocType (esign_mode, business_signatory)
- [x] `send_lease_for_signature`: require Settings.business_signatory, add as order-2 signer, record `Created` audit event
- [x] `send_for_signature`: email only the order-1 signer
- [x] `capture_signature`: after a signer signs, email the next pending signer; record `Counter Signed` for order>1
- [x] Verify finalize runs only when all signers signed
- [x] Counter-signature email wording: "{signer(s)} has already signed {doc}. Please review and sign to complete the contract"
- [x] `_get_signer` loops all signers matching email+token (same-email fix)

## Acceptance Criteria

- [x] Sending creates 2 signers (customer order 1, signatory order 2)
- [x] Business signatory emailed only after customer signs
- [x] Finalize only when both signed
- [x] Audit log: Created → Sent → (customer) OTP/Consent/Signed → Counter Signed → Completed
- [x] Signatory with the same email as customer can still sign via their own token

## Related

- Domain index: `vault/esign/esign.md`
- ADR: `vault/decisions/0005-counter-signature-pdc.md`
- Feature: `vault/payments-accounting/features/pdc-processing.md`
