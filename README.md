# homelab-ai-inference-starter

Kubernetes manifests for a self-hosted LLM inference stack: **llama.cpp** as the inference server (OpenAI-compatible API) and **Open WebUI** as the chat interface. Designed for a single-node k3s homelab but works on any Kubernetes cluster.

```
┌────────────────────────────────────────────┐
│  Browser → Open WebUI (:80)                │
│              │                             │
│              │ OpenAI-compatible API       │
│              ▼                             │
│         llama-server (:8080)               │
│              │                             │
│              │ reads GGUF model            │
│              ▼                             │
│         hostPath /srv/ai-models            │
│         (model files on k8s node)          │
└────────────────────────────────────────────┘
```

No GPU required. Runs on CPU, tested on a single-node k3s cluster with 16 GB RAM.

## Prerequisites

- Kubernetes cluster (k3s or any distribution)
- `kubectl` configured
- ~8 GB free RAM for a 4-bit quantised 3.8B parameter model
- Disk space for the model file (~2–4 GB per model)

## Quickstart

### 1. Download a model to the node

SSH into your Kubernetes node and download a GGUF model:

```bash
sudo mkdir -p /srv/ai-models

# Example: Phi-3.5 Mini (3.8B, Q4_K_M quantisation — ~2.4 GB)
sudo wget -P /srv/ai-models \
  "https://huggingface.co/bartowski/Phi-3.5-mini-instruct-GGUF/resolve/main/Phi-3.5-mini-instruct-Q4_K_M.gguf"
```

Other good CPU-friendly models:
- **Llama-3.2-3B** — fastest, lowest RAM (~2 GB)
- **Phi-3.5-mini** (default) — fast, good quality (~2.4 GB)
- **Mistral-7B-Instruct** — better quality, needs ~5 GB RAM
- **Llama-3.1-8B** — high quality, needs ~6–8 GB RAM

