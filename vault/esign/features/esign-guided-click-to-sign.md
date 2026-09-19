---
status: in-progress
owner: developer-1
domain: esign
created: 2026-09-03
updated: 2026-09-03
related_adr: ["0020-esign-pdf-upload-and-guided-signing"]
---

# E-Sign Guided Click-to-Sign (Phase 4)

## Summary

Replaces the guest signing page's flat, position-blind single widget (one jSignature canvas + one
typed-name input, submitted once for the whole document, confirmed by full code read to have zero
document-position awareness) with a guided flow: the signer creates their mark once (drawn, or
typed and rendered in a bundled cursive font), then taps each of their own assigned field positions
on the actual rendered document before submitting. Depends on Phase 2
(`esign-field-placement-designer.md`) already being live for the field-placement data to exist.

## Requirements

- Sign card in `apps/inbuilt_esign/inbuilt_esign/www/esign/index.html` becomes an explicit choice:
  "Draw my signature" or "Type my name" — replacing today's always-both-visible canvas + text input.
- A typed name is checked client-side (only) against the already-rendered `signer_name`, normalized
  (trim, case-insensitive); mismatch shows an inline, non-blocking warning with a way to continue
  anyway. Never a server-side hard gate — this is UX guidance, not identity verification (OTP +
  consent remain the actual compliance gates, unchanged).
- Initials are auto-derived from the name (first letter of each word, uppercase) — no new input, no
  new stored field.
- A bundled cursive font (Dancing Script, OFL-licensed) renders: the live client-side preview of a
  typed name, the stamped Signature/Initial fields on the final PDF (replacing today's
  `Helvetica-Oblique` fallback in `_build_field_overlay`), and the typed-name display on the
  Certificate of Completion page — same font file, all three places, so what the signer previews is
  what actually gets stamped.
- The signing page renders the document via PDF.js (not a flat iframe) and shows the current
  signer's own assigned `E-Sign Template Field` positions as tappable markers. Tapping a Signature/
  Initial marker fills it with their created mark (preview only, client-side); tapping a Date marker
  fills today's date; tapping a Text marker fills their name — mirroring exactly what
  `_build_field_overlay` already does per `field_type` at real stamping time, so the tap-preview and
  the eventual stamp are consistent by construction.
- "Sign document" stays disabled until every one of the signer's assigned fields has been tapped.
  This is a client-side-only confirmation gate — it does not change `capture_signature`'s
  request/response shape or add any new server-side requirement; the actual stamping-everywhere
  behavior already happens automatically today via `_stamp_template_fields` once the document is
  fully signed (Phase 2), so tapping is a confirmation affordance, not new plumbing.
- A document whose template has no field placements (or no template at all — the legacy
  default-Print-Format path) falls back to exactly today's flat single-widget flow — no visual or
  functional regression for existing/in-flight documents.
- V1 scope: `Lease Agreement` only, consistent with prior phases.

## Design

**Font**: `apps/inbuilt_esign/inbuilt_esign/public/fonts/DancingScript-Regular.ttf` (one file, three
uses):
- `pdfmetrics.registerFont(TTFont("DancingScript", <path>))` in `workflow.py`, used by
  `_build_field_overlay`'s typed-name fallback branch (Signature/Initial field types) in place of
  `Helvetica-Oblique`.
- Base64-embedded `@font-face` in `_certificate_html`'s generated HTML (wkhtmltopdf renders in a
  sandboxed context — a `file://` path isn't reliable, so the font bytes are embedded directly).
- `@font-face` served from the app's own `/assets/inbuilt_esign/fonts/...` path, referenced in
  `www/esign/index.html`'s CSS for the live typed-name preview.

**Signer-field resolution** (`www/esign/index.py`): needs the same role-to-field matching
`_stamp_template_fields`/`_resolve_stamp_signer` (`workflow.py`) already do server-side, applied in
the opposite direction (given a signer, find their fields rather than given a field, find the
signer) — factor this into one shared helper in `workflow.py` that both the finalize-time stamping
path and the page-context-building path in `index.py` call, rather than duplicating the role/
signing_order matching logic. Resolved fields (page_no, x_pct, y_pct, width_pct, height_pct,
field_type) get baked into the page's existing `window.esignData` server-rendered context, matching
the page's current pattern (no new round-trip needed).

**Signing page** (`www/esign/index.html`): **implemented differently from the original design
note** — kept the `#pdf-frame` iframe as-is (still useful for a full, un-annotated review before
OTP/consent even happen) and added a *separate* PDF.js-rendered, page-stacked view with tappable
field markers inside the sign card itself, shown only when the signer has assigned fields. Less
disruptive to the existing review step, same end result for the guided-signing goal. Uses the
`pdfjs-dist` legacy ES-module build (dynamic `import()` from a plain classic `<script>` — no need to
convert the whole inline script to `type="module"`, since dynamic `import()` works in classic
scripts too) rather than a bundler, since this page has none. New client-side state: which mode
(draw/type) is active, which fields have been tapped/filled, gating `#btn-sign`'s disabled state on
all-fields-tapped in addition to today's existing `otp_verified && consented` check.

