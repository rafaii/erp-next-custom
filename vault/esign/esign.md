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
| [provider-abstraction](features/provider-abstraction.md) | in-progress | developer-1 | 2026-08-29 |
| [esign-template-management](features/esign-template-management.md) | in-progress | developer-1 | 2026-09-02 |
| [esign-field-placement-designer](features/esign-field-placement-designer.md) | in-progress | developer-1 | 2026-09-03 |
| [esign-pdf-template-upload](features/esign-pdf-template-upload.md) | in-progress | developer-1 | 2026-09-03 |
| [esign-guided-click-to-sign](features/esign-guided-click-to-sign.md) | in-progress | developer-1 | 2026-09-03 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/esign/features"
SORT updated DESC
```

## Related ADRs

- `0002-esign-approach` — in-built vs third-party
- `0019-esign-template-designer` — business-owner-controlled contract templates + signing-field placement
- `0020-esign-pdf-upload-and-guided-signing` — upload a PDF template instead of authoring HTML; signers tap placed fields with a generated cursive signature
