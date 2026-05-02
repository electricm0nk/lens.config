---
feature: terminus-inference-qwen-adapter
story_id: S10
title: Production Checklist
epic: E3
sprint: 3
points: 3
status: not-started
assignee: null
depends_on: [S9]
blocks: []
updated_at: '2026-04-26T14:55:00Z'
---

# S10 — Production Checklist

## As a
Platform engineer / tech lead

## I want
A documented production readiness checklist completed and signed off

## So that
The feature is safe to promote to production with a clear runbook, rollback plan, and monitoring in place

## Context

Integration test (S9) validates functionality. This story validates operational readiness: is the team prepared to run this in production, handle failures, and recover quickly?

## Acceptance Criteria

### 1. Resource Limits
- [ ] K8s deployment manifest has `resources.limits` and `resources.requests` set:
  - `nvidia.com/gpu: 1`
  - `memory: 48Gi` (limit); `memory: 48Gi` (request)
  - `cpu: 4` (limit); `cpu: 2` (request)
- [ ] `nvidia.com/gpu: 1` is set in both requests and limits (GPU must be requested explicitly)

### 2. Monitoring
- [ ] `startup_warmup_duration_seconds` metric is confirmed flowing in production Prometheus
- [ ] `QwenAdapterWarmupSlow` alert rule is deployed to production alert manager (threshold: >300s)
- [ ] Grafana dashboard panel exists for warmup duration and model uptime
- [ ] Alert routing is configured: who gets paged if warmup alert fires?

### 3. Runbook
A runbook document is created in the team wiki/docs covering:
- [ ] **Normal startup behavior:** expected log sequence, timing benchmarks
- [ ] **Warmup failure:** symptoms, how to diagnose (check logs, check GPU), how to recover (restart pod, check VRAM availability)
- [ ] **OOM / GPU memory exhaustion:** how to detect (nvidia-smi), how to recover (delete pod, check node capacity)
- [ ] **Model eviction after warmup:** how to detect (liveness probe restarts pod), expected behavior (warm restart <60s)
- [ ] **Rolling update:** how to deploy a new image version with minimal downtime (readiness probe gates traffic)
- [ ] **Emergency rollback:** steps to roll back to the previous image version

### 4. Rollback Plan
- [ ] Previous Qwen adapter image tag is documented
- [ ] `kubectl rollout undo deployment/<name>` command verified to work in dev
- [ ] After rollback, prior behavior (JIT cold-start) is acceptable as temporary fallback

### 5. Load Test
- [ ] Simulated load test: 10 concurrent requests sent to the pod after warmup
- [ ] All 10 requests complete without error
- [ ] No memory pressure or OOM events during load test
- [ ] GPU memory usage remains stable under load (no growth leak)

## Definition of Done

- [ ] All checklist items above signed off by tech lead
- [ ] Runbook published to team wiki
- [ ] Alert routing confirmed with on-call rotation
- [ ] Load test results documented
- [ ] Feature is cleared for production promotion
