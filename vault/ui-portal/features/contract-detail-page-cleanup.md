---
status: done
owner: developer-1
domain: ui-portal
created: 2026-08-27
updated: 2026-08-27
related_adr: []
---

# Contract Detail Page Cleanup

## Summary

Three things Imran found looking at a signed contract's detail page
(`/contracts/LSE-2026-00328`): "Payment schedule" and "Post-Dated Cheques"
looked like two sections showing the same thing once signed; the
"Signing" section's name was unclear; and the "E-Sign Document" linked
section's rows 404'd, with unclear value even if they worked.

## Requirements

- No redundant/empty-looking section duplicating the real PDC rows.
- Clearer section naming.
- No dead links; no section without clear business value.

## Design

- `ui/src/pages/Dashboard.tsx::PaymentSchedulePanel`: once
  `alreadySent (Sent or Signed) && hasPdcs`, the panel now renders nothing.
  Its only job in that state was a static "this is locked" message — no
  Generate/Regenerate button (already blocked), no rows (those live in the
  separate "Post-Dated Cheques" linked-records section, which already
  shows the real schedule). Deliberately **kept** for Draft leases — its
  Generate/Regenerate button there is the only way to create a schedule at
  all (see `pdc-schedule-generation-and-reconciliation.md`'s two-step
  flow); removing it unconditionally would have broken lease creation,
  not just decluttered a signed contract's page.
- `LeaseSigningPanel`'s header renamed "Signing" → "Contract".
- `hooks.py::portal_detail_links`: removed the "E-Sign Document" entry
  from Lease Agreement's links. Root cause of the 404: `E-Sign Document`
  was never added to `DOCTYPE_SLUGS`, so `docPath("E-Sign Document", ...)`
  fell back to a slug (`e-sign-document`) with no matching route, and
  `SLUG_TO_DOCTYPE` lookup failed silently. Rather than wire up the
  missing route, removed the section entirely: it's the raw
  signing-audit-trail record (signers, hashes, PDF, dynamic
  reference_doctype/reference_name), not something a business owner
  needs to browse into — the Contract panel already surfaces what
  matters from it (esign_status, view/cancel actions, "view signed
  contract" link).

## Acceptance Criteria

- [x] `tsc -b && vite build` clean; hooks.py syntax valid
- [x] Deployed + `bench --site ... clear-cache` (hooks.py change only, no
      schema change)
- [x] Verified live against the actual lease from the report
      (`LSE-2026-00328`): `get_linked_records("Lease Agreement", ...)`
      now returns exactly `["Post-Dated Cheques", "Security Deposit"]` —
      "E-Sign Document" is gone.
- [ ] Not click-tested in an actual browser — no browser tool available
      in this environment.

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `contracts-page-fixes.md`, `pdc-schedule-generation-and-reconciliation.md`
