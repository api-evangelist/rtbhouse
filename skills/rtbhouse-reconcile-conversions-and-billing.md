---
name: Reconcile RTB House conversions against billing
description: Walk the cursor-paged conversions feed for an advertiser and reconcile it against the billing ledger and rate card, so reported conversion value can be tied to what was actually charged.
api: openapi/rtbhouse-advertisers-api-openapi.yml
operations:
  - GET /advertisers
  - GET /advertisers/{hash}/conversions
  - GET /advertisers/{hash}/billing
  - GET /advertisers/{hash}/rate-cards
  - GET /advertisers/{hash}/client
generated: '2026-08-13'
method: generated
source: openapi/_original/rtbhouse-client-panel-openapi.yml, data-model/rtbhouse-data-model.yml, conventions/rtbhouse-conventions.yml
---

# Reconcile RTB House conversions against billing

Base URL: `https://api.panel.rtbhouse.com/v5`. Authenticate first — see `rtbhouse-authenticate-and-rotate-token.md`.

> The published OpenAPI declares **no `operationId`**. Bind by method + path as written.

## 1. Scope to the advertiser

```
GET /advertisers/{hash}
```

Note the `currency` — the conversions feed, the rate card and the billing ledger are all denominated in it, and nothing in the payloads restates it.

Optionally `GET /advertisers/{hash}/client` for the invoicing counterparty.

## 2. Walk the conversions feed — the only paged collection in the API

```
GET /advertisers/{hash}/conversions?dayFrom=YYYY-MM-DD&dayTo=YYYY-MM-DD&limit=10000
```

The response is a container, not a bare array:

```json
{"status":"ok","data":{"rows":[...],"total":<int>,"nextCursor":"<opaque|null>"}}
```

Loop: read `data.rows`, then re-issue the same request with `nextCursor` set to the value returned. Stop when `nextCursor` is `null`. The first-party SDK sends `limit=10000`, which is the documented maximum (`MAX_CURSOR_ROWS`).

Each row carries `conversionHash`, `conversionIdentifier`, `conversionClass`, `conversionTime`, `conversionValue`, `commissionValue`, `cookieHash`, `lastClickTime`, `lastImpressionTime`.

- `conversionHash` is the stable key — dedupe on it, since a retried page can repeat rows.
- `cookieHash`, `lastClickTime` and `lastImpressionTime` are **nullable**. Never assume attribution timestamps are present.
- `commissionValue` is what RTB House charges for that conversion; `conversionValue` is the advertiser's own order value. They are different numbers and must not be summed together.

Every other collection in this API — advertisers, campaigns, offers, creatives, rate cards — returns the whole set unpaged. Only conversions use a cursor.

## 3. Pull the rate card that priced them

```
GET /advertisers/{hash}/rate-cards
```

Returns `id`, `version`, and the pricing dimensions `cpc`, `cpm`, `cpaPostClick`, `cpaPostView`, `cpsPostClick`, `cpsPostView`. Rate cards are **versioned** — record the `version` you reconciled against, because a later pull may price the same period differently.

`GET /advertisers/{hash}/campaigns` gives each campaign's `rateCardId`, so you can tell which card governed which campaign.

## 4. Pull the ledger

```
GET /advertisers/{hash}/billing
```

Returns `{"bills":[...],"initialBalance":<float>}`. Each bill row is `day`, `operation`, `position`, `credit`, `debit`, `balance`, `recordNumber` — a running ledger, not a set of keyed invoices.

Reconcile by walking `bills` in `recordNumber` order from `initialBalance` and checking that each row's `balance` equals the prior balance plus `credit` minus `debit`. A break in that chain is a paging or ordering bug on your side, not a billing discrepancy.

## 5. Tie the two together

Sum `commissionValue` over conversions in the period and compare against the debits in the ledger for the same days. Expect timing skew: `conversionTime` is when the conversion happened, ledger `day` is when it was billed, and the attribution convention used to count it is a query parameter on the stats endpoints (`countConvention`) rather than a property of the conversion row.

## 6. Failure handling

- `429` — the conversions walk is the second most expensive thing you can do to this API after wide statistics queries. Read `X-Resource-Usage`, wait out the named window (3600s or 21600s), and resume from the last `nextCursor` you successfully consumed. There is no `Retry-After`.
- `401 INVALID_CREDENTIALS` — token expired mid-walk. Re-authenticate and resume from the last cursor.
- There is **no idempotency key** on this API. Since every operation here is a `GET`, resuming from a stored cursor is safe; do not generalize that safety to `POST /tokens/current/rotate`.
