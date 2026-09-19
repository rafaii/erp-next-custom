---
status: in-progress
owner: developer-1
domain: esign
created: 2026-09-03
updated: 2026-09-03
related_adr: ["0020-esign-pdf-upload-and-guided-signing"]
---

# E-Sign PDF Template Upload (Phase 3)

## Summary

Lets a business owner upload a blank PDF contract instead of authoring the body as Jinja/HTML, and
use the existing Phase 2 field-placement designer to place data merge fields (tenant name, unit,
rent amount, etc.) directly on the uploaded PDF — no HTML/Jinja knowledge required. Depends on
Phase 1 (`esign-template-management.md`) and Phase 2 (`esign-field-placement-designer.md`) already
being live, which they are.

## Requirements

- When creating/editing an `E-Sign Template`, the business owner picks a source: "Design in HTML"
  (unchanged Phase 1 flow) or "Upload a PDF" (new).
- For "Upload a PDF": a file upload control replaces the body textarea and merge-field chip picker;
  the uploaded PDF becomes the template's base document.
- The field-placement designer (already built, Phase 2) gains a 5th palette option: "Merge Field."
  Placing one opens the same field picker `get_mergeable_fields` already powers (grouped by source
  doctype — Lease Agreement / Customer / Unit / Building, per the Phase 1 linked-field work), and
  binds the placed box to that field's token instead of a signer role.
- Document generation for a PDF-sourced template stamps every placed Merge Field's resolved value
  onto the uploaded PDF once, at generation time (before `document_hash` is computed) — the same
  moment HTML-sourced templates run their Jinja substitution. The generated document is
  content-final and hashed before any signer ever sees it, preserving ADR-0002's tamper-evidence
  guarantee.
- Everything downstream of generation (Signature/Initial/Date/Text field placement, finalize-time
  signer stamping, the PDF.js designer) works identically regardless of source — there is just a
  `document_pdf` by that point, HTML-sourced or PDF-sourced makes no difference.
- An HTML-sourced template's behavior is completely unchanged (regression requirement).
- V1 scope: `Lease Agreement` only, consistent with Phase 1/2.

## Design

**`E-Sign Template`** (existing doctype, modified): add `source_type` (Select: `HTML`\n`PDF`,
default `HTML`) and `pdf_file` (Attach). `body` stays required/relevant only for `source_type ==
"HTML"`.

**`E-Sign Template Field`** (existing child table, modified): add `Merge Field` to the `field_type`
Select options (alongside Signature/Initial/Date/Text), and a new `merge_token` (Data) field —
populated instead of `role_label` when `field_type == "Merge Field"`. Holds the same dotted-path
tokens `get_mergeable_fields`/`_render_context` already produce/resolve (e.g.
`links.customer.customer_name`, `links.unit.building.address`).

