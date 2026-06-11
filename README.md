# doctorSIM Agent Skills

Public mirror of the doctorSIM agent skill bundle. **Canonical, always-current source:** https://www.doctorsim.com/agents/SKILL.md

This repository is generated automatically from the doctorSIM website on every change. Do not edit by hand.

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | Primary agent skill (API v2 + MCP, OAuth) |
| `index.json` | agentskills.io discovery index (with `sha256`) |
| `references/` | API overview, order flow, webhooks, errors, local MCP |

## Connect via MCP

Remote connector URL (Claude.ai, ChatGPT): `https://api.doctorsim.com/mcp`

OAuth is dynamic (RFC 7591) with PKCE — leave any Client ID/Secret fields blank. Full guide: https://www.doctorsim.com/auth.md

## Verify integrity

```bash
shasum -a 256 SKILL.md   # compare to index.json -> skills[0].sha256
```
