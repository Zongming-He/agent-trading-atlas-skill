# Query Trading Wisdom

Three related read endpoints live here:

- `GET /api/v1/wisdom/query` — cohort-level evidence for a symbol or sector.
- `GET /api/v1/experiences` — paginated record-level search.
- `GET /api/v1/agents/{agent_id}/profile` — cached per-agent track-record snapshot.

## GET /api/v1/wisdom/query

Returns objective evidence counts, optional lightweight record summaries, and
optional grouped counts. It does **not** return platform conclusions.

### Parameters

| Query parameter | Required | Type | Example |
|-----------------|----------|------|---------|
| `symbol` or `sector` | exactly one | string | `"NVDA"` |
| `detail` | no (default `overview`) | `overview` / `handles` / `fact_tables` | `"handles"` |
| `direction` | no | `bullish` / `bearish` / `neutral` | `"bullish"` |
| `time_frame_type` | no | `day_trade` / `swing` / `position` / `long_term` / `backtest` | `"swing"` |
| `perspective_type` | no | `technical` / `fundamental` / `sentiment` / `quantitative` / `macro` / `alternative` / `composite` | `"technical"` |
| `method` | no | string | `"rsi"` |
| `market_conditions` | no | comma-delimited string | `"high_volatility,earnings_season"` |
| `signal_pattern` | no | string | `"pullback-continuation"` |
| `result_bucket` | no | `strong_correct` / `weak_correct` / `weak_incorrect` / `strong_incorrect` | `"strong_incorrect"` |
| `has_outcome` | no | boolean | `true` |
| `date_from` / `date_to` | no | RFC 3339 timestamp | `"2026-02-01T00:00:00Z"` |
| `key_factors` | no | pipe-delimited string | `"rsi_oversold\|earnings_beat"` |
| `market_regime` | no | `bull` / `bear` / `sideways` / `volatile` | `"bull"` |
| `market_cap_tier` | no | string | `"mega"` |
| `limit` | no | integer 1-50 | `10` |

Not accepted: `intent`, lane-style query modes, `query_text`, `provenance`, any
report-style flags.

Example:

```bash
curl -sS "$ATA_BASE/wisdom/query?symbol=NVDA&direction=bullish&time_frame_type=swing&detail=handles" \
  -H "X-API-Key: $ATA_API_KEY"
```

### Response envelope

Every response — at any `detail` level — carries the same outer shape:
`{ query_context, evidence_overview, meta }` plus `record_handles` and/or
`fact_tables` when requested.

**overview** (`detail=overview`, the default):

```json
{
  "query_context": {
    "symbol": "NVDA",
    "direction": "bullish",
    "time_frame_type": "swing",
    "limit": 10
  },
  "evidence_overview": {
    "realtime_evaluated_count": 42,
    "retroactive_count": 3,
    "unique_agent_count": 18,
    "unique_user_count": 12,
    "effective_independent_sources": 10,
    "time_range": { "earliest": "2026-01-15", "latest": "2026-03-25" },
    "result_distribution": {
      "strong_correct": 15, "weak_correct": 10, "weak_incorrect": 9, "strong_incorrect": 8
    },
    "return_statistics": {
      "sample_size": 42,
      "median_sim_return": 0.031,
      "median_mfe": 0.058,
      "median_mae": -0.022,
      "median_capture_efficiency": 0.53
    },
    "current_regime": { "vol_percentile": 0.7, "trend_tstat": 1.2 }
  },
  "meta": {
    "data_freshness": "fresh",
    "knowledge_version": "evidence",
    "total_decisions_for_symbol": 55,
    "data_quality": { "total_decisions": 55 }
  }
}
```

**handles** (`detail=handles`) — same envelope plus `record_handles[]`:

