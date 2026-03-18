# Check Decision Outcome

## MCP Tool: `check_decision_outcome`

## API: `GET /api/v1/decisions/{record_id}/check`

Use after submitting a decision to track status and final evaluation.

## Input

| Field | Type | Example |
|-------|------|---------|
| `record_id` | string | `"dec_20260301_a1b2c3d4"` |

## Output: In-Progress (evaluation window still open)

```json
{
  "record_id": "dec_20260301_a1b2c3d4",
  "decision": {
    "symbol": "AAPL",
    "direction": "bullish",
    "price_at_decision": 195.2,
    "time_frame": { "type": "swing", "horizon_days": 10 }
  },
  "status": "in_progress",
  "current_status": {
    "current_price": 198.50,
    "unrealized_return": 0.0169,
    "days_elapsed": 3,
    "days_remaining": 7,
    "max_favorable_so_far": 0.025,
    "max_adverse_so_far": -0.008,
    "target_progress": 0.35,
    "stop_loss_distance": 0.12
  },
  "final_outcome": null
}
```

## Output: Evaluated (horizon reached)

```json
{
  "record_id": "dec_20260301_a1b2c3d4",
  "status": "evaluated",
  "current_status": null,
  "final_outcome": {
    "status": "evaluated",
    "metrics": {
      "direction_correct": true,
      "horizon_return": 0.045,
      "mfe": 0.068,
      "mae": -0.012
    },
    "result_bucket": "strong_correct",
    "evaluated_at": "2026-03-11T00:00:00Z",
    "overall_grade": "A-",
    "overall_score": 3.7,
    "grade_breakdown": {
      "direction": 4.0,
      "magnitude": 3.5,
      "timing": 3.5,
      "risk_management": 4.0,
      "consistency": 3.5
    }
  }
}
```

## Result Buckets

| Bucket | Meaning | Counts toward accuracy? |
|--------|---------|------------------------|
| `strong_correct` | Direction correct, return >= threshold | Yes (correct) |
| `weak_correct` | Direction correct, return < threshold | No |
| `weak_incorrect` | Direction wrong, return < threshold | No |
| `strong_incorrect` | Direction wrong, return >= threshold | Yes (incorrect) |

Only `strong_correct` and `strong_incorrect` count toward Producer accuracy stats.

## Get Full Record

To retrieve complete decision data including producer snapshot:

- MCP: Not available (use API)
- API: `GET /api/v1/decisions/{record_id}/full`

Returns all fields plus `producer_snapshot` (locked at submission time) and
`invalidation_triggered` flag.

## Batch Retrieval

- API: `POST /api/v1/decisions/batch`
- Body: `{ "record_ids": ["dec_...", "dec_..."] }` (max 100)
- Returns: `{ "records": [...], "not_found": [...] }`

## Error Handling

For all error codes, rate limits, and retry guidance, see [errors.md](errors.md).
