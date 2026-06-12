# Order webhooks

Register via `POST /v2/webhooks/orders` with HTTPS callback URL.

## Events

- `order.created`
- `order.completed`
- `order.failed` — delivery or fulfillment could not complete; some cases that need manual follow-up are exposed as `MANUAL_REVISION` (operator or provider issue, out of stock, invalid MSISDN, etc.)

## Verification

Webhook payloads include HMAC signature headers documented in OpenAPI. Verify before acting on agent-side.

## API and MCP orders

Webhook events apply to orders placed through the API or MCP connector. Poll `GET /v2/orders/{id}` if you do not use webhooks.
