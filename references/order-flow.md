# Order placement flow

Every vertical uses the same Core order and monitoring endpoints (`/orders*`, `/balance`, `/webhooks`). Only the catalog differs, and each vertical has its own prefix: `/topup/*`, `/giftcards/*`, `/esim/*`. Un-prefixed catalog paths (`/countries`, `/operators/*`, `/products/*`, `/carriers/*`) remain as aliases.

**Base URL:** `https://api.doctorsim.com/v2`

## Mobile top-up

### Recommended agent conversation

When the user gives a phone number (e.g. `+52 222 123 1231`):

1. **Always run carrier lookup first** — do not ask the user to guess the network if lookup is available.  
   MCP: `lookup_carrier` · API: `GET https://api.doctorsim.com/v2/topup/carriers/lookup/{phone}`  
   Requires API key or OAuth (anonymous guest is rejected — paid HLR abuse hardening #9877).

2. **If lookup fails**, ask which carrier/network the number uses, or list operators for the country.

3. **Ask what they want to buy** — e.g. *"Do you want airtime (saldo), a bundle/package, or mobile data? If bundle or data, what are you looking for (e.g. 5GB, WhatsApp)?"*

4. **List service types** for the identified operator — each type (airtime vs bundle vs data) has its own `id_operator`.  
   MCP: `get_operator_service_types` · API: `GET https://api.doctorsim.com/v2/topup/operators/{id}/service-types`

5. **Fetch and filter rates** for the chosen service type. Search bundle/data descriptions with `q` (e.g. `q=5GB whatsapp`).  
   MCP: `get_operator_rates` · API: `GET https://api.doctorsim.com/v2/topup/operators/{child_id}/rates?q=…`

6. **Guest / consumer:** skip preview. Call `create_order` as soon as the product is chosen.  
   MCP: `create_order` · API: `POST https://api.doctorsim.com/v2/orders`

7. **PRO credits:** preview the exact cost (mandatory — credits will be deducted), show the breakdown, then place the order after confirmation.  
   MCP: `preview_order` then `create_order` · API: `POST https://api.doctorsim.com/v2/orders/preview` then `POST https://api.doctorsim.com/v2/orders`

### API steps

1. **Lookup phone number** — identify carrier and country from E.164 MSISDN  
   `GET https://api.doctorsim.com/v2/topup/carriers/lookup/{phone}`

2. **Select airtime, data, or bundle** — list service types, then pick a rate `token` (use `q` on rates for bundle/data search)  
   `GET https://api.doctorsim.com/v2/topup/operators/{id}/service-types`  
   `GET https://api.doctorsim.com/v2/topup/operators/{id}/rates?q=5GB+whatsapp`

3. **Guest:** place the order immediately (`POST https://api.doctorsim.com/v2/orders`). **PRO:** preview first (`POST https://api.doctorsim.com/v2/orders/preview`), then place the order.

5. **Monitor** — poll order status or register a webhook (see [Webhooks](https://www.doctorsim.com/agents/references/webhooks.md))  
   `GET https://api.doctorsim.com/v2/orders/{id}`

### Top-up order body (typical)

```json
{
  "price_token": "<token from /operators/{id}/rates>",
  "phone": "+34612345678",
  "lang": "en"
}
```

Optional: `operator_id` / `country_id` as cross-checks when using `price_token`.

---

## Gift cards

1. **Select country** — countries with gift card brands  
   `GET https://api.doctorsim.com/v2/giftcards/countries`

2. **Search brands** — find the brand (an `id_operator`), then load denominations and copy the `token`  
   `GET https://api.doctorsim.com/v2/giftcards/brands?country={iso_or_id}` · MCP: `get_giftcard_brands` (default 50/page; when `meta.has_more` is true, pass `cursor=meta.cursor`)  
   `GET https://api.doctorsim.com/v2/giftcards/brands/{id}/products` · MCP: `get_giftcard_brand_products`

3. **Guest:** `POST https://api.doctorsim.com/v2/orders` with `price_token` only (no `phone`). **PRO:** preview, then place.  
   MCP: `create_order` (guest) or `preview_order` / `create_order` (PRO)

4. **Monitor** — same as top-up  
   `GET https://api.doctorsim.com/v2/orders/{id}`

### Gift card order body (typical)

```json
{
  "price_token": "<token from rates>",
  "lang": "en"
}
```

Add `phone` or email fields only when the selected product requires them.

---

## Travel eSIM

Catalog lives under `/esim/*`; checkout, history, and status use the shared Core `/orders` spine (the body selects the vertical). MCP mirrors this: `get_esim_*` for the catalog, `preview_order` / `create_order` for checkout (`preview_esim_order` / `create_esim_order` remain as aliases).

1. **Browse destinations** — list countries/regions with plans  
   `GET https://api.doctorsim.com/v2/esim/destinations` · MCP: `get_esim_destinations`

2. **List plans** — pick `catalog_id` and `price_token`. `country_iso` is ISO-2 (`es`) or a region slug (`eu`, `ww`, `as`). Each plan includes `unlimited` (boolean). Optional `data_type=unlimited` or `limited`  
   `GET https://api.doctorsim.com/v2/esim/products?country_iso=es` · MCP: `get_esim_plans`

3. **Guest / consumer:** skip preview. Place the order as soon as the plan is chosen. Returns `payment_link` + `checkout_hash` to `/{locale}/esim/checkout/{hash}` on doctorsim.com (plan preloaded; buyer types email there).  
   `POST https://api.doctorsim.com/v2/orders` · MCP: `create_order` (alias: `create_esim_order`; REST alias `/esim/orders`)

4. **PRO credits:** preview first (mandatory), then place. Debits credits and returns `order_id`. Sandbox keys (`test_*`) preview the sandbox catalog and create a fake eSIM (`sandbox_esim_{id}`, no credit debit). Response carries `type: "esim"`  
   `POST https://api.doctorsim.com/v2/orders/preview` then `POST https://api.doctorsim.com/v2/orders` · MCP: `preview_order` / `create_order`

5. **Monitor** — `GET https://api.doctorsim.com/v2/orders/{id}` or `list_orders?type=esim`  
   Fulfilled eSIM includes `iccid`, `lpa_string`, Apple `install_url`, and a hosted PNG `qr_code_url`. Single-order status also returns remaining `balance_amount` (provider refresh if last check is older than five minutes). List rows set `balance_amount` to `per order basis`.

6. **Line remaining data** — `GET https://api.doctorsim.com/v2/esim/lines/{iccid}` · MCP: `get_esim_line`  
   Requires OAuth/API key (`orders:read`). The ICCID line is the same on every device. Provider refresh only when last check is older than five minutes.

### eSIM order body (typical)

```json
{
  "catalog_id": 42,
  "price_token": "esim__42__9.99__EUR__0__1710000000__a1b2c3d4e5f6g7h8",
  "lang": "en",
  "currency": "EUR"
}
```

---

## Before ordering (both)

Check PRO credit balance:

`GET https://api.doctorsim.com/v2/balance`

Requires `orders:write` scope (OAuth) or a `read-write` API key for `POST /orders`.

## Idempotency

Send an `Idempotency-Key` header on `POST /orders` to safely retry agent actions without duplicate charges.

## Sandbox

Use sandbox API keys (`test_*`) or an OAuth client on a test titular. Sandbox orders simulate success without carrier or provider delivery.

## Manual revision

Some orders enter `MANUAL_REVISION` when automatic fulfillment cannot complete — for example operator or provider failure, product temporarily out of stock, an invalid or mismatched MSISDN, or other data that needs correction.

Poll `GET https://api.doctorsim.com/v2/orders/{id}` for status updates, or rely on webhooks if configured. The account holder may also receive email about the order; check that channel when status stays in manual review.