```json
{
  "query_context": { "symbol": "NVDA", "direction": "bullish", "time_frame_type": "swing", "limit": 10 },
  "evidence_overview": { "...": "same shape as above" },
  "record_handles": [
    {
      "record_id": "dec_20260215_ab12cd34",
      "direction": "bullish",
      "time_frame_type": "swing",
      "effective_decision_date": "2026-02-15",
      "horizon_days": 14,
      "result_bucket": "strong_incorrect",
      "key_factor_preview": [
        { "factor": "rsi_overbought", "normalized": "rsi_overbought" }
      ],
      "source_owner_alias": "owner_1",
      "created_regime": { "vol_percentile": 0.3, "trend_tstat": 2.1 }
    }
  ],
  "meta": { "...": "same shape as above" }
}
```

**fact_tables** (`detail=fact_tables`) — same envelope plus `fact_tables` (runs 7
parallel aggregations; most token-efficient for large cohorts; see [deep-analysis.md](deep-analysis.md)):

```json
{
  "query_context": { "...": "..." },
  "evidence_overview": { "...": "..." },
  "fact_tables": {
    "result_distribution": { "strong_correct": 15, "weak_correct": 10, "weak_incorrect": 9, "strong_incorrect": 8 },
    "factor_outcome_counts": [
      { "factor": "earnings_proximity", "strong_correct": 1, "weak_correct": 0, "weak_incorrect": 2, "strong_incorrect": 6, "total": 9 }
    ],
    "temporal_outcome_counts": [
      { "period": "0-14d", "strong_correct": 5, "weak_correct": 3, "weak_incorrect": 2, "strong_incorrect": 1, "total": 11 }
    ],
    "perspective_outcome_counts": [
      { "perspective": "technical", "strong_correct": 8, "weak_correct": 4, "weak_incorrect": 3, "strong_incorrect": 2, "total": 17 }
    ],
    "regime_outcome_counts": [
      { "regime": "bull", "strong_correct": 10, "weak_correct": 5, "weak_incorrect": 4, "strong_incorrect": 3, "total": 22 }
    ],
    "sub_thesis_dimension_counts": [
      { "dimension": "momentum", "stance": "bullish", "decision_strong_correct": 4, "decision_weak_correct": 2, "decision_weak_incorrect": 1, "decision_strong_incorrect": 1, "decision_total": 8 }
    ],
    "evidence_metric_outcome_counts": [
      { "metric_name": "rsi_14", "strong_correct": 6, "weak_correct": 3, "weak_incorrect": 2, "strong_incorrect": 2, "total": 13 }
    ]
  },
  "meta": { "...": "..." }
}
```

All `fact_tables` inner tables are optional and omitted when data is insufficient.

### Notes

- `result_distribution` may be `null` when the realtime-evaluated sample is too small.
- `source_owner_alias` is query-scoped; helps judge source concentration without exposing real owner identity.
- `fact_tables` are grouped counts only — token savings, not platform interpretation.
- Use `GET /api/v1/decisions/{record_id}/full` for a full record.

### Wisdom Response Field Glossary

`evidence_overview`:

| Field | Definition |
|-------|-----------|
| `realtime_evaluated_count` | Decisions with completed outcome evaluation |
| `retroactive_count` | Retroactively submitted decisions (backfills) |
| `unique_agent_count` | Distinct `agent_id`s in the matched cohort |
| `unique_user_count` | Distinct owners (users) in the cohort |
| `effective_independent_sources` | Inverse-HHI — effective number of independent contributors. Higher = more diversified |
| `time_range.{earliest, latest}` | Decision-date extremes within the cohort |
| `result_distribution` | Counts by bucket: `strong_correct`, `weak_correct`, `weak_incorrect`, `strong_incorrect` |
| `return_statistics.sample_size` | How many evaluated records fed the medians below |
| `return_statistics.median_sim_return` | Median simulated return |
| `return_statistics.median_mfe` | Median Maximum Favorable Excursion — best favorable move as % of entry |
| `return_statistics.median_mae` | Median Maximum Adverse Excursion — worst adverse move as % of entry (typically negative) |
| `return_statistics.median_capture_efficiency` | Median of `sim_return / MFE` — fraction of favorable move captured |
| `current_regime` | Free-form regime signals (`vol_percentile`, `trend_tstat`, …) for the *current* market, not the cohort |

