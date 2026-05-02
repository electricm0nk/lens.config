---
feature: terminus-inference-qwen-adapter
story_id: S6
title: Health Endpoints
epic: E2
sprint: 2
points: 5
status: not-started
assignee: null
depends_on: [S3]
blocks: [S4, S9]
updated_at: '2026-04-26T14:55:00Z'
---

# S6 — Health Endpoints

## As a
Agent developer

## I want
`/health` and `/health/ready` endpoints in the agent service

## So that
Kubernetes can probe the agent's liveness and readiness state accurately

## Context

The K8s probes (S4, S5) need agent-side endpoints to query. S5 probes ollama directly on :11434, but S4 needs a readiness endpoint on the agent's port (:8000) that checks whether warmup has completed. This story implements both endpoints.

Check if the agent already has a `/health` stub — if so, extend it rather than creating from scratch.

## Acceptance Criteria

1. `GET /health` returns `HTTP 200` with body `{"status": "alive"}` at all times (including during warmup) — this is the liveness-style check
2. `GET /health/ready` returns `HTTP 200` with body `{"status": "ready", "model": "<model_tag>"}` when:
   - `/tmp/warmup.ready` file exists, AND
   - The model listed in `OLLAMA_MODEL` is present in ollama's `/api/tags` response
3. `GET /health/ready` returns `HTTP 503` with body `{"status": "not-ready", "reason": "<reason>"}` when warmup is not complete
4. `reason` field values: `"warmup-in-progress"` (marker file absent), `"model-not-loaded"` (file present but model not in ollama), `"ollama-unavailable"` (cannot reach ollama)
5. `/health/ready` never returns 200 unless the model is confirmed present in ollama (not just the marker file)
6. Both endpoints are accessible on the agent's primary port (`:8000` or whatever the agent uses — confirm with team)
7. Response times for both endpoints are <100ms

## Technical Notes

```python
import os, requests
from pathlib import Path

@app.get("/health")
def health():
    return {"status": "alive"}

@app.get("/health/ready")
def health_ready():
    warmup_ready = Path("/tmp/warmup.ready").exists()
    if not warmup_ready:
        return Response(status_code=503, content={"status": "not-ready", "reason": "warmup-in-progress"})
    
    model = os.environ.get("OLLAMA_MODEL", "")
    try:
        tags = requests.get("http://localhost:11434/api/tags", timeout=2).json()
        loaded = any(m["name"].startswith(model.split(":")[0]) for m in tags.get("models", []))
    except Exception:
        return Response(status_code=503, content={"status": "not-ready", "reason": "ollama-unavailable"})
    
    if not loaded:
        return Response(status_code=503, content={"status": "not-ready", "reason": "model-not-loaded"})
    
    return {"status": "ready", "model": model}
```

## Definition of Done

- [ ] `/health` returns 200+alive at all times
- [ ] `/health/ready` returns 503 during warmup
- [ ] `/health/ready` returns 200 only after warmup.ready exists AND model in ollama
- [ ] `/health/ready` distinguishes warmup-in-progress vs model-not-loaded vs ollama-unavailable
- [ ] Both endpoints respond in <100ms
- [ ] Code reviewed and merged to feature branch
