---
name: everbridge-manage-contacts
description: Create, find, partially update and bulk-load Everbridge contacts and groups without round-tripping whole objects or leaving orphaned records.
api: Everbridge Suite API
operations:
  - EBS List Contacts
  - EBS Get Contact
  - EBS Create Contact
  - EBS Update Contact Partial
  - EBS Update Contact
  - EBS Query Contacts Batch
  - EBS Create Contacts Batch
  - EBS Update Contacts Batch
  - EBS Delete Contact
  - EBS Upload Contacts
  - EBS Get Contacts Upload Status
  - EBS List Groups
  - EBS Create Group
  - EBS Add Contacts Group
generated: '2026-08-27'
method: generated
source: openapi/everbridge-eb-suite-openapi.json
---

# Manage Everbridge contacts and groups

## 1. Authenticate

Follow `everbridge-authenticate`. Every path is organization-scoped: `{organizationId}` is the first
path parameter and a wrong one returns `403`, not `404`.

## 2. Find before you create

- `EBS List Contacts` — `GET /contacts/{organizationId}` with `pageNumber` + `pageSize`
- `EBS Query Contacts Batch` — `POST /contacts/{organizationId}/query` for filtered lookups
- `EBS Get Contact` — `GET /contacts/{organizationId}/{id}`

Contact search supports relative-timeframe filtering on date fields (added 25.11) and filtering on
bounced email state via `emailStatus=bounced` (added 25.9) — useful for data-quality sweeps.

## 3. Write

| Intent | Operation | Path |
|---|---|---|
| Create one | `EBS Create Contact` | `POST /contacts/{organizationId}` |
| Change a few fields | `EBS Update Contact Partial` | `PATCH /contacts/{organizationId}/{id}` |
| Replace the whole record | `EBS Update Contact` | `PUT /contacts/{organizationId}/{id}` |
| Create many | `EBS Create Contacts Batch` | `POST /contacts/{organizationId}/batch` |
| Update many | `EBS Update Contacts Batch` | `PUT /contacts/{organizationId}/batch` |

**Use PATCH.** Everbridge added partial update specifically so integrations would stop retrieving and
re-sending entire objects. A PUT with a field you did not intend to change will change it.

Since release 26.3, creating or updating a contact with a duplicate **Location Name + Location Type**
combination is rejected. Code that used to succeed against older releases will now return `400`.

## 4. Bulk load

- `EBS Upload Contacts` — `POST /uploads/{organizationId}`
- `EBS Get Contacts Upload Status` — `GET /uploads/{organizationId}/{uploadBatchId}`
- `EBS Get Contacts Upload Details` — `GET /uploadContacts/{organizationId}/{uploadBatchId}`

Uploads are asynchronous. Poll the status endpoint with backoff; do not re-submit the batch because
there is no idempotency key and a resubmission creates duplicates.

## 5. Groups

- `EBS Create Group` — `POST /groups/{organizationId}`
- `EBS Add Contacts Group` — `POST /groups/{organizationId}/contacts`
- `EBS Replace Contacts SeqGroup` — `POST /groups/{organizationId}/contacts/sequence` for ordered
  escalation groups
- `EBS List Dynamic Groups` — `GET /dynamicGroups/{organizationId}` for rule-based membership

## 6. Deletion is permanent

`EBS Delete Contact` (`DELETE /contacts/{organizationId}/{id}`) and
`EBS Delete Contacts Batch` (`DELETE /contacts/{organizationId}/batch`) have **no restore, no
undelete and no published retention window**. Read the record first, keep your own copy, and require
explicit confirmation. Deleting a contact who is a member of a notification group silently shrinks
the audience of every future notification that targets it.

## Conventions

- Pagination: `pageNumber` + `pageSize`, default page size 10. Some endpoints use `pageNo` or `page`
  instead — check the operation, do not assume.
- Ids are bare numeric strings with no type prefix. Track which resource an id came from yourself.
- Errors are plain JSON with a free-text `message` field. There is no machine-readable error code.
