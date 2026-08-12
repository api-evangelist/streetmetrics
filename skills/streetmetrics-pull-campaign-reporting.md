---
name: streetmetrics-pull-campaign-reporting
description: Pull StreetMetrics delivery reporting for a campaign — impressions by date, ad group or unit, plus demographics, affinities and uniques-and-frequency — with the right date window, timezone and summary bin.
api: StreetMetrics Public API
base_url: https://dashboard.streetmetrics.io/v3/public/
operations:
  - AuthController_authenticate
  - CampaignController_getCampaigns
  - AdGroupController_getAdGroups
  - ReportingController_getImpressionsByDate
  - ReportingController_getImpressionsByAdGroup
  - ReportingController_getImpressionsByUnit
  - ReportingController_getDemographics
  - ReportingController_getAffinities
  - ReportingController_getUfCampaign
  - ReportingController_getUfAdGroup
  - ReportingController_getUfUnit
generated: '2026-08-12'
method: generated
source: openapi/streetmetrics-public-api-openapi.json
---

# Pull StreetMetrics campaign reporting

Reporting is read-only and comes in three grains — **campaign**, **ad group**, **unit** — across five
measures. Pick the grain first, then the measure.

Authenticate first — see `streetmetrics-authenticate`.

## 1. Resolve the campaign

`CampaignController_getCampaigns` — `GET /campaigns?search=campaignRef.ilike:<your-ref>`.
Take `campaignId` from `data`. If you need ad-group ids too, `AdGroupController_getAdGroups` —
`GET /ad-groups` (the id field is `flightId`).

## 2. Impressions

| Grain | Operation | Path |
|---|---|---|
| date | `ReportingController_getImpressionsByDate` | `GET /reporting/impressions/date` |
| ad group | `ReportingController_getImpressionsByAdGroup` | `GET /reporting/impressions/ad-group` |
| unit | `ReportingController_getImpressionsByUnit` | `GET /reporting/impressions/unit` |

Scope with `campaignId`, `startDate`, `endDate`. All three can return **`204 No Content`** — that is a
successful call with no delivery in the window, not an error. Handle it before parsing.

## 3. Audience

- `ReportingController_getDemographics` — `GET /reporting/demographics` (campaign grain)
- `ReportingController_getAffinities` — `GET /reporting/affinities`

## 4. Uniques & frequency

- `ReportingController_getUfCampaign` — `GET /reporting/uf/campaign/{campaignId}`
- `ReportingController_getUfAdGroup` — `GET /reporting/uf/ad-group/campaign/{campaignId}`
- `ReportingController_getUfUnit` — `GET /reporting/uf/unit/campaign/{campaignId}`

Note the shape: the campaign id is a **path** parameter here, unlike the impressions endpoints where it
is a query parameter.

## Time handling

- `startDate` / `endDate` bound the window.
- `timezone` controls how days are bucketed — set it explicitly rather than relying on a server
  default, or day boundaries will not match the client's reporting.
- On the attribution conversion endpoints the same idea appears as `summaryBin`:
  `LIFETIME | WEEK | MONTH | DAY | HOUR | DOW`. When `summaryBin` is set the response carries a
  `summaryBinGrouping` field.

## Operating rules

- Impression and uniques-and-frequency reporting declares `429 Too Many Requests` but publishes no
  limit and returns no `RateLimit-*` or `Retry-After` header. Back off on your own schedule and do not
  poll tightly when backfilling many campaigns.
- Collections paginate by cursor (`limit`, `cursor`, `sort`, `order`); follow `nextCursor` until null.
- Every payload is wrapped as `{statusCode, message, meta, data}`.
