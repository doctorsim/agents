---
name: doctorsim
description: Place mobile top-up, gift card, and travel eSIM orders on doctorSIM via API v2 or MCP. Browse products, check balance, manage webhooks.
metadata:
  author: doctorSIM
  version: "1.0.0"
  homepage: https://www.doctorsim.com/api-docs/
---

# doctorSIM Agent Skill

Use this skill when the user wants to recharge a mobile phone, buy a gift card, or automate doctorSIM PRO orders programmatically.

## When to use

- User asks to top up / recharge a phone number internationally
- User wants gift cards (gaming, streaming, retail) — `get_giftcard_brands` → `get_giftcard_brand_products` → `preview_order` / `create_order` (no phone)
- User wants a travel eSIM plan for international data
- User needs to check PRO credit balance or order history
- User wants to integrate doctorSIM into an agent workflow (MCP, API, OAuth)

## Authentication

**Default (ChatGPT / MCP): guest checkout — no login required.** Browse catalog, preview prices, and get a `payment_link` on doctorsim.com without linking an account.

**Optional OAuth** — link your doctorSIM account when the user wants:
- PRO prepaid credits (`checkout_mode=credits`)
- Order history (`list_orders`)
- Credit balance or webhooks (PRO only)

OAuth discovery: `https://www.doctorsim.com/.well-known/oauth-authorization-server`

**API keys** (server integrations only): `Authorization: Bearer {api_id}:{api_secret}` from Mi Cuenta → API

Full guide: `https://www.doctorsim.com/auth.md`

## Quick start (API)

```bash
# Health check (no auth)
curl -s https://api.doctorsim.com/v2/status

# List countries (guest when agentic_mcp_guest is ON, else API key/OAuth)
curl -s https://api.doctorsim.com/v2/countries
```

## MCP (remote — Claude.ai, ChatGPT, Grok)

Remote connectors use **Streamable HTTP only**. The MCP endpoint runs on Cloudflare Workers and forwards an optional OAuth Bearer token to API v2.

**Connector URL:** `https://api.doctorsim.com/mcp`

**Server card:** `https://www.doctorsim.com/.well-known/mcp/server-card.json` (primary transport: `streamable-http`)

### Setup (guest-first)

1. Add a **custom MCP connector** with URL `https://api.doctorsim.com/mcp`.
2. **No OAuth required at setup** — catalog and payment-link checkout tools use `noauth`. ChatGPT can connect immediately.
3. **Optional account linking** — only when the user asks for PRO credits, order history, or saved settings:
   - Leave OAuth Client ID/Secret blank; DCR + PKCE handles registration.
   - Authorization server: `https://www.doctorsim.com/.well-known/oauth-authorization-server`
   - See `https://www.doctorsim.com/auth.md` for scopes.
4. Protected tools (`get_balance`, `list_orders`, `list_webhooks`) return an auth challenge when called without a token.

### Before create_order (model script)

> I can send you a secure payment link to complete this top-up on doctorsim.com — no account needed. If you have a **PRO account with credits** or want **saved account settings**, say "use my doctorSIM account" and I'll connect you.

### Mobile top-up flow (MCP)

When the user provides a **phone number**:

1. **`lookup_carrier`** with E.164 phone (`+522221231231`) — always first; returns `id_operator` and country.
2. If lookup fails, ask the carrier or use **`get_operators`** for the country.
3. Ask the user: **airtime, bundles, or data?** If bundle/data, ask what they need (e.g. 5GB, WhatsApp).
4. **`get_operator_service_types`** — pick the `id_operator` for the chosen type (bundle operators differ from airtime).
5. **`get_operator_rates`** with `q` to search `description` / `product_name` (e.g. `q="5GB whatsapp"`).
6. **`preview_order`** — **mandatory** before placement. Guest: `checkout_mode=payment_link`. OAuth PRO: `checkout_mode=credits`. Show breakdown, then confirm.
7. **`create_order`** — only after preview **and** explicit confirmation. Guest: returns `payment_link` + `checkout_hash`. PRO OAuth: debits credits and returns `order_id`.

Do **not** use **`search_products`** alone for phone top-ups — it browses the whole country catalog by brand name. Use **`search_products`** with `operator_id` + `q` only as an alternative rate search API.

### Gift card flow (MCP)

1. **`get_giftcard_brands`** with `country` (ISO-2 or id) and optional `q` (brand keyword) — each brand is an `id_operator`.
2. **`get_giftcard_brand_products`** with `brand_id` — copy the `token`.
3. **`preview_order`** with `price_token` only (no `phone`), then **`create_order`** after confirmation. Fulfilled orders return `redemption_code` / `redemption_url` on `get_order_status`.

