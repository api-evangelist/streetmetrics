---
name: streetmetrics-run-an-attribution-study
description: Stand up a StreetMetrics attribution study — create the conversion pixel, bind it to the campaign and the study, then read conversions by date, ad group or unit and download the study file.
api: StreetMetrics Public API
base_url: https://dashboard.streetmetrics.io/v3/public/
operations:
  - AuthController_authenticate
  - PixelController_createPixel
  - PixelController_getPixels
  - PixelController_getPixel
  - PixelController_updatePixel
  - PixelController_attachCampaignToPixel
  - PixelController_attachAttributionStudyToPixel
  - PixelController_detachCampaignFromPixel
  - PixelController_detachAttributionStudyFromPixel
  - AttributionStudyController_createAttributionStudy
  - AttributionStudyController_getAttributionStudies
  - AttributionStudyController_getAttributionStudy
  - AttributionStudyController_getConversionsByDate
  - AttributionStudyController_getConversionsByAdGroup
  - AttributionStudyController_getConversionsByUnit
  - AttributionStudyController_getAttributionStudyFile
generated: '2026-08-12'
method: generated
source: openapi/streetmetrics-public-api-openapi.json
---

# Run a StreetMetrics attribution study

An attribution study binds to a campaign and is fed by one or more conversion **pixels**. Conversions
are then read back at three grains, and the finished study can be downloaded as a ZIP.

Authenticate first — see `streetmetrics-authenticate`.

## 1. Create the study

`AttributionStudyController_createAttributionStudy` — `POST /attribution-studies`
(`CreateAttributionStudyDto`, returns `201`).

The study entity carries `attributionId`, `campaignId`, `studyName`, `studyType`, `analysisType`,
`status`, `startDate`, `endDate`, `postStudyEndDate`, and the design flags `isPartOfOtherEffort`,
`pauseOtherChannels`, `brandFocused`, `adsAligned`. Those flags describe study conditions — set them
truthfully, they are part of how the result should be read.

## 2. Create and bind the pixel

- `PixelController_createPixel` — `POST /pixels` (`201`). The response carries `pixelId` and
  `pixelUUID`; the UUID is the value that goes in the tag on the advertiser's site.
- `PixelController_attachCampaignToPixel` — `PATCH /pixels/{pixelId}/campaign`
- `PixelController_attachAttributionStudyToPixel` — `PATCH /pixels/{pixelId}/attribution-study`

Unbinding uses the DELETE twins: `PixelController_detachCampaignFromPixel` and
`PixelController_detachAttributionStudyFromPixel` on the same paths.

Inspect with `PixelController_getPixels` (`GET /pixels`) / `PixelController_getPixel`
(`GET /pixels/{pixelId}`), and amend with `PixelController_updatePixel` (`PATCH /pixels/{pixelId}`).

**Watch the error surface here.** The eight Pixel operations declare only `200/201/400/401` — no `404`
and no `429`. Do not assume a missing pixel returns `404`; check the `errorCode` in the body.

## 3. Read conversions

| Grain | Operation | Path |
|---|---|---|
| date | `AttributionStudyController_getConversionsByDate` | `GET /attribution-studies/conversions/date` |
| ad group | `AttributionStudyController_getConversionsByAdGroup` | `GET /attribution-studies/conversions/ad-group` |
| unit | `AttributionStudyController_getConversionsByUnit` | `GET /attribution-studies/conversions/unit` |

- Scope with `attributionId` (and `campaignId` where accepted).
- `summaryBin` — `LIFETIME | WEEK | MONTH | DAY | HOUR | DOW` — buckets the result and adds a
  `summaryBinGrouping` field; set `timezone` alongside it.
- On the unit grain, `useCreative=true` groups by creative instead of by matchable unit.
- These three are the only reporting operations that declare `400`, so validate `summaryBin` and the
  date window before calling.

## 4. Download the study

`AttributionStudyController_getAttributionStudyFile` — `GET /attribution-studies/files/{id}` returns a
**ZIP**, not JSON. Do not pipe it through a JSON parser; stream it to disk.

List and inspect with `AttributionStudyController_getAttributionStudies` (`GET /attribution-studies`)
and `AttributionStudyController_getAttributionStudy` (`GET /attribution-studies/{id}`); poll `status`
on the entity to know when results exist.

## Operating rules

- No idempotency key: a retried `POST /attribution-studies` or `POST /pixels` creates a duplicate. List
  first, create second.
- Cursor pagination (`limit`, `cursor`, `sort`, `order`) and the `search` DSL apply to the study
  collections.
- Errors: `{status, statusCode, errorCode, message, details, timestamp, path}` — see
  `errors/streetmetrics-problem-types.yml`.
