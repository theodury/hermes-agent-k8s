# hermes-agent-k8s

Kustomize manifests to deploy [Hermes Agent](https://github.com/nousresearch/hermes-agent) (NousResearch) on a Kubernetes cluster.

This repo contains **only the K8s deployment layer**. Application config (Ollama endpoint, Qdrant endpoint, Discord token) is injected via ConfigMap and SOPS-encrypted Secret managed in your homelab GitOps repo.

## Architecture

```
Discord (Hypnoga server)
    │
    ▼
Hermes Agent pod (ai namespace)
    ├── Ollama  → http://ollama.ai.svc.cluster.local:11434  (hermes3:8b)
    └── Qdrant  → http://qdrant.ai.svc.cluster.local:6333
```

## Prerequisites

- Kubernetes cluster with Flux CD
- Namespace `ai` already exists
- Traefik as ingress controller with `wildcard-lab-technoga-net-tls` secret
- Ollama running with `hermes3:8b` model pulled
- Qdrant running with a collection for RAG
- SOPS + age key configured in your cluster

## Usage in your homelab repo

Reference this repo as a Flux `GitRepository` source, or copy the manifests into your `apps/base/ai/hermes-agent/` directory.

### Quick copy

```bash
cp -r manifests/ ~/workspace/technoga/homelab/apps/base/ai/hermes-agent/
```

### Flux GitRepository (optional)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: hermes-agent-k8s
  namespace: flux-system
spec:
  interval: 10m
  url: https://github.com/theodury/hermes-agent-k8s
  ref:
    branch: main
```

## Secret setup

Before deploying, encrypt your Discord token with SOPS:

```bash
# 1. Edit the placeholder secret
vim manifests/hermes-agent-secrets.sops.yaml
# Replace CHANGE_ME with your actual token from BotFather

# 2. Encrypt with your age key
sops --encrypt --in-place manifests/hermes-agent-secrets.sops.yaml

# 3. Commit the encrypted file (never the plaintext)
git add manifests/hermes-agent-secrets.sops.yaml
git commit -m 'secret(ai): add encrypted hermes-agent discord token'
```

## Verify deployment

```bash
kubectl get pod -n ai -l app=hermes-agent
kubectl logs -n ai deployment/hermes-agent --tail 100
```

## Skills

Skills are stored in `skills/` and mounted via ConfigMap into `/opt/data/hermes/skills/`.

| Skill | Description |
|---|---|
| `check-homelab-status` | Lists nodes + pods health summary via Kubernetes API |
| `search-runbooks` | Queries Qdrant RAG knowledge base for runbooks |

## Upgrade

Update the image tag in `manifests/deployment.yaml`:

```yaml
image: nousresearch/hermes-agent:latest  # pin to a specific tag for prod
```

## License

MIT
