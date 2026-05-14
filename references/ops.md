# Operations: auth, quota, rate limit, errors

The single source of truth for **how to recover** when something goes
wrong with an ATA call. Every other reference file links here.

---

## `GET /auth/status` — verify your key

Free; consumes no quota. Call once at startup to confirm the key is
live and read your effective capabilities.

```bash
curl -sS "$ATA_BASE/api/v1/public/auth/status" -H "X-API-Key: $ATA_API_KEY"
```

```json
{
  "user_id": "8d2c…",
  "email": "owner@example.com",
  "tier": "free",
  "permission_mode": "read_write",
  "agent_id": "rsi-scanner-v2",
  "can_submit": true,
  "can_query": true
}
```

| Field | Meaning |
|-------|---------|
| `permission_mode` | `read_write` or `read_only`. If `read_only`, submits return 403 `PERMISSION_DENIED`. |
| `tier` | Billing tier label. Numeric quota limits vary by tier — read them from response headers, not by guessing. |
| `agent_id` | Identity bound to this key. Omit it from submit payloads. |
| `can_submit`, `can_query` | Effective capability after permission_mode + tier. |

The endpoint does not return a quota body. Quota is observed only via
response headers on metered calls (see below).

### Key discovery order

When picking up `ATA_API_KEY`, check in order:

1. `~/.ata/ata.json`
2. `ATA_API_KEY` env var
3. `.env` in the current directory

If none is found, tell the operator `"ATA_API_KEY is not configured"`
and stop. Do not try to create a key.

---

## Response headers (every metered endpoint)

| Header | Meaning |
|--------|---------|
| `x-quota-resource` | Which pool the call drew from: `query` / `read` / `check` |
| `x-quota-remaining` | Available balance in that pool after this call |
| `x-quota-limit` | Pool limit for the current tier |
| `x-quota-reset` | When the pool refills (unix timestamp or RFC 3339) |
| `x-ratelimit-limit` | Per-key request budget for the current rate-limit window |
| `x-ratelimit-remaining` | Requests left in the current window |
| `x-ratelimit-reset` | When the rate-limit window resets (unix timestamp) |
| `x-request-id` | Per-request UUID. Quote this when reporting issues. |
| `retry-after` | Set on 429 responses. Seconds to wait before retrying. |

Read `x-quota-remaining` after each call — when it hits 0, stop calling
that pool. Don't hard-code numeric limits; they vary by tier.

---

## Quota pools

| Pool | Endpoints | Shape |
|------|-----------|-------|
| `query` | `GET /wisdom/query`, `GET /experiences` | Tier base + earned bonus, daily |
| `read` | `GET /decisions/{id}/full`, `POST /decisions/batch`, `GET /experiences?detail=full` (1 Read per record returned) | Tier flat, daily |
| `check` | `GET /decisions/{id}/check` | Per-decision per-day cap |

Submissions (`POST /decisions/submit`) are not quota-metered. They are
gated by the 15-minute dedup window per `(agent_id, symbol, direction,
holding_seconds)`.

When `x-quota-remaining` reaches 0:

- **query / read**: stop calls of that pool until `x-quota-reset`.
- **check**: stop polling that record until reset; other records still
  work.

---

## Rate limits

Per-key rate limit is enforced per request. The exact budget is whatever
`x-ratelimit-limit` reports for your tier. On 429 the response carries a
`retry-after` header.

On 429, sleep exactly `retry-after` seconds, then retry **once**. The
window is fixed — exponential backoff just wastes time.

---

## Error response format

```json
{
  "error": {
    "code": "BAR_INTERVAL_HOLDING_MISMATCH",
    "message": "bar_interval=1d (86400s) × holding_seconds=86400 yields fewer than 2 evaluation bars",
    "category": "input_invalid",
    "suggestion": "Pick a bar_interval where holding_seconds / bar_interval.seconds falls in [2, 50000]"
  }
}
```

Branch on `error.category`. The third column is what you say to the
human user — translate; don't expose internal mechanics.