**Rendering** (`apps/inbuilt_esign/inbuilt_esign/inbuilt_esign/workflow.py`):
- `_generate_document_pdf`/`_render_template_pdf` branch on `template.source_type`:
  - `HTML` (default): unchanged — `frappe.render_template(template.body, _render_context(ref_doc))`
    → `get_pdf(html)`.
  - `PDF`: base PDF = `template.pdf_file`'s bytes. Then a new `_stamp_merge_fields(template,
    ref_doc, writer)` pass: for each `field_type == "Merge Field"` row, resolve `merge_token`
    against `_render_context(ref_doc)` (walk the dotted path — `doc.foo` or
    `links.bar.baz` — against the same dict `_render_context` already builds) and stamp the
    resolved string onto the target page, reusing a generalized version of `_build_field_overlay`
    (extended to accept plain text + position, not only a signer object with a `field_type`
    switch — the Merge Field case is simpler: just draw the resolved string, no image/signature
    branch needed).
- This stamping happens once, synchronously, as part of generating `document_pdf` — before hashing,
  before any signer interaction. It is NOT part of `_finalize()`/`_stamp_template_fields` (that
  stays exactly as-is, for Signature/Initial/Date/Text fields only, at signing-completion time).

**Frontend** (`apps/real_estate_os/ui`):
- `EsignTemplateEditorDialog`: add a source-type toggle (HTML/PDF) near the top; PDF mode shows a
  file upload control (reusing the existing `uploadFile` helper already used elsewhere in
  Dashboard.tsx) instead of the body textarea + merge-field chip picker.
- `EsignFieldPlacementDialog`: add a 5th palette button, "Merge Field." Placing one opens the
  existing grouped field picker (same data `get_mergeable_fields` already returns) instead of the
  role-label dropdown used for Signature/Initial; selecting a field writes `merge_token` on that
  `EsignTemplateField` row instead of `role_label`.

## Implementation Plan

- [x] Add `source_type`/`pdf_file` fields to `E-Sign Template`.
- [x] Add `Merge Field` option + `merge_token` field to `E-Sign Template Field`.
- [x] Generalized `_build_field_overlay`: extracted `_field_geometry` (percentage-to-points
      conversion) as a shared helper, and added `_build_text_overlay` as the merge-field-specific
      sibling (draws plain text, no signer/field_type branching needed).
- [x] Implemented `_stamp_merge_fields` + `_resolve_merge_token` (walks a dotted path like
      `links.unit.building.address` against `_render_context`'s `{"doc": ..., "links": ...}` dict
      directly, not via Jinja) and wired into `_render_template_pdf` for `source_type == "PDF"`.
- [x] Built the source-type toggle + PDF upload UI in `EsignTemplateEditorDialog`. **Found a
      sequencing issue while implementing**: a file can only attach to an already-saved doc, and
      the backend rejects activating a PDF template with no file — so creating a new active
      PDF-sourced template now saves inactive first, uploads, then activates as a follow-up call.
- [x] Added the "Merge Field" palette option (distinct emerald color from signer-bound fields) +
      grouped field picker to `EsignFieldPlacementDialog` — reused the exact same PDF.js rendering
      and `get_mergeable_fields` data already built for HTML templates' merge-field chips.
- [x] Deploy: merged `inbuilt-esign#5` + `real-estate#83` to `main`, VPS image rebuilt, containers
      recreated, `bench migrate` clean on both tenant sites (`source_type`/`pdf_file` columns
      confirmed present) — 2026-09-03.
- [x] Verified live via direct Python round-trip (no bench needed for this piece): `_resolve_merge_token`
      correctly walks both `doc.*` (attribute access) and `links.*.*` (nested dict, including a
      2-hop path) and returns `""` gracefully for a missing path; a full base-PDF + `_build_text_overlay`
      + `merge_page` + text-extraction round-trip confirmed both the original page content and the
      merged text survive.
- [ ] **Not yet verified**: a full human click-through (upload a real PDF, place merge + signing
      fields via the UI, send a real document, confirm the final stamped PDF) — needs a human, not
      reachable via SSH.
- [ ] Regression check that an HTML-sourced template's rendering is byte-for-byte unchanged — not
      explicitly re-verified this round (the code path is untouched by this change: the `PDF`
      branch is new, the `HTML` branch's two lines are identical to before), but worth a real send
      to double-check rather than relying on code-review alone.

## Acceptance Criteria

- [ ] Business owner can create a PDF-sourced template, place merge fields and signing fields, and
      send a real document through it with all data correctly stamped. (Mechanism verified via
      direct round-trip; full UI click-through still needs a human.)
- [ ] An HTML-sourced template's output is byte-for-byte unchanged from before this feature.
- [x] Merge-field resolution happens before the document is hashed/sent — `_stamp_merge_fields` runs
      inside `_render_template_pdf`, called from `_generate_document_pdf` before `document_hash` is
      computed, same as the HTML/Jinja path.

## Related

- Domain index: `vault/esign/esign.md`
- ADR: `vault/decisions/0020-esign-pdf-upload-and-guided-signing.md`
- Prerequisites: `vault/esign/features/esign-template-management.md`,
  `vault/esign/features/esign-field-placement-designer.md`
- Companion (Phase 4): `vault/esign/features/esign-guided-click-to-sign.md`
