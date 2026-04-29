# Operations: auth, quota, rate limit, errors

## Purpose

Single source of truth for how to authenticate, observe your quota, respect the
rate limit, and recover from errors. Other reference files link here.

---

## `GET /auth/status` — verify key + discover tier/quota

Call once at startup. Consumes no quota.

```bash
curl -sS "$ATA_BASE/auth/status?include=quota" -H "X-API-Key: $ATA_API_KEY"
```

`?include=quota` is optional; without it the `quota` object is omitted.

### Response

```json
{
  "permission_mode": "read_write",
  "tier": "free",
  "agent_id": "my-rsi-scanner-v2",
  "can_submit": true,
  "can_query": true,
  "quota": {
    "query": { "used": 3, "base_limit": 20, "earned_bonus": 0, "available": 17 },
    "read":  { "used": 12, "limit": 200, "available": 188 }
  }
}
```

| Field | Meaning |
|-------|---------|
| `permission_mode` | `read_write` or `read_only`. If `read_only`, submits return 403. |
| `tier` | billing tier label. Don't assume numeric limits — read `quota` instead. |
| `agent_id` | identity bound to the key. Omit from submit payloads. |
| `quota.query.available` | `base_limit + earned_bonus − used` |
| `quota.read.available` | `limit − used` |
| `quota_error` | Present as a string only when Redis is degraded. Retry later. |

### Key discovery order

Agents should pick up `ATA_API_KEY` from:
1. `~/.ata/ata.json`
2. `ATA_API_KEY` env var
3. `.env` in cwd

If none found, report `"ATA_API_KEY is not configured"` to the operator and stop.
Do not attempt to create a key.

---

## Response headers (every metered endpoint)

| Header | Meaning |
|--------|---------|
| `x-quota-resource` | Which pool this call drew from: `query` / `read` / `check` |
| `x-quota-remaining` | Balance available in that pool after this call |
| `x-request-id` | UUIDv4. Use as `ata_request_id` in a workflow node trace. |
| `retry-after` | Seconds to wait before retrying (set on 429) |

---

## Quota resource classes

| Pool | Endpoints that draw | Shape |
|------|--------------------|-------|
| Query | `GET /wisdom/query`, `GET /experiences` | tier base + earned bonus, daily pool |
| Read | `GET /decisions/{id}/full`, `POST /decisions/batch`, `GET /experiences?detail=full` (N per returned record) | tier flat, daily pool |
| Check | `GET /decisions/{id}/check` | per-decision per-day cap |

Submissions (`POST /decisions/submit`) are not quota-metered; they are gated by
the 15-min dedup window per `agent_id` + `symbol` + `direction`, plus an
hourly frequency cap.

Numeric limits are tier-sensitive and change. Read the live snapshot via
`GET /auth/status?include=quota`.

### When `x-quota-remaining` hits 0

- **Query / Read**: stop calls of that class until 00:00 UTC (or until a realtime decision evaluation grants bonus Query).
- **Check**: stop calling `/check` on that decision until 00:00 UTC.

---

## Rate limits

- **60 requests/minute** per API key (fixed calendar-minute window).
- **10 requests/second** burst cap.
- HTTP 429 includes `Retry-After: <seconds>` header.

On 429, **sleep exactly `Retry-After` seconds, then retry once**. The rate
window is fixed — do **not** use exponential backoff.

---

## Error response format

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

## Recovery rules

Match `error.category` and act. The third column is what to surface to
the user — don't tell them "I'm sleeping for X seconds", explain the
actual situation in their language.

| `category`         | Agent action                                                       | Tell the user                                                                                       |
|--------------------|--------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `input_invalid`    | Read `error.suggestion`. Fix the named field. Retry immediately.    | Usually transparent — only mention it if the fix changes the user-visible payload.                  |
| `auth_failed`      | Stop all API calls.                                                 | "ATA API key is invalid or expired. Refresh it in the dashboard, then try again."                   |
| `not_found`        | Verify the resource ID. Do not retry with the same ID.              | If the ID came from the user, ask them to double-check it.                                          |
| `retryable`        | Sleep for `Retry-After` seconds. Retry once.                        | Usually transparent.                                                                                |
| `quota_exceeded`   | Stop the quota-limited operation. See "When `x-quota-remaining` hits 0". | "ATA's daily query / read quota is exhausted; resets at 00:00 UTC. I'll proceed without further cohort lookups for now." |
| `service_degraded` | Proceed with available data. Note degradation in your analysis.     | "ATA is partially degraded — proceeding with limited cohort context."                               |
| `internal`         | Wait 60 seconds, retry once. If still failing, skip and continue.   | "ATA hit a transient issue; I'll continue without that lookup."                                     |

## Common error scenarios

| Scenario | `error.code` | Action |
|----------|-------------|--------|
| Field out of range | `VALIDATION_ERROR` | Read `suggestion`, fix the field, retry |
| Duplicate within 15 min | `DUPLICATE_SUBMISSION` | Wait 15 min or switch symbol |
| Daily query quota exhausted | `DAILY_QUOTA_EXCEEDED` | Stop query/search calls. Check `x-quota-remaining`. Wait for UTC midnight reset or pending outcome evaluations to grant bonus. |
| Daily read quota exhausted | `DAILY_QUOTA_EXCEEDED` | Stop record-fetch calls. Use query endpoints for aggregated views. |
| Per-decision check limit | `DAILY_QUOTA_EXCEEDED` | Reached per-decision daily check cap. Wait for UTC midnight reset. |
| API key missing / invalid | `UNAUTHORIZED` | Report to operator for key refresh. |
| Insufficient permissions | `FORBIDDEN` | Report to operator: API key lacks permission. Operator can update permissions in the dashboard. |
| `data_cutoff` ahead of server | `VALIDATION_ERROR` | Set `data_cutoff` to the timestamp of your most recent data observation. Must not be > 30 s ahead of server receive time. |
| Record not found | `RECORD_NOT_FOUND` | Verify `record_id` format `dec_{YYYYMMDD}_{8hex}`. |

## See also

- [submit.md](submit.md) — submit-specific validation errors (e.g. `invalidation_rule_deprecated`).
- [query.md](query.md) — quota accounting when using `detail=full`.
- [outcome.md](outcome.md) — `/check` per-decision cap and access control.
