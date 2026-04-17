# GET /api/v1/experiences

Record-level search. Consumes 1 Query; `detail=full` adds N Read (N = records returned).

## Parameters

| Param | Required | Notes |
|-------|----------|-------|
| `symbol` **or** `sector` | exactly one | mutually exclusive |
| `direction`, `time_frame_type`, `perspective_type`, `method`, `signal_pattern` | no | carried from submit |
| `market_cap_tier`, `market_regime`, `market_conditions` | no | filters |
| `result_bucket`, `has_outcome` | no | |
| `date_from`, `date_to` | no | RFC 3339 |
| `time_axis` | default `analysis` | `analysis` (use `effective_decision_date`) or `submission` (use `created_at`) |
| `sort_by` | default per-axis | `effective_decision_date` / `created_at` |
| `limit` | no | 1-50 |
| `offset` | no | ≥ 0 |
| `detail` | default `summary` | `summary` or `full` |

## Response

`detail=summary`:

```json
{
  "total": 42,
  "experiences": [
    {
      "record_id": "dec_20260215_ab12cd34",
      "symbol": "NVDA", "direction": "bullish", "action": "buy",
      "time_frame": { "type": "swing", "horizon_days": 14 },
      "content_tags": ["analysis", "technical"],
      "perspective_type": "technical",
      "method_name": null, "signal_pattern": "pullback-continuation",
      "market_conditions": ["earnings_season"],
      "result_bucket": "strong_correct",
      "created_at": "2026-02-15T13:30:00Z"
    }
  ]
}
```

`detail=full` replaces each item with the full decision record (see [check-outcome.md](check-outcome.md) § Get Full Record).
