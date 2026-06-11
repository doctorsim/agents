# doctorSIM API v2 Overview

Base URL: `https://api.doctorsim.com/v2`

Sandbox vs production is determined by credentials (`test_*` vs `live_*` API keys, or OAuth JWT `env` claim) — not by hostname.

## Auth header

```
Authorization: Bearer {api_id}:{api_secret}
```

Or OAuth JWT from `POST https://www.doctorsim.com/oauth/token`.

## Core endpoints

| Method | Path | Scope |
|---|---|---|
| GET | /status | public |
| GET | /countries | read |
| GET | /operators | read |
| GET | /operators/{id}/service-types | read |
| GET | /operators/{id}/rates | read |
| GET | /products/search | read |
| GET | /products/{operator_id} | read |
| GET | /carriers/lookup/{phone} | read |
| POST | /orders | write |
| GET | /orders | read |
| GET | /orders/{id} | read |
| GET | /balance | read |
| GET | /balance/history | read |
| GET/POST/PATCH/DELETE | /webhooks | read/write |

See OpenAPI for full schema: `https://www.doctorsim.com/api-docs/openapi.yaml`

## Pricing and orders

1. Fetch rates from `GET /operators/{id}/rates` or `GET /products/{operator_id}`.
2. Copy the rate `token` from the `rates` array.
3. Place the order with `POST /orders` and `"price_token": "<token>"`.

For mobile top-ups, include `phone` (E.164). Gift cards may omit `phone` when the product does not require it.

Legacy checkout with separate `tool_id`, `denomination_id`, `amount`, `operator_id`, and `country_id` is still accepted but deprecated.

## Environments

All requests go to **`https://api.doctorsim.com/v2`**. There is no separate staging hostname.

- `test_*` API IDs → sandbox (no real delivery)
- `live_*` API IDs → production

OAuth tokens include `env` claim (`sandbox` | `production`). OAuth issuer is always `https://www.doctorsim.com`.

## Media URLs

Country flags and operator logos in API responses are **absolute HTTPS URLs on `www.doctorsim.com`**. Do not use or expect legacy `img1.doctorsim.com` hosts.

| Field | Pattern |
|---|---|
| Country `img` | `https://www.doctorsim.com/img/country_flags/vectored/{iso_short_code}.svg` |
| Operator `img` / product `image` | `https://www.doctorsim.com/img/oper/{filename}` |

Use the URL returned by the API as-is.
