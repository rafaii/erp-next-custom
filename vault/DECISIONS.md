# Decisions Index (ADRs)

Approved architectural decisions, sequential numbering.

## Approved

| ADR | Title | Status |
| --- | --- | --- |
| [0001-app-repo-structure](0001-app-repo-structure.md) | App & Git Repository Structure | accepted |
| [0002-esign-approach](0002-esign-approach.md) | E-Signature — In-Built vs Third-Party | accepted |
| [0003-multi-tenancy](0003-multi-tenancy.md) | Multi-Tenancy — Tenant-per-Site vs Single-DB | accepted |
| [0004-recurring-invoicing](0004-recurring-invoicing.md) | Recurring Invoicing Mechanism | accepted |
| [0005-counter-signature-pdc](0005-counter-signature-pdc.md) | Two-Party Counter-Signature & PDC-Driven Invoicing | accepted |

## Drafts

_(none pending)_

## Numbering rules

- Sequential, zero-padded, chronological (`0001`, `0002`, ...). Never reuse or renumber.
- New ADR flow: create draft in `decisions/drafts/draft-adr-<slug>.md` → get approval → number + move to `decisions/000N-<slug>.md` → delete draft → add row above.
