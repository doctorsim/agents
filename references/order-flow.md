# Order placement flow

Both product types use the same order and monitoring endpoints. Discovery differs between mobile top-up and gift cards.

**Base URL:** `https://api.doctorsim.com/v2`

## Mobile top-up

1. **Lookup phone number** — identify carrier and country from E.164 MSISDN  
   `GET https://api.doctorsim.com/v2/carriers/lookup/{phone}`

2. **Select airtime, data, or bundle** — list service types, then pick a rate `token`  
   `GET https://api.doctorsim.com/v2/operators/{id}/service-types`  
   `GET https://api.doctorsim.com/v2/operators/{id}/rates`

3. **Place order** — send `price_token` and recipient `phone` (credits deducted on create)  
   `POST https://api.doctorsim.com/v2/orders`

4. **Monitor** — poll order status or register a webhook (see [Webhooks](https://www.doctorsim.com/agents/references/webhooks.md))  
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

1. **Select country** — browse supported destinations  
   `GET https://api.doctorsim.com/v2/countries`

2. **Search catalog** — find brand/product, then load rates and copy the `token`  
   `GET https://api.doctorsim.com/v2/products/search?country={iso_or_id}`  
   `GET https://api.doctorsim.com/v2/operators/{id}/rates`

3. **Place order** — send `price_token` (some products do not require `phone`)  
   `POST https://api.doctorsim.com/v2/orders`

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
