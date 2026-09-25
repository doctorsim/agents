# doctorSIM

Buy travel eSIMs, mobile top-ups, and digital gift cards through MCP.

Public mirror of the doctorSIM agent skill bundle. **Canonical, always-current source:** https://www.doctorsim.com/agents/SKILL.md

This repository is generated automatically from the doctorSIM website on every change. Do not edit by hand.

## What it does

- **Travel eSIMs** — browse destinations and plans (`get_esim_destinations`, `get_esim_plans`, `get_esim_plan_detail`), then create a guest `payment_link` or a PRO credit order (`create_order`).
- **Mobile top-ups** — identify the carrier from an E.164 number (`lookup_carrier`), pick airtime / bundles / data (`get_operator_service_types`, `get_operator_rates`), then place the order.
- **Digital gift cards** — list brands and denominations (`get_giftcard_brands`, `get_giftcard_brand_products`) and check out the same way.
- **Guest checkout** — catalog browse and `payment_link` checkout work without a doctorSIM login. The buyer pays on doctorsim.com.
- **Optional account linking** — OAuth for PRO prepaid credits, order history, remaining eSIM data, and webhooks.

## Hosted MCP server (recommended)

doctorSIM hosts the production MCP server. You do not need to clone or run this repository to connect an assistant.

| Field | Value |
|---|---|
| Endpoint | `https://api.doctorsim.com/mcp` |
| Transport | Streamable HTTP (JSON-RPC 2.0) |
| Authentication | OAuth 2.0 with Dynamic Client Registration + PKCE. Guest catalog and payment-link checkout work without a token. |
| Server card | https://www.doctorsim.com/.well-known/mcp/server-card.json |
| Health | `GET https://api.doctorsim.com/mcp/health` |
| Docs | https://www.doctorsim.com/api-docs/mcp.html |

```bash
npx add-mcp 'https://api.doctorsim.com/mcp'
```

Installs into Claude Code, Codex, Cursor, and other MCP clients that understand `add-mcp`.

Claude / ChatGPT: leave **Client ID** and **Client Secret** blank. The server issues a public client (`token_endpoint_auth_method: none`) and uses authorization code + PKCE. Full guide: https://www.doctorsim.com/auth.md

**PRO-only URL (optional):** `https://api.doctorsim.com/mcp/pro` — same tools, but every request needs an OAuth Bearer (`initialize` answers 401). Use it on hosts that only start OAuth when the first `initialize` is 401 (some Grok connectors).

## Hosted vs this repo

