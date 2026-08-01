# mealie-mcp deployment manifests

Kustomize base for deploying the Mealie MCP server into the cluster. Consumed by ArgoCD via [`dvystrcil/argocd-projects/mealie-mcp/`](https://github.com/dvystrcil/argocd-projects/tree/main/mealie-mcp).

## What this deploys

| Resource | Purpose |
| --- | --- |
| `Namespace mealie-mcp` | Dedicated namespace per the [homelab#33 AC8 decision](https://github.com/dvystrcil/homelab/issues/33) — separates the MCP wrapper from Mealie itself so they can be lifecycle-managed independently. |
| `InfisicalSecret mealie-mcp-infisical-secret` | Syncs `MEALIE_API_KEY` from Infisical's `homelab-bz-gt/prod` project into the `mealie-mcp-secrets` Opaque secret. |
| `InfisicalSecret mealie-mcp-harbor-pull` | Templates a `dockerconfigjson` for pulling the wrapper image from Harbor. Same pattern as other -docker services. |
| `Deployment mealie-mcp` | 1 replica, image pinned by sha tag. Image-updater watches Harbor for new `sha-*` tags and writes back to this file. |
| `Service mealie-mcp` (ClusterIP) | Exposes port 8765 (MCP streamable-http endpoint). Consumed by OWUI's MCP Tool Server config + opencode + future cluster-internal MCP clients. |

## Network model

```
OWUI / opencode / n8n MCP client
        │ POST /mcp (streamable-http)
        ▼
http://mealie-mcp.mealie-mcp.svc.cluster.local:8765/mcp
        │
        ▼ (wrapper)
http://mealie.mealie.svc.cluster.local:80   ← Mealie's HTTP API (port 80 → targetPort http)
```

The wrapper holds `MEALIE_API_KEY` server-side. MCP clients **don't need their own Mealie credential** — they authenticate to OWUI / opencode, which proxy to this server.

## Image promotion

Built by `dvystrcil/mealie-mcp-docker`'s `docker.yaml` workflow on every push to `main`. Tags pushed:
- `:dev` — moving tag for the most recent main commit
- `:sha-<7-char>` — immutable per-commit

The deployment pins `:sha-<7-char>`; argocd-image-updater watches Harbor for newer matching tags and rewrites this file via git write-back.

## Smoke test (post-deploy)

```bash
# From inside the cluster (e.g. an OWUI shell):
curl -sf -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  http://mealie-mcp.mealie-mcp.svc.cluster.local:8765/mcp \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | head -3
```

A non-error response with a `result.tools` array confirms the wrapper started successfully and is reaching Mealie.

## Operator notes

- `MEALIE_API_KEY` must be set in Infisical (`homelab-bz-gt/prod`) before first sync — otherwise the pod will crash-loop with a clear log message from the upstream's import-time validation.
- The wrapper logs to stdout; `kubectl -n mealie-mcp logs -l app=mealie-mcp -f` shows JSON-RPC traffic at `INFO` level.
- To temporarily disable: scale to 0 (`kubectl -n mealie-mcp scale deploy/mealie-mcp --replicas=0`). The InfisicalSecrets stay healthy on their own.
