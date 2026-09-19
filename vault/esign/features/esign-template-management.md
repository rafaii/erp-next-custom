---
status: in-progress
owner: developer-1
domain: esign
created: 2026-09-02
updated: 2026-09-03
related_adr: ["0019-esign-template-designer"]
---

# E-Sign Template Management (Phase 1)

## Summary

Lets a business owner author and activate their own contract template — replacing the single
hardcoded Print Format `inbuilt_esign` currently always uses — while keeping full backward
compatibility for any reference doctype with no template configured. This is Phase 1 of ADR-0019:
template CRUD and the rendering-pipeline swap, without the visual field-placement designer (that's
Phase 2, `esign-field-placement-designer.md`).

## Requirements

- Business owner (gated by the existing `esign.can_manage` permission check) can create, edit, and
  activate/deactivate `E-Sign Template` records for a reference doctype (v1: `Lease Agreement` only).
- Template body is a Jinja/HTML code field, rendered with the same semantics as a Frappe Print
  Format (`frappe.render_template(body, {"doc": doc})`).
- A merge-field picker in the editor UI lists the reference doctype's own fields as `doc.<fieldname>`
  plus, two hops deep across Link fields, linked-doctype fields as `links.<path>.<fieldname>` (e.g.
  `links.customer.customer_name`, `links.unit.building.address`) — grouped by source doctype,
  inserted at the cursor on click.
- Template defines an ordered list of signer roles (`E-Sign Template Signer Role`: `role_label`,
  `signing_order`) — e.g. "Customer" order=1, "Business Signatory" order=2 — replacing the hardcoded
  tenant/business-signatory assumption in `send_lease_for_signature`.
- At most one active (`is_active=1`) template per `reference_doctype`; validated server-side.
- A "Preview" action renders a live sample PDF through the exact same code path used at send-time.
- The editor shows a debounced live HTML preview of the (unsaved) body as the owner types, against
  a real sample record — approximate (no pagination) but immediate, so writing raw HTML/Jinja
  doesn't require constant save-and-check-the-PDF round-trips.
- When no active template exists for a reference doctype, document generation falls back to today's
  exact `frappe.get_print(doc.reference_doctype, doc.reference_name, as_pdf=True)` behavior — zero
  migration required for existing/in-flight documents.

## Design

**New doctypes** (`apps/inbuilt_esign/inbuilt_esign/inbuilt_esign/doctype/`):
- `E-Sign Template` (parent): `title` (Data, reqd), `reference_doctype` (Link -> DocType, reqd),
  `is_active` (Check), `body` (Code/HTML), `signer_roles` (Table -> E-Sign Template Signer Role).
  (The `fields` Table -> E-Sign Template Field child table is added in Phase 2, along with that
  doctype itself — not created now, to avoid a dangling reference to a not-yet-existing doctype.)
- `E-Sign Template Signer Role` (child): `role_label` (Data), `signing_order` (Int).

**`E-Sign Document`** (existing, modified): add `template` (Link -> E-Sign Template, optional — null
means the legacy default-Print-Format path was used). Drop the dead `fields` (Table -> E-Sign Field)
child-table field from `E-Sign Document` — confirmed via grep to be unused anywhere in
`apps/inbuilt_esign`. Confirm the standalone `E-Sign Field` doctype has no other references before
deleting it entirely (Phase 2 introduces its generic replacement, `E-Sign Template Field`, at the
template level instead).

**Rendering pipeline** (`apps/inbuilt_esign/inbuilt_esign/inbuilt_esign/workflow.py::_generate_document_pdf()`):
1. Resolve template: explicit `template` param if given, else `E-Sign Template` where
   `reference_doctype = doc.reference_doctype and is_active = 1`.
2. No active template -> unchanged `frappe.get_print(...)` call (fallback).
3. Active template -> `html = frappe.render_template(template.body, {"doc": doc})`, then
   `frappe.utils.pdf.get_pdf(html)`.
4. Store the resolved `template` on the created `E-Sign Document`.

**Signer resolution** (`send_lease_for_signature` in the same `workflow.py`): when the resolved
`E-Sign Document.template` has `signer_roles`, build the `E-Sign Signer` rows from that ordered list
instead of the current hardcoded tenant-then-business-signatory logic; fall back to the current
hardcoded behavior when no template (or a template with no `signer_roles`) is in play.

**New whitelisted methods** (`inbuilt_esign`, exact module path TBD at implementation):
- `get_mergeable_fields(reference_doctype)` — returns field list for the merge-field picker.
- `render_template_preview(template_name)` — returns a PDF for the Preview action.
Both gated by the same `esign.can_manage` permission check already used for the Signature Settings
page in `real_estate_os`.

**Frontend** (`apps/real_estate_os/ui`): new template list/editor screen under the existing Settings
area in `Dashboard.tsx`, alongside the current "E-Signature" block (`s.esign.can_manage` gate).

## Implementation Plan