**Unchanged**: `capture_signature`'s params/behavior, `request_otp`/`verify_otp`/`give_consent`,
the sequential OTP-then-consent-then-sign server-side gating, `_finalize()`'s multi-field stamping
(Phase 2, already stamps every assigned field automatically once `capture_signature` completes) —
this feature only changes what the signer sees and interacts with before making that one existing
API call.

## Implementation Plan

- [x] Sourced and bundled the Dancing Script TTF (OFL-licensed, fetched from the official
      `google/fonts` GitHub repo — no static weight exists, used the variable-font file, confirmed
      it registers and renders correctly with reportlab before committing to it) into
      `apps/inbuilt_esign/inbuilt_esign/public/fonts/`, alongside its OFL license text.
- [x] Wired the font into `_build_field_overlay` (reportlab, `pdfmetrics.registerFont`/`TTFont`,
      lazily registered once), `_certificate_html` (base64-embedded `@font-face` — a `file://` path
      isn't reliable inside wkhtmltopdf's sandboxed rendering), and the browser CSS preview (served
      from `/assets/inbuilt_esign/fonts/...`).
- [x] Factored `get_signer_fields(doc, signer)` into `workflow.py` — reuses the exact same
      `_resolve_stamp_signer` helper `_stamp_template_fields` already used, just applied from one
      signer's perspective instead of iterating every field for every signer.
- [x] Extended `www/esign/index.py`'s context with `context.signer_fields` (page_no/x_pct/y_pct/
      width_pct/height_pct/field_type/label per field) and `context.signer_fields`-adjacent data
      already available (`signer.signer_name` for the name-match check — no new field needed).
- [x] Reworked the sign card: draw/type mode toggle, live cursive preview + non-blocking
      name-mismatch warning, initials auto-derivation, a separate PDF.js tappable-field-marker view
      (kept the existing iframe for general review — see Design note above), per-field fill state,
      "Sign document" gating on all-fields-tapped.
- [x] **Found and fixed a real infra gap while implementing**: the deployed nginx has no MIME
      mapping for `.mjs` (same class of issue already hit and fixed for the React portal's PDF.js
      integration) — bundled pdfjs-dist's legacy build renamed `.mjs`→`.js`, and separately found
      `inbuilt_esign`'s `public/` folder had no VPS-level asset symlink at all (only `real_estate_os`
      had one) — fixed the VPS-side Dockerfile (not tracked in any git repo) to add it, confirmed
      live in the built image before deploying.
- [x] Deploy: merged `inbuilt-esign#6` to `main` (no `real_estate_os` changes needed — this phase is
      entirely guest-facing), VPS image rebuilt (verified the new asset symlink exists in the image
      first), containers recreated — no migration needed, no schema change — 2026-09-03.
- [x] Verified live: all three new static assets (font, `pdf.min.js`, `pdf.worker.min.js`) serve
      with correct MIME types (`application/javascript` for both JS files); hit the actual
      production signing URL for an existing document and confirmed it renders successfully through
      the real Frappe pipeline (`get_context` → `frappe.render_template`, not a simulated context)
      with `signerFields` correctly resolving to `[]` and `expectedName` correctly resolving to the
      real signer's name — confirms zero regression for the "no field placements" case every
      existing document is currently in.
- [ ] **Not yet verified**: the interactive JS (mode toggle, tap-to-place, PDF.js rendering,
      gating) against a document that actually *has* field placements assigned to a signer — needs
      a human click-through with real business data (create a PDF or HTML template, place
      Signature/Initial fields, send a real/test lease through it), not reachable via curl/SSH.
- [ ] **Not yet verified**: type mode's cursive rendering matching between the live browser preview,
      the final stamped PDF, and the certificate page — mechanism verified independently (font
      registration + overlay stamping tested via direct round-trip) but not visually confirmed
      side-by-side.

## Acceptance Criteria

- [ ] A test signer can choose "Type my name," see a cursive rendering, tap all assigned field
      markers on the real document, and complete signing. (Needs a human click-through.)
- [ ] The final signed PDF and the certificate page both show the same cursive-font rendering the
      signer previewed. (Font-registration/stamping mechanism verified; visual side-by-side
      comparison still needs a human.)
- [ ] "Sign document" is unreachable until every assigned field has been tapped, for a template with
      field placements. (Needs a human click-through — no field-placement document exists yet to
      test against.)
- [x] A document with no field placements signs exactly as it did before this feature — verified
      live: the real signing page for an existing document renders with `signerFields: []`, taking
      the same flat-flow code path as before this feature.

## Related

- Domain index: `vault/esign/esign.md`
- ADR: `vault/decisions/0020-esign-pdf-upload-and-guided-signing.md`
- Prerequisite: `vault/esign/features/esign-field-placement-designer.md`
- Companion (Phase 3): `vault/esign/features/esign-pdf-template-upload.md`
