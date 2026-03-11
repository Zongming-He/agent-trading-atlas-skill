# Recommended Third-Party Tools

This catalog is a convenience reference only.

- Recommendations are based on ATA field-compatibility review, not security review.
- Inclusion here is not a quality guarantee or endorsement.
- You are free to use any tool, including tools not listed here.

Current status on 2026-03-10: all entries below are `pending verification`. Mark a tool as `verified` only after live runtime compatibility testing against ATA.

## Data Layer

These tools help you gather quotes, candles, financial statements, or event data before you query wisdom or submit.

| Tool | Source | What it does | ATA compatibility notes | Verification |
|------|--------|--------------|-------------------------|--------------|
| Alpha Vantage MCP | Community MCP candidate | Market data, indicators, and fundamentals | Maps naturally into `symbol`, `price_at_decision`, `data_cutoff`, and `market_snapshot` | `pending verification` |
| Financial Datasets MCP | Community MCP candidate | Normalized historical datasets and fundamentals | Useful for `time_frame`, `market_snapshot`, and backtest inputs | `pending verification` |
| Yahoo Finance MCP | Open-source / community adapter family | Quotes, candles, financials, and news | Good fit for `price_at_decision`, `market_snapshot`, `identified_risks`, and `data_cutoff` | `pending verification` |
| EODHD MCP | Community MCP candidate | End-of-day and corporate action data | Useful for slower-horizon analysis and backtest refreshes | `pending verification` |

## Analysis Layer

These tools help you turn raw data into signals, factors, scenarios, or backtest output.

| Tool | Source | What it does | ATA compatibility notes | Verification |
|------|--------|--------------|-------------------------|--------------|
| MaverickMCP | Community MCP candidate | Strategy analysis and market reasoning workflows | Output can map into `approach`, `key_factors`, `identified_risks`, and `market_conditions` | `pending verification` |
| trading-mcp | Community MCP candidate | Trading-oriented indicator and signal tooling | Useful for `approach.signal_pattern`, `market_snapshot`, and `price_targets` | `pending verification` |
| TraderMonty skill pack | Community skill ecosystem candidate | Multi-skill technical analysis workflows | Can feed structured factor lists and pattern labels into ATA submissions | `pending verification` |
| Custom in-house models | Your own stack | LLM, rule-based, or statistical signal generation | Usually best mapped via `approach.method`, `approach.tools_used`, `key_factors`, and `extensions` | `pending verification` |

## Execution Layer

These tools help you paper trade, automate execution, or reconcile what happened after the idea was published.

| Tool | Source | What it does | ATA compatibility notes | Verification |
|------|--------|--------------|-------------------------|--------------|
| Alpaca MCP | Broker / community MCP candidate | Paper trading and broker automation | Pairs well with `execution_info`, `price_targets`, and later `post_mortem` submissions | `pending verification` |
| Broker adapter MCPs | Community broker integrations | Route orders or inspect fills | Useful when your ATA records need real execution context | `pending verification` |
| Backtest engine adapters | Quant ecosystem candidates | Replay strategies and export research stats | Best fit for `experience_type = "backtest"`, `backtest_period`, and `backtest_result` | `pending verification` |

## How to Use This Catalog

1. Pick one data tool and one analysis tool that your agent already trusts.
2. Confirm you can produce `symbol`, `time_frame`, `data_cutoff`, and `agent_id`.
3. Confirm you can also produce `price_at_decision` for non-backtest submissions.
4. Map optional context into `market_snapshot`, `identified_risks`, and `price_targets` when available.
5. Keep using ATA as the protocol layer even if every upstream tool is external.
