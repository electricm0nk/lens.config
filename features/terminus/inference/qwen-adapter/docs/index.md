# terminus-inference-qwen-warmup — Project Index

**Generated:** 2026-04-26  
**Feature:** qwen-adapter  
**Repository:** `electricm0nk/terminus-inference-qwen-warmup`  
**Domain/Service:** terminus/inference  
**Branch:** `main` (squash-merged from `feature/qwen-warmup` — PR #1)

---

## Purpose

GPU-dedicated container that bootstraps the Qwen 2.5:14b model via Ollama before serving inference requests. Implements the warmup container pattern: ollama server → model pull + test inference → FastAPI agent service.

---

## Project Type

**Backend service / infrastructure container** — Python, FastAPI, Ollama, Docker, Kubernetes

---

## Source Tree

```
terminus-inference-qwen-warmup/
├── warmup.py                          # GPU/model warmup script (stdlib only)
├── main.py                            # FastAPI agent service with health + Prometheus metrics
├── entrypoint.sh                      # Three-phase container startup script
├── Dockerfile                         # Container image definition
├── requirements.txt                   # Runtime deps: fastapi, uvicorn, requests, prometheus-client
├── requirements-dev.txt               # Dev deps: pytest, pytest-cov
├── tests/
│   └── test_warmup.py                 # 12 unit tests for warmup.py (all passing)
├── deploy/
│   ├── deployment.yaml                # K8s Deployment manifest
│   ├── pvc.yaml                       # PersistentVolumeClaim (25Gi) for model cache
│   ├── service.yaml                   # K8s Service (port 8000)
│   └── monitoring/
│       ├── service-monitor.yaml       # Prometheus ServiceMonitor
│       └── prometheus-rule.yaml       # PrometheusRules: WarmupSlow + WarmupFailed alerts
└── docs/
    ├── integration-test-procedure.md  # 7-test acceptance gate
    └── production-checklist.md        # Resource limits, runbook, sign-off table
```

---

## Documentation

| Document | Path | Description |
|----------|------|-------------|
| Project Overview | [project-overview.md](./project-overview.md) | Architecture, design decisions, operations |
| Integration Test Procedure | *(in repo)* `docs/integration-test-procedure.md` | 7-test acceptance gate |
| Production Checklist | *(in repo)* `docs/production-checklist.md` | Resource limits, runbook, sign-off |

---

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Warmup language | Python stdlib only | No extra container layer; `urllib.request` + `subprocess` sufficient |
| Readiness probe | exec: file marker + curl | Two-gate: warmup completion AND FastAPI serving |
| Liveness probe | httpGet /api/tags (ollama) | Direct model server heartbeat |
| KEEP_ALIVE | 0 (disabled) | Dedicated GPU container; Ollama must not unload the model |
| Model cache | PVC at `/models` | Survives pod restarts; avoids re-pull on warm restart |
| Metrics | prometheus-client 0.21.0 | Consistent with Terminus prometheus stack |
