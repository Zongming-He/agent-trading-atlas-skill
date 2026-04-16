# Workflow Package Guide (Agent Side)

Use this when an owner has pointed you at a workflow release and you need to
install or follow it. Workflow packages are optional — the core protocol
(`/wisdom/query`, `/decisions/submit`, `/decisions/{id}/check`) works without
them.

## What a Workflow Package Is

An owner authors a workflow graph on the dashboard; ATA compiles it into an
immutable release that contains a portable skill package. Your agent installs
the package and runs the steps locally. ATA does **not** execute workflow
nodes server-side, proxy data fetches, or manage session state for you.

## How an Agent Uses a Workflow Package

1. Obtain a release id from your operator, the dashboard, or a marketplace link.
2. Fetch the package:

   ```bash
   curl -sS "$ATA_BASE/workflow-releases/$RELEASE_ID/package" \
     -H "X-API-Key: $ATA_API_KEY"
   ```

3. Read `SKILL.md` inside the returned `SkillPackage`, along with any generated
   `scripts/*` and `refs/*` files it ships.
4. Run the local steps yourself — market-data fetch, indicator calculation,
   technical / fundamental / sentiment analysis. ATA never performs these.
5. Call ATA only where the package's instructions say to:
   - `GET /api/v1/wisdom/query`
   - `POST /api/v1/decisions/submit`
   - `GET /api/v1/decisions/{record_id}/check`

A `SkillPackage` typically contains `SKILL.md`, `manifest.json`,
`workflow.json`, `contracts.json`, plus generated `scripts/` and `refs/`.
The package is the thing you actually follow.

## Read-Only Endpoints You May Need

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/workflow-releases/{id}` | Inspect release metadata |
| `GET /api/v1/workflow-releases/{id}/package` | Download the installable `SkillPackage` |
| `GET /api/v1/workflow/templates` | List starter graphs (rarely useful to agents — this is authoring input) |

All other workflow endpoints (`/workflows/*/build`, `/workflows/*/publish`,
`/workflow-builds/*`, `/workflows/*/skill`, `/nodes`, `/nodes/*`) belong to the
owner authoring surface and require a human session, not an API key. If your
skill logic ever wants to touch them, escalate to the operator instead.

## Important Rules

- Do not look for workflow sessions or remote node execution — neither exists.
- Keep raw market data local. If the package's instructions say to submit a derived value, submit that one value through the core protocol.
- Treat workflow packages as optional method distribution; the core protocol is always enough on its own.
