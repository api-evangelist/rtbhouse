---
name: Pull RTB House campaign performance statistics
description: Read RTB stats, summary stats, win-rate, top-hosts and deduplication statistics for one or many advertisers without exhausting the account's resource-usage budget. Covers the required date window, the groupBy/metrics contract, and why the response has no fixed schema.
api: openapi/rtbhouse-statistics-api-openapi.yml, openapi/rtbhouse-advertisers-api-openapi.yml
operations:
  - GET /advertisers
  - GET /advertisers/{hash}/rtb-stats
  - GET /advertisers/{hash}/summary-stats
  - GET /advertisers-summary-stats
  - GET /advertisers/{hash}/win-rate-stats
  - GET /advertisers/{hash}/top-hosts-stats
  - GET /advertisers/{hash}/top-in-apps-stats
  - GET /advertisers/{hash}/last-seen-tags-stats
  - GET /advertisers/{hash}/rtb-deduplication-stats
generated: '2026-08-13'
method: generated
source: openapi/_original/rtbhouse-client-panel-openapi.yml, conventions/rtbhouse-conventions.yml, rate-limits/rtbhouse-rate-limits.yml
---

# Pull RTB House campaign performance statistics

Base URL: `https://api.panel.rtbhouse.com/v5`. Authenticate first — see `rtbhouse-authenticate-and-rotate-token.md`.

> The published OpenAPI declares **no `operationId`**. Bind by method + path as written.

## 1. Resolve the advertiser hash

```
GET /advertisers
```

Returns an array of advertisers, each with `hash`, `name`, `status`, `currency`, `url`, `createdAt`, `features`, `properties`. Every statistics call is scoped by that `hash`.

## 2. Choose the endpoint that answers the question

| Question | Call |
|---|---|
| Full performance breakdown | `GET /advertisers/{hash}/rtb-stats` |
| Headline totals for one advertiser | `GET /advertisers/{hash}/summary-stats` |
| Headline totals across advertisers | `GET /advertisers-summary-stats` |
| How often we win auctions | `GET /advertisers/{hash}/win-rate-stats` → `{day, won, total}` |
| Which publishers we run on | `GET /advertisers/{hash}/top-hosts-stats` → `{host, value}` |
| Which apps we run on | `GET /advertisers/{hash}/top-in-apps-stats` → `{appName, value}` |
| Tag freshness | `GET /advertisers/{hash}/last-seen-tags-stats` → `{lastTagHour, impsCount, clicksCount}` |
| Overlap with other channels | `GET /advertisers/{hash}/rtb-deduplication-stats` |

## 3. Build the query

`dayFrom` and `dayTo` are **required** on every statistics endpoint, in `YYYY-MM-DD` form.

On `rtb-stats`, `summary-stats` and `advertisers-summary-stats` you also send:

- `groupBy` — the dimensions to break the result down by
- `metrics` — the measures to return
- `countConvention` — the attribution convention for conversion counting (e.g. attributed post-click)
- `utcOffsetHours` — timezone offset applied to day bucketing
- `subcampaigns` — subcampaign hashes, multiple values concatenated with `-`
- `userSegments`, `deviceTypes`, `creativeNamesFamilies` — further filters

## 4. Expect a schema you cannot pre-generate

The spec says it outright: each row *"contains selected groupBy fields + selected metric fields."* There is no fixed response schema for `rtb-stats`, `summary-stats`, `advertisers-summary-stats` or `rtb-deduplication-stats`.

Read the rows dynamically against the `groupBy` and `metrics` you asked for. Do not assume a column exists because it existed on a previous call with a different selection.

Also expect **floats where you might expect integers**: `imps_count`, `clicks_count`, `conversions_count` and `video_complete_views` are floats, because custom grouping and manual adjustment can produce fractional values.

## 5. Stay inside the budget — this is the real constraint

RTB House does not rate-limit by request count. It meters **resource usage**: backend worker time over a 1-hour window (`WORKER_TIME`) and BigQuery terabytes billed over a 6-hour window (`BQ_TB_BILLED`). One wide query can exhaust the budget that thousands of narrow ones would not.

- Narrow `dayFrom`/`dayTo`. Ask for a month, not a year.
- Request only the `groupBy` dimensions and `metrics` you will actually use — every extra dimension multiplies the scan.
- Cache. Historical days do not change; re-query only the recent tail.
- On `429`, parse `X-Resource-Usage` (`METRIC-WINDOWSECONDS=used/limit;…`) to learn which budget you blew, then wait for that window — 3600s or 21600s. There is no `Retry-After`.

## 6. Unwrap and check

Success is `{"status":"ok","data":[...]}` — read `data`; a missing `data` key means the response is malformed, not empty.

Failure is `{"status":"error","message":...,"httpCode":...,"appCode":...}`. See `errors/rtbhouse-problem-types.yml`.
