# mealie-mcp-docker

Container build of the [`rldiao/mealie-mcp-server`](https://github.com/rldiao/mealie-mcp-server) MCP server adapted for cluster-resident deployment (**streamable-http transport** instead of the upstream's stdio default — matches OWUI's native MCP client which uses `mcp.client.streamable_http.streamablehttp_client`).

Pairs with [homelab#33](https://github.com/dvystrcil/homelab/issues/33) (Mealie MCP cluster deployment).

## What this image does

- Installs the upstream Mealie MCP server (`rldiao/mealie-mcp-server`) from git, pinned to a specific ref (`UPSTREAM_REF` build arg, default `main` — pin to a SHA for real builds).
- Replaces the upstream's stdio `__main__` with our wrapper at `src/server.py`, which runs the same `mcp` object via streamable-http transport on `0.0.0.0:${MCP_PORT}`.
- Exposes `:8765` by default. Configurable via the `MCP_PORT` env.

## Required env (set on the deployment, sourced from Infisical)

| Var | Source | Purpose |
|---|---|---|
| `MEALIE_BASE_URL` | configmap / value | URL of the Mealie API in-cluster (`http://mealie.mealie.svc.cluster.local`) |
| `MEALIE_API_KEY` | Infisical → `mealie/mealie-mcp-token` | Mealie API key (long-lived); enables CRUD as the bot user |
| `MCP_PORT` | optional, default `8765` | Listen port |
| `LOG_LEVEL` | optional, default `INFO` | Server log verbosity |

## CI shape

Identical to [`dvystrcil/ollama-docker`](https://github.com/dvystrcil/ollama-docker)'s sidecar pattern:

- `docker.yaml` runs on push to main: builds + pushes `:dev` and `:sha-XXX` to Harbor, then auto-creates a GitHub release with the next semver tag.
- `docker-release.yaml` runs on release publish: promotes `:dev` to `:vN.N.N`, `:N.N`, `:latest`.
- ImageUpdater watches `harbor.sirddail.net/ai/mealie-mcp:0.x` and writes back to the cluster manifest's overlay.

## Bumping upstream

Edit `Dockerfile`'s `ARG UPSTREAM_REF=<SHA>` to the new commit. Push. CI rebuilds. Verify the new image still imports + runs streamable-http; if upstream renamed `mcp` or moved registration, our `src/server.py` wrapper will fail loudly at startup (intentional — easy to spot).

For breaking upstream changes (new required env vars, schema changes, etc.), drop a sed-style patch under `patches/` and apply it in the Dockerfile before the `pip install`.

## Open questions before first ship

- **Verify the FastMCP streamable-http API**: `mcp.run(transport="streamable-http", host=..., port=...)` works on newer FastMCP versions; older ones use `FASTMCP_HOST`/`FASTMCP_PORT` env vars. The wrapper tries both — if either fails, log the FastMCP version + fix the call signature.
- **Healthcheck endpoint**: streamable-http servers respond at `/mcp` (POST for tool calls; GET typically 405). The HEALTHCHECK accepts 405/406/400 as proof-of-life since the server IS listening. Adjust if upstream uses a different mount point.
- **Mealie API key format**: confirm the upstream expects the bare token (not `Bearer <token>`). The user already added `MEALIE_API_KEY` to Infisical per [homelab#33 AC3](https://github.com/dvystrcil/homelab/issues/33).

## License

MIT for our wrapper code (`src/`, Dockerfile, CI workflows). Upstream code from `rldiao/mealie-mcp-server` retains its upstream license — bundled here only via runtime pip-install, not redistributed in this repo.
