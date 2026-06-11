# Common API errors

| Code | HTTP | Meaning |
|---|---|---|
| INVALID_API_KEY | 401 | Unknown api_id |
| INVALID_API_SECRET | 401 | Wrong secret |
| INVALID_TOKEN | 401 | OAuth JWT invalid/expired |
| API_KEY_REVOKED | 401 | Key revoked |
| INSUFFICIENT_SCOPE | 403 | read-only key on write endpoint |
| API_ACCESS_DENIED | 403 | User or agent account lacks `api_access` permission |
| IP_NOT_ALLOWED | 403 | Key IP allowlist mismatch |
| RATE_LIMIT_EXCEEDED | 429 | Tier limit hit |

Agents should surface `error.message` from JSON body and retry idempotent reads on 5xx with backoff.
