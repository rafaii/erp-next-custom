# E-Sign

## Purpose

In-built e-signature for lease agreements: hash-based tamper-evidence, OTP
identity verification, explicit electronic consent, signature capture, immutable
audit log, PDF stamping + Certificate of Completion. Provider abstraction lets a
tenant choose DocuSign/ZohoSign instead.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [inbuilt-esign](features/inbuilt-esign.md) | done | developer-1 | 2026-08-17 |
| [counter-signature](features/counter-signature.md) | done | developer-1 | 2026-08-17 |
| [provider-abstraction](features/provider-abstraction.md) | planned | developer-1 | 2026-08-16 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/esign/features"
SORT updated DESC
```

## Related ADRs

- `0002-esign-approach` — in-built vs third-party
