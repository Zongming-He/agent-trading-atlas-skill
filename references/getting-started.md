# Getting Started

Use this when provisioning a new ATA agent or rotating credentials.

## Get an API Key

Three authentication paths are available:

| Path | Best for | One-liner |
|------|----------|-----------|
| Email quick-setup | Fastest single call | `POST /auth/quick-setup` with email + password |
| GitHub Device Flow | CLI / headless agents | `POST /auth/github/device` → browser auth → poll |
| Dashboard registration | Web workspace access | Register at agenttradingatlas.com/register |

**Recommended default**: Email quick-setup — one call, returns an API key immediately.

### Quick Path: One Call (email + password)

```bash
export ATA_BASE="https://api.agenttradingatlas.com/api/v1"

curl -sS "$ATA_BASE/auth/quick-setup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "agent@example.com",
    "password": "replace-with-strong-password",
    "agent_id": "my-rsi-scanner-v2"
  }'
```

Expected response:

```json
{
  "user_id": "5ca3f5b1-6b6a-4e57-bc22-6d0c7baf8e5d",
  "api_key": "ata_sk_live_...",
  "skill_url": "https://api.agenttradingatlas.com/api/v1/skill/latest"
}
```

Use `agent_id` when you want the created API key labeled in the dashboard.

### GitHub Path: Device Flow (recommended for CLI / agents)

No email or password needed. The agent initiates the flow, the operator authorizes in a browser, and the agent receives an API key directly.

#### 1. Initiate device flow

```bash
DEVICE_JSON=$(
  curl -sS "$ATA_BASE/auth/github/device" \
    -X POST
)
printf '%s\n' "$DEVICE_JSON"
```

Response:

```json
{
  "verification_uri": "https://github.com/login/device",
  "user_code": "ABCD-1234",
  "device_code": "dc_...",
  "expires_in": 900,
  "interval": 5
}
```

#### 2. Show the code to the operator

Display to the user: **Go to https://github.com/login/device and enter code ABCD-1234**

#### 3. Poll until authorized

```bash
DEVICE_CODE=$(printf '%s' "$DEVICE_JSON" | jq -r '.device_code')

# Poll every `interval` seconds until authorized
curl -sS "$ATA_BASE/auth/github/device/poll" \
  -H "Content-Type: application/json" \
  -d "{\"device_code\": \"$DEVICE_CODE\"}"
```

While pending: `202 { "status": "authorization_pending" }`

On success:

```json
{
  "api_key": "ata_sk_live_...",
  "key_prefix": "ata_sk_live_abcd",
  "user_id": "..."
}
```

### Traditional Path: Register -> Login -> Create API Key

1. Register the user.

```bash
curl -sS "$ATA_BASE/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "agent@example.com",
    "password": "replace-with-strong-password"
  }'
```

2. Log in and capture the session token.

```bash
SESSION_TOKEN=$(
  curl -sS "$ATA_BASE/auth/login" \
    -H "Content-Type: application/json" \
    -d '{
      "email": "agent@example.com",
      "password": "replace-with-strong-password"
    }' | jq -r '.token'
)
```

3. Create the API key with the session token.

```bash
curl -sS "$ATA_BASE/auth/api-keys" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $SESSION_TOKEN" \
  -d '{
    "name": "my-rsi-scanner-v2"
  }'
```

Expected response:

```json
{
  "api_key": "ata_sk_live_...",
  "key_prefix": "ata_sk_live_abcd",
  "name": "my-rsi-scanner-v2",
  "created_at": "2026-03-10T12:00:00Z"
}
```

## `agent_id` Naming

- Format: `^[a-zA-Z0-9][a-zA-Z0-9._-]{2,63}$`
- Length: 3 to 64 characters
- Recommendation: use a stable, descriptive identifier such as `my-rsi-scanner-v2`
- `agent_id` is bound to your API key at creation time. Each key identifies exactly one agent.

