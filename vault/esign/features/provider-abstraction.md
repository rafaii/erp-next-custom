---
status: planned
owner: developer-1
domain: esign
created: 2026-08-16
updated: 2026-08-16
related_adr: ["0002-esign-approach"]
---

# E-Sign Provider Abstraction

## Summary

A `Signature Settings` singleton lets each tenant choose the e-sign mode:
In-Built, DocuSign, or ZohoSign — with the same downstream contract (send,
status, callback).

## Requirements

- `Signature Settings` DocType: `esign_mode` (In-Built / DocuSign / ZohoSign)
  + provider credentials.
- Common provider interface: `send_for_signature`, `get_status`, `webhook handler`.
- Webhook endpoint (whitelisted) to receive third-party completion callbacks.

## Design

- Python adapter classes under `real_estate_os/esign/providers/`
  (`base.py`, `inbuilt.py`, `docusign.py`, `zohosign.py`).
- `Lease Agreement.esign_provider` defaults from `Signature Settings`.
- Third-party credentials stored in site config / encrypted, never in fixtures.

## Implementation Plan

- [ ] Define provider interface (base class)
- [ ] Implement In-Built adapter (delegates to `inbuilt-esign.md`)
- [ ] Implement DocuSign adapter (envelope + webhook)
- [ ] Implement ZohoSign adapter
- [ ] Add webhook route + status sync

## Acceptance Criteria

- [ ] Switching `esign_mode` changes active provider without code changes
- [ ] DocuSign/ZohoSign callback updates `esign_status` correctly

## Related

- Domain index: `vault/esign/esign.md`
- Feature: `inbuilt-esign.md`
