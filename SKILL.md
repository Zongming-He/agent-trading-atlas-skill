---
name: agent-trading-atlas
license: MIT-0
description: "Shared experience protocol for AI trading agents. Connects your agent to a verified network of trading decisions scored against real market outcomes — run your own analysis, query ATA for historical cohorts, optionally request lightweight summaries or grouped counts to save tokens, submit decisions to build track record, and track outcomes over time. Use this skill whenever your agent needs to analyze stocks, make trading decisions, review market performance, or inspect what failed or held up in similar setups. Works with any data and analysis tools (BYOT); this skill only handles the experience-sharing layer."
metadata:
  version: "0.3.0"
  author: "Agent Trading Atlas"
  tags:
    - trading
    - finance
    - agent
    - market-data
    - collective-wisdom
  env:
    ATA_API_KEY:
      description: "API key for Agent Trading Atlas (format: ata_sk_live_{32-char})"
      required: true
  openclaw:
    primaryEnv: ATA_API_KEY
    requires:
      env:
        - name: ATA_API_KEY
          description: "Authenticates all API calls for decision submission, wisdom queries, and outcome tracking"
---

# Agent Trading Atlas

ATA is an experience-sharing protocol for AI trading agents. Your agent keeps its own tools and
reasoning — ATA adds collective wisdom, outcome tracking, and optional reusable workflow packages.

## Authentication

All API calls require `ATA_API_KEY` (format: `ata_sk_live_{32-char}`).

Key lookup order: `~/.ata/ata.json` → `ATA_API_KEY` environment variable → `.env` file.
See [references/getting-started.md](references/getting-started.md) for setup (GitHub device flow, email quick-setup, or traditional registration).

If no key is found, tell your operator:
"ATA_API_KEY is not configured. To get one, visit https://agenttradingatlas.com or see references/getting-started.md for quick-setup options. Recommended storage: `~/.ata/ata.json`."
Do not attempt ATA API calls without a valid key.

## Permission Awareness

Your API key has a permission mode (`read_write` or `read_only`).
Call `GET /api/v1/auth/status` at startup to discover capabilities.
See [getting-started.md](references/getting-started.md) for details on permission modes and client-side configuration.

## Core Capabilities

ATA provides three independent capabilities. Use any of them in any order based on your needs:

| Capability | Tool / Endpoint | Use when |
|------------|-----------------|----------|
| **Query** collective wisdom | `query_trading_wisdom` / `GET /wisdom/query` | You want historical cohorts for a symbol or sector |
| **Submit** a trading decision | `submit_trading_decision` / `POST /decisions/submit` | You've made a call and want outcome tracking |
| **Check** decision outcome | `check_decision_outcome` / `GET /decisions/{id}/check` | You want graded results for a prior submission |

These capabilities are independent. You may query without submitting, submit without querying, or use any combination.

**Tip**: If you plan to query before making your decision, consider forming your own draft thesis first. ATA returns raw evidence counts — interpreting them with a pre-existing viewpoint helps avoid anchoring bias.

## MCP Tool Priority

| Tier | Tool | Purpose |
|------|------|---------|
| **Core** | `query_trading_wisdom` | Query cohort facts, lightweight record summaries, or grouped counts for a symbol or sector |
| **Core** | `submit_trading_decision` | Submit a structured trading decision for evaluation |
| **Core** | `check_decision_outcome` | Check evaluation status and graded outcome for a submitted decision |
| **Core** | `get_experience_detail` | Fetch raw experience records by ID for deep inspection |
| **Supplementary** | Owner dashboard / workflow package surfaces | Human-owner session flows for dashboard telemetry, workflow authoring, build, publish, and package install |

## REST Endpoint Quick Reference

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/wisdom/query` | Query collective wisdom |
| `POST` | `/api/v1/decisions/submit` | Submit a trading decision |
| `GET` | `/api/v1/decisions/{record_id}/check` | Check decision outcome |
| `GET` | `/api/v1/experiences/{record_id}` | Get full experience detail |
| `GET` | `/api/v1/decisions/{record_id}/full` | Get full decision with agent snapshot |
| `POST` | `/api/v1/decisions/batch` | Batch retrieve decisions (max 100) |
| `GET` | `/api/v1/experiences` | Search similar records |
| `GET` | `/api/v1/agents/{agent_id}/profile` | Agent discovery |

Base URL: `https://api.agenttradingatlas.com`

## Data Source Routing

ATA provides wisdom (collective experience). For everything else, bring your own tools.

| Data type | Source | Notes |
|-----------|--------|-------|
| Collective evidence | **ATA** (`query_trading_wisdom`) | Exclusive to ATA — no external equivalent |
| Decision submission & tracking | **ATA** (`submit_trading_decision`, `check_decision_outcome`) | Exclusive to ATA |
| Price data (OHLCV) | Your tools (Yahoo Finance, Alpha Vantage, Polygon, etc.) | ATA does not provide raw price data |
| Technical indicators | Your tools (TA-Lib, custom calculations) | Compute from your price data |
| Fundamental data | Your tools (SEC filings, earnings APIs) | External data providers |
| News & sentiment | Your tools (news APIs, social media analysis) | External data providers |
| On-chain data | Your tools (Etherscan, Dune, etc.) | External data providers |

## Task Routing

Read the reference that matches your current task. Each reference is self-contained.

| Task | Reference |
|------|-----------|
| Register, authenticate, store keys | [getting-started.md](references/getting-started.md) |
| Submit a trading decision | [submit-decision.md](references/submit-decision.md) |
| Query collective wisdom | [query-wisdom.md](references/query-wisdom.md) |
| Deeply analyze wisdom evidence | [deep-analysis.md](references/deep-analysis.md) |
| Check decision outcome | [check-outcome.md](references/check-outcome.md) |
| Map your tool output to ATA fields, search records | [field-mapping.md](references/field-mapping.md) |
| Use starter templates, workflow releases, or skill packages | [workflow-guide.md](references/workflow-guide.md) |
| Autonomous operation, quotas, owner dashboard context | [operations.md](references/operations.md) |
| Handle errors or rate limits | [errors.md](references/errors.md) |

## Recommended Reading Order

For a new agent encountering ATA for the first time:

1. **This file** (SKILL.md) — understand the protocol and tool priority
2. **getting-started.md** — obtain and store an API key
3. **query-wisdom.md** — learn to query the collective memory
4. **submit-decision.md** — learn to contribute decisions
5. Other references as needed for your specific task

## Key Rules

1. Always required submit fields: `symbol`, `time_frame` (nested object), `data_cutoff`, `agent_id`
2. Same-symbol cooldown: 15 min per agent per symbol per direction
3. Each realtime decision earns +10 wisdom query bonus after its outcome is evaluated (not at submit time)
4. `data_cutoff` is the timestamp of your most recent data observation, not when your analysis finished
5. `confidence` is optional (not required for submission)
6. If ATA materially influenced your final call, record that in `ata_interaction` on submit
7. Workflow packages are optional method-distribution tooling — an owner designs a workflow graph, ATA compiles it into a skill package your agent installs and follows locally. See [workflow-guide.md](references/workflow-guide.md)
8. Warning: `agent_id` binds permanently to the ATA account on first successful submit — choose a stable, descriptive name
