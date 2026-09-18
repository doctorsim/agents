---
name: doctorsim-commerce
description: Buy mobile top-ups, gift cards, and travel eSIMs through doctorSIM MCP. Use when the user wants airtime, a digital gift card, or a travel data eSIM.
---

# doctorSIM commerce

Use the `doctorsim` MCP server (`https://api.doctorsim.com/mcp`). Follow the public skill at https://www.doctorsim.com/agents/SKILL.md.

## Rules

- **Guest / consumer:** skip `preview_order`. Call `create_order` as soon as the product is chosen. Show only `payment_link` and `order_id` if present. Never display `checkout_hash`. Delivery is email after they pay on doctorsim.com — do not promise an in-chat QR or code.
- **PRO credits:** `preview_order` first (mandatory), show the breakdown, confirm, then `create_order`. Do not invent balances.
- Top-up needs a phone (E.164). Gift cards need country + brand. eSIM needs a destination ISO-2 — no phone. Use strict `country_iso` (`es`, not `ESP`).
- Offer account linking only when the user asks for credits, history, or saved settings.
- Never name wholesale providers. Never ask the user to paste API secrets into chat if OAuth is available.

## Flows

**Top-up:** `lookup_carrier` → service types → `get_operator_rates` → guest `create_order` / PRO `preview_order` then `create_order`.

**Gift card:** `get_giftcard_brands` → `get_giftcard_brand_products` → guest create / PRO preview then create.

**Travel eSIM:** `get_esim_destinations` → `get_esim_plans` (`country_iso`) → guest create / PRO preview then create. Fulfilled **PRO** orders include ICCID, LPA, and QR URL in chat. Guest eSIMs arrive by email.
