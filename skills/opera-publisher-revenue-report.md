---
name: Pull an Opera Ads publisher revenue report
description: >-
  Query the Opera Ads publisher (OFP) Report API for revenue, requests,
  impressions, clicks, CTR, eCPM, fill rate and show rate, broken down by app,
  placement, country, traffic type, SDK ad format or OS.
api: openapi/opera-publisher-report-api-openapi.yml
operations:
  - queryPublisherReport
  - getInventoryReport
---

# Pull an Opera Ads publisher revenue report

Use this when a publisher needs their own delivery and revenue numbers out of
Opera Ads. Prefer `queryPublisherReport` — `getInventoryReport` is the older
token-in-query endpoint and returns a narrower row shape.

## Before you start

- You need an Opera Ads publisher auth token. It is issued by Opera through the
  publisher console; there is no self-serve key and no test token. See
  `authentication/opera-authentication.yml`.
- Send it as `Authorization: Bearer <token>`.

## Steps

1. **Build the filter set.** `filters[]` is required and MUST include a `day`
   filter whose value is a two-element `["YYYYMMDD", "YYYYMMDD"]` array. Add
   `app_id` or `slot_id` filters (array of IDs) to narrow the query.

2. **Choose your measurements.** `measurements[]` is required. Each entry is
   `{name, sort}` where `sort` is `"desc"`, `"asc"`, or `""` for no sorting.
   - Dimensions: `day`, `publisher_id`, `app_id`, `slot_id`, `country_code`,
     `traffic_type`, `sdk_ad_format`, `os_type`
   - Metrics: `costs`, `cost_ecpm`, `requests`, `responses`, `impressions`,
     `clicks`, `ctr`, `cpc`, `fillrate`, `showrate`
   Ask for a metric-only measurement list to get a daily summary; add
   dimensions to get a breakdown.

3. **Call `queryPublisherReport`** — `POST /openapi/report/v1/query` on
   `https://ofp.adx.opera.com`. `limit` is REQUIRED; `offset` defaults to 0.

4. **Read the envelope.** Success is `{message, rows, statusCode, total}` with
   `statusCode: 0`. Note that this endpoint's success shape differs from the
   `{code, msg, data}` shape used elsewhere in the Opera Ads API — do not
   assume one envelope across Opera endpoints
   (`conventions/opera-conventions.yml`).

5. **Page with `offset`/`limit`**, not `page`/`page_size`. Compare the running
   row count against `total`. The advertiser Report API uses `page`/`page_size`
   instead, so do not carry pagination code between the two.

6. **Numbers come back as strings.** `costs`, `ctr`, `showrate` and the rest
   are decimal strings, not JSON numbers. Parse before arithmetic.

## Failure handling

- `400` invalid parameters, `401` invalid or missing token, `403` insufficient
  scope, `500` server error — all with `{code, msg}`.
- `429` means rate limited. No `Retry-After` and no `RateLimit-*` headers are
  sent, so back off exponentially on your own schedule
  (`rate-limits/opera-rate-limits.yml`).
- **No idempotency key exists.** Reads are safe to retry; nothing else on this
  platform is made safe by the API.

## If you are using the legacy endpoint

`getInventoryReport` is `GET /openapi/reports?token=…&start_date=…&end_date=…`.
The credential travels in the URL — keep it out of logs and referrers. Dates are
`YYYY-MM-DD`. A `revenue` of `-1` means the figure is still being computed, not
zero revenue.
