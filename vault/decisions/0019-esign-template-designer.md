# ADR-0019: E-Sign Template & Field-Placement Designer

- **Status**: accepted
- **Date**: 2026-09-02
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Contract generation in the e-sign flow is currently a single, hardcoded developer-authored Print
Format per reference doctype. `inbuilt_esign`'s `_generate_document_pdf()`
(`apps/inbuilt_esign/inbuilt_esign/inbuilt_esign/workflow.py`) always calls
`frappe.get_print(doc.reference_doctype, doc.reference_name, as_pdf=True)`, which resolves to
`Lease Agreement`'s fixed `default_print_format`
(`apps/real_estate_os/real_estate_os/real_estate/print_format/lease_agreement/lease_agreement.html`).
A business owner has no way to edit contract wording, choose which data fields appear where, or
control where each signer's signature/initial/date lands. Signature capture today is only ever
stamped onto a trailing Certificate-of-Completion appendix page — never onto the contract itself —
even though an `E-Sign Field` child table (`page_no`/`x_coord`/`y_coord`/`field_type`/`signer`)
already exists in the schema for exactly this purpose. It is dead code: nothing in
`apps/inbuilt_esign` reads or writes it.

We want to bring this up to the level of DocuSign/PandaDoc/HelloSign template editors — the business
owner controls the template body and pre-places signature/initial/date fields — while preserving
what's deliberately different about this system: we generate the original document server-side from
the template before sending it out, rather than handing the signer a blank form.

This changes the core document-generation mechanism, introduces a genuinely new PDF-stamping
pipeline, and spans the `inbuilt_esign` / `real_estate_os` app boundary established by ADR-0017.

## Decision

1. Add three new doctypes to `inbuilt_esign` (not `real_estate_os`) — `E-Sign Template`,
   `E-Sign Template Signer Role` (child), `E-Sign Template Field` (child) — keeping the e-sign engine
   generic and reusable across future OS apps per ADR-0017, rather than coupling template/
   field-placement logic to real-estate-specific code.
2. Reuse Frappe's existing Jinja/`frappe.render_template` merge-field semantics for the template
   body — the same mechanism Print Formats already use — instead of inventing a new templating
   language or adopting a third-party template engine.
3. Template fields bind to a `role_label` (e.g. "Customer", "Business Signatory"), resolved to an
   actual `E-Sign Signer` at send-time by signing order — not a literal signer email — since
   templates are meant to be reused across many documents.
4. Field coordinates are stored as percentage-of-page (`x_pct`/`y_pct`/`width_pct`/`height_pct`),
   not absolute pixels/points, for resolution independence across different page sizes/renders.
5. Rendering falls back to today's exact `frappe.get_print()` Print Format path whenever no active
   `E-Sign Template` exists for a reference doctype — existing Lease Agreements and any in-flight
   `E-Sign Document`s keep working with zero migration required.
6. The visual field-placement designer renders the *actual* generated PDF client-side via PDF.js
   (not the raw HTML source) so that placed coordinates are guaranteed to match the final signed
   document — HTML page breaks in an editor don't reliably match where wkhtmltopdf actually
   paginates.
7. The designer UI lives in `real_estate_os`'s existing React admin SPA (`apps/real_estate_os/ui`),
   under the existing Settings/E-Signature area (same `esign.can_manage` gate already in place),
   calling `inbuilt_esign` only via whitelisted methods over HTTP — never a direct Python import,
   consistent with the existing `e_sign/dispatch.py` hook-based pattern.
8. At finalize time, placed fields get burned onto the actual contract PDF pages via a
   pypdf/reportlab overlay, in addition to (not instead of) the existing Certificate-of-Completion
   appendix page, which remains the audit record.
9. V1 scope: template body editing is a code editor (Jinja/HTML) plus a merge-field picker sidebar,
   not a full WYSIWYG editor; reference-doctype scope is `Lease Agreement` only for v1, though the
   doctypes themselves stay generic (`reference_doctype` is a Link, not hardcoded).
10. Rollout is two separately shippable phases: Phase 1 (template CRUD + rendering swap with
    fallback) ships and is verified live before Phase 2 (PDF.js field-placement designer + real
    stamping) is built on top of it.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Extend Frappe's native Print Format doctype directly with x/y metadata | Reuses a built-in Frappe primitive | Print Format is a shared, generic Frappe concept used well beyond esign (reports, other doctypes) — repurposing it for esign-specific field-placement pollutes a core primitive; Frappe Desk (where Print Format is edited) is explicitly never tenant-facing per the platform strategy, so business owners can't reach it there anyway | rejected |
| Adopt a third-party PDF-template/form-designer library | Faster initial build for the drag/drop UI | New external dependency (license/cost/maintenance surface); doesn't naturally reuse Frappe's existing Jinja merge-field conventions; less control over the percentage-based, resolution-independent coordinate model this needs | rejected |
| Put the new doctypes in `real_estate_os` instead of `inbuilt_esign` | Slightly less cross-app surface for this one feature | Breaks ADR-0017's reusability goal — any future OS app would have to reimplement template/field-placement from scratch instead of getting it "for free" from the shared e-sign engine | rejected |
| New doctypes in `inbuilt_esign`, Jinja reuse, PDF.js-based designer under existing Settings area | Keeps the engine generic and reusable, reuses proven rendering/permission mechanisms, guarantees placement matches the final signed PDF | Meaningful new surface (3 doctypes, a rendering-path branch, a new stamping pipeline, a PDF.js frontend) | chosen |

## Consequences

### Positive

- Business owners get real self-service control over contract content and signing-field placement
  without touching Frappe Desk.
- The e-sign engine (`inbuilt_esign`) stays reusable for future OS apps per ADR-0017 — a second OS
  gets template/field-placement for free.
- Zero-migration backward compatibility: any reference doctype without an active template keeps
  working exactly as it does today.

### Negative

- Meaningful new surface: 3 new doctypes, a rendering-path branch, a new PDF-stamping pipeline, and
  a PDF.js-based frontend designer.
- `render_template` executing business-owner-authored Jinja is a code-execution surface — needs
  Frappe's existing Print-Format-style sandboxing confirmed to apply here too, not assumed.
- The preview-render (used by the designer) and the finalize-render (used for actual stamping) must
  produce identical page dimensions, or placed fields will drift from where they land on the signed
  PDF — an explicit acceptance check is needed for this, not just visual spot-checking.

## Implementation

- Feature file (Phase 1): `vault/esign/features/esign-template-management.md`
- Feature file (Phase 2): `vault/esign/features/esign-field-placement-designer.md`