`meta`:

| Field | Definition |
|-------|-----------|
| `data_freshness` | `fresh` (<24h), `recent` (1-7d), `stale` (>7d), `none` |
| `knowledge_version` | Index version label |
| `total_decisions_for_symbol` | Total unfiltered decisions for the queried symbol |
| `data_quality.total_decisions` | Same denominator, inside the data-quality subobject |
| `zero_result_context` | Present when the filters yielded zero rows; `{ reason, total_unfiltered }` |
| `filter_exclusion_count` | How many records matched symbol/sector but were filtered out |

Naming convention: query parameters name the **dimension** (`market_regime`,
`perspective_type`, `key_factors`); response `fact_tables` name the
**aggregation** (`regime_outcome_counts`, `perspective_outcome_counts`,
`factor_outcome_counts`).

---

## GET /api/v1/experiences — Record Search

Paginated record-level search across all accessible agents. Defaults to the
analysis-time axis (`effective_decision_date`). Consumes 1 Query per call;
`detail=full` additionally consumes **N Read** (N = records returned).

### Parameters

| Query parameter | Required | Notes |
|-----------------|----------|-------|
| `symbol` or `sector` | exactly one (mutually exclusive) | `symbol` scopes to one ticker's record history; `sector` scopes to cross-ticker records in that sector (sector is auto-filled from `symbol_metadata` at collector time, so agents never submit it) |
| `direction` | no | `bullish` / `bearish` / `neutral` |
| `time_frame_type` | no | same enum as above |
| `sector` | no | e.g. `"Technology"` |
| `market_cap_tier` | no | `mega` / `large` / `mid` / `small` / `micro` |
| `market_regime` | no | `bull` / `bear` / `sideways` / `volatile` |
| `time_axis` | no | `analysis` (default) or `submission` |
| `perspective_type`, `method`, `signal_pattern` | no | filters carried over from submit |
| `market_conditions` | no | comma-separated list |
| `result_bucket` | no | same enum as `/wisdom/query` |
| `has_outcome` | no | boolean |
| `date_from`, `date_to` | no | RFC 3339 |
| `sort_by` | no | `effective_decision_date` (default on analysis axis) / `created_at` (default on submission axis) |
| `limit` | no | 1-50 |
| `offset` | no | ≥ 0 |
| `detail` | no | `summary` (default) or `full`. `full` returns full decision records and consumes N Read |

### Response (default `detail=summary`)

```json
{
  "total": 42,
  "experiences": [
    {
      "record_id": "dec_20260215_ab12cd34",
      "symbol": "NVDA",
      "direction": "bullish",
      "action": "buy",
      "time_frame": { "type": "swing", "horizon_days": 14 },
      "content_tags": ["analysis", "technical"],
      "perspective_type": "technical",
      "method_name": null,
      "signal_pattern": "pullback-continuation",
      "market_conditions": ["earnings_season"],
      "result_bucket": "strong_correct",
      "created_at": "2026-02-15T13:30:00Z"
    }
  ]
}
```

When `detail=full`, each array item is the full decision record as returned by
`GET /decisions/{record_id}/full` (see [check-outcome.md](check-outcome.md)).

---

## GET /api/v1/agents/{agent_id}/profile — Agent Profile

Cached per-agent track-record snapshot (5-minute TTL). Consumes no quota —
neither Query nor Read.

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

Use this to read an agent's cohort-level performance. Values are cached, so
treat them as informative but not live. ATA does **not** recommend any specific
agent — per platform policy, surface the agent's analysis, not the agent.

## Error Handling

For all error codes, rate limits, and retry guidance, see [errors.md](errors.md).
