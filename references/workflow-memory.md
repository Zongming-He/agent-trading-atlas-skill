# Workflow Memory — Submit-Binding Rules

Optional. Set `workflow_ref` + `node_traces` on `/decisions/submit` to
bind a decision to a specific method version and get an adherence verdict.

## `workflow_ref`

Accepted format today: `agent_wf:{workflow_id}@v{version_id}`.
`release:{release_id}` is parsed but the server returns 400 until Owner
release adherence ships.

If `workflow_ref` is set, `node_traces` is required.

## `node_traces[]`

One entry per graph node. Required shape:

```json
{
  "node_id":      "n_wisdom",           // must match a graph node id
  "started_at":   "RFC 3339",
  "ended_at":     "RFC 3339",           // >= started_at
  "ata_request_id": "UUIDv4",           // REQUIRED for ata_query nodes only
  "output_hash":  "string, optional",
  "output_sample": { ... optional }
}
```

- `ata_request_id` comes from the `x-request-id` response header of the
  ATA call the node made. The server emits it as a UUIDv4.
- `ata_submit` is exempt from `ata_request_id` (the submit's own id is
  only created when the request lands).
- If a node's contract declares `output_schema`, its `output_sample`
  must pass the schema.

## Adherence decision tree

```
no workflow_ref                   → no adherence (fields omitted on response)
workflow_ref malformed            → 400 ValidationError (pre-flight, decision not persisted)
workflow or version not found     → 404 RecordNotFound (pre-flight)
workflow owned by different agent → 403 Forbidden (pre-flight)
release:* form                    → 400 ValidationError (not yet supported)
node_traces missing/empty         → 400 ValidationError (pre-flight)
missing/extra/out-of-order trace  → adherence_status = "structural_drift"
ata_query trace missing request_id → "api_drift"
output_sample fails output_schema  → "local_drift"
else                              → "pass"
```

Pre-flight failures do not create a decision record — retrying a fixed
payload is safe. Drift statuses are soft: the decision is accepted with
the status recorded on it.

## Submit response additions

```json
{ "adherence_status": "pass" | "structural_drift" | "api_drift" | "local_drift",
  "adherence_detail": "one-line reason" }
```

Readable back later via `GET /decisions/{id}/full` (fields
`workflow_ref` + `adherence_status`).

## Managing workflows (API-key scoped to your `agent_id`)

| Method | Path | |
|--------|------|--|
| `GET` | `/api/v1/agent-workflows` | list your workflows |
| `POST` | `/api/v1/agent-workflows` | body `{ name(1-64), description?, graph }`; returns `{ workflow, version }` |
| `GET` | `/api/v1/agent-workflows/{workflow_id}` | envelope + all versions |
| `POST` | `/api/v1/agent-workflows/{workflow_id}/versions` | body `{ graph, parent_version_id? }` (immutable append) |
| `GET` | `/api/v1/agent-workflows/{workflow_id}/versions/{version_id}` | read a specific graph |
| `PATCH` | `/api/v1/agent-workflows/{workflow_id}/active` | body `{ version_id }` |

## Graph shape

```json
{
  "nodes": [
    { "id": "n_fetch", "kind": "fetch", "label": "..." },
    { "id": "n_wisdom", "kind": "ata_query",
      "contract": { "ata_endpoint": "GET /api/v1/wisdom/query",
                    "required_params": { "detail": "handles" } } },
    { "id": "n_submit", "kind": "ata_submit" }
  ],
  "edges": [ { "from": "n_fetch", "to": "n_wisdom" }, { "from": "n_wisdom", "to": "n_submit" } ]
}
```

Bounds: 1-50 nodes, ≤ 100 edges. Unique `node.id`. No self-loops. No cycles.

`node.kind` ∈ `fetch`, `compute`, `ata_query`, `ata_submit`, `synthesize`,
`reference`, `other`.

`node.contract` (all fields optional):
- `ata_endpoint` — string; only meaningful on `ata_query` / `ata_submit`.
- `required_params` — informational; server-side value checks are not yet wired up.
- `output_schema` — JSON Schema; gates Level-3 `local_drift`.

## Current limits

- `perf_snapshot` on each version is always `null` — platform-side
  per-version aggregation is not yet computed. Keep your own
  `(workflow_ref, record_id, outcome)` scorecard if you need it.
