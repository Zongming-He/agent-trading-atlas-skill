# Aggregation Layer Reference

## Purpose

When a wisdom cohort contains hundreds or thousands of records, fetching them individually wastes tokens and quota. The `detail=fact_tables` parameter runs 7 parallel aggregation queries server-side and returns grouped outcome counts — giving you multi-dimensional statistical breakdowns in a single API call.

For query parameters and response format basics, see [query-wisdom.md](query-wisdom.md).

## When to Use

- `detail=overview` gives cohort-level counts (cheapest call, good for checking if evidence exists)
- `detail=handles` gives per-record previews (useful when the cohort is small enough to scan)
- **`detail=fact_tables`** gives grouped aggregations (most token-efficient for large cohorts)
- `GET /api/v1/decisions/{record_id}/full` gives raw records (when you need original reasoning)

Use `fact_tables` when you want to analyze patterns across a large cohort without fetching individual records.

## Fact Table Specifications

All tables only include realtime submissions with evaluated outcomes. Each row contains outcome bucket counts: `strong_correct`, `weak_correct`, `weak_incorrect`, `strong_incorrect`, `total`.

### 1. `factor_outcome_counts`

Groups by **normalized key-factor name** extracted from each record.

- Minimum 3 occurrences per factor
- Top 20 by total count descending
- Use to identify which factors historically correlated with correct or incorrect outcomes

### 2. `temporal_outcome_counts`

Groups by **decision age** relative to today, in 4 fixed buckets:

| Period | Range |
|--------|-------|
| `0-14d` | Last 2 weeks |
| `15-60d` | 2 weeks to 2 months |
| `61-180d` | 2 to 6 months |
| `180d+` | Older than 6 months |

Use to check whether recent evidence diverges from historical patterns.

### 3. `perspective_outcome_counts`

Groups by **perspective_type** (`technical`, `fundamental`, `sentiment`, `quantitative`, `macro`, `alternative`, `composite`).

- Ordered by total count descending
- Use to see which analytical approaches produced better outcomes for this cohort

### 4. `regime_outcome_counts`

Groups by **market_regime** (`bull`, `bear`, `sideways`, `volatile`).

- Ordered by total count descending
- Gracefully omitted if the query fails (not all records have regime data)
- Use to check whether outcomes vary by market regime

### 5. `sub_thesis_dimension_counts`

Groups by **reasoning dimension x stance** from structured `reasoning_dag` sub-theses.

- Joins `decision_sub_theses` table (relational, not JSONB expansion)
- Groups by `normalized_dimension` + `stance`
- Minimum 3 occurrences, top 30 by count descending
- Use to analyze which reasoning dimensions (e.g. `momentum × bullish`) led to correct outcomes

### 6. `evidence_metric_outcome_counts`

Groups by **evidence metric name** from structured `reasoning_dag` evidence.

- Joins `decision_evidence` table (relational, not JSONB expansion)
- Groups by `metric_name` (e.g. `rsi_14`, `pe_ratio`, `macd_signal`)
- Minimum 3 occurrences, top 50 by count descending
- Use to identify which quantitative metrics were associated with correct or incorrect calls

### 7. `result_distribution`

The overall outcome distribution for the entire matched cohort. Same data as in `detail=overview`, included here for convenience so you do not need a separate call.

## Progressive Refinement Strategy

1. Start with `detail=overview` to check if the cohort has meaningful data.
2. If `realtime_evaluated_count` is large, use `detail=fact_tables` to get aggregated breakdowns.
3. If a specific table row is interesting (e.g. a factor with high `strong_incorrect`), use `detail=handles` or fetch raw records to inspect individual cases.
4. If `detail=fact_tables` is too coarse for your question, fetch raw records and compute your own grouping.

## Fallback Rules

- If `detail=overview` shows no useful evidence, stop and rely on your own analysis.
- If `result_distribution` is `null`, the evaluated sample is too small for bucket summaries.
- Do not infer a base rate from a tiny sample.
- Tables 5-6 (`sub_thesis_dimension_counts`, `evidence_metric_outcome_counts`) require submissions with structured `reasoning_dag` data and may be empty for older cohorts.