REST: `GET /giftcards/countries`, `GET /giftcards/brands`, `GET /giftcards/brands/{id}/products`, `POST /orders/preview`, `POST /orders`. Top-up REST lives under `/topup/*` (`/topup/carriers/lookup/{phone}`, `/topup/operators/{id}/rates`); the un-prefixed paths remain as aliases.

`get_operator_rates` totals are indicative; **`preview_order`** is the authoritative checkout breakdown.

### Travel eSIM flow (MCP)

When the user wants **international data / travel eSIM** (no phone number required):

1. **`get_esim_destinations`** — list countries/regions with plans; pick `country_iso` (ISO-2: `es`, not `ESP`. Regions: `eu`, `ww`, `as`).
2. **`get_esim_plans`** with `country_iso` — browse plans; copy `catalog_id` and `price_token`. Each plan includes `unlimited` (boolean). Optional `data_type` (`all` / `unlimited` / `limited`) filters the list; omit or `all` returns both.
3. Optional: **`get_esim_plan_detail`** for a single SKU (includes `unlimited`).
4. **`preview_order`** with `catalog_id` + `price_token` (the eSIM token selects the vertical; `type: "esim"` optional) — mandatory before placement. Same body as create; no credits deducted. Returns `unlimited` and `data_gb` with the price breakdown. Guest / consumer OAuth: `checkout_mode=payment_link`. PRO: `checkout_mode=credits`. Sandbox keys (`test_*`) preview the **sandbox** catalog. (`preview_esim_order` is an alias.)
5. **`create_order`** — only after preview and explicit confirmation. PRO debits credits and returns `order_id`; guest / consumer OAuth return `payment_link` + `checkout_hash` (share the link; poll `get_order_status` with the hash). Sandbox keys create a fake eSIM (`sandbox_esim_{id}`, no credit debit). Response carries `type: "esim"`. (`create_esim_order` is an alias.)
6. Monitor with **`get_order_status`** or **`list_orders`** (`type=esim`). Fulfilled eSIM includes `iccid`, `lpa_string`, Apple `install_url`, and a hosted PNG `qr_code_url`. Single-order status also includes remaining `balance_amount` (provider refresh if last check is older than five minutes). List rows set `balance_amount` to `per order basis`. Use `iccid` with **`get_esim_line`**.
7. Optional: **`get_esim_line`** with the ICCID (OAuth `orders:read`) for remaining data. The line is the same on every device. The API refreshes from the provider only when the last check is older than five minutes.

REST equivalents: `GET /esim/destinations`, `GET /esim/products`, `POST /orders/preview`, `POST /orders`, `GET /esim/lines/{iccid}` (`/esim/orders/preview` and `/esim/orders` are aliases).

> Travel eSIM checkout uses the same modes as top-up / gift cards: PRO credits, or a `payment_link` for guest / consumer OAuth.

### Verify MCP Worker (no auth)

These hit the **Cloudflare MCP Worker** (`https://api.doctorsim.com/mcp`), not the website.
`GET /mcp` returns the server descriptor (protocol + tools). `GET /mcp/health` is a liveness check.
Neither requires a Bearer token.

```bash
curl -s https://api.doctorsim.com/mcp | jq .
curl -s https://api.doctorsim.com/mcp/health | jq .
```

> **Do not configure stdio for remote connectors.** stdio is optional and only for local IDE plugins (Cursor, Claude Desktop). See [local MCP setup](https://www.doctorsim.com/agents/references/local-mcp.md) if you need a local process — not required for Claude.ai or ChatGPT.

## Credits

PRO orders debit prepaid account credits. Guest and consumer OAuth users pay via `payment_link` on doctorsim.com. Fund PRO credits via the web dashboard.

## References

- [MCP Server guide](https://www.doctorsim.com/api-docs/mcp.html)
- [MCP reference (markdown)](https://www.doctorsim.com/agents/references/mcp-server.md)
- [API overview](https://www.doctorsim.com/agents/references/api-overview.md)
- [Order flow](https://www.doctorsim.com/agents/references/order-flow.md)
- [Webhooks](https://www.doctorsim.com/agents/references/webhooks.md)
- [Error codes](https://www.doctorsim.com/agents/references/errors.md)
- [Local MCP — Cursor / Claude Desktop only](https://www.doctorsim.com/agents/references/local-mcp.md)

## Discovery URLs

| Resource | URL |
|---|---|
| API catalog | https://www.doctorsim.com/.well-known/api-catalog |
| OpenAPI | https://www.doctorsim.com/api-docs/openapi.yaml |
| API explorer | https://www.doctorsim.com/api-docs/ |
| MCP server card | https://www.doctorsim.com/.well-known/mcp/server-card.json |
| MCP endpoint (Streamable HTTP) | https://api.doctorsim.com/mcp |
| Protected resource | https://api.doctorsim.com/.well-known/oauth-protected-resource/v2 |
| Agent skills index | https://www.doctorsim.com/.well-known/agent-skills/index.json |
