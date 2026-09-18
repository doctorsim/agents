# Install doctorSIM on Grok

Add doctorSIM to **Grok Bot**, **Grok Build**, or **grok.com** so the assistant can buy mobile top-ups, gift cards, and travel eSIMs.

**MCP URL:** `https://api.doctorsim.com/mcp`

This is the same hosted Worker used by Claude and ChatGPT. Add a **custom MCP** (not a marketplace plugin). Leave headers empty. Do **not** paste an API key. Do **not** ask the bot to add the URL — that often attaches a saved key and leaves the host with 0 tools.

Grok and Grok Bot decide auth **once, while adding**. After save you either click **Authenticate** (Grok Bot / Cursor host) or paste the published public Client ID (grok.com). Then doctorsim.com offers **Sign in** (PRO credits) or **Continue as guest** (catalog + payment-link checkout).

## 1. Grok Bot (any bot)

Plugins are account-wide. Add doctorSIM once, then enable it on every bot (or `@` it).

1. Open **Settings → Plugins**.
2. Add a **custom MCP** (Marketplace is for catalog plugins — doctorSIM is not listed there yet):
   - Name: `doctorSIM`
   - Server URL: `https://api.doctorsim.com/mcp`
   - Headers: leave empty
3. Save. Click **Authenticate**. On doctorsim.com choose **Continue as guest** or **Sign in**.
4. If Grok shows **OAuth Credentials Required** instead, paste Client ID `cli_e97f6079db704bcc0a0c39b8`, leave the secret empty, keep PKCE on, Save.
5. Confirm ~20 tools under **Plugins → Yours**. Enable the plugin on the bot, or type `@doctorSIM`.
6. First proof (guest): “List the doctorSIM tools you have,” then “What travel eSIM plans exist for Spain?” Then Demo B — a checkout **link**, not a gift card on credits.

If the host shows 0 tools / a Connect card, **remove** every doctorSIM connector and add it again from the form. Dismiss any “PRO API key” card.

## 2. grok.com connectors

1. Go to [grok.com/connectors](https://grok.com/connectors).
2. If an old `doctorSIM` connector exists (added with no OAuth screen), **remove** it first.
3. **New Connector → Custom**. Paste `https://api.doctorsim.com/mcp`. Headers empty.
4. **OAuth Credentials Required:** Client ID `cli_e97f6079db704bcc0a0c39b8`, secret empty, scopes `openid profile orders:read orders:write balance:read webhooks:read`, PKCE on.
5. doctorsim.com → **Sign in** (PRO) or **Continue as guest**.

Guest tokens cannot call balance / history. Use **Reauthenticate → Sign in** (do not expect a mid-chat OAuth popup).

## 3. Grok Build / official plugin list

Until doctorSIM is in the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace), add the server yourself:

```bash
grok mcp add doctorsim --transport http --url https://api.doctorsim.com/mcp
```

After xAI merges the catalog PR, install from `/plugin` by searching **doctorSIM**. That PR is a human step: publish public `doctorsim/agents`, pin the 40-char SHA in `grok-plugin/marketplace-entry.json`, then open the catalog PR.

## 4. What to expect

| You ask | Guest (Continue as guest) | Signed-in PRO |
|---|---|---|
| Browse countries / plans | Yes | Yes |
| Place order | Show only `payment_link` (+ `order_id` if any). Pay on doctorsim.com. Product arrives by **email**. Never read `checkout_hash` aloud. | Preview → confirm → credits; gift card code or eSIM QR in chat |
| Order history / balance | Reauthenticate → Sign in | Yes |

Guest and consumer skip preview and create the payment link as soon as the product is chosen. PRO must `preview_order` first.

Full tool list and flows: [SKILL.md](https://www.doctorsim.com/agents/SKILL.md) · [MCP server](https://www.doctorsim.com/agents/references/mcp-server.md)
