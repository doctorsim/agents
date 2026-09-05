# doctorSIM MCP Server

Public consumer documentation: [MCP Server guide](/api-docs/mcp.html)

## Endpoint

- **Streamable HTTP:** `https://api.doctorsim.com/mcp` (Cloudflare Worker — same URL on mydev3; the website does not serve `/mcp`)
- **Streamable HTTP, account required:** `https://api.doctorsim.com/mcp/pro` — same tools, but every request needs an OAuth Bearer (`initialize` answers 401 + `WWW-Authenticate`). Use it on hosts that only start OAuth when the first `initialize` is 401 (Grok connectors) and you want PRO credits / order history.
- **Server card:** `/.well-known/mcp/server-card.json`

## Authentication

- **Remote connectors (Claude, ChatGPT):** OAuth 2.0 with Dynamic Client Registration + PKCE. See [auth.md](/auth.md).
- **API keys:** PRO accounts only — `Authorization: Bearer {api_id}:{api_secret}`

## Tools

| Tool | Purpose |
|------|---------|
| `lookup_carrier` | Identify operator from E.164 phone |
| `get_countries` | List supported countries |
| `get_operators` | Operators for a country |
| `get_operator_service_types` | Airtime / data / bundle types |
| `get_operator_rates` | Live pricing + `token` for orders |
| `search_products` | Unfiltered catalog search (top-up operators + gift card brands) |
| `get_esim_destinations` | eSIM destination list |
| `get_esim_plans` | eSIM plans for a country |
| `get_esim_plan_detail` | Single eSIM plan detail |
| `preview_esim_order` | Alias of `preview_order` with `type=esim` (kept for existing connectors) |
| `create_esim_order` | Alias of `create_order` with `type=esim` (kept for existing connectors) |
| `get_esim_line` | Remaining data for a titular-owned ICCID (OAuth) |
| `get_giftcard_brands` | Gift card brands for a country (`/giftcards/brands`) |
| `get_giftcard_brand_products` | Gift card denominations + `token` (`/giftcards/brands/{id}/products`) |
| `preview_order` | Price breakdown without placing an order. **Required for PRO credits.** Optional for guest / consumer `payment_link`. |
| `create_order` | Place order for any vertical (PRO credits, or guest / consumer payment link) |
| `get_order_status` | Single order status |
| `list_orders` | Order history |
| `get_balance` | PRO prepaid credits |
| `list_webhooks` | PRO webhook subscriptions |

## Checkout modes

### PRO (credit checkout)

`create_order` debits prepaid credits and returns `order_id` immediately.

### Regular consumer (payment link)

`create_order` returns a `payment_link` (the only URL to show the buyer) and `order_id` when one exists. `checkout_hash` is a machine identifier for `get_order_status` — never display or read it aloud. Prefer `get_order_status({ payment_link })`.

```json
{
  "checkout_mode": "payment_link",
  "payment_link": "https://www.doctorsim.com/en/es/topup-phone/checkout/abc123...",
  "order_id": null,
  "status": "awaiting_payment",
  "expires_at": "2026-06-12T15:00:00Z",
  "show_to_user": { "payment_link": "https://www.doctorsim.com/en/es/topup-phone/checkout/abc123..." }
}
```

The user must open `payment_link`, enter email, and pay on doctorSIM. Guest products are delivered by email. Poll with `payment_link` or `order_id` if you need status — do not promise in-chat QR/code for guests.

## Purchase flow

### Mobile top-up

1. `lookup_carrier` → `get_operator_service_types` → `get_operator_rates` → copy `token`
2. Guest: `create_order` as soon as the product is chosen (preview optional). PRO: `preview_order` → show breakdown → `create_order` after confirmation
3. Guest / consumer: share `payment_link`. PRO: `order_id` after credit debit
4. Monitor via `get_order_status` / `list_orders`

### Travel eSIM

1. `get_esim_destinations` → pick `country_iso`
2. `get_esim_plans` → copy `catalog_id` and `price_token`
3. Guest: `create_order` with `catalog_id` + `price_token` as soon as the plan is chosen (preview optional) → `payment_link` to `/{locale}/esim/checkout/{hash}`
4. PRO: `preview_order` then `create_order` → credits debit + `order_id`; response carries `type: "esim"`
5. Monitor via `get_order_status` / `list_orders` (`type=esim`)
6. Optional: `get_esim_line` with ICCID for remaining data (OAuth)

### Gift cards

1. `get_giftcard_brands` → `get_giftcard_brand_products` → copy `token`
2. Guest: `create_order` as soon as the denomination is chosen (preview optional). PRO: `preview_order` → show breakdown → `create_order` after confirmation
3. Guest / consumer: share `payment_link`. PRO: `order_id` after credit debit
4. Monitor via `get_order_status` / `list_orders`
