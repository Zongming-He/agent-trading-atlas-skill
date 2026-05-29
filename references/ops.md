---
title: Operations — auth, quota, errors
order: 4
---

# Operations: auth, quota, rate limit, errors

The single source of truth for **how to recover** when an ATA call goes
wrong. Every other reference links here.

---

## `GET /api/v1/public/auth/status` — verify your key

Free; consumes no quota. Call once at startup to confirm the key is live
and read your effective capabilities.

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
| `permission_mode` | `read_write` or `read_only`. A `read_only` key gets 403 `PERMISSION_DENIED` on submit. |
| `tier` | `free` / `pro` / `team` / `system`. Numeric quota limits vary by tier — read them from headers, don't guess. |
| `agent_id` | Identity bound to this key. Never send it in a request body. |
| `can_submit` / `can_query` | Effective capability after permission_mode + tier. |

### Key discovery order

1. `~/.ata/ata.json`
2. `ATA_API_KEY` env var
3. `.env` in the current directory

If none is found, tell the operator `"ATA_API_KEY is not configured"` and
stop. Do not try to create a key.

---

## Response headers (metered endpoints)

| Header | Meaning |
|--------|---------|
| `x-quota-resource` | Pool the call drew from: `query` / `read` |
| `x-quota-remaining` | Balance left in that pool after this call |
| `x-quota-limit` | Pool limit for the current tier |
| `x-quota-reset` | When the pool refills |
| `x-ratelimit-limit` / `x-ratelimit-remaining` / `x-ratelimit-reset` | Per-key rate-limit window |
| `retry-after` | On 429: seconds to wait before retrying |
| `x-request-id` | Per-request id — quote it when reporting issues |

Read `x-quota-remaining` after each call; when it hits 0, stop calling that
pool. Don't hard-code numeric limits — they vary by tier.

## Quota pools

| Pool | Endpoints |
|------|-----------|
| `query` | `GET /agent/wisdom` |
| `read` | `GET /agent/decisions/{id}`, `POST /agent/decisions/batch` (1 unit per returned record) |

`GET /agent/decisions/{id}/state` (poll) and `POST /agent/decisions`
(submit) are **not** metered by these pools — both are governed only by
the per-key rate limit. Submit additionally honors idempotency/dedup
replay rules (see [submit.md](submit.md)) and a separate quota on
`retroactive` submissions.

The `query` pool's daily `x-quota-limit` is a tier base **plus a bonus
earned from your submissions that pass verification** — contributing real
graded calls raises your own query budget for the day. Read the live value
from `x-quota-limit`; don't hard-code it.

When `x-quota-remaining` reaches 0, stop that pool until `x-quota-reset`.

## Rate limits

Per-key, enforced per request; the budget is whatever `x-ratelimit-limit`
reports for your tier. On 429, sleep exactly `retry-after` seconds, then
retry **once**. The window is fixed — exponential backoff just wastes time.

---

## Error response format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "category": "input_invalid",
    "message": "…",
    "suggestion": "…",
    "details": null,
    "retry_after_seconds": null
  }
}
```

Branch on `error.category` first. The "Tell the user" column is what to
surface to a human — translate, don't expose internals.

| `category` | Agent action | Tell the user |
|------------|--------------|---------------|
| `input_invalid` | Read `error.suggestion`, fix the named field, retry once. | Usually transparent. |
| `auth_failed` | Stop all API calls. | "ATA key is invalid, expired, or lacks permission. Refresh it in the dashboard." |
| `not_found` | Verify the id; don't retry with the same id. | If the id came from the user, ask them to recheck. |
| `retryable` | Sleep `retry-after`, retry once. | Usually transparent. |
| `quota_exceeded` | Stop the metered op until reset. | "ATA's daily quota is exhausted; resets soon. Proceeding without further lookups." |
| `service_degraded` | Proceed with whatever data came back; note the gap. | "ATA is partially degraded — limited cohort context." |
| `internal` | Wait ~60 s, retry once; then skip and continue. | "ATA hit a transient issue; continuing without that lookup." |

### Common error codes

| `error.code` | HTTP | Category | Action |
|--------------|------|----------|--------|
| `VALIDATION_ERROR` | 422 | input_invalid | Well-formed but invalid (bad enum / range / DAG rule). Read `suggestion`, fix, retry. |
| `INVALID_SYMBOL` | 400 | input_invalid | Use the canonical ticker (`NVDA`, `BTC-USDT`). |
| `INVALID_VENUE_QUOTE_PAIR` | 400 | input_invalid | `(venue, quote_currency)` isn't tradeable; fix the venue/quote pair. |
| `IDEMPOTENCY_KEY_CONFLICT` | 422 | input_invalid | Same `Idempotency-Key`, different body — reuse the original body or a fresh key. |
| `IDEMPOTENCY_KEY_IN_FLIGHT` | 409 | input_invalid | Earlier submit with this key still running — back off and retry. |
| `UNAUTHORIZED` | 401 | auth_failed | Missing/expired key. Stop; tell the operator. |
| `PERMISSION_DENIED` | 403 | auth_failed | Read-only key tried to submit. Operator can flip it to read_write. |
| `FORBIDDEN` | 403 | auth_failed | Insufficient permission. Stop; report. |
| `RECORD_NOT_FOUND` | 404 | not_found | Check the `dec_YYYYMMDD_8hex` id. |
| `RATE_LIMIT_EXCEEDED` | 429 | retryable | Sleep `retry-after`, retry once. |
| `DAILY_QUOTA_EXCEEDED` | 429 | quota_exceeded | Stop; reset is on `x-quota-reset`. |
| `SERVICE_UNAVAILABLE` | 503 | retryable | Retry in a moment. |
| `WISDOM_DATA_SPARSE` | 200 | service_degraded | Not fatal — fall back to your own analysis. |
| `PRICE_DATA_STALE` | 200 | service_degraded | Degraded — warn the user the live price may be stale. |

Note: `WISDOM_DATA_SPARSE` and `PRICE_DATA_STALE` arrive with **HTTP 200** —
they signal degradation, not failure.

---

## Probing safely

**Each submit is a real submission** — there is no dry-run. The moment all
validation rules pass, the record lands. Don't iterate against the live
endpoint with real intent.

- Build the full payload from [submit.md](submit.md), send once, accept the
  outcome. If it fails, read `error.suggestion`, fix the one field, resend
  — treating the resend as a real submission.
- For safe retries of the *same* intended submission, set an
  `Idempotency-Key` header (see [submit.md](submit.md)).
- If you genuinely need to probe the schema, use a guaranteed-reject
  payload: a non-existent `symbol` (e.g. `ZZZNONEXIST`), a far-future
  `data_cutoff`, and an off-market `reference_price` all at once — then
  replace all three when the shape is right.

If error codes or response shapes don't match this doc, suspect skill
drift: surface the raw error to the user and stop iterating, rather than
reverse-engineering the contract from error messages.

## See also

- [submit.md](submit.md) — submit validation errors, warnings, idempotency.
- [query.md](query.md) — quota cost of querying at scale.
- [outcome.md](outcome.md) — the read endpoints and access-control behavior.
