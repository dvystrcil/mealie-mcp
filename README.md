# mealie-mcp

Deploy manifests for the [mealie-mcp-docker](https://github.com/dvystrcil/mealie-mcp-docker) container image.

This repo holds Kustomize bases, overlays, and the ImageUpdater CRD for
cluster-resident deployment. Build artifacts (Dockerfile, source code, CI)
live in [dvystrcil/mealie-mcp-docker][docker].

## What it deploys

- **Deployment** `mealie-mcp` - runs the Mealie MCP server with streamable-http transport
- **Service** `mealie-mcp` - exposes port 8765
- **ImageUpdater** - auto-updates deployment when Harbor image tag changes

See [homelab#33][issue] for background.

[issue]: https://github.com/dvystrcil/homelab/issues/33
[docker]: https://github.com/dvystrcil/mealie-mcp-docker

## Environment (Infisical → Secret)

| Var | Source |
|---|---|
| `MEALIE_BASE_URL` | `mealie/mealie-base-url` (config) |
| `MEALIE_API_KEY` | `mealie/mealie-mcp-token` |

## CI workflow

- Push to this repo updates the cluster via ArgoCD
- PR validation: kustomize build + dry-run

## Bumping image versions

Update `overlays/prod/kustomization.yaml`:

```yaml
images:
  - name: harbor.sirddail.net/ai/mealie-mcp
    newTag: v1.2.3
```

Push → ArgoCD syncs.
