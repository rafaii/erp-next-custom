---
status: planned
owner: ui-designer-1
domain: ui-portal
created: 2026-08-16
updated: 2026-08-16
related_adr: []
---

# Maintenance Portal

## Summary

Tenant-facing web form to submit maintenance requests with photo upload, and a
staff queue to assign/track.

## Requirements

- Tenant form: issue type, description, priority, photo upload.
- Status tracking for the submitting tenant.
- Staff queue with assignment.

## Design

- Frappe Web View/Form backed by `Maintenance Request` DocType.
- File upload field on the request.

## Implementation Plan

- [ ] Tenant submission form + photo upload
- [ ] Status-tracking view for tenant
- [ ] Staff queue + assignment UI

## Acceptance Criteria

- [ ] Tenant submits request with photo; sees status updates

## Related

- Domain index: `vault/ui-portal/ui-portal.md`
- Feature: `maintenance/features/maintenance-request-doctype.md`
