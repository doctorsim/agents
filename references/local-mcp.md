# Local MCP (stdio) — Cursor & Claude Desktop only

**Not for Claude.ai, ChatGPT, or other remote MCP connectors.** Those products use Streamable HTTP at `https://api.doctorsim.com/mcp` — see the main [SKILL.md](https://www.doctorsim.com/agents/SKILL.md).

Use this guide only when your IDE spawns a **local Node process** (stdio transport).

## When to use

- **Cursor** or **Claude Desktop** with a local MCP server entry
- Local development without OAuth (API key in env vars)
- **Not** required for production agent workflows

## Build

From a clone of the doctorSIM repository:

```bash
cd cms/docs/agentic/mcp-server
npm ci
npm run build    # compiles TypeScript → dist/
npm start        # runs dist/stdio.js
```

## Environment variables

| Variable | Required | Example |
|---|---|---|
| `DSIM_API_BASE_URL` | No (default prod) | `https://api.doctorsim.com/v2` |
| `DSIM_API_ID` | Yes | `test_…` or `live_…` |
| `DSIM_API_SECRET` | Yes | `sk_…` |

Generate keys in **Mi Cuenta → API** (PRO account with `api_access`).

## Cursor configuration

See `cms/docs/agentic/mcp-server/cursor-mcp.example.json` in the repo. Example:

```json
{
  "mcpServers": {
    "doctorsim-api": {
      "command": "node",
      "args": ["/ABS/PATH/TO/www/cms/docs/agentic/mcp-server/dist/stdio.js"],
      "env": {
        "DSIM_API_BASE_URL": "https://api.doctorsim.com/v2",
        "DSIM_API_ID": "test_XXXXXXXXXXXXXXXXXXXXXXXX",
        "DSIM_API_SECRET": "sk_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
      }
    }
  }
}
```

## Claude Desktop

Add the same `command` / `args` / `env` block to your Claude Desktop MCP config (stdio transport).

## Relationship to the Cloudflare Worker

| Transport | Where | Audience |
|---|---|---|
| **Streamable HTTP** | `https://api.doctorsim.com/mcp` (Cloudflare Worker) | Claude.ai, ChatGPT, remote agents |
| **stdio** | Local Node (`dist/stdio.js`) | Cursor, Claude Desktop |

Both share the same tool definitions. Tool logic lives in `cms/docs/agentic/mcp-server/src/tools.ts`; the Worker uses a JS port at `cloudflare-workers/mcp-server/src/tools.js`. When tools change, sync both — see `cms/docs/agentic/mcp-server/DEPLOY.md`.
