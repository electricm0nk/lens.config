---
feature: terminus-inference-qwen-adapter
story_id: S9
title: Integration Test
epic: E3
sprint: 3
points: 8
status: not-started
assignee: null
depends_on: [S4, S5, S6, S7, S8, S11]
blocks: [S10]
updated_at: '2026-04-26T14:55:00Z'
---

# S9 — Integration Test

## As a
QA engineer / platform engineer

## I want
End-to-end verification of the warmup lifecycle deployed in the dev environment

## So that
We have confirmed evidence that warmup completes in <5 min, first requests are fast, model persists, and Prometheus metrics flow

## Context

This story is the primary acceptance gate for the entire feature. It deploys the updated Qwen adapter to dev and verifies all success criteria from the business-plan hold in a real environment.

**Measurement point:** First-request latency is measured at the **pod boundary** (inside k8s, at the service endpoint — not the user-facing API gateway). This separates inference latency from network/gateway latency.

## Acceptance Criteria

### 1. Pod Startup and Warmup
- [ ] Deploy updated Qwen adapter to dev environment
- [ ] Pod starts in NotReady state
- [ ] Warmup completes and pod transitions to Ready within **5 minutes** of container start
- [ ] Container logs show all three startup phases: ollama start, warmup, agent start
- [ ] `/tmp/warmup.ready` marker present in running pod

### 2. First Request Latency
- [ ] Send first inference request to the pod via the **service endpoint** (inside k8s, bypassing API gateway)
- [ ] First request completes in **<2 seconds** (measured pod-boundary to pod-boundary)
- [ ] Response is valid (non-empty, coherent text output)
- [ ] Measurement is documented in the PR: timestamp of request, response time, measuring tool/method

### 3. Model Persistence
- [ ] Wait 10 minutes with no requests
- [ ] Send another inference request — response time is still <2 seconds
- [ ] `ollama /api/tags` confirms model still loaded (not unloaded/evicted)
- [ ] Wait 30 minutes — repeat check; model still loaded

### 4. Warm Restart
- [ ] Delete the running pod (simulates crash/restart)
- [ ] Pod restarts and reaches Ready state within **60 seconds** (model already cached in PVC)
- [ ] First request after warm restart completes in <2 seconds

### 5. Prometheus Metrics
- [ ] Confirm `startup_warmup_duration_seconds` metric is emitted after warmup
- [ ] Confirm `model_load_time_seconds` is visible in Prometheus/Grafana
- [ ] Confirm `ollama_process_uptime_seconds` is increasing over time

### 6. Alert Rule
- [ ] Prometheus alert rule `QwenAdapterWarmupSlow` is active: fires when `startup_warmup_duration_seconds > 300`
- [ ] Rule is defined in the cluster's alerting configuration
- [ ] Test the alert fires by simulating a slow warmup (or injecting a fake metric value)

### 7. Probe Behavior
- [ ] Liveness probe does NOT kill the pod during normal warmup
- [ ] Simulate liveness failure (kill ollama): pod restarts within ~30s
- [ ] After restart (warm), pod Ready again within 60s

## Technical Notes

**Measuring pod-boundary latency:**
```bash
# From within the cluster (or via kubectl exec into a test pod)
time curl -X POST http://qwen-adapter-svc:8000/inference \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Hello, how are you?", "max_tokens": 50}'
```

**Check warmup duration from logs:**
```bash
kubectl logs <pod-name> | grep "\[warmup\]" | head -20
```

**Check model persistence:**
```bash
kubectl exec <pod-name> -- curl -s http://localhost:11434/api/tags | jq '.models[].name'
```

## Definition of Done

- [ ] All 7 acceptance criterion sections verified and documented
- [ ] Test results recorded in PR description (warmup time, latency measurements, metric screenshots)
- [ ] Alert rule deployed and tested
- [ ] No regressions on existing inference behavior
- [ ] Code reviewed and merged to feature branch
