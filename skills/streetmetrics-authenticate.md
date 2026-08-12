---
name: streetmetrics-authenticate
description: Exchange StreetMetrics platform credentials for the JWT bearer token every other StreetMetrics Public API operation requires, and handle the token-expiry and error cases correctly.
api: StreetMetrics Public API
base_url: https://dashboard.streetmetrics.io/v3/public/
operations:
  - AuthController_authenticate
generated: '2026-08-12'
method: generated
source: openapi/streetmetrics-public-api-openapi.json + https://docs.streetmetrics.com/recipes/how-to-authenticate-requests-and-create-tokens
---

# Authenticate against the StreetMetrics Public API

Every operation except this one requires a JWT bearer token. There is no API-key-only path and no
OAuth flow — you trade platform credentials for a token, then attach it.

## Step 1 — mint the token

`AuthController_authenticate` — `POST /auth/authenticate`

- Base URL: `https://dashboard.streetmetrics.io/v3/public/`
- Required header: `api-key: <your StreetMetrics api key>` (declared required on this operation)
- Body: `{ "email": "<platform email>", "password": "<platform password>" }` — the same credentials
  used to sign in at https://platform.streetmetrics.com/login. Only `email` is marked required in the
  schema; send both.
- Response (`AuthResponse`): `{ "statusCode": 201, "message": "...", "meta": {}, "data": "<token>" }`.
  **The token is the `data` string**, not an object.

## Step 2 — attach it

Send `Authorization: Bearer <token>` on every subsequent request.

## What the contract gets wrong — do not trust it blindly

- The spec defines `components.securitySchemes.bearer` but never applies it: there is no root
  `security` block and no operation carries one. A generated client will look anonymous. Add the
  header yourself. (Our correction is captured in `overlays/streetmetrics-public-api-overlay.yaml`.)
- `POST /auth/authenticate` declares only a `201` response. It really can fail — an invalid email
  returns `400 BAD_REQUEST` with `"message": "Email must be an email"`.
- The published recipe still shows `/v3/api/auth/authenticate`. That path returns **404** today. Use
  `/v3/public/auth/authenticate`. The older `/v3/auth/authenticate` also still answers, with a
  different error envelope — do not build against it.

## Failure handling

| Status | errorCode | What to do |
|---|---|---|
| 400 | `BAD_REQUEST` | Fix the credential payload; the message names the offending field. |
| 401 | `UNAUTHORIZED` | Token missing, expired, or the account lacks permission — re-mint and retry once. |
| 429 | — | Back off exponentially. No `Retry-After` or `RateLimit-*` header is returned, so use your own schedule. |
| 500 | — | Retry with backoff; capture `timestamp` and `path` from the body, which are the only handles for support. |

No token lifetime is published. Treat a `401` on a previously-working token as expiry, re-run step 1,
and retry the request once.