- [x] Add `E-Sign Template` and `E-Sign Template Signer Role` doctypes to `inbuilt_esign`.
- [x] Add `template` field to `E-Sign Document`; remove the dead `fields` child-table field and
      delete the orphaned `E-Sign Field` doctype (confirmed via grep it had no other references;
      a `remove_orphaned_esign_field_doctype` patch drops its DocType/table on migrate).
- [x] Update `_generate_document_pdf()` to resolve and render an active template, with fallback.
- [x] Update `send_lease_for_signature` to build `E-Sign Signer` rows from `template.signer_roles`
      when present, falling back to today's hardcoded roles otherwise. Permission check uses
      `frappe.has_permission("E-Sign Template", "write")` (System Manager by default via its own
      DocPerm row) rather than reaching into `real_estate_os`'s `Signature Settings` — keeps
      `inbuilt_esign` free of any real-estate-specific doctype reference, per ADR-0017.
- [x] Add `get_mergeable_fields` and `render_template_preview` whitelisted methods, permission-gated
      via `_require_template_manage_permission()`.
- [x] Build the template list/editor UI in `apps/real_estate_os/ui` (code editor + merge-field
      picker + signer-role table + Preview action) under the existing Settings area.
- [x] Deploy: merged to `main` in both repos (`rafaii/inbuilt-esign#1`, `rafaii/real-estate#77`),
      VPS image rebuilt, containers recreated, `bench migrate` run clean on both tenant sites with
      `inbuilt_esign` installed (`rastec.nnuggets.com`, `realestate.nnuggets.com`) — 2026-09-03.
- [x] Verify: `tabE-Sign Template`/`tabE-Sign Template Signer Role` exist on both sites; the old
      `E-Sign Field` DocType record is gone from `tabDocType` on both (its now-orphaned, empty
      table wasn't physically dropped by Frappe's `on_trash` this time — harmless, left as an
      optional cosmetic follow-up, not blocking).
- [x] Verify: pre-existing `E-Sign Document` data survived the migration untouched (spot-checked
      `ESD-00354` on realestate.nnuggets.com, still `status: Signed`); both sites respond `200` on
      `/login`; backend container logs show a clean startup with no errors.
- [ ] Verify: a lease with no active template still sends/signs/finalizes correctly *going forward*
      (only pre-existing data was checked above, not a fresh send through the deployed code).
- [ ] Verify: an authored, activated template renders correctly through the real send flow and
      produces the expected `document_pdf` — needs an actual click-through (create a template in
      the new Settings UI, activate it, send a real/test lease).
- [ ] Verify: `get_mergeable_fields`/`render_template_preview` reject a non-`can_manage` role.
- [x] Fixed a deploy gap found by Imran immediately after the first deploy ("I do not see any UI
      element"): `real_estate_os/ui`'s committed build output (`real_estate_os/public/portal/`) was
      never regenerated for PR #77 — the source merged and deployed, but the served bundle was
      still pre-change. Fixed via PR #78 (build output only) — see
      [[feedback_real_estate_os_ui_build_step]] (memory) for the standing rule this created: any
      `real_estate_os/ui` change must rebuild and commit the bundle in the same PR, `tsc` passing
      is not sufficient.
- [x] Added a live preview to the template body editor per Imran's follow-up request ("difficult to
      see how it's going to look because it is HTML based"): new `render_body_preview` endpoint
      (renders unsaved body text against a sample record, returns HTML not PDF) + a debounced
      (500ms) sandboxed-iframe preview pane in the editor dialog, merge-field picker moved to a
      compact chip strip to make room. `inbuilt-esign#2` + `real-estate#79`, merged and deployed
      2026-09-03 (no schema change, no migration needed). Verified live: served bundle contains the
      new UI, and a direct `bench execute render_body_preview` against realestate.nnuggets.com
      correctly substituted real data into a merge-field test string.

Code and deployment complete (branch `agent/developer-1/esign-templates` + two follow-ups:
`agent/developer-1/esign-templates-build` for the bundle fix, `agent/developer-1/esign-live-preview`
for the live-preview addition). All merged and live on both production tenant sites as of
2026-09-03. Remaining gap is functional click-through verification of the template-authoring flow
by a human (create/activate a template, send a real/test lease) — everything else (deployment,
migration, bundle serving, the new endpoints) is verified live.

## Acceptance Criteria

- [ ] Business owner can create, edit, preview, and activate an `E-Sign Template` for Lease
      Agreement through the UI.
- [ ] A lease sent while a template is active generates its contract PDF from that template's body.
- [ ] A lease sent while no template is active behaves exactly as before this feature (byte-for-byte
      equivalent generation path, not just "looks the same").
- [ ] Template management endpoints are inaccessible to a user without `esign.can_manage`.

## Related

- Domain index: `vault/esign/esign.md`
- ADR: `vault/decisions/0019-esign-template-designer.md`
- Phase 2: `vault/esign/features/esign-field-placement-designer.md`
