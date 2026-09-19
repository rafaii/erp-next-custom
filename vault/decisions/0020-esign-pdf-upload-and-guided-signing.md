# ADR-0020: PDF Template Upload & Guided Click-to-Sign

- **Status**: accepted
- **Date**: 2026-09-03
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

ADR-0019's Phase 1 (business-owner-authored HTML/Jinja contract templates) and Phase 2 (visual
Signature/Initial/Date/Text field placement + PDF stamping) are both built and live. Two gaps
remain, both raised directly by Imran after using the shipped designer:

1. **Authoring still requires HTML/Jinja knowledge.** The only way to create a template body today
   is to write Jinja/HTML in a code editor. A business owner who just has a lawyer-drafted PDF
   contract has no way to use it directly — they'd have to manually retype it as HTML.
2. **Signing is still "blind."** The guest signing page (`apps/inbuilt_esign/inbuilt_esign/www/esign/index.html`,
   confirmed by full code read, not assumed) has zero awareness of where on the document a
   signature goes. It's one flat page: view a PDF in an iframe, then a single jSignature canvas +
   typed-name text input below it, submitted once for the whole document. There's no per-field
   interaction, and a typed name is stored and shown as plain italic serif text — no generated
   signature look at all.

Two research findings materially simplify what's needed:

- **Multi-location stamping already works.** Phase 2's `_stamp_template_fields` already iterates
  every `E-Sign Template Field` and stamps whichever signer's role matches — a signer with 3
  assigned field positions already gets their one captured signature/typed-name burned onto all 3,
  automatically. Nothing server-side needs to change for "one signature, applied everywhere it's
  needed" — that's already true. What's missing is purely the signer-facing UX to show them where
  those positions are and let them confirm.
- **A "generated signature" doesn't need to be a separately-generated image.** The stamping code
  (`_build_field_overlay`) already has a typed-name fallback path — it just draws
  `Helvetica-Oblique` text today. Swapping in a bundled cursive TTF font there — and reusing the
  same font via CSS `@font-face` in the browser preview and the certificate page — *is* "generating
  a signature." No new image-generation pipeline, no canvas-to-PNG conversion, no new storage field.

## Decision

Extend the existing `E-Sign Template`/`E-Sign Template Field` system (ADR-0019) with two
separately-shippable phases, rather than introducing new doctypes or a parallel system:

**Phase 3 — PDF Template Upload**
1. `E-Sign Template` gains `source_type` (Select: `HTML`/`PDF`, default `HTML`) and `pdf_file`
   (Attach). `HTML` keeps today's `frappe.render_template` + `get_pdf` rendering path unchanged.
2. `E-Sign Template Field.field_type` gains a `Merge Field` option, with a new `merge_token` (Data)
   field (e.g. `links.customer.customer_name`) used instead of `role_label` for that type.
3. For `source_type == "PDF"`, document generation uses the uploaded PDF as the base and runs a new
   merge-field stamping pass — resolving each `Merge Field` row's `merge_token` against the same
   `_render_context` (`doc`/`links`) resolution Jinja rendering already uses, and stamping the
   resolved text via a generalized version of the existing `_build_field_overlay` primitive
   (extended to draw arbitrary text, not only signer data). This runs once, before
   `document_hash` is computed, preserving ADR-0002's tamper-evidence guarantee.
4. Signature/Initial placement, finalize-time stamping, and the PDF.js field-placement designer
   are all unchanged regardless of `source_type` — downstream of generation there is just a
   `document_pdf`.
5. V1 stays `Lease Agreement`-only, consistent with Phase 1/2.

**Phase 4 — Guided Click-to-Sign with Generated Cursive Signatures**
1. Bundle Dancing Script (OFL-licensed) as a TTF in `apps/inbuilt_esign/inbuilt_esign/public/fonts/`,
   used in three places from the one file: `pdfmetrics.registerFont`/`TTFont` in
   `_build_field_overlay`'s typed-name fallback (replacing `Helvetica-Oblique`); a base64-embedded
   `@font-face` in `_certificate_html`'s wkhtmltopdf-rendered HTML; and a `@font-face` served from
   the app's own `/assets/inbuilt_esign/...` path for the signing page's live CSS preview.
