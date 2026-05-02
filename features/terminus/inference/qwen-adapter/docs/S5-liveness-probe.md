---
feature: terminus-inference-qwen-adapter
story_id: S5
title: Liveness Probe
epic: E2
sprint: 2
points: 3
status: not-started
assignee: null
depends_on: [S3]
blocks: [S9]
updated_at: '2026-04-26T14:55:00Z'
---

# S5 — Liveness Probe

## As a
Platform engineer

## I want
A K8s liveness probe that calls the ollama `/api/tags` endpoint to detect ollama process crashes

## So that
If ollama dies after warmup, the pod is automatically restarted to restore service

## Context

The readiness probe (S4) gates initial traffic routing. The liveness probe is a separate concern: it detects post-warmup failures. If ollama crashes or hangs while the agent is receiving traffic, the liveness probe ensures the pod is recycled rather than staying live in a broken state.

The `initialDelaySeconds` must be long enough to not fire during warmup — set to 30s, which is before warmup completes, but that's acceptable because liveness failures require multiple consecutive failures.

## Acceptance Criteria

1. K8s deployment manifest includes a `livenessProbe` on the agent container
2. Probe type: `httpGet` with `path: /api/tags` and `port: 11434` (ollama's port)
3. Probe timing: `initialDelaySeconds: 30`, `periodSeconds: 10`, `failureThreshold: 3`
4. Probe causes pod restart only after 3 consecutive failures (30s window)
5. Probe does NOT fire during normal warmup (warmup completes in <300s; probe allows 30s before starting)
6. Pod is restarted if ollama crashes or stops responding after warmup

## Technical Notes

```yaml
livenessProbe:
  httpGet:
    path: /api/tags
    port: 11434
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Why `/api/tags` not the agent's `/health`?** The liveness concern is ollama, not the agent. If ollama is down, inference is impossible. The agent `/health` endpoint is handled by the readiness probe logic.

**Trade-off:** The 30s initialDelay means during warmup the probe could temporarily fail, but with `failureThreshold: 3` and the pod in NotReady state (S4 controls that), restarts during warmup won't be triggered unless ollama crashes.

## Definition of Done

- [ ] K8s deployment manifest updated with liveness probe
- [ ] Probe timing configured as specified (30/10/3)
- [ ] Verified in dev: probe does not interfere with normal startup
- [ ] Verified in dev (manually): killing ollama process triggers pod restart within ~30s
- [ ] Code reviewed and merged to feature branch
