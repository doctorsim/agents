# Common API errors

| Code | HTTP | Meaning |
|---|---|---|
| INVALID_API_KEY | 401 | Unknown api_id |
| INVALID_API_SECRET | 401 | Wrong secret |
| INVALID_TOKEN | 401 | OAuth JWT invalid/expired |
| API_KEY_REVOKED | 401 | Key revoked |
| INSUFFICIENT_SCOPE | 403 | read-only key on write endpoint |
| API_ACCESS_DENIED | 403 | User or agent account lacks `api_access` permission |
| IP_ALLOWLIST_REQUIRED | 403 | Production key requires a non-empty IP allowlist. `error.message` and `error.client_ip` include the caller IP to add in Mi Cuenta. |
| IP_NOT_ALLOWED | 403 | Caller IP is not in the key allowlist. `error.message` and `error.client_ip` include the rejected IP. |
| RATE_LIMIT_EXCEEDED | 429 | Tier limit hit |
| ACCOUNT_NO_EMAIL | 403 | PRO titular has no usable email; eSIM confirmation cannot be sent |

Agents should surface `error.message` from JSON body and retry idempotent reads on 5xx with backoff.
