# Analysis Frameworks

## Technical Analysis

MCP Tools: `compute_technical_indicators` + `identify_trend` (`@ata/mcp-indicators`)

### compute_technical_indicators

Input: `{ candles }` — Array of OHLCV candles (minimum 200 for all indicators).

Output:

```json
{
  "latest": {
    "sma_20": 185.4,
    "sma_50": 180.2,
    "sma_200": 170.8,
    "rsi_14": 55.3,
    "macd": 2.1,
    "macd_signal": 1.8,
    "macd_histogram": 0.3,
    "bb_upper": 195.0,
    "bb_mid": 185.4,
    "bb_lower": 175.8,
    "bb_position": 0.65,
    "atr": 5.2,
    "atr_pct": 2.8,
    "volume_ratio": 1.15
  }
}
```

Error: `INSUFFICIENT_DATA` if < 200 candles provided.

### identify_trend

Input: `{ indicators }` — Output from `compute_technical_indicators`.

Output:

```json
{
  "trend": "up",
  "strength": 0.72,
  "signals": [
    { "signal": "SMA20 > SMA50 golden cross", "direction": "bullish", "strength": 0.8 },
    { "signal": "RSI neutral zone", "direction": "neutral", "strength": 0.5 }
  ]
}
```

Trend values: `up`, `sideways`, `down`

### Technical Framework

1. **Trend**: SMA20 vs SMA50 vs SMA200 alignment
2. **Momentum**: RSI14 zones + MACD crossovers
3. **Volatility**: Bollinger Band position + ATR percentage
4. **Synthesis**: Score bullish/bearish/neutral with confidence range

## Fundamental Analysis

No dedicated MCP tool — use `get_financials` + `get_key_stats` from Yahoo Finance.

### Framework

1. **Valuation**: PE, PB, EV/EBITDA vs sector median
2. **Growth**: Revenue YoY, earnings growth, earnings surprise
3. **Financial health**: Leverage, free cash flow, liquidity ratios
4. **Industry comparison**: Relative positioning

### Output Structure

Provide structured summary with bull/bear drivers and key risks:
- Bull case factors
- Bear case factors
- Key risks and catalysts

## Composite Analysis

Combine technical, fundamental, and sentiment dimensions.
When submitting, use `"perspective_type": "composite"` in the `approach` object.

### Weighting Baseline

| Dimension | Weight |
|-----------|--------|
| Technical | 40% |
| Fundamental | 40% |
| Sentiment/news | 20% |

Adjust weights based on time_frame:
- `day_trade`: Technical 70%, Sentiment 20%, Fundamental 10%
- `swing`: Technical 50%, Fundamental 30%, Sentiment 20%
- `position`/`long_term`: Fundamental 50%, Technical 30%, Sentiment 20%

### Output

Produce: final direction, confidence, risk checklist, action plan,
and structured `market_snapshot` for decision submission.

## Risk Metrics (optional)

MCP Tool: `compute_risk_metrics` (`@ata/mcp-indicators`)

Input: `{ candles, benchmark_candles (optional) }`

Output: `{ volatility_30d, max_drawdown_90d, sharpe_30d, beta, var_95 }`
