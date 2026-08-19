---
name: Authenticate against the RTB House Client Panel API and keep the token alive
description: Obtain, present and rotate an RTB House API v5 token so an unattended integration never loses access. Covers the three auth styles, the rotation operation, and the expiry trap that silently kills idle integrations.
api: openapi/rtbhouse-user-api-openapi.yml, openapi/rtbhouse-tokens-api-openapi.yml
operations:
  - GET /user/info
  - POST /tokens/current/rotate
  - GET /healthcheck
generated: '2026-08-13'
method: generated
source: openapi/_original/rtbhouse-client-panel-openapi.yml, conventions/rtbhouse-conventions.yml, https://github.com/rtbhouse-apps/rtbhouse-python-sdk
---

# Authenticate against the RTB House Client Panel API

Base URL: `https://api.panel.rtbhouse.com/v5`

> The published OpenAPI declares **no `operationId` on any operation**. Bind every call by HTTP method + path, exactly as written below. Do not invent operation identifiers.

## 1. Get a credential

There is no self-service signup. A Client Panel account is issued by the RTB House account team. Once you have one, mint an API token at `https://panel.rtbhouse.com/user/api-tokens`.

## 2. Pick an auth style

Three styles work; the API declares `basicAuth` and `bearerAuth` as its security schemes.

| Style | Header | Use when |
|---|---|---|
| API token (preferred) | `Authorization: Bearer <token>` | Any integration |
| Fixed token | `Authorization: Token <token>` | Legacy clients (SDK `BasicTokenAuth`) |
| HTTP Basic | `Authorization: Basic <base64(user:password)>` | Interactive/manual use only |

Never put a panel username and password in an unattended integration — use a token so it can be rotated independently of a human account.

## 3. Verify the credential

```
GET /user/info
```

A success returns the envelope `{"status":"ok","data":{...}}` where `data` carries `hashId`, `login`, `email`, `isClientUser`, `isDemoUser` and `permissions`.

Check `permissions` before attempting anything else — it is the only authorization signal the API exposes. Check `isDemoUser`: on a demo account the statistics you read are not real campaign data.

If you need a liveness check that requires no credential at all, use `GET /healthcheck`, which returns the string `SUCCESS`.

## 4. Rotate before you expire — this is the trap

RTB House API tokens have a limited lifetime **and expire if they are not actively used**. An integration that runs weekly will lose its token.

```
POST /tokens/current/rotate
```

Returns `{"status":"ok","data":{"token":"<new-token>","expiresAt":"<ISO-8601>"}}`.

Rules:

- The rotation response is the **only** time the new token is shown. Persist it atomically before you make another call; there is no read operation for the current token.
- Rotation is **not idempotent and not retry-safe**. If the call succeeds but your write fails, you have locked yourself out. Take an exclusive lock on your token store around the whole rotate-and-persist sequence.
- Rotate inside the rotation window, not on expiry. If you use the Python SDK, `ApiTokenManager` checks eligibility on every request and rotates automatically; for infrequent integrations schedule `python -m rtbhouse_sdk.api_tokens keep-alive-json` at least once a day.

## 5. Handle failure

| Status | Meaning | Do this |
|---|---|---|
| 401 `INVALID_CREDENTIALS` | Missing, malformed, expired or revoked credential | Re-mint the token in the panel. Do not retry with the same credential. |
| 410 | The API version is retired | Read `X-Current-Api-Version` and move off `/v5`. |
| 429 | Resource-usage budget exhausted | Read `X-Resource-Usage`; wait for the named window. No `Retry-After` is sent. |

Errors arrive as `{"status":"error","message":...,"httpCode":...,"appCode":...,"errors":...}` — a vendor envelope, **not** RFC 9457 problem+json.

Watch `X-Current-Api-Version` on every successful response too: if it differs from `v5`, you are on an outdated version and should plan the move before you get a 410.
