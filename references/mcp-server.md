# doctorSIM MCP Server

Public consumer documentation: [MCP Server guide](/api-docs/mcp.html)

## Endpoint

- **Streamable HTTP:** `https://api.doctorsim.com/mcp` (or your environment's MCP URL from the server card)
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
| `search_products` | Catalog search |
| `preview_order` | **Required** price breakdown before checkout |
| `create_order` | Place order (credits or payment link) |
| `get_order_status` | Single order status |
| `list_orders` | Order history |
| `get_balance` | PRO prepaid credits |
| `list_webhooks` | PRO webhook subscriptions |

## Checkout modes

### PRO (credit checkout)

`create_order` debits prepaid credits and returns `order_id` immediately.

### Regular consumer (payment link)

`create_order` returns:

```json
{
  "checkout_mode": "payment_link",
  "checkout_hash": "abc123...",
  "payment_link": "https://www.doctorsim.com/en/es/topup-phone/checkout/abc123...",
  "status": "awaiting_payment",
  "expires_at": "2026-06-12T15:00:00Z"
}
```

The user must open `payment_link` and pay on doctorSIM. Poll with `checkout_hash` until `order_id` appears.

## Purchase flow

1. `lookup_carrier` (top-ups) or `search_products` (gift cards)
2. `get_operator_rates` → copy `token`
3. `preview_order` → show breakdown, ask for confirmation
4. `create_order` → call as soon as the user confirms (e.g. "yes"); reuse the same `price_token`, no need to re-preview
5. Monitor via `get_order_status` / `list_orders`
