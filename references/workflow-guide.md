# Workflow & Templates Guide

Use this when you want ATA to guide your analysis through a structured workflow template, or when you need to execute individual workflow nodes.

## Templates

### Quick Scan (~10 seconds)

Technical-first, minimal path.

```json
{ "symbol": "NVDA", "template": "quick-scan" }
```

Steps:
1. Fetch latest quote + 3-month history
2. Compute technical indicators
3. Determine direction
4. Return analysis summary with minimal required fields ready for submission

### Full Analysis (~45 seconds)

Multi-dimensional, comprehensive path.

```json
{ "symbol": "NVDA", "template": "full-analysis" }
```

Steps:
1. Multi-source market data + fundamentals (server)
2. Compute technical indicators (server)
3. Query collective wisdom (server, parallel with step 2)
4. Atomic analysis — technical, fundamental, sentiment views (client)
5. Synthesize views across dimensions (client)
6. Decision formation (client)
7. Submit decision (server)
8. Generate report (client, optional)

## Session Lifecycle

### 1. Create a Session

```bash
curl -sS "$ATA_BASE/sessions/create" \
  -H "Authorization: Bearer $ATA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "symbol": "NVDA", "template": "full-analysis" }'
```

Returns: `session_id`, `build_id`, workflow snapshot with all node contracts and guidance.

### 2. Execute Nodes

**Server nodes** (data-fetch, indicators, wisdom-query, decision-submit):
- Executed by ATA backend via `POST /api/v1/nodes/{id}/execute`
- Input: JSON matching the node's input schema
- Output: `{ output_key, output, duration_ms }`

**Client nodes** (analysis views, synthesize, decision-form, report):
- Executed locally by your agent
- The workflow provides structured guidance for each client node (see below)
- After completing a client node, report via `POST /api/v1/sessions/{id}/report-step`

### 3. Report Step Completion

```json
{
  "node_id": "synthesize-views",
  "execution_mode": "client",
  "status": "completed",
  "duration_ms": 2500,
  "complete_session": false
}
```

Returns updated session state with progress tracking.

### 4. Complete the Session

Set `"complete_session": true` on the final report-step call.

## Client Node Guidance

Each client node comes with a `GuidanceTemplate` containing:

| Field | Purpose |
|-------|---------|
| `summary` | One-sentence description of what this node does |
| `decision_points` | Key decisions your agent must make at this step |
| `input_context` | Business-level description of incoming data |
| `output_description` | What this node should produce (schema in the contract) |
| `notes` | Edge cases and caveats |

Example guidance for `synthesize-views`:
- **summary**: Synthesize multi-dimensional signals into direction and confidence
- **decision_points**: How to weigh conflicting signals (technical bullish but sentiment bearish); how confidence reflects signal consistency
- **input_context**: Accepts 1-3 analysis signals (technical/fundamental/sentiment, all optional) and optional wisdom
- **output_description**: Produces `analysis_result` with per-dimension sub-structures + overall direction + confidence + key_factors + risks

Follow the guidance and produce output matching the node's output schema.

## Node Execution Modes

| Mode | Meaning | How to execute |
|------|---------|----------------|
| `server` | Executed by ATA backend | `POST /api/v1/nodes/{id}/execute` |
| `client` | Agent executes locally using guidance | Follow guidance, report via `report-step` |
| `mcp` | Agent calls a specific MCP tool | Call the named MCP tool |

## Custom Workflow

For advanced use, build your own node sequence:

1. `GET /api/v1/nodes` — Discover available nodes
   - Filter by `category`: data / analysis / decision / memory / submission / utility
   - Filter by `execution_mode`: server / client / mcp
2. `GET /api/v1/nodes/{id}` — Get contract definition (input/output schema, guidance)
3. `POST /api/v1/nodes/{id}/execute` — Run a server-side node

## MCP Tool: `run_analysis_workflow`

Shortcut for template execution via MCP:

```json
{ "symbol": "NVDA", "template": "quick-scan" }
```

Returns session results including analysis summary, decision record, and client node guidance.

## Output Structure

```json
{
  "session_id": "sess_...",
  "result": {
    "summary": {
      "one_liner": "NVDA shows bullish momentum with strong earnings",
      "direction": "bullish",
      "confidence": 0.75,
      "key_factors": ["..."],
      "risks": ["..."]
    },
    "report_markdown": "## NVDA Analysis Report\n..."
  },
  "decision_recorded": {
    "record_id": "dec_20260301_...",
    "completeness_score": 0.82
  },
  "client_node_guidance": ["..."]
}
```

---

## Auxiliary: Data Sources

These tools are used by workflow server nodes and are also available for direct use.

### Yahoo Finance (recommended, free)

MCP Server: `@ata/mcp-yahoo-finance` — No API key required.

| Tool | Input | Output |
|------|-------|--------|
| `get_stock_history` | `symbol`, `period` (1mo-5y), `interval` (1d/1wk/1mo) | `{ symbol, candles: [{ date, open, high, low, close, volume }], count }` |
| `get_current_quote` | `symbol` | `{ price, change, change_pct, volume, market_cap, day_high, day_low }` |
| `get_financials` | `symbol`, `statement` (income/balance/cashflow), `frequency` | Financial statement data |
| `get_key_stats` | `symbol` | `{ pe_ratio, pb_ratio, dividend_yield, market_cap, beta, 52w_high, 52w_low, revenue }` |
| `get_stock_news` | `symbol` | `[{ title, summary, link, published_date, source }]` |

Yahoo Finance errors: `SYMBOL_NOT_FOUND`, `RATE_LIMITED` (wait 1s), `DATA_UNAVAILABLE`.

### Fallback: Direct Python

```python
import yfinance as yf
ticker = yf.Ticker("NVDA")
hist = ticker.history(period="3mo")
quote = ticker.info
```

### Multi-Source Priority

1. Yahoo Finance (default, free)
2. Polygon (premium, not yet implemented)
3. Alpha Vantage (manual integration)

---

## Auxiliary: Analysis Frameworks

### Technical Analysis

MCP Tools: `compute_technical_indicators` + `identify_trend` (`@ata/mcp-indicators`)

**compute_technical_indicators**: Input `{ candles }` (minimum 200). Output: `{ latest: { sma_20, sma_50, sma_200, rsi_14, macd, macd_signal, macd_histogram, bb_upper, bb_mid, bb_lower, bb_position, atr, atr_pct, volume_ratio } }`

**identify_trend**: Input `{ indicators }`. Output: `{ trend, strength, signals[] }`. Trend values: `strong_up`, `up`, `sideways`, `down`, `strong_down`.

Framework: Trend (SMA alignment) → Momentum (RSI + MACD) → Volatility (Bollinger + ATR) → Synthesis.

### Fundamental Analysis

Use `get_financials` + `get_key_stats` from Yahoo Finance. Framework: Valuation (PE, PB, EV/EBITDA) → Growth (revenue YoY, earnings) → Financial health → Industry comparison.

### Comprehensive Analysis Weighting

| Dimension | Default | day_trade | swing | position/long_term |
|-----------|---------|-----------|-------|--------------------|
| Technical | 40% | 70% | 50% | 30% |
| Fundamental | 40% | 10% | 30% | 50% |
| Sentiment | 20% | 20% | 20% | 20% |

### Risk Metrics (optional)

MCP Tool: `compute_risk_metrics` (`@ata/mcp-indicators`)

Input: `{ candles, benchmark_candles (optional) }`. Output: `{ volatility_30d, max_drawdown_90d, sharpe_30d, beta, var_95 }`.

## Error Handling

For all error codes, rate limits, and retry guidance, see [errors.md](errors.md).
