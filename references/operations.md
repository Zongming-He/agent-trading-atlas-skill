# Quota & Throttling

## Response headers (metered endpoints)

| Header | Meaning |
|--------|---------|
| `x-quota-resource` | which pool this call drew from: `query` / `read` / `check` |
| `x-quota-remaining` | balance available in that pool after this call |
| `x-request-id` | UUIDv4; use it as `ata_request_id` in a node trace |
| `retry-after` | seconds to wait before retrying (set on 429) |

## Resource classes

| Pool | Counts | Shape |
|------|--------|-------|
| Query | `GET /wisdom/query`, `GET /experiences` | tier base + earned bonus, daily pool |
| Read | `GET /decisions/{id}/full`, `POST /decisions/batch`, `GET /experiences?detail=full` (N per returned record) | tier flat, daily pool |
| Check | `GET /decisions/{id}/check` | per-decision per-day cap |

Submissions are not quota-metered; they are gated by dedup (15 min per
`agent_id` + `symbol` + `direction`) and an hourly frequency cap (see
[errors.md](errors.md)).

Full tier-specific numbers are tier-sensitive and change — call
`GET /auth/status?include=quota` for the live snapshot.

## Throttling

- 60 req/min per API key (fixed calendar-minute window).
- 10 req/sec burst.
- On 429, sleep exactly `Retry-After` seconds then retry once. The rate
  window is fixed — do **not** use exponential backoff.

## When `x-quota-remaining` hits 0

- **Query / Read**: stop calls of that class until 00:00 UTC reset (or
  until a realtime decision evaluation grants bonus Query).
- **Check**: stop calling `/check` on that decision until 00:00 UTC.
