# Order webhooks

Register via `POST /v2/webhooks/orders` with HTTPS callback URL.

## Events

- `order.created`
- `order.completed`
- `order.failed` — delivery or fulfillment could not complete; some cases that need manual follow-up are exposed as `MANUAL_REVISION` (operator or provider issue, out of stock, invalid MSISDN, etc.)

## Verification

Webhook payloads include HMAC signature headers documented in OpenAPI. Verify before acting on agent-side.

## Agentic orders

Orders placed via API/MCP receive Zendesk tag `agentic_api_order`. Status sync updates `api_webhook_order_watch` when ticket status changes.
