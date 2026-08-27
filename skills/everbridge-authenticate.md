---
name: everbridge-authenticate
description: Mint an Everbridge Suite bearer token with an OAuth 2.0 service account, and pick the right grant type for the job.
api: Everbridge Suite Authentication API
operations:
  - Generate Token
generated: '2026-08-27'
method: generated
source: openapi/everbridge-authentication-openapi.json, https://developers.everbridge.net/home/docs/ebs-gs-guide-authentication-types
---

# Authenticate against the Everbridge Suite API

Every other Everbridge skill depends on this one. Do it first, cache the token, and re-mint on `401`.

## 1. Know which grant you need

| You are | Grant | Token to use | Scope string |
|---|---|---|---|
| A machine integration doing a narrow, fixed set of calls | `client_credentials` | `access_token` | `openid user-profile role service-profile` |
| An automation acting as a specific Everbridge user across many features | `password` | `id_token` | `openid user-profile role` |

Using the wrong field is the most common first failure: the client-credentials response and the
password response both contain several tokens, and only the one named above is accepted downstream.

## 2. Get credentials before you call anything

Client ID and Client Secret are issued from the Manager Portal, not from the API:

- Account level (Users, Audit logs, Roles): **Users → Service Accounts**
- Organization level (everything else): **Settings → Access → Service Accounts**

Scope the service account to the exact resource + action pairs you need. Everbridge's own example
scopes a contact workflow to *List contacts*, *Create contact*, *Get contact* while **rejecting**
*Delete contact* — do the same. Not every EB Suite method supports resource+action permissions yet.

## 3. Call the token endpoint

`POST https://api.everbridge.net/authorization/v1/tokens`
Content-Type: `application/x-www-form-urlencoded`

```
grant_type=client_credentials
client_id=<MyClientId>
client_secret=<MyClientSecret>
scope=openid user-profile role service-profile
```

If the API user holds multiple roles, or is an account admin, send a `roleId` header to pick the role
the token is minted for. **For an Account Admin user, send the Org Admin role id, not the Account
Admin role id** — this catches people out.

## 4. Use it

```
Authorization: Bearer <access_token|id_token>
```

The token is valid for **28800 seconds (8 hours)**. There is no refresh endpoint documented — mint a
new one. A `401` mid-session almost always means expiry, not bad credentials; a `403` means the token
is fine but the service account lacks the permission or the organization.

## Alternatives you will meet

- **HTTP Basic** is still supported: `Authorization: Basic base64(username:password)`. The API user
  must have API Access enabled.
- **Access Token authentication is legacy.** Everbridge labels it as such. Do not build new
  integrations on it.
- **SnapComms** (`https://api.snapcomms.com/v1`) authenticates separately with its own bearer JWT.
  An Everbridge Suite token will not work there.

## Rules that apply to every subsequent call

- Throttling is per-organization and measured in **requests per second**, not per minute. The default
  Bronze tier allows 10 reads/s and 20 writes/s with a 50,000 daily quota.
- On `429` or `5xx`, back off exponentially with jitter:
  `sleep = random_between(0, min(cap, base * 2 ** attempt))`. Reset on the first `2xx`.
- **No rate-limit response headers are published.** You must track your own budget.
