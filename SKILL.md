---
name: doctorsim
description: Place mobile top-up, travel eSIM, and gift card orders on doctorSIM via API v2 or MCP. Browse products, check balance, manage webhooks.
metadata:
  author: doctorSIM
  version: "1.1.0"
  homepage: https://www.doctorsim.com/api-docs/
---

# doctorSIM Agent Skill

Use this skill when the user wants to recharge a mobile phone, buy a travel eSIM, buy a gift card, or automate doctorSIM PRO orders programmatically.

## When to use

- User asks to top up / recharge a phone number internationally
- User wants a travel eSIM plan for international data — `get_esim_destinations` → `get_esim_plans` → `create_order` (guest) or `preview_order` then `create_order` (PRO credits); no phone
- User wants gift cards (gaming, streaming, retail) — `get_giftcard_brands` → `get_giftcard_brand_products` → `create_order` (guest) or `preview_order` then `create_order` (PRO credits); no phone
- User needs to check PRO credit balance or order history
- User wants to integrate doctorSIM into an agent workflow (MCP, API, OAuth)

## Authentication

**Default (ChatGPT / MCP): guest checkout — no login required.** Browse the catalog and, once the product is chosen, call `create_order` for a `payment_link` on doctorsim.com. Guest preview is optional (no credits at risk).

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

> I can send you a secure payment link to complete this top-up, travel eSIM, or gift card on doctorsim.com — no account needed. If you have a **PRO account with credits** or want **saved account settings**, say "use my doctorSIM account" and I'll connect you.

### Mobile top-up flow (MCP)

When the user provides a **phone number**:

1. **`lookup_carrier`** with E.164 phone (`+522221231231`) — always first; returns `id_operator` and country.
2. If lookup fails, ask the carrier or use **`get_operators`** for the country.
3. Ask the user: **airtime, bundles, or data?** If bundle/data, ask what they need (e.g. 5GB, WhatsApp).
4. **`get_operator_service_types`** — pick the `id_operator` for the chosen type (bundle operators differ from airtime).
5. **`get_operator_rates`** with `q` to search `description` / `product_name` (e.g. `q="5GB whatsapp"`).
6. **Guest / consumer:** skip preview. Call **`create_order`** as soon as the product is chosen (`checkout_mode=payment_link`). Share the `payment_link`.
7. **PRO credits:** **`preview_order`** first (mandatory — credits will be deducted), show the breakdown, confirm, then **`create_order`**. Returns `order_id`.

Do **not** use **`search_products`** alone for phone top-ups — it browses the whole country catalog by brand name. Use **`search_products`** with `operator_id` + `q` only as an alternative rate search API.

### Travel eSIM flow (MCP)

When the user wants **international data / travel eSIM** (no phone number required):

