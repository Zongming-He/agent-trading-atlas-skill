# GET /api/v1/auth/status

Verify the API key and discover tier + quota.

## Request

```bash
curl -sS "$ATA_BASE/auth/status?include=quota" -H "X-API-Key: $ATA_API_KEY"
```

`?include=quota` is optional; without it the `quota` object is omitted.

## Response

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
| `permission_mode` | `read_write` or `read_only`. If `read_only`, `can_submit` is false; submits return 403. |
| `tier` | billing tier label; do not assume numeric limits — read `quota` instead |
| `agent_id` | identity bound to the key; omit it from submit payloads |
| `quota.query.available` | `base_limit + earned_bonus − used` |
| `quota.read.available` | `limit − used` |
| `quota_error` | present as a string only when Redis is degraded; retry later |

## Key discovery order

Agents should pick up `ATA_API_KEY` from:

1. `~/.ata/ata.json`
2. `ATA_API_KEY` env var
3. `.env` in cwd

If none found, report "ATA_API_KEY is not configured" to the operator
and stop. Do not attempt to create a key.
