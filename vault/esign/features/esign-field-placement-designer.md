---
status: in-progress
owner: developer-1
domain: esign
created: 2026-09-02
updated: 2026-09-03
related_adr: ["0019-esign-template-designer"]
---

# E-Sign Field-Placement Designer (Phase 2)

## Summary

Lets a business owner visually place Signature/Initial/Date/Text fields on their contract template,
and actually stamps signer input onto the generated contract PDF at those positions when signing
completes — the DocuSign-style capability this whole effort is aimed at. Depends on Phase 1
(`esign-template-management.md`) already being shipped and live.

## Requirements

- Field-placement designer opens against the *actual* generated PDF for the template (via the
  `render_template_preview` method from Phase 1), rendered client-side page-by-page with PDF.js —
  not the raw HTML/Jinja source, since wkhtmltopdf's real pagination doesn't reliably match an HTML
  editor's page breaks.
- Business owner drags a palette item (Signature/Initial/Date/Text) onto a rendered page, positions
  and resizes it, and assigns it a `role_label` from the template's own `signer_roles` list (so it's
  explicit which signer's box is which).
- Placed fields are stored as percentage-of-page coordinates (`x_pct`/`y_pct`/`width_pct`/
  `height_pct`), not absolute pixels, so they stay correct regardless of render resolution.
- At finalize time (after all signers have completed, per the existing counter-signature flow —
  ADR-0005, unchanged), each placed field is resolved to its signer (by `role_label` /
  `signing_order` match) and burned onto the actual contract PDF page at its position — signature
  image for `Signature`/`Initial`, text for `Date`/`Text`.
- The existing Certificate-of-Completion appendix page is unchanged and still appended — it remains
  the audit record; in-place stamping is the new "the contract itself is visibly signed" artifact,
  not a replacement for the certificate.
- Templates with no placed fields (or no template at all) continue to finalize exactly as they do
  today — appending only the certificate page, no in-place stamping attempted.

## Design

**`E-Sign Template Field`** (new child table on `E-Sign Template`, added in Phase 1's schema but
populated starting here): `role_label` (Data — binds to a `signer_roles` row), `field_type` (Select:
Signature/Initial/Date/Text), `page_no` (Int), `x_pct`/`y_pct`/`width_pct`/`height_pct` (Float),
`label` (Data, optional, designer-UI-only).

**Stamping** (`_finalize()` in `apps/inbuilt_esign/inbuilt_esign/inbuilt_esign/workflow.py`):
1. Skip entirely if `E-Sign Document.template` is unset or has no `E-Sign Template Field` rows —
   preserves today's exact appendix-only behavior.
2. For each field row: resolve `role_label` -> matching `E-Sign Signer` (by `signing_order`/role);
   pull `signature` (PNG) / `typed_name` / current date depending on `field_type`.