1. **`get_esim_destinations`** — list countries/regions with plans; pick `country_iso` (ISO-2: `es`, not `ESP`. Regions: `eu`, `ww`, `as`).
2. **`get_esim_plans`** with `country_iso` — browse plans for that destination only. ISO-2 (`es`) is a strict match (does **not** include `eu`/`ww` regional or global plans). Use region slugs (`eu`, `ww`, `as`) for those catalogs. Copy `catalog_id` and `price_token`. Each plan includes `unlimited` (boolean). Optional `data_type` (`all` / `unlimited` / `limited`) filters the list; omit or `all` returns both.
3. Optional: **`get_esim_plan_detail`** for a single SKU (includes `unlimited`).
4. **Guest / consumer:** skip preview. Call **`create_order`** with `catalog_id` + `price_token` as soon as the plan is chosen. Show the buyer only `payment_link` and `order_id` when one exists. Never display `checkout_hash` (machine id for `get_order_status` — prefer passing `payment_link`). The buyer types their email on the doctorsim.com checkout page and receives the eSIM **by email** after payment — do not promise an in-chat QR for guests. (`create_esim_order` is an alias.)
5. **PRO credits:** **`preview_order`** first (mandatory — same body; no credits deducted yet; returns `unlimited` and `data_gb`), confirm, then **`create_order`**. Debits credits and returns `order_id`. Sandbox keys (`test_*`) preview the **sandbox** catalog and create a fake eSIM (`sandbox_esim_{id}`, no credit debit). Response carries `type: "esim"`. (`preview_esim_order` is an alias.)
6. Monitor with **`get_order_status`**. **Guest:** pass `payment_link` (preferred) or `checkout_hash` — no login; the hash is unguessable. **Never** look up a sequential `order_id` without OAuth (IDs are enumerable). **PRO:** pass `order_id`, or **`list_orders`** (`type=esim`). When status is `fulfilled`, the same poll includes `iccid`, `lpa_string`, Apple `install_url`, and a hosted PNG `qr_code_url` — show those `show_to_user` fields in chat (do not wait for email). If fulfilled but those fields are missing, poll the same `payment_link` again; guests cannot call **`get_esim_line`**. Linked accounts may use `iccid` with **`get_esim_line`** for remaining data. Single-order status also includes remaining `balance_amount` (provider refresh if last check is older than five minutes). List rows set `balance_amount` to `per order basis`. Never display `checkout_hash`.
7. Optional: **`get_esim_line`** with the ICCID (OAuth `orders:read`) for remaining data. The line is the same on every device. The API refreshes from the provider only when the last check is older than five minutes.

REST equivalents: `GET /esim/destinations`, `GET /esim/products`, `POST /orders/preview`, `POST /orders`, `GET /esim/lines/{iccid}` (`/esim/orders/preview` and `/esim/orders` are aliases).

> Travel eSIM checkout uses the same modes as top-up and gift cards: PRO credits, or a `payment_link` for guest / consumer OAuth.

### Gift card flow (MCP)

1. **`get_giftcard_brands`** with `country` (ISO-2 or id) and optional `q` (brand keyword) — each brand is an `id_operator`.
2. **`get_giftcard_brand_products`** with `brand_id` — copy the `token`.
3. **Guest:** **`create_order`** with `price_token` only (no `phone`) as soon as the denomination is chosen. Show only `payment_link` and `order_id` if present. Guest gift cards are emailed after they pay on doctorsim.com — do not promise the code in chat. **PRO:** **`preview_order`** then **`create_order`** after confirmation. Fulfilled PRO orders return `redemption_code` / `redemption_url` on `get_order_status`.

REST: `GET /giftcards/countries`, `GET /giftcards/brands`, `GET /giftcards/brands/{id}/products`, `POST /orders/preview`, `POST /orders`. Top-up REST lives under `/topup/*` (`/topup/carriers/lookup/{phone}`, `/topup/operators/{id}/rates`); the un-prefixed paths remain as aliases.

`get_operator_rates` totals are indicative. **PRO:** **`preview_order`** is the authoritative debit breakdown. **Guest:** skip preview and create the payment link.

### Verify MCP Worker (no auth)

These hit the **Cloudflare MCP Worker** (`https://api.doctorsim.com/mcp`), not the website.
`GET /mcp` returns the server descriptor (protocol + tools). `GET /mcp/health` is a liveness check.
Neither requires a Bearer token.

```bash
curl -s https://api.doctorsim.com/mcp | jq .
curl -s https://api.doctorsim.com/mcp/health | jq .
```

> **Do not configure stdio for remote connectors.** stdio is optional and only for local IDE plugins (Cursor, Claude Desktop). See [local MCP setup](https://www.doctorsim.com/agents/references/local-mcp.md) if you need a local process — not required for Claude.ai or ChatGPT.

## Errors

Every failed tool result includes `CODE — message. Next: …`. Follow **Next** (link account / pick another plan / retry). Do not invent carriers, tokens, or hashes.

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
| Protected resource (API v2) | https://api.doctorsim.com/.well-known/oauth-protected-resource/v2 |
| MCP protected resource | {{MCP_PROTECTED_RESOURCE_URL}} |
| Agent skills index | https://www.doctorsim.com/.well-known/agent-skills/index.json |
