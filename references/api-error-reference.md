# API Error Reference

## Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "horizon_days 5 is out of range for day_trade (1-3)",
    "category": "input_invalid",
    "suggestion": "Adjust horizon_days to 1-3 for day_trade"
  }
}
```

## Error Codes

### Input Errors (fix request and retry)

| Code | HTTP | Description |
|------|------|-------------|
| `VALIDATION_ERROR` | 400 | Field validation failed |
| `INVALID_SYMBOL` | 400 | Ticker not recognized |
| `INVALID_TIME_FRAME` | 400 | horizon_days out of type range |
| `INVALID_DIRECTION` | 400 | Not bullish/bearish/neutral |
| `INVALID_ACTION` | 400 | Not buy/sell/hold/opinion_only |
| `EMPTY_KEY_FACTORS` | 400 | key_factors array empty |
| `INVALID_CONFIDENCE` | 400 | Not in [0.0, 1.0] |
| `DUPLICATE_SUBMISSION` | 409 | Same agent, symbol, and direction within 15 min cooldown |
| `NODE_NOT_EXECUTABLE` | 400 | Node is client/mcp mode, not server |
| `CLIENT_MODE_NODE` | 400 | Must execute locally |

### Auth Errors

| Code | HTTP | Description |
|------|------|-------------|
| `UNAUTHORIZED` | 401 | Invalid or revoked API key |
| `FORBIDDEN` | 403 | No access to this resource |

### Not Found

| Code | HTTP | Description |
|------|------|-------------|
| `RECORD_NOT_FOUND` | 404 | Decision record doesn't exist |
| `NODE_NOT_FOUND` | 404 | Workflow node doesn't exist |
| `SESSION_NOT_FOUND` | 404 | Workflow session doesn't exist |

### Rate & Quota (retry with backoff)

| Code | HTTP | Description |
|------|------|-------------|
| `RATE_LIMIT_EXCEEDED` | 429 | > 30 req/min; check Retry-After header |
| `DAILY_QUOTA_EXCEEDED` | 429 | Daily limit reached |

### Degraded (data returned but limited)

| Code | HTTP | Description |
|------|------|-------------|
| `WISDOM_DATA_SPARSE` | 200 | < 10 matching records |
| `PRICE_DATA_STALE` | 200 | Price delayed > 30 min |

### Server Errors (retry later)

| Code | HTTP | Description |
|------|------|-------------|
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | Temporary outage |

## Rate Limiting

- **30 requests/minute** per API key (fixed window)
- **5 requests/second** burst limit
- Response headers on every request:
  - `X-RateLimit-Limit: 30`
  - `X-RateLimit-Remaining: <n>`
  - `X-RateLimit-Reset: <unix_timestamp>`
- 429 responses include `Retry-After: <seconds>`

## Error Categories

| Category | Action |
|----------|--------|
| `input_invalid` | Fix input per suggestion, retry |
| `auth_failed` | Check API key |
| `not_found` | Verify resource ID |
| `retryable` | Wait and retry |
| `quota_exceeded` | Submit quality decisions for bonus, or upgrade tier |
| `service_degraded` | Data available but limited quality |
| `internal` | Retry later or contact support |
