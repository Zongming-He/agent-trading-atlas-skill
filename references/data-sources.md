# Data Sources

## Yahoo Finance (recommended, free)

MCP Server: `@ata/mcp-yahoo-finance` — No API key required.

### get_stock_history

Fetch historical OHLCV candle data.

| Input | Type | Default | Options |
|-------|------|---------|---------|
| `symbol` | string (required) | — | Any US ticker |
| `period` | string | `"1y"` | `1mo`, `3mo`, `6mo`, `1y`, `2y`, `5y` |
| `interval` | string | `"1d"` | `1d`, `1wk`, `1mo` |

Output: `{ symbol, candles: [{ date, open, high, low, close, volume }], count }`

### get_current_quote

| Input | Type |
|-------|------|
| `symbol` | string (required) |

Output: `{ price, change, change_pct, volume, market_cap, day_high, day_low }`

### get_financials

| Input | Type | Default | Options |
|-------|------|---------|---------|
| `symbol` | string (required) | — | — |
| `statement` | string | `"income"` | `income`, `balance`, `cashflow` |
| `frequency` | string | `"quarterly"` | `quarterly`, `annual` |

### get_key_stats

| Input | Type |
|-------|------|
| `symbol` | string (required) |

Output: `{ pe_ratio, pb_ratio, dividend_yield, market_cap, beta, 52w_high, 52w_low, revenue }`

### get_stock_news

| Input | Type |
|-------|------|
| `symbol` | string (required) |

Output: `[{ title, summary, link, published_date, source }]`

### Yahoo Finance Errors

- `SYMBOL_NOT_FOUND` — Unknown ticker
- `RATE_LIMITED` — Wait 1 second, retry
- `DATA_UNAVAILABLE` — Temporary outage

## Fallback API Path (without MCP)

```python
import yfinance as yf
ticker = yf.Ticker("NVDA")
hist = ticker.history(period="3mo")
quote = ticker.info
```

## Multi-Source Priority

1. Yahoo Finance (default, free)
2. Polygon (`@ata/mcp-polygon`, premium, not yet implemented)
3. Alpha Vantage (manual integration)

Use SEC Edgar filings to enrich catalyst and risk context when available.
