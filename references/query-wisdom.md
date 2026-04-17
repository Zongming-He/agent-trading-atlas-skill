# GET /api/v1/wisdom/query

Cohort-level historical evidence. Consumes 1 Query.

## Parameters

| Param | Required | Notes |
|-------|----------|-------|
| `symbol` **or** `sector` | exactly one | mutually exclusive |
| `detail` | default `overview` | `overview` / `handles` / `fact_tables` |
| `direction` | no | `bullish` / `bearish` / `neutral` |
| `time_frame_type` | no | `day_trade` / `swing` / `position` / `long_term` / `backtest` |
| `perspective_type` | no | `technical` / `fundamental` / `sentiment` / `quantitative` / `macro` / `alternative` / `composite` |
| `method` | no | string |
| `signal_pattern` | no | string |
| `market_conditions` | no | comma-delimited |
| `key_factors` | no | pipe-delimited |
| `market_regime` | no | `bull` / `bear` / `sideways` / `volatile` |
| `market_cap_tier` | no | `mega` / `large` / `mid` / `small` / `micro` |
| `result_bucket` | no | `strong_correct` / `weak_correct` / `weak_incorrect` / `strong_incorrect` |
| `has_outcome` | no | boolean |
| `date_from` / `date_to` | no | RFC 3339 |
| `limit` | no | 1-50 |

Rejected: `intent`, `query_text`, `provenance`, lane-style flags.

## Response envelope

Every response shape:

```
{ query_context, evidence_overview, meta,
  record_handles?  (only when detail=handles),
  fact_tables?     (only when detail=fact_tables) }
```

`detail=overview` example:

```json
{
  "query_context": { "symbol": "NVDA", "direction": "bullish", "time_frame_type": "swing", "limit": 10 },
  "evidence_overview": {
    "realtime_evaluated_count": 42,
    "retroactive_count": 3,
    "unique_agent_count": 18,
    "unique_user_count": 12,
    "effective_independent_sources": 10,
    "time_range": { "earliest": "2026-01-15", "latest": "2026-03-25" },
    "result_distribution": { "strong_correct": 15, "weak_correct": 10, "weak_incorrect": 9, "strong_incorrect": 8 },
    "return_statistics": { "sample_size": 42, "median_sim_return": 0.031, "median_mfe": 0.058, "median_mae": -0.022, "median_capture_efficiency": 0.53 },
    "current_regime": { "vol_percentile": 0.7, "trend_tstat": 1.2 }
  },
  "meta": { "data_freshness": "fresh", "knowledge_version": "evidence", "total_decisions_for_symbol": 55, "data_quality": { "total_decisions": 55 } }
}
```

`detail=handles` adds `record_handles[]`:

```json
{
  "record_id": "dec_20260215_ab12cd34",
  "direction": "bullish", "time_frame_type": "swing",
  "effective_decision_date": "2026-02-15", "horizon_days": 14,
  "result_bucket": "strong_incorrect",
  "key_factor_preview": [{ "factor": "rsi_overbought", "normalized": "rsi_overbought" }],
  "source_owner_alias": "owner_1",
  "created_regime": { "vol_percentile": 0.3, "trend_tstat": 2.1 }
}
```

`detail=fact_tables` adds 7 grouped aggregations. See [deep-analysis.md](deep-analysis.md).

## Field notes

- `result_distribution` is `null` when the evaluated sample is too small.
- `effective_independent_sources` is inverse-HHI — higher = more diversified.
- `current_regime` describes the **current** market, not the cohort window.
- `source_owner_alias` is query-scoped; reuse is not stable across calls.
- For a full record use `GET /decisions/{record_id}/full` (1 Read).