See [Hugging Face GGUF models](https://huggingface.co/models?library=gguf) for more options.

### 2. Create the namespace

```bash
kubectl create namespace ai-inference
```

### 3. Create the secret

```bash
kubectl create secret generic open-webui-secrets \
  --namespace ai-inference \
  --from-literal=secret-key="$(openssl rand -hex 32)"
```

Or use External Secrets Operator + Vault — see `k8s/external-secret.yaml`.

### 4. Update the model filename

Edit `k8s/llama-server.yaml` and change the `-m` argument to match your downloaded model filename:

```yaml
args:
  - -m
  - /models/your-model-filename.gguf  # ← update this
```

### 5. Apply the manifests

```bash
kubectl apply -f k8s/llama-server.yaml
kubectl apply -f k8s/open-webui.yaml
```

### 6. Access Open WebUI

**Port-forward (quickest test):**

```bash
kubectl port-forward svc/open-webui 8080:80 -n ai-inference
# Open http://localhost:8080
```

**Ingress (permanent access):**  
Edit `k8s/ingress.yaml`, uncomment the block for your ingress controller, update the hostname, and apply:

```bash
kubectl apply -f k8s/ingress.yaml
```

## Files

```
k8s/
├── llama-server.yaml      # llama.cpp Deployment + Service (--metrics enabled)
├── open-webui.yaml        # Open WebUI Deployment + PVC + Service
├── secret.yaml            # Secret setup options (plain kubectl)
├── external-secret.yaml   # ESO + Vault alternative for the secret
├── ingress.yaml           # Traefik / nginx / NodePort ingress options
├── grafana-dashboard.yaml # Grafana dashboard ConfigMap (auto-loaded by sidecar)
└── prometheus-scrape.yaml # Prometheus scrape config snippet + metric reference
```

## Metrics

llama-server exposes a Prometheus metrics endpoint at `:8080/metrics` — enabled by the `--metrics` flag already present in `k8s/llama-server.yaml`.

Key metrics:

| Metric | Type | What it shows |
|--------|------|---------------|
| `llamacpp:predicted_tokens_seconds` | gauge | Generation throughput (tok/s) |
| `llamacpp:tokens_predicted_total` | counter | Total output tokens (cumulative) |
| `llamacpp:prompt_tokens_total` | counter | Total input tokens (cumulative) |
| `llamacpp:requests_processing` | gauge | Requests currently running |
| `llamacpp:requests_deferred` | gauge | Requests queued, waiting for a slot |

**Grafana dashboard:** apply `k8s/grafana-dashboard.yaml`. If your Grafana uses the sidecar pattern with `label: grafana_dashboard`, the dashboard loads automatically. Otherwise import the JSON from the ConfigMap data directly.

**Prometheus scrape config:** see `k8s/prometheus-scrape.yaml` for the config snippet and the full metric list.

**Network policy note:** if your namespaces are isolated (Cilium, Calico), allow inbound connections from the Prometheus namespace to the llama-server namespace on port 8080.

## Configuration reference

### llama-server

Key arguments in `k8s/llama-server.yaml`:

| Arg | Default | Description |
|-----|---------|-------------|
| `-m` | `Phi-3.5-mini-instruct-Q4_K_M.gguf` | Model filename under `/models` |
| `--ctx-size` | `4096` | Context window size (tokens) |
| `--n-predict` | `1024` | Max output tokens per request |
| `--parallel` | `1` | Parallel request slots — increase if you have more RAM |

Model directory: `hostPath: /srv/ai-models` on the k8s node. Change this path to wherever your models live.

Memory limit: set to ~1.5–2× the model's RAM footprint. A 2.4 GB Q4_K_M model typically uses ~3–4 GB at runtime.

### Open WebUI

Key environment variables in `k8s/open-webui.yaml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_BASE_URL` | `http://llama-server.ai-inference.svc:8080/v1` | llama-server endpoint |
| `WEBUI_AUTH` | `false` | Set to `true` to require Open WebUI's own login |
| `ENABLE_RAG_WEB_SEARCH` | `false` | Enable web search in RAG (requires extra config) |

## Namespace

All manifests default to namespace `ai-inference`. To change it:

```bash
sed -i 's/namespace: ai-inference/namespace: your-namespace/g' k8s/*.yaml
```

Also update the `OPENAI_API_BASE_URL` in `open-webui.yaml` to reflect the new namespace:

```
http://llama-server.your-namespace.svc:8080/v1
```

## Resource requirements

| Component | RAM request | RAM limit | Notes |
|-----------|-------------|-----------|-------|
| llama-server | 1 Gi | 6 Gi | Adjust limit to ~2× model size |
| open-webui | 512 Mi | 1.5 Gi | Stable, minimal footprint |

Total minimum: ~8 GB free RAM recommended for a 3–4 GB model.

## Secret model

| Approach | When to use |
|----------|-------------|
| `kubectl create secret` | Single-node homelab, simplest |
| Declarative YAML (not committed) | GitOps without a secrets manager |
| External Secrets Operator + Vault | Full GitOps with secrets management |

The secret contains only `WEBUI_SECRET_KEY` — used to sign Open WebUI sessions. It is not the LLM inference API key (llama-server does not require authentication).

## Using a different inference backend

llama-server exposes an OpenAI-compatible API. You can swap it for any other OpenAI-compatible server (Ollama, vLLM, etc.) by updating `OPENAI_API_BASE_URL` in `open-webui.yaml` to point to the alternative endpoint.

## Example: k3s + Traefik + OAuth2 proxy

This is the author's setup. Traefik is the default ingress controller in k3s, with an OAuth2 proxy sidecar for authentication.

1. Traefik `IngressRoute` is used (see `k8s/ingress.yaml`, Option A).
2. An OAuth2 proxy middleware (`oauth-auth`) in a separate namespace handles authentication — `WEBUI_AUTH` is set to `false` since the proxy handles it upstream.
3. The secret is managed with HashiCorp Vault + External Secrets Operator (`k8s/external-secret.yaml`).
4. Models live at `/srv/ai-models` on the single k3s node.

## License

MIT — see [LICENSE](LICENSE).  
Maintainer: Janos Gyorgy
