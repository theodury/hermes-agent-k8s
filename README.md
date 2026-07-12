# hermes-agent-k8s

Production-ready Kustomize manifests to deploy [Hermes Agent](https://github.com/nousresearch/hermes-agent) (NousResearch) on any Kubernetes cluster.

This repo contains **only the K8s deployment layer** — generic and reusable. Your cluster-specific config (endpoints, node affinity, namespace, secrets) lives in your own GitOps overlay.

## Architecture

```
Discord (private server)
    │
    ▼
Hermes Agent pod
    ├── Ollama  →  http://<your-ollama-endpoint>:11434  (hermes3:8b by default)
    └── Qdrant  →  http://<your-qdrant-endpoint>:6333
```

## Prerequisites

- Kubernetes 1.26+
- Kustomize or Flux CD
- Traefik (optional, for API server mode)
- Prometheus Operator (optional, for alerts)
- SOPS + age key (for secret encryption)
- Ollama running with a Hermes model pulled: `ollama pull hermes3:8b`
- Qdrant running with a collection for RAG memory

## Quick start

### 1. Create your overlay

```bash
cp -r overlays/example overlays/mycluster
cd overlays/mycluster
```

Edit `kustomization.yaml`:
- Set your `namespace`
- Set your `OLLAMA_BASE_URL` and `QDRANT_URL`
- Optionally pin to a specific node
- Optionally change `storageClassName`

### 2. Set up your Discord bot

1. Go to [discord.com/developers](https://discord.com/developers/applications) → New Application
2. Bot section → Reset Token → copy token
3. Enable: `Message Content Intent`, `Server Members Intent`
4. Invite bot to your server with `bot` scope + `Send Messages` + `Read Message History` permissions
5. Find your Discord user ID: Settings → Advanced → Developer Mode → right-click your avatar → Copy User ID

### 3. Encrypt your secret

```bash
# Edit the placeholder
vim manifests/hermes-agent-secrets.sops.yaml
# Replace CHANGE_ME with your real token and user ID

# Encrypt with your age key
sops --encrypt --in-place manifests/hermes-agent-secrets.sops.yaml

# Commit only the encrypted version — NEVER the plaintext
git add manifests/hermes-agent-secrets.sops.yaml
git commit -m 'secret(ai): add encrypted hermes-agent credentials'
```

### 4. Deploy

```bash
# Via kubectl
kubectl apply -k overlays/mycluster/

# Via Flux — add to your GitOps repo kustomization
```

### 5. Verify

```bash
kubectl get pod -n <your-namespace> -l app=hermes-agent
kubectl logs -n <your-namespace> deployment/hermes-agent --tail 100
```

## Customization

| What | How |
|---|---|
| Namespace | `namespace:` in your overlay `kustomization.yaml` |
| Ollama / Qdrant endpoints | Patch `OLLAMA_BASE_URL` / `QDRANT_URL` env vars |
| LLM model | Edit `configmap-config.yaml` → `llm.model` (e.g. `hermes3:14b`, `hermes3:70b`) |
| Node affinity | Patch in overlay (see `overlays/example/`) |
| Storage class | Patch `storageClassName` in overlay |
| Gateway type | Edit `configmap-config.yaml` → `gateway.type` (`discord`, `telegram`, `matrix`) |
| Enable API server | Change Deployment `command` to `["hermes", "serve"]` + uncomment IngressRoute |
| TLS secret | Uncomment and edit `manifests/ingressroute.yaml` with your hostname + TLS secret |

## Skills

Skills are defined in `manifests/configmap-skills.yaml` and mounted into the pod at startup via an init container.

| Skill | Trigger | Description |
|---|---|---|
| `check-homelab-status` | `@bot check status` | Lists K8s nodes + pod health summary |
| `search-runbooks` | `@bot search runbooks <query>` | Queries Qdrant RAG for runbook excerpts |

To add your own skill: add a new entry in `configmap-skills.yaml` following the same pattern (`skill.yaml` + Python entrypoint).

## Enabling the API server (phase 2)

The default mode is `hermes gateway run` (Discord bot only, no HTTP port).

To expose an OpenAI-compatible HTTP API:
1. Change `command` in `deployment.yaml` to `["hermes", "serve"]`
2. Uncomment the IngressRoute in `manifests/ingressroute.yaml`
3. Set your hostname and TLS secret name
4. Choose a middleware (OIDC forward-auth or IP whitelist)

## Alerts

Three PrometheusRules are included (requires Prometheus Operator):

| Alert | Condition | Severity |
|---|---|---|
| `HermesAgentDown` | Pod not ready for 5m | critical |
| `HermesAgentHighMemory` | Memory > 90% of limit for 15m | warning |
| `HermesAgentCrashLoop` | > 3 restarts in 1h | warning |

## License

MIT — contributions welcome.
