---
feature: terminus-inference-qwen-adapter
doc_type: epics
status: approved
goal: "Eliminate JIT cold-start by preloading Qwen into GPU memory and persisting indefinitely"
key_decisions:
  - Warmup script runs at container init, not as init-container
  - OLLAMA_KEEP_ALIVE=0 hard-coded in Dockerfile (not runtime-configurable)
  - PVC for /models directory to survive pod restarts
  - K8s readiness probe gates traffic until warmup completes
open_questions: []
depends_on: []
blocks: []
updated_at: '2026-04-26T14:55:00Z'
---

# Epics — Qwen Adapter Container Warmup

## Epic Summary

| Epic | Title | Stories | Points | Sprint |
|------|-------|---------|--------|--------|
| E1 | Warmup Core | S1, S2, S3 | 18 pts | Sprint 1 |
| E2 | Kubernetes Integration | S4, S5, S6, S7, S8, S11 | 26 pts | Sprint 2 |
| E3 | Validation & Production | S9, S10 | 11 pts | Sprint 3 |

**Total:** 11 stories, 55 points, 3 sprints (~3 weeks)

---

## Epic E1 — Warmup Core

**Goal:** Implement the warmup script and Dockerfile configuration that preloads Qwen into GPU memory on container startup.

**Scope:**
- Python warmup script (`warmup.py`) that waits for ollama, pulls the model, runs a test query, writes the readiness marker
- Ollama persistence configuration (`OLLAMA_KEEP_ALIVE=0`, `OLLAMA_MODELS=/models`)
- Container entrypoint wiring: `ollama serve` → warmup → agent start

**Success Criteria:**
- Container starts and automatically loads Qwen into VRAM within 5 minutes
- `/tmp/warmup.ready` marker file written on successful warmup
- Model persists indefinitely with no idle shutdown

**Dependencies:**
- GPU-capable node with sufficient VRAM (≥24GB for Qwen 3.6)
- Ollama base image running in container
- Qwen model tag confirmed (`OLLAMA_MODEL` env var)

**Stories:**

| Story | Title | Points |
|-------|-------|--------|
| S1 | Warmup Script | 8 pts |
| S2 | Ollama Persistence Config | 5 pts |
| S3 | Container Entrypoint | 5 pts |

---

## Epic E2 — Kubernetes Integration

**Goal:** Wire the warmup lifecycle into Kubernetes via readiness/liveness probes, health endpoints, and a model cache PVC.

**Scope:**
- K8s readiness probe: checks `/tmp/warmup.ready` + `/health/ready` endpoint
- K8s liveness probe: checks `ollama /api/tags` to detect crashes
- Health endpoints in agent: `/health` (liveness) and `/health/ready` (warmup-gated)
- PVC for `/models` directory (25GB ReadWriteOnce) to persist model cache across pod restarts
- Prometheus metrics endpoint (`/metrics`), ServiceMonitor, PrometheusRule with warmup alerts
- Loki log stream labeling via `app.kubernetes.io/name: qwen-adapter` pod label
- Unit tests for warmup.py using pytest + mocks

**Success Criteria:**
- Pod marked Ready only after warmup completes
- Liveness probe triggers pod restart on ollama crash
- `/health/ready` returns 503 before warmup, 200 after
- Model cache survives pod restart; warm restart takes <30s

**Dependencies:** Epic E1 complete (warmup script and entrypoint wiring)

**Stories:**

| Story | Title | Points |
|-------|-------|--------|
| S4 | Readiness Probe | 5 pts |
| S5 | Liveness Probe | 3 pts |
| S6 | Health Endpoints | 5 pts |
| S7 | Model Cache Volume | 3 pts |
| S8 | Unit Tests | 5 pts |
| S11 | Prometheus and Loki Observability Wiring | 5 pts |

---

## Epic E3 — Validation & Production

**Goal:** Verify end-to-end warmup behavior in the dev environment, confirm production-readiness, and document operations.

**Scope:**
- Integration test: deploy to dev, verify warmup completion, first-request latency, model persistence over time, Prometheus metrics flow
- Production checklist: resource limits, monitoring alert on warmup duration > 300s, runbook, rollback plan, load test

**Success Criteria:**
- Warmup completes in <5 min in dev environment
- First request latency (pod-boundary) <2 sec post-warmup
- Model remains resident for 60+ minutes without idle shutdown
- Prometheus metrics flowing; alert rule active
- Runbook covers startup failure, OOM, model eviction

**Dependencies:** Epic E2 complete (K8s integration)

**Stories:**

| Story | Title | Points |
|-------|-------|--------|
| S9 | Integration Test | 8 pts |
| S10 | Production Checklist | 3 pts |

---

## Dependency Graph

```
S1 (Warmup Script) ──────┐
                          ▼
S2 (Ollama Config) ──► S3 (Entrypoint) ──► S4 (Readiness Probe) ──┐
                                       └──► S5 (Liveness Probe)    │
                                       └──► S6 (Health Endpoints)  ├──► S9 (Integration Test) ──► S10 (Prod Checklist)
                                       └──► S7 (Cache Volume) ─────┘
                                       └──► S8 (Unit Tests) ───────┘
                                       └──► S11 (Observability) ───┘
```

---

## Risk Register

| Risk | Mitigation | Owner |
|------|-----------|-------|
| OOM at warmup | Check VRAM ≥24GB in warmup.py; fail-fast with clear error | Dev (S1) |
| Slow warmup (>5 min) | Generous probe timeout (360s window); alert on >300s | Dev (S4, S9) |
| Model name drift | `OLLAMA_MODEL` env var; validate in warmup | Dev (S2) |
| Model eviction after warmup | Liveness probe detects; log monitoring; document recovery | Dev (S5, S10) |
| Warm restart cache miss | PVC mounted at /models; verify in S9 | Dev (S7, S9) |
