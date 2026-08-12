---
name: streetmetrics-build-a-campaign
description: Build an out-of-home campaign in StreetMetrics end to end — create the campaign, add a transit or stationary ad group, attach the inventory (assets or frames) and the creatives.
api: StreetMetrics Public API
base_url: https://dashboard.streetmetrics.io/v3/public/
operations:
  - AuthController_authenticate
  - CampaignController_createCampaign
  - CampaignController_getCampaigns
  - AdGroupController_createTransitAdGroup
  - AdGroupController_createStationaryAdGroup
  - AssetController_getCompatibleAssets
  - AssetController_getCompatibleAssetsByAdGroupId
  - AdGroupController_attachAssets
  - AdGroupController_putUpdateAssets
  - FramesController_getFrames
  - AdGroupController_attachFrames
  - AdGroupController_putUpdateFrames
  - PublicCreativesController_getCreatives
  - AdGroupController_attachCreatives
  - AdGroupController_getAdGroup
generated: '2026-08-12'
method: generated
source: openapi/streetmetrics-public-api-openapi.json + https://docs.streetmetrics.com/recipes/creating-base-campaigns + https://docs.streetmetrics.com/recipes/attach-inventory-to-your-ad-group
---

# Build a StreetMetrics campaign

The hierarchy is **Campaign → Ad Group (called a "flight" in payloads) → inventory + creatives**.
An ad group is either *transit* (carries **assets** — vehicles) or *stationary* (carries **frames** —
fixed faces). The two branches never mix: attaching frames to a transit ad group is not a supported
path.

Authenticate first — see `streetmetrics-authenticate`.

## 1. Create the campaign

`CampaignController_createCampaign` — `POST /campaigns` (body `CreateCampaignDto`, returns `201`).

Keep your own identifier in `campaignRef`; that is the field you will match on later.
Verify with `CampaignController_getCampaigns` (`GET /campaigns`) using the search DSL, e.g.
`?search=campaignRef.ilike:<your-ref>`.

## 2. Create the ad group

- Transit: `AdGroupController_createTransitAdGroup` — `POST /ad-groups/transit`
  (`CreateTransitAdGroupDto`)
- Stationary: `AdGroupController_createStationaryAdGroup` — `POST /ad-groups/stationary`
  (`CreateStationaryAdGroupDto`)

Both return `201`. The response id field is `flightId`, not `adGroupId` — the resource is named
`ad-groups` but the payload vocabulary is "flight".

## 3. Attach inventory

**Transit** — find eligible vehicles first:
- `AssetController_getCompatibleAssets` — `GET /assets/compatible`
- or, once the ad group exists, `AssetController_getCompatibleAssetsByAdGroupId` —
  `GET /assets/ad-group/{adGroupId}/compatible`

Then `AdGroupController_attachAssets` — `POST /ad-groups/{id}/assets` (`201`).
To replace the whole set rather than add to it, use `AdGroupController_putUpdateAssets` —
`PUT /ad-groups/{id}/assets` (`200`).

**Stationary** — list faces with `FramesController_getFrames` (`GET /frames`), then
`AdGroupController_attachFrames` — `POST /ad-groups/{id}/frames` (`201`), or
`AdGroupController_putUpdateFrames` — `PUT /ad-groups/{id}/frames` to replace.

POST adds, PUT replaces. There is no partial-remove operation for attached inventory.

## 4. Attach creatives

`PublicCreativesController_getCreatives` — `GET /creatives` to find them, then
`AdGroupController_attachCreatives` — `POST /ad-groups/{id}/creatives` (`201`). This one operation
serves both stationary and transit ad groups.

## 5. Verify

`AdGroupController_getAdGroup` — `GET /ad-groups/{id}?include=media,assets,frames`. The `include`
parameter is the only expansion mechanism, and `media`, `assets`, `frames` are the only documented
values.

## Rules that apply to every step

- **Retries are not safe.** There is no idempotency key. A retried `POST /campaigns` creates a second
  campaign. Read back with the search DSL before retrying a write.
- **Pagination is cursor-based**: `?limit=&cursor=&sort=&order=ASC|DESC`; follow `nextCursor` from the
  previous response until it is null. Documented max `limit` is 10000.
- **Filtering** uses `search=fieldName.operation:value`, chained with commas —
  e.g. `search=status.eqo:PENDING,status.eqo:APPROVED` (the trailing `o` means OR, `a` means AND).
- **Errors** come back as `{status, statusCode, errorCode, message, details, timestamp, path}` — not
  RFC 9457. See `errors/streetmetrics-problem-types.yml`.
- Every response wraps its payload as `{statusCode, message, meta, data}`; read your entity from `data`.
