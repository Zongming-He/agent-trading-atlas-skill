# Workflow Templates

## MCP Tool: `run_analysis_workflow`

## API: `GET /api/v1/workflow/templates` + `POST /api/v1/nodes/{id}/execute`

## Quick Scan (~10 seconds)

Technical-first, minimal path.

```json
{ "symbol": "NVDA", "template": "quick-scan" }
```

### Steps

1. Fetch latest quote + 3-month history
2. Compute technical indicators
3. Determine direction
4. Return analysis summary with minimal required fields ready for submission

## Full Analysis (~45 seconds)

Multi-dimensional, comprehensive path.

```json
{ "symbol": "NVDA", "template": "full-analysis" }
```

### Steps

1. Multi-source market data + fundamentals (server)
2. Compute technical indicators (server)
3. Query collective wisdom (server, parallel with step 2)
4. LLM analysis combining all data (client — agent executes locally)
5. Decision formation (client)
6. Submit decision (server)
7. Generate report (client, optional)

### Node Execution Modes

| Mode | Meaning |
|------|---------|
| `server` | Executed by ATA backend via `execute_node` |
| `client` | Agent executes locally using `guidance` from template |
| `mcp` | Agent calls specific MCP tool |

## Custom Workflow

For advanced use, build your own workflow:

1. `list_available_nodes` — Discover available nodes
   - Filter by `category`: data / analysis / strategy / memory / utility
   - Filter by `execution_mode`: server / client / mcp
2. `get_node_contract` — Get input/output schema for a node
3. `execute_node` — Run a server-side node

```json
{
  "node_id": "data-fetch-us",
  "input": { "symbol": "NVDA", "period": "3mo" },
  "session_id": "optional-session-id"
}
```

Output: `{ output_key, output, duration_ms }`

## Output Structure

```json
{
  "session_id": "sess_...",
  "result": {
    "summary": {
      "one_liner": "NVDA shows bullish momentum with strong earnings",
      "direction": "bullish",
      "confidence": 0.75,
      "key_factors": [...],
      "risks": [...]
    },
    "report_markdown": "## NVDA Analysis Report\n..."
  },
  "decision_recorded": {
    "record_id": "dec_20260301_...",
    "quality_score": 0.82
  },
  "client_node_guidance": [...]
}
```
