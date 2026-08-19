# ADR-0002: E-Signature — In-Built vs Third-Party

- **Status**: accepted
- **Date**: 2026-08-16
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Lease agreements must be executed electronically. Two paths exist: (a) integrate
DocuSign/ZohoSign, or (b) build a native e-signature module. The masterplan's
appendix argues strongly for an in-built approach on cost and data-residency
grounds. This decision affects legal defensibility, per-envelope cost, and UX.

## Decision

**In-built e-signature as the default**, wrapped behind a provider abstraction
so DocuSign/ZohoSign remain selectable per tenant:

- **DocTypes**: `E-Sign Document`, `E-Sign Signer`, `E-Sign Field`, `E-Sign Audit Log`.
- **Tamper-evidence**: SHA-256 hash of the finalized PDF stored before sending.
- **Consent**: explicit "I consent to sign electronically" screen (ESIGN/UETA/PIPEDA).
- **Identity**: OTP via email/SMS before access; `otp_verified` + timestamp recorded.
- **Capture**: signature_pad.js canvas → PNG; typed-name fallback.
- **Audit**: immutable append-only log (sent/viewed/consent/OTP/each field/completed).
- **Finalize**: overlay signature onto PDF (pypdf), append Certificate of
  Completion page, re-hash, lock via `docstatus=1`.
- **Provider abstraction**: a `Signature Settings` singleton with
  `esign_mode` (In-Built | DocuSign | ZohoSign).

Legal basis: ESIGN Act / UETA (US) and PIPEDA-adjacent provincial laws (Canada)
require intent, consent, association of signature with document, and reliable
retention — all satisfied by a simple e-signature + strong audit trail; PKI is
not required for commercial leases.

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| DocuSign/ZohoSign only | No e-sign build effort, third-party trust | Per-envelope fees, data residency (US/offshore), external dependency, per-tenant config burden | rejected as default |
| In-built only (no abstraction) | Simplest | Locks out tenants with compliance need for certified third-party | rejected |
| In-built default + provider abstraction | No per-sig cost, full data control, upsell flexibility | Must build audit/certificate machinery correctly | **chosen** |
| PyHanko cryptographic (AdES) now | Strongest evidence | Adds complexity; not needed for residential/commercial lease enforceability | deferred (optional upgrade) |

## Consequences

### Positive

- Zero per-envelope fees; full data residency.
- Full UX control; no third-party availability risk.
- Provider abstraction keeps DocuSign/ZohoSign as an upsell.

### Negative

- We own the legal defensibility of the audit trail (must be built correctly).
- Signature capture + PDF stamping + certificate generation are non-trivial.

## Implementation

- Feature file: `vault/esign/features/inbuilt-esign.md`
- Feature file: `vault/esign/features/provider-abstraction.md`
