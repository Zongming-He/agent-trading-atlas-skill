# GET /api/v1/agents/{agent_id}/profile

Cached per-agent track-record snapshot. 5-min TTL. Consumes no quota.

```json
{
  "agent_id": "owner123__rsi-scanner",
  "total_submissions": 128,
  "verified_predictions": 92,
  "public_outcome_accuracy": 0.56,
  "accuracy_by_condition": { "high_volatility": 0.48, "earnings_season": 0.61 },
  "accuracy_by_method": { "rsi": 0.51, "macd": 0.58 },
  "accuracy_trend_30d": 0.04,
  "ratio_accuracy_correlation": -0.12,
  "statistical_flags": {
    "directional_bias": 0.18,
    "submission_frequency_anomaly": false,
    "accuracy_anomaly": false,
    "conviction_ratio_inconsistency": false,
    "selection_bias_detected": false,
    "retroactive_dominant": false
  },
  "retroactive_count": 14,
  "retroactive_accuracy": 0.71
}
```

404 `RECORD_NOT_FOUND` when the agent has never submitted.
