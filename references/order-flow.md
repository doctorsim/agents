# Order placement flow

Both product types use the same order and monitoring endpoints. Discovery differs between mobile top-up and gift cards.

**Base URL:** `https://api.doctorsim.com/v2`

## Mobile top-up

### Recommended agent conversation

When the user gives a phone number (e.g. `+52 222 123 1231`):

1. **Always run carrier lookup first** — do not ask the user to guess the network if lookup is available.  
   MCP: `lookup_carrier` · API: `GET https://api.doctorsim.com/v2/carriers/lookup/{phone}`

2. **If lookup fails**, ask which carrier/network the number uses, or list operators for the country.

3. **Ask what they want to buy** — e.g. *"Do you want airtime (saldo), a bundle/package, or mobile data? If bundle or data, what are you looking for (e.g. 5GB, WhatsApp)?"*

4. **List service types** for the identified operator — each type (airtime vs bundle vs data) has its own `id_operator`.  
   MCP: `get_operator_service_types` · API: `GET https://api.doctorsim.com/v2/operators/{id}/service-types`

5. **Fetch and filter rates** for the chosen service type. Search bundle/data descriptions with `q` (e.g. `q=5GB whatsapp`).  
   MCP: `get_operator_rates` · API: `GET https://api.doctorsim.com/v2/operators/{child_id}/rates?q=…`

6. **Preview the exact cost** — mandatory before placement; show the user the full breakdown (recharge, service fee, SMS if any, credit applied, amount due).  
   MCP: `preview_order` · API: `POST https://api.doctorsim.com/v2/orders/preview`

7. **Ask for explicit confirmation** only after showing the preview breakdown.

8. **Place order** with the same fields as preview.  
   MCP: `create_order` · API: `POST https://api.doctorsim.com/v2/orders`

### API steps

1. **Lookup phone number** — identify carrier and country from E.164 MSISDN  
   `GET https://api.doctorsim.com/v2/carriers/lookup/{phone}`

2. **Select airtime, data, or bundle** — list service types, then pick a rate `token` (use `q` on rates for bundle/data search)  
   `GET https://api.doctorsim.com/v2/operators/{id}/service-types`  
   `GET https://api.doctorsim.com/v2/operators/{id}/rates?q=5GB+whatsapp`

3. **Preview order** — mandatory; present full price breakdown to the user  
   `POST https://api.doctorsim.com/v2/orders/preview`

4. **Place order** — only after preview and explicit user confirmation  
   `POST https://api.doctorsim.com/v2/orders`

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

1. **Select country** — browse supported destinations  
   `GET https://api.doctorsim.com/v2/countries`

2. **Search catalog** — find brand/product, then load rates and copy the `token`  
   `GET https://api.doctorsim.com/v2/products/search?country={iso_or_id}` (default 50/page; when `meta.has_more` is true, pass `cursor=meta.cursor`)  
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
