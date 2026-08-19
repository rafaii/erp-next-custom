---
status: done
owner: developer-1
domain: compliance-security
created: 2026-08-16
updated: 2026-08-17
related_adr: ["0002-esign-approach"]
---

# E-Sign Audit Trail & Legal Defensibility

## Summary

Immutable, append-only event log that satisfies ESIGN Act / UETA (US) and
PIPEDA-adjacent provincial e-signature laws (Canada): intent, consent,
association, retention. Full trail surfaced in the signing portal and in the
Certificate of Completion appended to the signed PDF.

## Requirements

- `E-Sign Audit Log` DocType: immutable after insert (no edit permission).
- Events: sent, viewed, consent, OTP verified, each field signed, completed.
- Records timestamp, IP, user-agent per event.
- Certificate of Completion appended to signed PDF (hash + signer + timestamps).
- Signing portal shows live audit trail (timestamp / event / details / IP).
- Signing portal shows each party's signature + verification evidence.
- `Viewed` event persists immediately when the PDF is streamed (GET request).

## Design

- Audit Log DocType with `create`-only permission (no write/delete).
- Every workflow step writes an entry.
- `view_document_pdf` commits the `Viewed` audit event (Frappe GET requests
  roll back uncommitted writes otherwise).
- `_certificate_html` renders the full audit log as a structured table.
- `/esign` page (`index.py` + `index.html`) renders signers/signatures card
  and audit trail card server-side.

## Implementation Plan

- [x] Audit Log DocType + immutable permissions (create+read only; no write/delete)
- [x] Instrument every e-sign event (Sent, Viewed, Consent, OTP, Signed, Completed)
- [x] Certificate of Completion generation (appended to signed PDF)
- [x] Certificate includes structured Audit Trail table (timestamp, event, details, IP) + signer evidence table
- [x] Signing portal displays Signing Parties & Signatures card + full Audit Trail table
- [x] `view_document_pdf` commits `Viewed` audit event so it survives the GET request

## Acceptance Criteria

- [x] No post-insert edit/delete possible
- [x] Full event trail with IP/timestamp/user-agent
- [x] Signed PDF certificate shows the complete audit trail
- [x] Partial-flow (e.g. customer signed, owner pending) link shows customer signature + events so far

## Related

- Domain index: `vault/compliance-security/compliance-security.md`
- Feature: `esign/features/inbuilt-esign.md`
