# doctorSIM plugin for Grok

Install doctorSIM into **Grok Bot**, **Grok Build**, or **grok.com** so an agent can request mobile top-ups, gift cards, and travel eSIMs.

This package is a `.grok-plugin/plugin.json` manifest, a remote `.mcp.json`, and a commerce skill. Nothing runs locally. The plugin only points at the hosted MCP Worker.

## What it does

After install, Grok stores a guest or account token from the add-time authorize page. Guest catalog and `payment_link` checkout work without a doctorSIM login. PRO credits need **Sign in**.

| Vertical | How the user asks | Tools |
|---|---|---|
| Mobile top-up | Phone number + amount or bundle | `lookup_carrier` → rates → guest `create_order` / PRO `preview_order` then `create_order` |
| Gift cards | Country + brand | `get_giftcard_brands` → products → guest create / PRO preview then create |
| Travel eSIM | Destination (no phone) | `get_esim_destinations` → `get_esim_plans` → guest create / PRO preview then create |

PRO accounts can debit prepaid credits. Everyone else gets a doctorsim.com payment link. Guest products are emailed after payment — do not promise an in-chat QR or code.

## Install (any Grok Bot)

### A. Custom MCP (works today, no marketplace review)

1. Open **Grok Bot → Settings → Plugins**.
2. Add a custom MCP (not Marketplace):
   - **Name:** `doctorSIM`
   - **URL:** `https://api.doctorsim.com/mcp`
   - **Headers:** leave empty
3. Click **Authenticate**. On doctorsim.com choose **Continue as guest** or **Sign in**. Do not paste an API key. Do not ask the bot to add the URL.
4. If Grok shows **OAuth Credentials Required**, paste Client ID `cli_e97f6079db704bcc0a0c39b8` and leave the secret empty.
5. Enable the connector on the bot (or attach with `@doctorSIM`).
6. Start with a guest catalog question, then a travel eSIM checkout **link** (Demo B). Do not start with a PRO gift card.

Same URL on **grok.com → Connectors → New → Custom** (use the Client ID on Grok’s credentials screen).

### B. Grok Build CLI

```text
/plugin
```

Search **doctorSIM** after the official marketplace PR merges. Until then, add the remote server:

```bash
grok mcp add doctorsim --transport http --url https://api.doctorsim.com/mcp
```

### C. Official Grok plugin list

xAI’s catalog is an index, not an upload portal: [xai-org/plugin-marketplace](https://github.com/xai-org/plugin-marketplace).

1. Publish this folder on the public Git repo (`https://github.com/doctorsim/agents` path `grok-plugin` — repo exists; this folder is not on `main` yet).
2. Pin a full 40-character commit SHA in `marketplace-entry.json`.
3. Fork the marketplace repo, append that entry to `.grok-plugin/marketplace.json`.
4. Run `python3 scripts/generate-plugin-index.py` and `python3 scripts/validate-catalog.py`.
5. Open a PR. Code-owner review is required. See `CONTRIBUTING.md` in that repo.

We cannot merge that PR from the www repo. A human with GitHub access to `doctorsim` plus an xAI marketplace PR is required.

## Authentication

| Mode | When | What the user does |
|---|---|---|
| Guest | Browse + payment-link checkout | Continue as guest on the authorize page |
| OAuth | PRO credits, history, saved settings | Sign in on doctorsim.com (add or Reauthenticate) |
| API key | PRO only, scripts | `Authorization: Bearer {api_id}:{api_secret}` — do not put keys in the plugin JSON |

MCP endpoint: `https://api.doctorsim.com/mcp`  
OAuth: `https://www.doctorsim.com/auth.md`

## Resources

- Public install: https://www.doctorsim.com/agents/references/grok-bot.md
- Skill: https://www.doctorsim.com/agents/SKILL.md
- MCP docs: https://www.doctorsim.com/api-docs/mcp.html
- Developers: https://www.doctorsim.com/developers-agents.html
- Staff plan: `cms/docs/agentic/grok-bot-plugin.html`

## License

MIT