| `category` | Agent action | Tell the user |
|------------|--------------|---------------|
| `input_invalid` | Read `error.suggestion`. Fix the named field. Retry once. | Usually transparent. |
| `auth_failed` | Stop all API calls. | "ATA API key is invalid, expired, or lacks the needed permission. Refresh it in the dashboard, then try again." |
| `not_found` | Verify the resource ID. Do not retry with the same ID. | If the ID came from the user, ask them to double-check it. |
| `retryable` | Sleep `retry-after` seconds. Retry once. | Usually transparent. |
| `quota_exceeded` | Stop the metered operation. See "When `x-quota-remaining` reaches 0" above. | "ATA's daily query/read quota is exhausted; resets soon. Proceeding without further cohort lookups." |
| `service_degraded` | Proceed with whatever data did come back. Note the degradation in your analysis. | "ATA is partially degraded — proceeding with limited cohort context." |
| `internal` | Wait 60 seconds, retry once. If still failing, skip and continue. | "ATA hit a transient issue; continuing without that lookup." |

### Common error codes

| Scenario | `error.code` | HTTP | Action |
|----------|--------------|------|--------|
| Field out of range / unknown field | `VALIDATION_ERROR` | 400 | Read `suggestion`, fix, retry |
| Bad symbol shape | `INVALID_SYMBOL` | 400 | Use canonical ticker (`NVDA`, `BTC-USDT`) |
| `bar_interval` × `holding_seconds` not usable | `BAR_INTERVAL_HOLDING_MISMATCH` | 400 | Pick a coarser bar or longer hold |
| Missing or expired key | `UNAUTHORIZED` | 401 | Stop; tell operator to refresh the key |
| Read-only key tried to submit | `PERMISSION_DENIED` | 403 | Stop; operator can flip the key to read_write |
| Insufficient permission | `FORBIDDEN` | 403 | Stop; report to operator |
| `data_cutoff` ahead of server clock | `VALIDATION_ERROR` | 400 | Use the timestamp of your most recent data observation; must not be > 30 s ahead of server time |
| `record_id` not found | `RECORD_NOT_FOUND` | 404 | Verify the format `dec_{YYYYMMDD}_{8hex}`. Note: `/agents/{agent_id}/profile` also returns this when the agent isn't yours — don't infer existence from it |
| Same-shape duplicate within 15 min | `DUPLICATE_SUBMISSION` | 409 | Wait the rest of the cooldown, or change one of `(symbol, direction, holding_seconds)` |
| Per-key rate limit | `RATE_LIMIT_EXCEEDED` | 429 | Sleep `retry-after`, retry once |
| Daily quota exhausted | `DAILY_QUOTA_EXCEEDED` | 429 | Stop; reset is on `x-quota-reset` |
| Wisdom query had too few records | `WISDOM_DATA_SPARSE` | 200 | Degraded, not fatal — fall back to your own analysis |
| Price feed temporarily missing | `PRICE_DATA_STALE` | 200 | Degraded; warn the user the live price may be stale |

---

## Probing safely

Validation short-circuits: the server returns the first failed rule,
not all of them. **Each submit is a real submission** — the moment all
rules pass, the record lands permanently. There is no dry-run mode.
Iterating fixes one field at a time with otherwise-real values is the
single fastest way to land a record you didn't mean to publish.

Two patterns to avoid accidental commits:

1. **Don't iterate with real intent.** Construct the full payload from
   [submit.md](submit.md), send once, accept the outcome. If validation
   fails, read `error.suggestion`, fix the field, resubmit — and treat
   that resubmit as a real submission, not a probe.

2. **Use a guaranteed-rejected probe payload** if you genuinely need
   to discover the schema (e.g. you suspect this skill has drifted
   from the current server contract). Three independent defenses, all
   present at once:

   - `symbol`: a non-existent instrument (e.g. `ZZZNONEXIST-USDT`)
   - `data_cutoff`: far in the future (e.g. `2099-01-01T00:00:00Z`)
   - `price_at_decision`: orders of magnitude off market (e.g. `0.000001`)

   Any one of these makes the record reject; all three together make
   accidental commit impossible during iteration. Once the shape is
   right, replace all three at once and submit.

If error codes or response shapes don't match what this doc describes,
suspect skill drift — surface the raw error to the user and stop
iterating. Don't try to reverse-engineer the contract from error
messages.

## See also

- [submit.md](submit.md) — submit-specific validation errors and the response branches.
- [query.md](query.md) — quota cost of `detail=full` searches.
- [outcome.md](outcome.md) — the per-decision `/check` cap and access-control behavior.
