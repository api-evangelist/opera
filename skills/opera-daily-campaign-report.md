---
name: Pull an Opera Ads daily campaign report
description: Authenticate to the Opera Ads Report API and retrieve daily performance metrics for advertiser campaigns.
api: openapi/opera-report-api-openapi.yml
operations: [getCampaignDailyReport]
---

# Pull an Opera Ads daily campaign report

Use the Opera Ads Report API to fetch daily performance metrics (impressions,
clicks, conversions, spend) for your advertiser campaigns.

## Auth
- Add header `Authorization: Bearer <token>` (token issued by Opera Ads — contact them to obtain one).
- Add header `Content-Type: application/json`.

## Steps
1. Call `getCampaignDailyReport` — `POST https://ofa.adx.opera.com/oapi/v1/report/campaign_daily`.
   - Include a `filters` array with a required `day` range as `["YYYYMMDD","YYYYMMDD"]` (max 90 days).
   - Optionally narrow by `order_id`, `lineitem_id`, or `creative_id`.
   - Optionally set `measurements` (e.g. `spent`, `impressions`, `clicks`, `conversions`).
   - Page with `page` and `page_size` (default 50, max 1000).
2. Read the response envelope: `code` must be `0` for success; results are in `data.list` with `data.total` for pagination.

## Rules
- Rate limit is 100 requests/minute — back off with exponential retry on HTTP 429 (`code` 3).
- Error envelope is `{ code, message }` (see errors/opera-problem-types.yml): 1 invalid params, 2 auth, 3 rate limit, 4 server.
