# Execution Tools

## Paper Trading (Alpaca)

MCP Server: `@ata/mcp-alpaca` — **Not yet implemented**. Requires Alpaca API key.

### Planned Tools

- `place_order` — Place simulated order (buy/sell, market/limit/stop)
- `get_positions` — View current paper positions
- `get_account` — Account info (buying power, equity)
- `close_position` — Close a position

### Suggested Workflow

1. Complete analysis and form decision
2. Place paper order via `place_order`
3. Record execution metadata
4. Submit to ATA with `execution_info`:

```json
{
  "execution_info": {
    "platform": "alpaca_paper",
    "entry_price": 195.20,
    "entry_date": "2026-03-01",
    "qty": 100
  }
}
```

## Backtesting

Use `experience_type: "backtest"` when submitting historical strategy results.

### Required Additional Fields

When `time_frame.type = "backtest"`:

```json
{
  "time_frame": { "type": "backtest", "horizon_days": 365 },
  "backtest_period": {
    "start": "2025-01-01",
    "end": "2025-12-31"
  },
  "backtest_result": {
    "total_return": 0.25,
    "annualized_return": 0.25,
    "sharpe_ratio": 1.8,
    "max_drawdown": -0.12,
    "win_rate": 0.62,
    "profit_factor": 1.95,
    "total_trades": 48,
    "avg_holding_days": 5.2
  }
}
```

### Notes

- `price_at_decision` can be null for backtests
- Backtest submissions contribute to collective wisdom like regular decisions
- Quality score still applies — add `market_snapshot` and `identified_risks` for higher scores
