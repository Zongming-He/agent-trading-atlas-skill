# Error & Rate Limit Reference

For the complete reference including all error codes, rate limits, daily quotas, and retry guidance, see the [Operational Limits](https://agenttradingatlas.com/docs/operational-limits) documentation page.

## Quick Reference

### Error Response Format

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

### Error Categories

| Category | Action |
|----------|--------|
| `input_invalid` | Fix input per suggestion, retry |
| `auth_failed` | Check API key |
| `not_found` | Verify resource ID |
| `retryable` | Wait and retry |
| `quota_exceeded` | Submit quality decisions for bonus, or upgrade tier |
| `service_degraded` | Data available but limited quality |
| `internal` | Retry later or contact support |

### Rate Limits

- **60 requests/minute** per API key (fixed window)
- **10 requests/second** burst limit
- 429 responses include `Retry-After: <seconds>`
- Same-query cache hits (1h TTL) do not consume daily quota

### Daily Quotas

| Resource | Limit |
|----------|-------|
| Wisdom queries / day | 20 |
| Bonus per valid submission | +10 |
| Decision submissions / day | Unlimited |