3. Use `pypdf` to read the target page's actual `mediabox` dimensions; convert `x_pct/y_pct/
   width_pct/height_pct` into absolute PDF points for that page.
4. Build an overlay page via `reportlab.pdfgen.canvas` at those coordinates, then
   `PdfWriter.merge_page()` it onto the target page — standard pypdf/reportlab overlay pattern, no
   new dependency (reportlab is already a transitive Frappe dependency).
5. Append the Certificate-of-Completion page exactly as before.

**Saving**: no dedicated whitelisted method — `fields` is saved through the same generic
`E-Sign Template` REST update path `signer_roles` already uses (`updateResource("E-Sign Template",
name, { fields })`), gated by `E-Sign Template`'s own DocPerms.

**Frontend** (`apps/real_estate_os/ui`): PDF.js-based designer screen, opened from the template
editor built in Phase 1. Palette of field types, drag/resize on canvas, role assignment dropdown
sourced from the template's `signer_roles`.

**Critical invariant to test explicitly**: the preview PDF rendered for the designer
(`render_template_preview`) and the PDF actually stamped at finalize time must produce identical
page dimensions for the same template/data shape — any drift misplaces stamped fields. Don't rely on
visual spot-checking alone for this; compare page dimensions programmatically as part of the
acceptance check.

## Implementation Plan

- [x] Add `E-Sign Template Field` child table to `E-Sign Template`.
- [x] Build the PDF.js field-placement designer UI — palette (Signature/Initial/Date/Text),
      click-to-place, drag-to-reposition, corner-handle resize, role assignment from
      `signer_roles`. Positions are authored directly as CSS percentages against each rendered
      page's container, which IS the `x_pct`/`y_pct`/`width_pct`/`height_pct` storage format — no
      separate pixel-to-percent conversion step needed.
- [x] **Simplified from the original plan**: no dedicated `save_template_fields` whitelisted
      method — `fields` is just another child table on `E-Sign Template`, saved through the same
      generic `updateEsignTemplate`/REST update path `signer_roles` already used. Less code, same
      permission model (`E-Sign Template`'s own DocPerms).
- [x] Implement the `_finalize()` stamping overlay (`_stamp_template_fields` /
      `_resolve_stamp_signer` / `_build_field_overlay` / `_signature_image_reader` in
      `workflow.py`) — role resolution via `signer_roles`' `signing_order` (falling back to the
      same customer/tenant heuristic `_resolve_template_signer` uses at send time) -> pypdf/
      reportlab overlay -> `merge_page` onto the target page, before the certificate append.
      **Found and fixed a wrong assumption from the original plan**: `reportlab` was NOT already a
      transitive Frappe dependency (verified live against the deployed backend — `pypdf` is,
      `reportlab` wasn't) — added explicitly to `inbuilt_esign`'s `pyproject.toml`; confirmed it
      installs cleanly in the real Docker build before deploying.
- [x] Deploy: merged `inbuilt-esign#4` + `real-estate#81` to `main`, VPS image rebuilt (verified
      `reportlab` importable in the fresh image before proceeding), containers recreated,
      `bench migrate` clean on both tenant sites (`realestate.nnuggets.com`, `rastec.nnuggets.com`)
      — `tabE-Sign Template Field` confirmed present on both — 2026-09-03.
- [x] Verified live (without a full guest-flow signing, which needs a human — see below): a direct
      Python round-trip on the deployed backend built a base PDF, called `_build_field_overlay` and
      `merge_page` exactly as `_finalize()` does, and confirmed via text extraction that BOTH the
      original page content and the stamped field text survive in the merged output. Also confirmed
      `_build_field_overlay` correctly returns `None` (skips stamping) for a signer with nothing to
      draw (no signature image, no typed name).
- [ ] **Not yet verified**: a full end-to-end test signing through the real guest flow (OTP,
      consent, signature capture) confirming the final signed PDF shows fields stamped in the
      correct visual position when opened — needs a human click-through, not reachable via SSH.
- [ ] **Not yet verified**: a human click-through of the designer UI itself (drag/resize/role
      assignment actually feel right, multi-page templates render/scroll correctly).
- [ ] Dropped the "page-dimension-consistency check between preview-render and finalize-render" as
      a separate acceptance item — both paths call the identical `_render_template_pdf`/
      `_render_context` code, so there is no code path where they could diverge (not a
      "should match" property to verify, but a structural guarantee).

## Acceptance Criteria

- [ ] Business owner can place Signature/Initial/Date/Text fields on the real rendered PDF and save
      them against the template. (Code deployed and the save path verified generically works via
      the same REST mechanism `signer_roles` already used successfully in Phase 1; the placement UI
      itself needs a human click-through to confirm the interaction feels right.)
- [ ] A fully counter-signed document produces a final PDF with each placed field visibly stamped
      in the correct position on the correct page, in addition to the unchanged certificate page.
      (Overlay+merge mechanism verified correct via direct Python round-trip; needs a real signing
      to confirm end-to-end through the guest flow.)
- [x] A template/document with no placed fields is unaffected — `_stamp_template_fields` returns
      immediately when `doc.template` is unset or has no `fields` rows, leaving `writer` untouched.

## Related

- Domain index: `vault/esign/esign.md`
- ADR: `vault/decisions/0019-esign-template-designer.md`
- Phase 1 (prerequisite): `vault/esign/features/esign-template-management.md`
