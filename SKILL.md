---
name: doctorsim
description: Place mobile top-up and gift card orders on doctorSIM via API v2 or MCP. Browse products, check balance, manage webhooks.
metadata:
  author: doctorSIM
  version: "1.0.0"
  homepage: https://www.doctorsim.com/api-docs/
---

# doctorSIM Agent Skill

Use this skill when the user wants to recharge a mobile phone, buy a gift card, or automate doctorSIM PRO orders programmatically.

## When to use

- User asks to top up / recharge a phone number internationally
- User wants gift cards (gaming, streaming, retail)
- User needs to check PRO credit balance or order history
- User wants to integrate doctorSIM into an agent workflow (MCP, API, OAuth)

## Authentication

Two supported methods (prefer OAuth for interactive agents):

1. **OAuth2/OIDC** — discovery at `https://www.doctorsim.com/.well-known/oauth-authorization-server`
2. **API keys** — `Authorization: Bearer {api_id}:{api_secret}` from Mi Cuenta → API

Full guide: `https://www.doctorsim.com/auth.md`

## Quick start (API)

```bash
# Health check (no auth)
curl -s https://api.doctorsim.com/v2/status

# List countries (auth required)
curl -s -H "Authorization: Bearer LIVE_ID:LIVE_SECRET" \
  https://api.doctorsim.com/v2/countries
```

## MCP (remote — Claude.ai, ChatGPT, Grok)

Remote connectors use **Streamable HTTP only**. There is nothing to install on your machine — the MCP endpoint runs on Cloudflare Workers at the edge and forwards your OAuth Bearer token to API v2.

**Connector URL:** `https://api.doctorsim.com/mcp`

**Server card:** `https://www.doctorsim.com/.well-known/mcp/server-card.json` (primary transport: `streamable-http`)

### Setup

1. In your agent product, add a **custom MCP connector** with URL `https://api.doctorsim.com/mcp`.
2. **Leave any "OAuth Client ID / Secret" fields blank.** This server supports Dynamic Client Registration (RFC 7591) and issues a **public client** (`token_endpoint_auth_method: none`) authenticated with **authorization code + PKCE** — there is no client secret to paste. The connector handles discovery → register → authorize → token automatically.
3. Complete OAuth in the browser popup when prompted:
   - Authorization server: `https://www.doctorsim.com/.well-known/oauth-authorization-server`
   - Dynamic Client Registration: `POST https://www.doctorsim.com/oauth/register`
   - User authorize + token exchange per `https://www.doctorsim.com/auth.md`
4. At the consent screen, approve the connector's requested scopes. The recommended grant is **`orders:read orders:write balance:read`** — placing orders is the primary purpose of this connector, so `orders:write` is included (it debits real PRO credits on `create_order`). Approve a read-only subset only if the integration genuinely never places orders.
5. The connector sends `Authorization: Bearer {access_token}` on every MCP `POST`. Access tokens last 1 hour; the connector refreshes automatically.
6. Use MCP tools (`get_countries`, `get_operator_rates`, `preview_order`, `create_order`, `get_balance`, …). `get_operator_rates` returns the exact `service_fee_eur/usd` and `total_cost_eur/usd` per price point; call `preview_order` to confirm the full cost (incl. optional 0.99 EUR SMS fee) before `create_order`.

> The `api_key` path (`Authorization: Bearer {api_id}:{api_secret}`) is for **direct API / server integrations only** — remote MCP connectors must use the OAuth flow above.

### Verify (no auth)

```bash
curl -s https://api.doctorsim.com/mcp | jq .
curl -s https://api.doctorsim.com/mcp/health | jq .
```

> **Do not configure stdio for remote connectors.** stdio is optional and only for local IDE plugins (Cursor, Claude Desktop). See [local MCP setup](https://www.doctorsim.com/agents/references/local-mcp.md) if you need a local process — not required for Claude.ai or ChatGPT.

## Credits

Orders debit PRO account credits. Fund credits via the web dashboard (pro-pro checkout). Programmatic credit purchase (x402/ACP) is not yet available.

## References

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
