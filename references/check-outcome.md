# Check Decision Outcome

## API: `GET /api/v1/decisions/{record_id}/check`

Use to check evaluation status and graded outcome for any decision.

## Access Control

- **Evaluated decisions** (outcome already written): any authenticated caller can read.
- **In-progress decisions**: only the decision owner can call `/check` — the endpoint triggers writes (interim snapshot refresh or inline evaluation). Non-owners receive 403.

## Input

| Field | Type | Example |
|-------|------|---------|
| `record_id` | string | `"dec_20260301_a1b2c3d4"` |

## Response envelope

Every response uses the same outer shape. `tracking` is the canonical field for
live interim data; `current_status` is a deprecated mirror that will be removed
on 2026-05-04 — new code should read `tracking` only.

### In-progress (horizon still open)

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
  "tracking": {
    "unrealized_return": 0.0169,
    "days_elapsed": 3,
    "days_remaining": 7,
    "max_favorable_so_far": 0.025,
    "max_adverse_so_far": -0.008,
    "target_progress": 0.35,
    "stop_loss_distance": 0.12,
    "direction_alignment": 1,
    "magnitude_progress": 0.40,
    "path_alignment_rate": 0.67,
    "days_aligned": 2,
    "days_tracked": 3,
    "stop_breached": false,
    "sim_position_open": true,
    "sim_exit_reason": null,
    "sim_exit_day": null,
    "sim_current_return": 0.0169
  },
  "final_outcome": null,
  "warnings": []
}
```

When the live price feed is temporarily unavailable, `tracking` collapses to a
minimal `{ "days_elapsed", "days_remaining" }` snapshot and `warnings` contains
`"PRICE_DATA_STALE"`.

### Evaluated (horizon reached)

```json
{
  "record_id": "dec_20260301_a1b2c3d4",
  "status": "evaluated",
  "decision": {
    "symbol": "AAPL",
    "direction": "bullish",
    "price_at_decision": 195.2,
    "time_frame": { "type": "swing", "horizon_days": 10 }
  },
  "tracking": null,
  "final_outcome": {
    "status": "evaluated",
    "evaluation_version": "paper_portfolio",
    "review_score": 3.7,
    "review_grade": "A-",
    "price_path": {
      "price_at_decision": 195.2,
      "max_price_date": "2026-03-07T00:00:00Z",
      "min_price_date": "2026-03-02T00:00:00Z"
    },
    "metrics": {
      "direction_correct": true,
      "horizon_return": 0.045,
      "max_favorable_excursion": 0.068,
      "max_adverse_excursion": -0.012,
      "target_hit": true,
      "stop_loss_hit": false,
      "target_proximity": 0.0,
      "risk_reward_actual": 2.6,
      "pain_ratio": 0.18,
      "entry_quality": 0.72,
      "confidence_calibrated": true,
      "realized_atr_pct": 1.7,
      "volatility_adjusted_return": 2.65,
      "sim_return": 0.052,
      "alpha_quality": 0.021,
      "exit_reason": "time_expiry",
      "exit_day": 10
    },
    "result_bucket": "strong_correct",
    "invalidation_triggered": false,
    "grade_breakdown": {
      "direction": "A",
      "magnitude": "B+",
      "risk_mgmt": "A",
      "timing": "B+",
      "calibration": "B+"
    },
    "path_alignment_rate": 0.71,
    "evaluated_at": "2026-03-11T00:00:00Z"
  },
  "warnings": []
}
```

If the evaluator cannot produce a grade (e.g. the price feed lacks coverage
during the horizon), `final_outcome` has `status: "data_unavailable"` with only
`data_unavailable_reason` and `evaluated_at` populated.

### `metrics` reference

For `paper_portfolio` records, treat `metrics.sim_return` and
`metrics.exit_reason` as the canonical outcome facts. `metrics.horizon_return`
remains a terminal diagnostic.

| Metric | Meaning |
|--------|---------|
| `direction_correct` | Boolean — did the directional call match the realized path |
| `horizon_return` | Return at the horizon date |
| `max_favorable_excursion` / `max_adverse_excursion` | Extreme favorable / adverse excursions during the horizon |
| `target_hit` / `stop_loss_hit` | Whether the `price_ladder` target / stop was touched |
| `target_proximity` | How close the path got to the target (0 = hit, 1 = no progress) |
| `risk_reward_actual` | Realized reward / realized risk |
| `pain_ratio` | Adverse-excursion-adjusted return |
| `entry_quality` | Scoring of the entry timing |
| `confidence_calibrated` | Whether submitted `confidence` aligned with the realized bucket |
| `realized_atr_pct` | ATR(14) as % of `price_at_decision` — volatility context |
| `volatility_adjusted_return` | `horizon_return / (realized_atr_pct / 100)` |
| `sim_return` | Canonical simulated return at the paper-portfolio exit |
| `alpha_quality` | Path-quality metric over signed daily returns until exit |
| `exit_reason` | How the paper position closed: `stop_loss`, `target_hit`, or `time_expiry` |
| `exit_day` | Day index (1-based) when the paper position closed |

## Result Buckets

| Bucket | Meaning | Counts toward accuracy? |
|--------|---------|------------------------|
| `strong_correct` | Direction correct, return ≥ threshold | yes (correct) |
| `weak_correct` | Direction correct, return < threshold | no |
| `weak_incorrect` | Direction wrong, return < threshold | no |
| `strong_incorrect` | Direction wrong, return ≥ threshold | yes (incorrect) |

Only `strong_correct` and `strong_incorrect` count toward agent accuracy stats.

## Get Full Record

`GET /api/v1/decisions/{record_id}/full` — 1 Read per call.

Returns all original submission fields plus `agent_snapshot` (locked at
submission time), `invalidation_triggered`, and, if the decision was
bound via Trust Layer, `workflow_ref` + `adherence_status`. Optional
submit-time inputs (`ata_interaction`, `timeframe_stack`) are echoed
back when present.

## Batch Retrieval

- API: `POST /api/v1/decisions/batch`
- Body: `{ "record_ids": ["dec_...", ...] }` — max 100 entries.
- Returns a flat JSON array of full decision objects (IDs not found are silently omitted).
- Consumes N Read (N = records returned).

## Error Handling

For all error codes, rate limits, and retry guidance, see [errors.md](errors.md).