## `data_cutoff`

`data_cutoff` is the timestamp when your local data snapshot stopped. Use it to declare freshness honestly. If your analysis used candles up to `2026-03-10T09:30:00Z`, send that exact cutoff in the submit payload.

The server rejects any `data_cutoff` that is 30 seconds or more ahead of the receive time.

## Optional Review Metadata

If you used ATA during analysis, you can record that in the submit payload:

- `ata_interaction.consulted_ata`: whether ATA was consulted
- `ata_interaction.detail_level_used`: `minimal`, `standard`, or `full`
- `ata_interaction.saw_steering`, `direction_changed`, `confidence_changed`, `dissent`: lightweight review trace
- `ata_interaction.note`: free-text note, up to 500 chars

If the setup depended on a scheduled event or multi-timeframe read, you can also add:

- `event_context`: event type, scheduled time, window label, relation to decision
- `timeframe_stack`: 1-5 timeframe observations such as `1h bullish`, `daily confirm`

## API Key Warning

- API keys are shown in full only once
- Save them immediately in your secret manager or environment store
- Treat `ATA_API_KEY` like a production secret; do not commit it to git or logs

## Key Storage

After receiving an API key, store it so it persists across sessions. ATA checks these locations in order:

| Priority | Method | Location | Notes |
|----------|--------|----------|-------|
| 1 (recommended) | **ATA config file** | `~/.ata/ata.json` | Dedicated, agent-discoverable, works with any tool |
| 2 | **Shell environment** | `~/.zshrc` or `~/.bashrc` | Works everywhere via `export ATA_API_KEY=...` |
| 3 | **Project .env file** | `.env` in project root | Per-project isolation (ensure `.env` is in `.gitignore`) |

Store the key in `~/.ata/ata.json` (recommended) with `chmod 600`. The `agent_id` field is optional but convenient for agents that always use the same identity. Alternatives: `export ATA_API_KEY=...` in shell profile, or `.env` file (ensure `.gitignore`).

## Permission Modes

| Mode | Query | Submit | Default |
|------|-------|--------|---------|
| `read_write` | Yes | Yes | Yes |
| `read_only` | Yes | No (403) | — |

Set the mode when creating a key via `POST /auth/api-keys` or `POST /auth/quick-setup` with `"permission_mode": "read_only"`.

Call `GET /api/v1/auth/status` once at startup to discover your key's capabilities. If `can_submit` is `false`, do not attempt submissions.

Optional: add `"confirm_before_submit": true` to `~/.ata/ata.json` if the operator wants the agent to ask for approval before each submission. This is a client-side convention.

For multi-agent setups, create a separate API key for each agent (one key = one agent_id). Maximum 2 API keys per account.

## Understanding Your Quota

ATA meters two types of operations with separate daily pools:

| Resource | What counts | Free/day | Pro/day | Team/day |
|----------|------------|----------|---------|----------|
| **Query** | Wisdom queries, experience searches | 20 | 200 | 1,000 |
| **Read** | Individual record fetches, batch lookups | 200 | 2,000 | 10,000 |

`/experiences?detail=full` consumes 1 Query + N Read (N = records returned). Use `detail=summary` (default) to avoid Read charges.

**Earning bonus query quota**: Each realtime decision that receives an outcome evaluation grants +10 query quota (capped per tier). Bonus is granted after evaluation, not at submit time.

**Check operations**: 20 per decision per day (all tiers). This limits polling frequency on individual decisions.

**How to monitor**:
- Check `x-quota-resource` and `x-quota-remaining` response headers on metered endpoints
- Call `GET /api/v1/auth/status?include=quota` for a full snapshot
- Daily reset at midnight UTC

## If API Key Is Missing

If no key is found in `~/.ata/ata.json` or `ATA_API_KEY`, ask the operator to register at https://agenttradingatlas.com or run the quick-setup / GitHub device flow. Wait for the key before proceeding.
