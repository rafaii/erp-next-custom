# ADR-0005: Two-Party Counter-Signature & PDC-Driven Invoicing

- **Status**: accepted
- **Date**: 2026-08-16
- **Deciders**: Imran
- **Supersedes**: -
- **Superseded by**: -

## Context

Lease execution today is one-party: the customer signs and the contract is
finalized. In practice the agreement must also be executed by the landlord
(business signatory) to be binding, and payment is typically collected as
post-dated cheques (PDCs) whose dates define when rent is invoiced.

We need: (1) a second, sequential signer (customer → business signatory),
(2) a PDC model, (3) invoicing scheduled from PDC dates instead of the fixed
monthly cadence, (4) idempotent creation (no duplicates), and (5) a complete
audit trail covering created/opened/signed/counter-signed.

## Decision

1. **Two-party sequential signing.** `send_lease_for_signature` adds two signers:
   customer (`signing_order=1`) and the business signatory (`signing_order=2`).
   The document finalizes only when **all** signers have signed. After the
   customer signs, the business signatory is emailed to counter-sign.

2. **`Signature Settings` singleton** (`is_single`) with `esign_mode`
   (In-Built | DocuSign | ZohoSign) and `business_signatory` (Link → User).
   Sending requires `business_signatory` to be set; if missing, throw a clear
   error. This is the same singleton planned for provider abstraction (ADR-0002).

3. **`PDC Entry` standalone DocType** (per MASTERPLAN §3.4): `pdc_id` (auto),
   `lease`, `customer` (fetched), `check_number`, `check_date`, `amount`,
   `bank_account`, `status` (Pending/Deposited/Cleared/Bounced), `deposit_date`.

4. **PDC-driven payment schedule.** `build_payment_schedule` derives the `Rent
   Schedule` rows from the lease's PDC `check_date`s (amount = PDC amount) when
   PDCs exist; otherwise it falls back to the existing monthly/quarterly/annual
   cadence. It is **idempotent** — it builds only when the schedule is empty.

5. **Idempotent triggers with dedup.** The schedule is created at send time (if
   PDCs exist) and again at finalize as a fallback; both paths check
   "already set?" before creating, so nothing is created twice. First invoice is
   created at finalize (both-signed), reusing the existing dedup-by-status logic.

6. **Extended audit trail.** `E-Sign Audit Log` gains `Created` (on ESD insert)
   and `Counter Signed` (when a `signing_order>1` signer signs) events; existing
   `Viewed`/`Signed` events already record the signer's email, satisfying
   "opened by customer / signed by customer / counter-signed by admin".

## Alternatives Considered

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| Single-signer (status quo) | Simple | Not legally complete; no landlord execution | rejected |
| PDC as child table on Lease | One less DocType | No independent PDC lifecycle (deposit/cleared/bounced) | rejected |
| Monthly-only schedule | Already built | Ignores PDC dates; wrong invoicing dates | rejected |
| Third-party counter-sign (DocuSign) | No build | Cost, data residency, breaks in-built default (ADR-0002) | rejected |

## Consequences

### Positive

- Contract is executed by both parties (legal completeness).
- Invoicing aligns to actual PDC cashflow dates.
- Duplicate-safe (idempotent schedule + first-invoice creation).
- Complete, tamper-evident audit trail per document.

### Negative

- Sequential signing adds coordination (customer must sign before counter-sign).
- Requires `Signature Settings` to be configured before sending.
- PDC completeness is the operator's responsibility (schedule only covers the
  PDC dates supplied).

## Implementation

- Feature file: `vault/esign/features/counter-signature.md`
- Feature file: `vault/payments-accounting/features/pdc-processing.md`
