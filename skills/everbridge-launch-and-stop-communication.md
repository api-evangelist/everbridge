---
name: everbridge-launch-and-stop-communication
description: Launch an Everbridge communication and stop or expire it while it is still in flight — the reversible alternative to a mass notification, including CAP/IPAWS public warning delivery.
api: Everbridge Communications API
operations:
  - Launch Communication
  - patchStopCem-comms
  - patchStopComm
  - deactivateCem-commsPublishOptionMessage
  - getCem-commsCommIdActivities
generated: '2026-08-27'
method: generated
source: openapi/everbridge-communications-openapi.json
---

# Launch a communication you can stop

The Communications API (`https://api.everbridge.net/managerapps/communications/v1`) is the surface to
build on when a workflow may need to be recalled. Unlike EB Suite mass notifications, it exposes an
explicit stop.

## 1. Authenticate

Follow `everbridge-authenticate`. Communications accepts the same bearer token.

## 2. Launch

`Launch Communication` — `POST /`

For a public-safety message, the payload carries a **CAP** block. Everbridge models the OASIS Common
Alerting Protocol natively: `CAPPublicMessage` requires `paths[]` (delivery modalities `IPAWS`,
`CAP_GOOGLE`, `CAP_RSS`) and a `cap` object whose required fields are `capCategory`, `capEventType`,
`capEventCode`, `capSeverity`, `capUrgency`, `capCertainty`. If you already hold a CAP alert, these
map straight across — no translation layer needed. A modality may be targeted by at most one CAP
public message per communication.

Capture the returned `commId`. Everything below needs it.

## 3. Watch it

`getCem-commsCommIdActivities` — `GET /{commId}/activities`

Contacts can respond to polling requests **more than once** (26.1), so activity state evolves; read
the latest, do not cache the first response.

`Get Communication Settings` — `GET /{commId}/settings`

## 4. Stop it

| Intent | Operation | Path |
|---|---|---|
| Stop an in-flight communication | `patchStopCem-comms` | `PATCH /{commId}/stop` |
| Same, alias route | `patchStopComm` | `PATCH /comm/{commId}/stop` |
| Deactivate or expire one publish-option message | `deactivateCem-commsPublishOptionMessage` | `PATCH /{commId}/publish-options` |

> **Everbridge publishes no window for any of these.** The contract names the stop operation but no
> documentation states how long after launch it remains effective, or what happens to messages
> already handed to a carrier. Assume anything already delivered stays delivered, and treat stop as
> best-effort suppression of the remainder — not as an undo.

## 5. Templates and plans

- `deleteTemplate` — `DELETE /templates/{templateId}`
- `deleteGlobalTemplate` — `DELETE /global-templates/{id}`
- `deletePlan` — `DELETE /plans/{planId}`
- `delete-template-reservations` — `DELETE /templates/reservations/{templateId}`

All permanent, all with no restore.

## Error handling

The Communications API declares `409 Conflict` on several operations — reconcile the resource state
rather than retrying. Everything else follows the platform table: `400` payload/business-logic,
`401` expired token, `403` permission or wrong organization, `404` missing object, `429` throttled,
`500` server-side with detail in `message`. Six operations in this contract declare **no
operationId**, so bind those by method + path.
