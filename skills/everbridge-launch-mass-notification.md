---
name: everbridge-launch-mass-notification
description: Launch an Everbridge mass notification to contacts or groups and read its delivery report — including what you must check first, because this action cannot be taken back.
api: Everbridge Suite API
operations:
  - EBS List Groups
  - EBS List Contacts
  - EBS List Notification Templates
  - EBS Launch Mass Notification
  - EBS List Mass Notifications
  - EBS Get Notification
  - EBS Get Notification Report
  - EBS Mass Notification Report
generated: '2026-08-27'
method: generated
source: openapi/everbridge-eb-suite-openapi.json
---

# Launch a mass notification

> **STOP AND READ.** `EBS Launch Mass Notification` places phone calls, sends SMS and pushes alerts to
> real people. The EB Suite contract exposes **no cancel, stop or recall operation** for a launched
> mass notification, and Everbridge supports **no idempotency key**, so a retry after a network
> timeout may send the whole thing twice with no way to tell. Treat this as a one-way door. Require a
> human confirmation before calling it, and never retry it blindly.

## 1. Authenticate

Follow `everbridge-authenticate`. You need a bearer token and an `organizationId` — every path below
is organization-scoped.

## 2. Resolve your audience before you compose

- `EBS List Groups` — `GET /groups/{organizationId}`
- `EBS List Groups By Page` — `GET /groups/{organizationId}/pagination` for large accounts
- `EBS List Contacts` — `GET /contacts/{organizationId}` (`pageNumber` + `pageSize`, default page
  size 10, up to 1000 on Users)
- `EBS List Group Contacts` — `GET /contacts/groups/{organizationId}` to see who is actually in a group

Confirm the audience size here. An empty or wrong group is the failure mode that matters.

## 3. Prefer a template

- `EBS List Notification Templates` — `GET /notificationTemplates/{organizationId}`
- `EBS Get Notification Template` — `GET /notificationTemplates/{organizationId}/{notificationTemplateId}`

Templates carry the approved wording, delivery paths and escalation your organization already agreed
to. Composing from scratch bypasses that review.

## 4. Launch

`EBS Launch Mass Notification` — `POST /notifications/{organizationId}`

For push-only delivery use `EBS Launch Push Notification` — `POST /notifications/push/{organizationId}`.

Record the returned notification id immediately. Without it you cannot read the report, and there is
no idempotency key to recover the identity of a launch whose response you lost.

## 5. Read the outcome

- `EBS Get Notification` — `GET /notifications/{organizationId}/{notificationId}`
- `EBS Get Notification Report` — `GET /notificationReports/{organizationId}/{notificationId}`
- `EBS Mass Notification Report` — `GET /notifications/{organizationId}/reports`
- `EBS List Mass Notifications` — `GET /notifications/{organizationId}`

`EBS Update Notification` (`PUT /notifications/{organizationId}/{notificationId}`) is the only
post-launch write. The contract does not document it as a stop, so do not rely on it as one.

## Error handling

| Status | What it means here | What to do |
|---|---|---|
| 400 | Payload or business-logic violation | Fix the payload. Retrying the same body fails identically. |
| 401 | Token expired (8h TTL) | Re-mint and retry once. |
| 403 | Service account lacks the permission, or the wrong organizationId | Do not retry. Check the service-account scope. |
| 429 | Over the per-second or daily quota | Exponential backoff with jitter. |
| 500 | Server-side; detail is in the free-text `message` field | Backoff. **Do not blind-retry a launch** — you cannot tell whether it sent. |

## If you need a reversible surface instead

The newer **Communications API** (`https://api.everbridge.net/managerapps/communications/v1`) does
have a reversal: `patchStopCem-comms` — `PATCH /{commId}/stop`. If your workflow needs the ability to
stop an in-flight message, build on Communications rather than on EB Suite mass notifications. See
`everbridge-launch-and-stop-communication`.
