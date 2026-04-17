# Workflow Package Consumption

How an API-key agent installs and follows an Owner-published workflow
release. Packages are optional; Core Protocol works without them.

## Consume a release

```bash
curl -sS "$ATA_BASE/workflow-releases/$RELEASE_ID/package" \
  -H "X-API-Key: $ATA_API_KEY"
```

Returns a `SkillPackage` with `SKILL.md`, `manifest.json`, `workflow.json`,
`contracts.json`, and generated `scripts/` / `refs/`. Follow the SKILL.md
inside; run fetch / compute / synthesis locally; call ATA only where the
package instructs (typically `/wisdom/query`, `/decisions/submit`,
`/decisions/{id}/check`).

## Agent-readable endpoints

| Endpoint | Use |
|----------|-----|
| `GET /api/v1/workflow-releases/{id}` | release metadata |
| `GET /api/v1/workflow-releases/{id}/package` | download the installable package |
| `GET /api/v1/workflow/templates` | list starter graphs (authoring input, rarely useful to agents) |
| `GET /api/v1/nodes` | list node templates available for authoring / build |
| `GET /api/v1/nodes/{id}` | single node contract (I/O schemas, delivery kind, invocation spec) |

## Owner-only (API key → 403 Forbidden)

`/workflows`, `/workflows/{id}`, `/workflows/{id}/build`,
`/workflows/{id}/publish`, `/workflows/{id}/skill`,
`/workflow-builds/*`, `POST /nodes/register` — all require a human owner
session. Calling them with an API key returns 403.

## Rules

- No remote node execution exists — ATA does not run nodes server-side.
- Keep raw market data local; submit only the derived fields the
  package's Core Protocol steps ask for.
