# doctorSIM Agent Skills

Public mirror of the doctorSIM agent skill bundle. **Canonical, always-current source:** https://www.doctorsim.com/agents/SKILL.md

This repository is generated automatically from the doctorSIM website on every change. Do not edit by hand.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | Primary agent skill (API v2 + MCP, OAuth) |
| `index.json` | agentskills.io discovery index (with `sha256`) |
| `references/` | API overview, order flow, webhooks, errors, Grok Bot, local MCP |
| `grok-plugin/` | Tavily-shaped Grok Bot / Grok Build plugin (MCP + skill) |

## Connect via MCP

Remote connector URL (Claude.ai, ChatGPT, Grok, Grok Bot): `https://api.doctorsim.com/mcp`

Claude / ChatGPT: leave Client ID/Secret blank (DCR + PKCE). Grok Bot: Authenticate on add. grok.com may ask for the published public Client ID in grok-bot.md. Full guide: https://www.doctorsim.com/auth.md

## Verify integrity

```bash
shasum -a 256 SKILL.md   # compare to index.json -> skills[0].sha256
```
