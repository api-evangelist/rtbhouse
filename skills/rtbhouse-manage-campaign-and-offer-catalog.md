---
name: Inspect and control RTB House campaigns, creatives and the product offer catalog
description: Read an advertiser's campaigns, creatives, offer categories and product offers, and safely toggle advertiser or campaign status — the only two write operations in the v5 API, both of which start or stop live ad spend.
api: openapi/rtbhouse-advertisers-api-openapi.yml
operations:
  - GET /advertisers
  - GET /advertisers/{hash}/campaigns
  - PUT /advertisers/{hash}/campaigns/{campaignHash}/status
  - PUT /advertisers/{hash}/status
  - GET /advertisers/{hash}/rtb-creatives
  - GET /advertisers/{hash}/offer-categories
  - GET /advertisers/{hash}/offers
generated: '2026-08-13'
method: generated
source: openapi/_original/rtbhouse-client-panel-openapi.yml, data-model/rtbhouse-data-model.yml, agentic-access/rtbhouse-agentic-access.yml
---

# Inspect and control RTB House campaigns, creatives and offers

Base URL: `https://api.panel.rtbhouse.com/v5`. Authenticate first — see `rtbhouse-authenticate-and-rotate-token.md`.

> The published OpenAPI declares **no `operationId`**. Bind by method + path as written.

## 1. Read the campaign surface

```
GET /advertisers/{hash}/campaigns
```

Each campaign returns `hash`, `name`, `campaignType`, `status`, `isEditable`, `creativeIds`, `rateCardId`, `advertiserLimits`, `updatedAt`.

Two fields govern what you may do next:

- **`isEditable`** — if false, do not attempt a status change. Check it before every write.
- **`status`** — the current delivery state. Read it and confirm the change you are about to make is actually a change.

`creativeIds` resolves against `GET /advertisers/{hash}/rtb-creatives` (each has `hash`, `names`, `previews`, `status`). `rateCardId` resolves against `GET /advertisers/{hash}/rate-cards`.

## 2. Read the product catalog

```
GET /advertisers/{hash}/offer-categories
GET /advertisers/{hash}/offers
```

Categories carry `categoryId`, `identifier`, `name`, `activeOffersNumber`. Offers carry `id`, `identifier`, `name`, `fullName`, `categoryName`, `price`, `currency`, `url`, `images`, `status`, `customProperties`, `updatedAt`.

Note the join: an offer names its category by **`categoryName`**, not by `categoryId`. If two categories share a name, that join is ambiguous — match on the category `identifier` where you can and treat the name join as best-effort.

Neither collection is paged. An advertiser with a large feed returns the whole array in one response, which is the expensive part of this call.

## 3. Write — both writes move money

These are the only two mutating operations in the entire v5 API, and neither is cosmetic. Both start or stop live ad delivery and therefore spend.

```
PUT /advertisers/{hash}/campaigns/{campaignHash}/status
PUT /advertisers/{hash}/status
```

Before either call:

1. `GET` the current object and read its present `status`. Skip the write if it already matches the target.
2. For a campaign, confirm `isEditable` is true.
3. Confirm the advertiser hash is the one you intend — hashes are opaque strings with no readable prefix, and there is nothing in the identifier to catch a mix-up.
4. Confirm the account is not a demo account if the intent was to affect production (`isDemoUser` from `GET /user/info`).

There is **no idempotency key** on this API. `PUT` is idempotent by HTTP semantics, so replaying the identical status change is safe; a *different* status in a retry is not a retry, it is a second decision.

An agent should treat both writes as requiring human confirmation. See `agentic-access/rtbhouse-agentic-access.yml`.

## 4. Verify

Re-`GET` the campaign or advertiser and assert the new `status` and a moved `updatedAt`. The `PUT /advertisers/{hash}/status` response returns the full advertiser object; the campaign status `PUT` returns an empty object, so a read-back is the only confirmation available there.

## 5. Failure handling

`{"status":"error","message":...,"httpCode":...,"appCode":...,"errors":...}`. The `errors` object carries per-field validation detail when the request body is rejected. `401 INVALID_CREDENTIALS`, `410` on a retired version, `429` with `X-Resource-Usage` on budget exhaustion — see `errors/rtbhouse-problem-types.yml`.
