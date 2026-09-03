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
| GET | /topup/carriers/lookup/{phone} | read |
| GET | /topup/countries | read |
| GET | /topup/operators | read |
| GET | /topup/operators/{id}/service-types | read |
| GET | /topup/operators/{id}/rates | read |
| GET | /topup/products/search | read |
| GET | /topup/products/{operator_id} | read |
| GET | /giftcards/countries | read |
| GET | /giftcards/brands | read |
| GET | /giftcards/brands/{id}/products | read |
| GET | /giftcards/products/{operator_id} | read |
| GET | /esim/destinations | read |
| GET | /esim/products | read |
| GET | /esim/products/{catalog_id} | read |
| GET | /esim/lines/{iccid} | read |
| POST | /orders/preview | read (every vertical; `/esim/orders/preview` is an alias) |
| POST | /orders | write (every vertical; `/esim/orders` is an alias) |
| GET | /orders | read |
| GET | /orders/{id} | read |
| GET | /balance | read |
| GET | /balance/history | read |
| GET/POST/PATCH/DELETE | /webhooks | read/write |

See OpenAPI for full schema: `https://www.doctorsim.com/api-docs/openapi.yaml`

## Pricing and orders

All three products share the same Core endpoints (`POST /orders/preview`, `POST /orders`). The **catalog** and the **JSON body** decide the vertical. **PRO credits:** always preview, show the breakdown, then create. **Guest / consumer `payment_link`:** skip preview and create as soon as the product is chosen.

| | Top-up | Gift cards | Travel eSIM |
|---|---|---|---|
| Catalog | `/topup/*` (aliases: `/countries`, `/operators/*`, `/products/*`, `/carriers/*`) | `/giftcards/*` | `/esim/*` only — do **not** use `/countries` or `/operators` |
| How you pick a product | Phone → carrier → `GET /topup/operators/{id}/rates` → copy rate `token` | Country → brand → `GET /giftcards/brands/{id}/products` → copy rate `token` | Destination → `GET /esim/products` → copy `catalog_id` **and** eSIM `price_token` |
| Create body | `price_token` + `phone` (E.164) | `price_token` only (no phone) | `catalog_id` + eSIM `price_token` (no phone). `type: "esim"` is optional — the token already selects the vertical |
| What create returns | PRO: `order_id`. Guest / consumer OAuth: `payment_link` + `checkout_hash` | Same | Same |
| After payment | Airtime / bundle on the phone | Redemption code or URL | ICCID, LPA, QR, Apple `install_url`. Titular gets the confirmation email |

### Top-up

1. Identify the line: `GET /topup/carriers/lookup/{phone}` (or list operators for a country).
2. `GET /topup/operators/{id}/rates` — copy `token` from the `rates` array.
3. Guest: `POST /orders` with `"price_token": "<token>"` and `"phone": "+34…"`. PRO: preview first, then the same create body.

### Gift cards

1. `GET /giftcards/brands?country={iso}` then `GET /giftcards/brands/{id}/products`.
2. Copy `token` from the `rates` array.
3. Guest: `POST /orders` with `"price_token": "<token>"` only. PRO: preview first, then create.

### Travel eSIM (not a top-up)

eSIM is a **plan SKU**, not an operator rate. There is no phone number and no `/topup/operators/{id}/rates` step.

1. `GET /esim/destinations` — pick `country_iso` (ISO-2 such as `es`, or region `eu` / `ww` / `as`).
2. `GET /esim/products?country_iso=es` — pick a plan. Copy **both** `catalog_id` and `price_token` from that row (the eSIM token is not a top-up rate `token`).
3. Guest: `POST /orders` as soon as the plan is chosen:
   ```json
   { "catalog_id": 42, "price_token": "<esim price_token>", "lang": "en" }
   ```
   Returns `payment_link` + `checkout_hash` to `/{locale}/esim/checkout/{hash}` on doctorsim.com (plan preloaded; buyer types email there). Poll `GET /orders?checkout_hash=<hash>` until `order_id` appears. Preview is optional for guests.
4. PRO: `POST /orders/preview` with that same body (`checkout_mode=credits`), then `POST /orders`. Returns `order_id` immediately.
5. Fulfilled eSIM: `GET /orders/{id}` (or `GET /orders?type=esim`) for `iccid`, `lpa_string`, `qr_code_url`, `install_url`. Remaining data: `GET /esim/lines/{iccid}`.

`POST /esim/orders/preview` and `POST /esim/orders` are aliases of the Core `/orders*` calls (same body).

Full conversation scripts: [order-flow.md](https://www.doctorsim.com/agents/references/order-flow.md).

Legacy checkout with separate `tool_id`, `denomination_id`, `amount`, `operator_id`, and `country_id` is still accepted for top-up / gift cards but deprecated. Do not use that shape for eSIM.

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
