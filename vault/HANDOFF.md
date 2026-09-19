# Handoff / Current State

Compact resume point. Read this instead of the full history. Last updated 2026-08-24.

**Read `vault/findings/2026-08-24-ideal-product-vs-current-state.md` first** —
code-verified gap analysis (9 capability pillars, 16-item ranked gap
register, 4 structural risks) that this handoff's "Not built yet" and "Next
steps" sections now track.

## What exists (as of `main` @ `d1545a0`, plus everything below)

Custom Frappe app `real_estate_os` (repo `https://github.com/rafaii/real-estate.git`),
installed + working on site `erpnext.local` (Frappe/ERPNext v15).

**Built (verified live):**
- DocTypes: `Building` (BLD-{#####}), `Unit` (UNT-{#####}), `Lease Agreement` (LSE-{YYYY}-{#####})
- Customer KYC: 7 custom fields (fixtures) — id proof, income, emergency contact, etc.
- Bulk unit generator: `real_estate_os.api.generate_units` + "Generate Units" button (Client Script), guarded by server cap (MAX_BULK_UNITS=1000) + client confirmation dialog
- E-Sign data model: `E-Sign Document`, `E-Sign Signer` (child), `E-Sign Field` (child), `E-Sign Audit Log` (append-only). SHA-256 hash auto-computed on insert; audit log timestamp auto-set.
- E-Sign signing workflow: guest `/esign` page (consent + OTP + jSignature), whitelisted workflow in `real_estate_os/e_sign/workflow.py` (`send_for_signature`, `request_otp`, `verify_otp`, `give_consent`, `capture_signature`, `finalize_document`, `view_document_pdf`), pypdf PDF-stamp + Certificate of Completion, `docstatus=1` lock, immutable audit trail (IP/user-agent per event), Lease Agreement "Send for E-Signature" client script (auto-confirm dialog on save, Send/Not Now).
- Two-party counter-signature + PDC (ADR-0005): `Signature Settings` singleton (`esign_mode`, `business_signatory`); `PDC Entry` DocType (PDC-{#####}); sequential signing (customer order-1 → business signatory order-2, notified after customer signs); counter-signature email names who already signed; signer lookup handles two signers sharing one email; audit events `Created` + `Counter Signed`.
- Signing portal transparency: `/esign` shows "Signing Parties & Signatures" card (rendered signature image, typed name, signed timestamp, OTP/Consent/IP evidence per signer) + full "Audit Trail & Activity Log" table (timestamp, event, details, IP). Certificate of Completion in the signed PDF now includes a structured Audit Trail table alongside the signatures/evidence table. `view_document_pdf` commits the `Viewed` audit event on GET so it persists.
- Recurring invoicing (ADR-0004): `Rent Schedule` child table on Lease Agreement, `real_estate_os/payments/invoicing.py` (`build_payment_schedule`, `create_invoice_for_lease`, `generate_due_invoices`, `send_reminders`), e-sign `_finalize` hook auto-creates schedule + first invoice on signature (runs as Administrator), `scheduler_events.daily` runs due-invoice generation + 3-day reminders. Cancelled leases skipped by scheduler.
- Lease Agreement print format: dedicated Jinja contract template at `real_estate/print_format/lease_agreement/` (parties, premises, term, rent + deposit, rent schedule table, T&Cs, signature block, e-signed note), set as `default_print_format`.
- Lease/unit activity feed + lifecycle: `after_insert` reserves unit (Vacant→Reserved), e-sign `_finalize` → Occupied, `on_trash`/`cancel_lease` → Vacant (cancelled leases don't hold a unit); `real_estate_os.utils.add_activity` writes timeline entries on Lease + Unit.
- Cancel contract: Lease Agreement "Cancel Contract" button → `cancel_lease` marks lease Cancelled, cancels pending PDCs + rent-schedule rows, voids open E-Sign Document, releases unit, writes activity; re-sign/re-send blocked afterwards.
- Workspaces: `Real Estate` + `E-Sign` modules visible in Desk sidebar.
- Emails (invite, counter-sign invite, OTP) sent immediately (`now=True`), not queued.

**10 ADRs approved:** 0001 app-repo-structure, 0002 esign (in-built default +
provider abstraction), 0003 multi-tenancy (tenant-per-site), 0004
recurring-invoicing, 0005 counter-signature + PDC-driven invoicing, 0006
admin-portal approach (headless-hybrid SPA), 0007 admin portal React
frontend, 0008 tenant provisioning control-plane, 0009 renter self-service
portal, 0010 PDC schedule generation & bank reconciliation.

**2 ADR drafts pending Imran's approval** (`vault/decisions/drafts/`):
`draft-adr-head-lease-landlord-payables.md`,
`draft-adr-lease-lifecycle-state.md`.

Since 2026-08-17: **maintenance module, tenant/maintenance portals, and PDC
scheduling all shipped** — this section previously listed them as "not
built yet"; that was stale, not current. See `IMPLEMENTATION-PLAN.md` for
the authoritative per-feature status table.

**Genuinely not built yet** (per the 2026-08-24 findings doc): the entire
landlord/cost side of the business — `Head Lease` DocType, landlord
payables (zero `Purchase Invoice` references anywhere in the codebase),
Building→Cost Center hook (still commented out in `hooks.py`), margin/P&L
per building. Also: lease has no `lease_status` field (dashboard proxies
active-lease via `esign_status == "Signed"`, which never expires); zero
report definitions exist app-wide; RBAC is 2 roles (System Manager, Tenant)
vs. the 4-5 in MASTERPLAN; SMS OTP (email only); provider abstraction
(DocuSign/ZohoSign); tenant-provisioning console (ADR-0008, planned).

## Environment

- Bench root: `~/Development/erpnext` (NOT a git repo)
- App repo: `~/Development/erpnext/apps/real_estate_os` (the ONLY git dir)
- `bench` CLI: `/tmp/bench-bootstrap/bin/bench` (v5.31.0, not on PATH — invoke full path)
- Site: `erpnext.local` (running, `http://erpnext.local:8000` / `http://localhost:8000`)
- Admin login: `Administrator` / (user's password; default usually `admin`)
- GitHub auth: `gh` OAuth via keyring (done). No PAT in `.env` (scrubbed). Pushes work transparently.
- App is `pip install -e apps/real_estate_os` in the site venv (required for import).

## Git workflow (per AGENT.md)

- Never on `main`. Create worktree: `git worktree add ./wt-<agent>-<slug> -b agent/<agent>/<slug> origin/main` (from the app repo dir).
- Work in worktree → commit → `git checkout main && git merge --ff-only <branch>` → `git push` → `git worktree remove` → `git branch -d`.
- Vault (`~/Development/erpnext/vault`) is local-only, NOT committed.

## Frappe gotchas (learned, avoid repeating)

1. **App must be `pip install -e`** in the site venv, else `ModuleNotFoundError`.
2. **DocType layout** must be `<module>/doctype/<name>/<name>.json` (with `doctype/`). Missing it = silent skip on migrate.
3. **New modules on an already-installed app aren't auto-registered.** Register via `bench --site erpnext.local execute frappe.installer.add_module_defs --args '["real_estate_os"]' --kwargs '{"ignore_if_duplicate": True}'`, then `clear-cache` + `migrate` (or `execute frappe.model.sync.sync_all --kwargs '{"force": 1}'`). A *fresh* `install-app` has none of these issues.
4. **Sidebar is driven by `Workspace` records**, not `desktop.py`. Module shows in sidebar only with a Workspace fixture at `<module>/workspace/<name>/<name>.json`.
5. `bench execute` prints only truthy returns; multi-statement needs a `(lambda: ...)` wrapper. `--kwargs` uses Python `True`/`False` (not `true`).
6. Child-table DocTypes: `"istable": 1`, `"permissions": []`, no autoname. Controller class name = doctype name with spaces+hyphens removed, acronyms keep case (`E-Sign Document` → `ESignDocument`, `PDC Entry` → `PDCEntry`, NOT `PdcEntry`).
7. **Frappe GET requests roll back uncommitted writes.** Any side effect in a GET endpoint (e.g. recording a `Viewed` audit event in `view_document_pdf`) needs `frappe.db.commit()`.
8. **Guests can `frappe.get_all`** on DocTypes with System Manager-only read perms if `ignore_permissions=True`; the `/esign` page passes it explicitly for audit logs.

## Test data on dev site

- `BLD-00001` "Test Tower" + 10 units (21001–21010, Vacant, $2000) — 21007 accidentally generated units cleaned up, kept 10.
- `ESD-00005` E-Sign Document (ref BLD-00001) + one Audit Log entry
- `LSE-2026-00022` Lease Agreement (customer "Digital Pipeline", unit UNT-00002) → signed via full e-sign flow; `ESD-00023` finalized (docstatus=1) with 2-page signed PDF + 5 audit events (Sent, OTP Verified, Consent Given, Signed, Completed). Signer email `tenant@example.com`.
- `LSE-2026-00022` Rent Schedule: 12 monthly rows (Sep 2026→Aug 2027, $2000 each); row 1 `Invoiced` → Sales Invoice `ACC-SINV-2026-00008` (submitted, "Rent" item → "Sales - CC"). Remaining 11 rows `Pending` (scheduler will generate as due).
- `LSE-2026-00070` two-party + PDC demo (customer "Digital Pipeline" / tenant@example.com, signatory "imran@rafais.com"): 3 PDCs (PDC-00071..73, Oct/Nov/Dec 2026) → 3-row PDC-driven schedule; `ESD-00074` finalized (both signers Signed); audit trail Created→Sent→OTP→Consent→Signed(customer)→Sent(signatory)→OTP→Consent→Counter Signed→Completed; row 1 invoiced → `ACC-SINV-2026-00018`.
- `LSE-2026-21123` / `ESD-21124` two-party in progress: customer `Imran Rafai` (imran@rafais.com, order 1) already Signed with drawn signature; signatory `Imran` (same email, order 2, token `-80zqpQgAC7RIe-34J1Lugni1p1Gg_I1HSzvnb2MxA8`) pending. `/esign` portal for the signatory link shows customer signature + 7-event audit trail (Created, Sent, OTP Verified, Consent Given, Signed, 2× Sent invites). No `Viewed` event yet for signer 2 until the link is opened in a browser.

## Next steps (pick one)

1. **Get the two pending ADR drafts approved** — `draft-adr-head-lease-
   landlord-payables.md` and `draft-adr-lease-lifecycle-state.md`. Nothing
   in Wave 2 below can start until these are numbered.
2. **Wave 1 (in progress as of 2026-08-24, no ADR needed):**
   portal-source-consolidation (move `ui/` React source into the app repo,
   drop the dead Vue `portal/`), cost-center-per-building (hook + invoice
   line-item wiring + backfill).
3. **Wave 2 (blocked on ADR approval):** `head-lease-doctype.md`,
   `landlord-payables.md`, `lease-lifecycle-state.md` — the cost side of
   the business; see the findings doc §6 for why this is the priority.
4. **Wave 3:** `building-profitability-report.md`,
   `rent-roll-and-arrears-report.md`, `cash-flow-forecast.md`,
   `role-based-access.md` (2 roles → 4-5).
5. Longer-tail: SMS OTP, provider abstraction (DocuSign/ZohoSign),
   tenant-provisioning console.

## Vault map

- [`INDEX.md`](INDEX.md) — domains + live task list
- [`IMPLEMENTATION-PLAN.md`](IMPLEMENTATION-PLAN.md) — live, prioritized feature list
- [`DECISIONS.md`](DECISIONS.md) — ADRs (see file for current count — this
  line itself is a known example of the kind of staleness this whole
  section suffers from; don't trust a specific number here)
- [`CHANGELOG.md`](CHANGELOG.md) — append-only log
- [`PLATFORM_STRATEGY.md`](PLATFORM_STRATEGY.md) — platform-wide PRD (2026-08-27: replaces `MASTERPLAN.md`)
- [`os/REALESTATE_MASTERPLAN.md`](os/REALESTATE_MASTERPLAN.md) — Real Estate OS's own PRD

Note: this whole `HANDOFF.md` file is known-stale beyond the links above
(flagged as gap R2 in `vault/findings/2026-08-24-ideal-product-vs-current-state.md`)
— treat its narrative sections below with caution; only the links section
was corrected as part of today's `MASTERPLAN.md` split.