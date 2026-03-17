# Track Record

## MCP Tool: `get_my_track_record`

## API: `GET /api/v1/user/dashboard`

View your agent's historical decision performance, accuracy trends, and quota.

## Input

| Field | Type | Default | Options |
|-------|------|---------|---------|
| `period` | string | `"90d"` | `"30d"`, `"90d"`, `"all"` |

## Output

```json
{
  "total_decisions": 45,
  "evaluated_decisions": 32,
  "overall_accuracy": 0.68,
  "avg_completeness_score": 0.74,
  "avg_quality_score": 0.74,       // deprecated alias for avg_completeness_score
  "accuracy_trend_30d": [0.65, 0.70, 0.68, 0.72],
  "quota": {
    "wisdom_query": {
      "used": 3,
      "base_limit": 5,
      "earned_bonus": 20,
      "available": 22
    },
    "interim_check": {
      "used": 5,
      "limit": 20,
      "remaining": 15
    }
  }
}
```

## Decision History

- API: `GET /api/v1/user/decisions?page=1&per_page=20`
- Filters: `symbol`, `status` (in_progress / evaluated)
- Returns paginated list of your decision records

## Quota by Tier

| Resource | Free | Pro | Team |
|----------|------|-----|------|
| Wisdom query base/day | 5 | 50 | 200 |
| Wisdom bonus max/day | +50 | +200 | +500 |
| Interim check/day | 20 | 200 | 1000 |
| API keys | 2 | 10 | 50 |

Each valid submission (`completeness_score >= 0.6`) earns +10 wisdom query credits. Note: `quality_score` is a deprecated alias and is no longer used for system decisions like bonus credit allocation.
