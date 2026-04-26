# Project Overview: terminus-inference-qwen-warmup

**Generated:** 2026-04-26  
**Repository:** `electricm0nk/terminus-inference-qwen-warmup`  
**Feature:** terminus-inference-qwen-adapter  
**Version:** `main` — squash-merged from `feature/qwen-warmup` (PR #1, 2026-04-26)

---

## Purpose

A GPU-dedicated container that handles the full Qwen 2.5:14b model lifecycle before serving inference requests. The container runs three sequential phases:

1. **Ollama start** — `ollama serve` in background
2. **Warmup** — GPU check, model pull, test inference, ready marker written to `/tmp/warmup.ready`
3. **Agent service** — FastAPI server for health endpoints and Prometheus metrics

This pattern prevents cold-start failures in Kubernetes where the readiness probe fires before the model is loaded.

---

## Technology Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Base image | `ollama/ollama:latest` | latest |
| Language | Python | 3.x (system) |
| HTTP framework | FastAPI | 0.115.6 |
| ASGI server | uvicorn[standard] | 0.32.1 |
| HTTP client | requests | 2.32.3 |
| Metrics | prometheus-client | 0.21.0 |
| Warmup HTTP | urllib.request (stdlib) | — |
| Test framework | pytest | 8.3.4 |
| Test coverage | pytest-cov | 6.0.0 |
| Container runtime | Docker + Kubernetes | — |
| Model server | Ollama | latest |
| Model | Qwen 2.5:14b | — |

---

## Architecture

### Container Startup Flow

```
[K8s starts pod]
      │
      ▼
entrypoint.sh
      │
      ├── ollama serve &          ← background
      │
      ├── python3 warmup.py       ← blocks until done
      │       │
      │       ├── check_gpu_memory()    → nvml check
      │       ├── wait_for_ollama()     → polls /api/tags (60s timeout)
      │       ├── pull_model()          → ollama pull qwen2.5:14b
      │       ├── test_generation()     → small inference to warm GPU
      │       └── writes /tmp/warmup.ready
      │
      └── exec python3 main.py    ← FastAPI (replaces process)
```

### Probe Design

| Probe | Type | Target | Purpose |
|-------|------|--------|---------|
| Readiness | exec | `test -f /tmp/warmup.ready && curl -sf http://localhost:8000/health/ready` | Two-gate: file marker + FastAPI serving |
| Liveness | httpGet | `http://localhost:11434/api/tags` | Direct model server heartbeat |

Readiness timing: `initialDelaySeconds: 60`, `periodSeconds: 10`, `failureThreshold: 30` (up to 5 minutes for model pull).

### API Endpoints

| Endpoint | Method | Response | Purpose |
|----------|--------|----------|---------|
| `/health` | GET | 200 `{"status": "alive"}` | Liveness — always returns |
| `/health/ready` | GET | 200/503 JSON | Readiness — checks marker + ollama |
| `/metrics` | GET | Prometheus text | prometheus-client scrape target |

`/health/ready` 503 response includes a `reason` field: `warmup-in-progress`, `ollama-unavailable`, or `model-not-loaded`.

### Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `qwen_adapter_warmup_duration_seconds` | Gauge | Total warmup duration |
| `qwen_adapter_model_load_time_seconds` | Gauge | Time to first token on warmup inference |
| `qwen_adapter_gpu_memory_used_bytes` | Gauge | GPU memory at warmup start |
| `qwen_adapter_ollama_uptime_seconds` | Counter | Ollama uptime counter |
| `qwen_adapter_warmup_success_total` | Counter | Successful warmup count |
| `qwen_adapter_warmup_failure_total` | Counter | Failed warmup count |

### Alert Rules

| Alert | Severity | Condition |
|-------|----------|-----------|
| `QwenAdapterWarmupSlow` | warning | warmup > 300s |
| `QwenAdapterWarmupFailed` | critical | `warmup_failure_total` increments |

---

## Key Files

### `warmup.py`
Core warmup script (stdlib only — no third-party dependencies). Five functions:
- `check_gpu_memory()` — NVML check via subprocess
- `wait_for_ollama(timeout=60)` — polls `http://localhost:11434/api/tags` with backoff
- `pull_model(model_name)` — subprocess `ollama pull`
- `test_generation(model_name)` — HTTP POST to `/api/generate` with minimal prompt
- `main()` — orchestrates all phases, writes `/tmp/warmup.ready` on success, exits non-zero on failure

**Known advisory:** `os.environ["OLLAMA_MODEL"]` in `main()` is outside try/except — raises `KeyError` if var absent. Mitigated by `entrypoint.sh` guard (`set -e; [ -z "$OLLAMA_MODEL" ] && exit 1`).

### `main.py`
FastAPI service. Mounts prometheus metrics ASGI app at `/metrics`. Health endpoints read the ready marker and check Ollama via `requests.get`. Feature label `qwen-adapter` on all metrics.

### `entrypoint.sh`
```bash
set -e
if [ -z "$OLLAMA_MODEL" ]; then echo "[entrypoint] ERROR: OLLAMA_MODEL is not set" >&2; exit 1; fi
ollama serve &
python3 /app/warmup.py
exec python3 /app/main.py
```

### `Dockerfile`
Extends `ollama/ollama:latest`. Installs `python3` and `curl` (curl required for readiness probe exec). Sets `OLLAMA_KEEP_ALIVE=0` (dedicated GPU — disable idle unload). Model cache at `/models` via `ENV OLLAMA_MODELS=/models` and `VOLUME ["/models"]`.

---

## Kubernetes Resources

| Resource | Name | Notes |
|----------|------|-------|
| Deployment | `qwen-adapter` | namespace: `inference` (TODO: confirm) |
| PVC | `qwen-adapter-models` | 25Gi ReadWriteOnce (storageClassName: TODO) |
| Service | `qwen-adapter` | port 8000 |
| ServiceMonitor | `qwen-adapter` | 30s scrape interval |
| PrometheusRule | `qwen-adapter.rules` | WarmupSlow + WarmupFailed |

Resource requests: `cpu: 2, memory: 48Gi, nvidia.com/gpu: 1`  
Resource limits: `cpu: 4, memory: 48Gi, nvidia.com/gpu: 1`

---

## Test Coverage

12 unit tests in `tests/test_warmup.py` — all passing.

| Test | Description |
|------|-------------|
| `test_check_gpu_memory_success` | GPU memory check — subprocess success path |
| `test_check_gpu_memory_failure` | GPU check failure → warning, returns 0 |
| `test_wait_for_ollama_success` | Ollama ready on first poll |
| `test_wait_for_ollama_retry` | Ollama ready after retries |
| `test_wait_for_ollama_timeout` | Timeout → raises RuntimeError |
| `test_pull_model_success` | `ollama pull` success |
| `test_pull_model_failure` | `ollama pull` non-zero exit → raises RuntimeError |
| `test_test_generation_success` | Inference HTTP roundtrip success |
| `test_test_generation_http_error` | HTTP error → raises RuntimeError |
| `test_main_success` | Full main() happy path, writes ready marker |
| `test_main_failure` | main() exception → exits non-zero, no marker |
| `test_main_missing_env_var` | Missing OLLAMA_MODEL → KeyError propagates |

Mocking: `urllib.request.urlopen` (context manager pattern), `subprocess.run`.

---

## Deferred / Known TODOs

| Item | Priority | Notes |
|------|----------|-------|
| storageClassName in pvc.yaml | High (pre-deploy) | Confirm with ops |
| Namespace `inference` exists | High (pre-deploy) | Confirm with ops |
| Alert routing (alertmanager) | Medium | Confirm on-call rotation |
| Grafana dashboard | Low | Post-deployment |
| S9 integration tests executed | High (pre-production) | Requires GPU dev environment |
| main.py health endpoint unit tests | Medium | Future sprint |
| `pull_model()` total stream timeout | Low | Socket-level only in v1 |

---

## Operations

See `docs/production-checklist.md` for the full runbook covering:
- Normal startup procedure
- Warmup failure response
- OOM recovery
- Rolling update procedure
- Rollback steps
- Load test approach
- Sign-off table

See `docs/integration-test-procedure.md` for the 7-test acceptance gate.
