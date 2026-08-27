---
generated: '2026-08-27'
method: probed
source: openapi/everbridge-cem-alerts-query-public-openapi.json, openapi/everbridge-cem-alerts-query-stream-openapi.json
introspection: gated
---

# Everbridge CEM Alerts GraphQL API

Everbridge's Critical Event Management Alerts service is GraphQL-native. Two separate GraphQL
endpoints are published, each declared in its own OpenAPI document on the Everbridge developer hub
with `POST /graphql` as the single operation.

## Endpoints

| Service | Endpoint | Transport |
|---|---|---|
| CEM Alerts Query Public | `https://api.everbridge.net/cem-alerts/query/v1/graphql` | HTTP request/response |
| CEM Alerts Query Stream | `https://api.everbridge.net/cem-alerts/stream/v1/graphql` | WebSocket subscription |

## Introspection is gated

```
POST https://api.everbridge.net/cem-alerts/query/v1/graphql
{"query":"{__schema{queryType{name}}}"}
-> 401 {"message":"Unauthorized"}

POST https://api.everbridge.net/cem-alerts/stream/v1/graphql
-> 401 (empty body)
```

Probed 2026-08-27. No SDL is saved in this repo, because none could be read without credentials and
fabricating one would be worse than recording the gap. The root fields below are transcribed verbatim
from the request examples Everbridge publishes inside its own OpenAPI documents — they are real, but
they are not the whole schema.

## Known query root fields

```graphql
query { alert(id: "<alertId>") { alertId organization status } }
```

```graphql
query {
  alerts(filter: {}, token: "") {
    results { alertId organization status expiresAt snoozedUntil isActive owner systemLinks }
    token
  }
}
```

```graphql
query {
  alert(id: "<alertId>") {
    summaries(filter: {summaryKinds: COORDINATES}) { summaryId summaryName }
  }
}
```

## Known subscription

```graphql
query { subscription read { alertsStream {
  alertId organization status owner snoozedUntil isActive
  lastEvent { title correlations { impactGeometry } }
  alertActions {
    actionType
    extendedAttributes { key value }
    user
    userInfo { email firstName id lastName userName userStatus }
  }
} } }
```

This subscription is the only push/event surface Everbridge publishes across its entire API estate.
There is no webhook catalogue and no AsyncAPI document.

## Notes

- Authentication is the standard Everbridge Suite bearer token — see
  `authentication/everbridge-authentication.yml`.
- The stream operation sets `x-readme.explorer-enabled: false` and cannot be executed from the
  developer hub's Try It console.
- The query service declares `403`, `404` and `424 Failed Dependency` alongside `200`.
- `token` on `alerts` is a continuation cursor — the only cursor-style paging anywhere in the
  Everbridge surface; the REST APIs are all `pageNumber`/`pageSize`.

**Documentation:** https://developers.everbridge.net/home/reference/cema-query-public