| Surface | What it is |
|---|---|
| **Hosted MCP** `https://api.doctorsim.com/mcp` | Production connector. Use this for Claude, ChatGPT, Cursor (remote), Grok, and Grok Bot. |
| **This GitHub repo** | Skill bundle mirror (`SKILL.md`, references, `index.json`). Not a self-hosted MCP server. |
| **Local stdio** | Optional IDE-only process for Cursor / Claude Desktop. Not for Claude.ai or ChatGPT. See [local-mcp.md](https://www.doctorsim.com/agents/references/local-mcp.md). |

Unlike a sample implementation you would run with an API key, the hosted server is the supported production path.

## Tools

Live tool names from the hosted server. Aliases stay for existing connectors. Do not invent extra tools.

| Tool | Auth | Purpose |
|---|---|---|
| `get_countries` | Guest | Countries for top-up, travel eSIM, and gift cards |
| `lookup_carrier` | Guest | Operator + country from an E.164 phone. Call this first for top-ups. |
| `get_operators` | Guest | Operators for a country when lookup fails |
| `get_operator_service_types` | Guest | Airtime / bundles / data types for an operator |
| `get_operator_rates` | Guest | Live rates + `token` for the chosen type |
| `search_products` | Guest | Catalog search (top-up operators + gift card brands). Do not use alone for phone top-ups. |
| `get_esim_destinations` | Guest | Travel eSIM destinations (ISO-2 or region slugs such as `eu`, `ww`) |
| `get_esim_plans` | Guest | Plans for one destination (`catalog_id`, `price_token`) |
| `get_esim_plan_detail` | Guest | Single eSIM SKU |
| `get_giftcard_brands` | Guest | Gift card brands for a country |
| `get_giftcard_brand_products` | Guest | Denominations + `token` |
| `preview_order` | Guest optional / **required for PRO credits** | Price breakdown. Does not place the order. |
| `create_order` | Guest or OAuth | Place the order (any vertical). Guest returns `payment_link`. PRO debits credits. |
| `preview_esim_order` | Guest optional | Alias of `preview_order` with `type=esim` |
| `create_esim_order` | Guest or OAuth | Alias of `create_order` with `type=esim` |
| `get_order_status` | Guest via `payment_link` / OAuth | Single-order status. Guests: pass `payment_link`, never a sequential `order_id`. |
| `get_esim_line` | OAuth `orders:read` | Remaining data for a titular-owned ICCID |
| `list_orders` | OAuth `orders:read` | Order history |
| `get_balance` | OAuth `balance:read` (PRO) | Prepaid credit balance |
| `list_webhooks` | OAuth `webhooks:read` (PRO) | Webhook subscriptions |

Flows and field rules: [SKILL.md](https://www.doctorsim.com/agents/SKILL.md) · [MCP reference](https://www.doctorsim.com/agents/references/mcp-server.md)

## How to connect

Same hosted URL everywhere: `https://api.doctorsim.com/mcp`.

### Claude

1. Settings → Connectors → Add custom connector (or use the approved Remote MCP listing when Claude shows it — do not invent a public directory URL).
2. Paste `https://api.doctorsim.com/mcp`. Leave Client ID / Secret blank (DCR + PKCE).
3. Approve consent if Claude opens it. Catalog tools work immediately as a guest.

### ChatGPT

1. Add a custom MCP connector with `https://api.doctorsim.com/mcp`, or use the approved ChatGPT plugin 2.0 when ChatGPT shows it — do not invent a plugin-store URL.
2. Leave Client ID / Secret blank.
3. Ask in plain language (examples below).

### Cursor

Remote (preferred):

```json
{
  "mcpServers": {
    "doctorsim": {
      "url": "https://api.doctorsim.com/mcp"
    }
  }
}
```

Leave OAuth client fields blank. Or run `npx add-mcp 'https://api.doctorsim.com/mcp'`. Local stdio is optional and only for this IDE — see [local-mcp.md](https://www.doctorsim.com/agents/references/local-mcp.md).

### Grok / Grok Bot

Grok chooses auth **on add**, not mid-chat.

- **Grok Bot:** Settings → Plugins → custom MCP. Name `doctorSIM`. URL `https://api.doctorsim.com/mcp`. Headers empty. Click **Authenticate**, then Sign in or Continue as guest.
- **grok.com:** [connectors](https://grok.com/connectors) → New → Custom. If Grok asks for OAuth credentials, use the published public Client ID in [grok-bot.md](https://www.doctorsim.com/agents/references/grok-bot.md). Leave the secret empty. Keep PKCE on.
- Guest tokens cannot call balance or history. Use Reauthenticate → Sign in to upgrade.

Step-by-step: [Install on Grok](https://www.doctorsim.com/agents/references/grok-bot.md)

### Other MCP clients

Point the client at `https://api.doctorsim.com/mcp` (Streamable HTTP). Register as a public client with PKCE, or stay guest for catalog + payment links. Auth walkthrough: https://www.doctorsim.com/auth.md

## Example prompts

English:

- “What travel eSIM plans exist for Spain?”
- “I’m travelling to Japan for two weeks — which eSIM data plans do you have?”
- “Look up the carrier for +522221231231 and show 5GB WhatsApp bundles.”
- “Buy a Steam gift card in Spain and send me the payment link.”

Español:

- “¿Qué planes eSIM de viaje hay para Japón?”
- “Recarga el +522221231231 con saldo. Enséñame el enlace de pago.”
- “Quiero una gift card de Spotify en España.”

Guest / consumer: as soon as the product is chosen, call `create_order` and share only the `payment_link`. PRO credits: `preview_order` first, confirm, then `create_order`.

## Scope and limitations

**MCP can**

- Browse live catalogs for travel eSIMs, mobile top-ups, and digital gift cards across 200+ countries.
- Create a guest / consumer `payment_link` so the buyer pays on doctorsim.com.
- Debit PRO prepaid credits after a confirmed `preview_order` when the account is linked.
- Poll guest status with `payment_link` (or an unguessable `checkout_hash` — never display the hash).
- Return eSIM QR / LPA fields or gift-card redemption codes on **fulfilled PRO** orders (and on guest eSIM polls after payment, when those `show_to_user` fields are present).

**MCP cannot**

- Collect card numbers or complete card payments inside the connector. MCP does not take card details; the buyer pays on doctorsim.com (or with PRO credits).
- Promise an in-chat QR or gift-card code for **guest** checkout before the buyer pays and the product is emailed.
- Look up a sequential `order_id` without OAuth (IDs are enumerable).
- Read PRO balance, order history, webhooks, or `get_esim_line` without the matching OAuth scopes.
- Replace the website dashboard for funding PRO credits.

Rate totals from `get_operator_rates` are indicative. **PRO:** `preview_order` is the authoritative debit breakdown.

## Status, privacy, and support

| Resource | URL |
|---|---|
| Auth guide | https://www.doctorsim.com/auth.md |
| Agent skill | https://www.doctorsim.com/agents/SKILL.md |
| MCP docs | https://www.doctorsim.com/api-docs/mcp.html |
| Developers landing | https://www.doctorsim.com/en/developers-agents.html |
| Privacy | https://www.doctorsim.com/en/privacy.html |
| API terms | https://www.doctorsim.com/en/legal_specific.html#agentic-api-terms |
| Support | https://www.doctorsim.com/en/contact-us.html |
| OpenAPI | https://www.doctorsim.com/api-docs/openapi.yaml |

Hosted MCP is live. ChatGPT official plugin 2.0 and Claude Connectors Directory (Remote MCP) are approved — do not invent those listing URLs. This GitHub mirror is what [mcpservers.org/servers/doctorsim/agents](https://mcpservers.org/servers/doctorsim/agents) scrapes.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | Primary agent skill (API v2 + MCP, OAuth) |
| `index.json` | agentskills.io discovery index (with `sha256`) |
| `references/` | API overview, order flow, webhooks, errors, Grok Bot, local MCP |
| `grok-plugin/` | Tavily-shaped Grok Bot / Grok Build plugin (MCP + skill) |

## Verify integrity

```bash
shasum -a 256 SKILL.md   # compare to index.json -> skills[0].sha256
```
