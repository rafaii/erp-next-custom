# Email System

## Purpose

Platform-wide, OS-agnostic outgoing mail: strips ERPNext's own branding from
every email the platform sends and replaces it with the sending business's
own logo/address plus a consistent "Powered by Aetris" credit — identical
whether the tenant sends through their own SMTP or Aetris-managed (Brevo)
delivery. Any OS app declares this module as a `required_apps` dependency,
the same pattern `inbuilt_esign`/`accounts_portal` already follow.

## Features

| Feature | Status | Owner | Updated |
| --- | --- | --- | --- |
| [outgoing-mail-branding](features/outgoing-mail-branding.md) | in-progress | developer-1 | 2026-09-04 |

## Dataview (auto)

```dataview
TABLE status, owner, updated
FROM "vault/email-system/features"
SORT updated DESC
```

## Related ADRs

- `0021-email-system-outgoing-mail-branding` — new module app, replaces
  ERPNext branding with business + Aetris branding