2. The sign card in `www/esign/index.html` becomes an explicit "Draw my signature" / "Type my name"
   choice, replacing today's always-both-visible canvas-plus-text-input.
3. A typed name is checked client-side only against the already-rendered `signer_name` (normalized:
   trim, case-insensitive) and shown as a soft, non-blocking warning on mismatch — never a
   server-side hard gate.
4. Initials are auto-derived from the signer's name (first letter of each word) — no new stored
   field, rendered in the same cursive font.
5. `www/esign/index.py` resolves the current signer's own assigned `E-Sign Template Field` rows
   (the same role-to-field matching `_stamp_template_fields`/`_resolve_stamp_signer` already do
   server-side, factored into one shared helper both call) and bakes them into the page context.
6. The signing page renders the document via PDF.js (reusing the technique already proven in the
   business-owner's `EsignFieldPlacementDialog`) with each of the signer's fields shown as a
   tappable marker; "Sign document" stays disabled until every field is tapped. This is a
   client-side-only confirmation layer — OTP verification and consent remain the sole
   server-enforced preconditions, and `capture_signature`'s request/response shape is unchanged.
7. A document whose template has no field placements (or no template at all) falls back to
   exactly today's flat single-widget signing flow — no regression for existing/in-flight
   documents.
8. V1 stays `Lease Agreement`-only.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| A separate "PDF Template" doctype instead of a `source_type` branch on `E-Sign Template` | Cleaner separation on paper | Duplicates Signer Roles/Fields/permissions across two doctypes for no real benefit — the field-placement designer, stamping pipeline, and permission model are identical either way | rejected |
| Client-side canvas rendering of typed names into a PNG signature image | Works without touching server-side stamping code | A whole new image-generation pipeline (canvas, font loading, PNG export, new storage) for something a font swap at the two existing render/stamp sites already achieves, with better cross-renderer consistency | rejected |
| Server-side hard-blocking name validation before allowing signature | Stronger identity-consistency enforcement | Risks locking out a legitimate signer over a minor name variation (nickname, missing middle name); OTP + consent already establish the actual compliance-relevant identity check | rejected (Imran's explicit choice: soft warning only) |
| `source_type`/`pdf_file` on `E-Sign Template`, `Merge Field` as a new `E-Sign Template Field.field_type`, font-swap for generated signatures, client-side click-to-place confirmation | Reuses the entire Phase 1/2 pipeline unchanged; minimal new surface; each piece maps to something already built | Two new doctype fields, one new field_type + generalized overlay function, a bundled font asset, and a meaningfully reworked signing-page UI | chosen |

## Consequences

### Positive

- A business owner with only a PDF contract (no HTML/Jinja skill) can use the template system
  fully, via the same visual designer already built for signature placement.
- Signers get a guided, position-aware signing experience with a real generated-signature look,
  without any new compliance/audit surface — OTP and consent remain the only server-enforced gates.
- Both phases are additive and backward-compatible: existing HTML templates and existing/in-flight
  documents with no field placements are completely unaffected.

### Negative

- The signing page (`www/esign/index.html`) gets meaningfully more complex — PDF.js integration,
  per-field tap state, a mode toggle — on a guest-facing, no-login page where bugs are more visible
  and costlier (a broken signing page blocks real contracts from being signed).
- A bundled font file and its three separate usage sites (reportlab, wkhtmltopdf HTML, browser CSS)
  need to render consistently or the "what you see is what gets stamped" guarantee breaks.

## Implementation

- Feature file (Phase 3): `vault/esign/features/esign-pdf-template-upload.md`
- Feature file (Phase 4): `vault/esign/features/esign-guided-click-to-sign.md`
