---
name: everbridge-query-cem-alerts
description: Query Everbridge CEM alerts over GraphQL and subscribe to the live alerts stream — Everbridge's only push surface.
api: Everbridge CEM Alerts API
operations:
  - GraphQL Query
generated: '2026-08-27'
method: generated
source: openapi/everbridge-cem-alerts-query-public-openapi.json, openapi/everbridge-cem-alerts-query-stream-openapi.json
---

# Query and stream CEM alerts

CEM Alerts is GraphQL-native. The REST contract declares exactly one operation per service —
`POST /graphql` — and everything else is expressed in the query document.

## 1. Authenticate

Follow `everbridge-authenticate` and send the bearer token. An anonymous POST returns
`401 {"message":"Unauthorized"}`, and **introspection is gated too** — you cannot read the schema
without credentials, so build queries from the documented examples below rather than expecting
`__schema` to answer.

## 2. Query (request/response)

`POST https://api.everbridge.net/cem-alerts/query/v1/graphql`

Get one alert:
```graphql
query { alert(id: "<alertId>") { alertId organization status } }
```

List alerts with paging (`token` is the continuation cursor returned in the response):
```graphql
query {
  alerts(filter: {}, token: "") {
    results { alertId organization status expiresAt snoozedUntil isActive owner systemLinks }
    token
  }
}
```

Pull impact geometry for an alert:
```graphql
query {
  alert(id: "<alertId>") {
    summaries(filter: {summaryKinds: COORDINATES}) { summaryId summaryName }
  }
}
```

## 3. Subscribe (streaming)

`POST https://api.everbridge.net/cem-alerts/stream/v1/graphql`, over an open WebSocket:

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

This is **the only push/event surface Everbridge publishes** — there is no webhook catalogue and no
AsyncAPI document anywhere in the platform. If you need event-driven behaviour, it is this or polling.

The stream operation sets `x-readme.explorer-enabled: false`, so it cannot be exercised from the
developer hub's Try It console; test it from your own WebSocket client.

## 4. Errors

The query service declares `403`, `404` and `424 Failed Dependency` in addition to `200`. A `424`
means an upstream CEM Alerts dependency failed — retry with exponential backoff and jitter. Note that
GraphQL may also return `200` with an `errors[]` array; check the body, not only the status.

## Related

For country-level risk ratings rather than live alerts, use the Travel Risk Intelligence API
(`https://api.everbridge.net/travel/v1`, ISO-2 country codes, added 26.1). For a bulk risk-event
export, the Risk Intelligence Feed (`https://api.everbridge.net/riskevents/v1/riskevents`) returns up
to 1,000 results per invocation over a year of history — it requires a separate subscription.
