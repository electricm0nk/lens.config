---
feature: terminus-inference-qwen-adapter
story_id: S11
title: Prometheus and Loki Observability Wiring
epic: E2
sprint: 2
points: 5
status: not-started
assignee: null
depends_on: [S3]
blocks: [S9]
updated_at: '2026-04-26T15:30:00Z'
---

# S11 — Prometheus and Loki Observability Wiring

## As a
Platform engineer / SRE

## I want
The Qwen adapter pod wired into the cluster's Prometheus and Loki observability stack

## So that
Warmup duration, GPU memory, and model state metrics are visible in Grafana, log streams are queryable in Loki, and the warmup alert fires correctly

## Context

The cluster runs:
- **Prometheus** via `kube-prometheus-stack` — ServiceMonitor resources in `monitoring` namespace with `release: prometheus` label scrape pods
- **Loki** via Grafana Alloy DaemonSet — auto-collects all pod logs from `/var/log/pods/`; uses `app.kubernetes.io/name` pod label as the `app` Loki stream label
- **Alerts** via `PrometheusRule` resources in `monitoring` namespace with `release: prometheus` label

The tech-plan lists required metrics (startup_warmup_duration_seconds, model_load_time_seconds, gpu_memory_used_bytes, ollama_process_uptime_seconds). This story implements those metrics in the agent, creates the ServiceMonitor, creates the PrometheusRule with the warmup-slow alert, and verifies Loki stream labeling.

## Acceptance Criteria

### 1. Agent Prometheus Metrics Endpoint
- [ ] Agent exposes a `/metrics` endpoint on its primary port (`:8000/metrics`) in Prometheus exposition format
- [ ] The following metrics are emitted:
  | Metric | Type | Description |
  |--------|------|-------------|
  | `qwen_adapter_warmup_duration_seconds` | Gauge | Total warmup duration in seconds (set once on completion) |
  | `qwen_adapter_model_load_time_seconds` | Gauge | Time from pull start to model-loaded state |
  | `qwen_adapter_gpu_memory_used_bytes` | Gauge | GPU VRAM used (from nvidia-smi), updated every 60s |
  | `qwen_adapter_ollama_uptime_seconds` | Counter | Seconds since ollama last started |
  | `qwen_adapter_warmup_success_total` | Counter | Increments by 1 on each successful warmup |
  | `qwen_adapter_warmup_failure_total` | Counter | Increments by 1 on each failed warmup attempt |
- [ ] All metrics have label `feature="qwen-adapter"` for namespace-independent filtering
- [ ] `/metrics` is reachable without authentication

### 2. Service Definition
- [ ] K8s `Service` for the qwen-adapter pod exposes a named port `metrics` mapping to `:8000` (or a separate `:9090/metrics` port if preferred — confirm with team)
- [ ] Service has label `app.kubernetes.io/name: qwen-adapter` matching the pod label

### 3. ServiceMonitor
A `ServiceMonitor` is deployed to the `monitoring` namespace:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: qwen-adapter
  namespace: monitoring
  labels:
    release: prometheus
    terminus.io/domain: terminus
    terminus.io/service: inference
    terminus.io/component: qwen-adapter
spec:
  namespaceSelector:
    matchNames:
      - <qwen-adapter-namespace>
  selector:
    matchLabels:
      app.kubernetes.io/name: qwen-adapter
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```
- [ ] ServiceMonitor has `release: prometheus` label (required for kube-prometheus-stack pickup)
- [ ] Targets appear in Prometheus UI → Status → Targets as `UP`

### 4. PrometheusRule (Alert)
A `PrometheusRule` is deployed to the `monitoring` namespace:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: terminus-qwen-adapter-rules
  namespace: monitoring
  labels:
    release: prometheus
    terminus.io/domain: terminus
    terminus.io/service: inference
spec:
  groups:
    - name: qwen-adapter.warmup
      interval: 1m
      rules:
        - alert: QwenAdapterWarmupSlow
          expr: qwen_adapter_warmup_duration_seconds > 300
          for: 0m
          labels:
            severity: warning
            component: qwen-adapter
          annotations:
            summary: "Qwen adapter warmup took {{ $value | humanizeDuration }}"
            description: "Pod {{ $labels.pod }} warmup exceeded 300s threshold. Check GPU availability and model cache."

        - alert: QwenAdapterWarmupFailed
          expr: increase(qwen_adapter_warmup_failure_total[5m]) > 0
          for: 0m
          labels:
            severity: critical
            component: qwen-adapter
          annotations:
            summary: "Qwen adapter warmup failed on pod {{ $labels.pod }}"
            description: "Warmup failure detected. Pod will not accept traffic. Check logs: kubectl logs {{ $labels.pod }} -n {{ $labels.namespace }}"
```
- [ ] PrometheusRule has `release: prometheus` label
- [ ] Both alert rules appear in Prometheus UI → Alerts

### 5. Loki Log Stream Labeling
- [ ] Qwen adapter pod template has label `app.kubernetes.io/name: qwen-adapter`
- [ ] Alloy DaemonSet picks this up automatically — verify in Loki that log stream `{app="qwen-adapter"}` returns warmup logs
- [ ] Warmup logs use consistent prefix `[warmup]` for easy filtering: `{app="qwen-adapter"} |= "[warmup]"`
- [ ] Agent logs use consistent prefix `[agent]` for separation from warmup logs

### 6. Verification
- [ ] In Prometheus UI: query `qwen_adapter_warmup_duration_seconds` — returns a value after pod warmup
- [ ] In Loki/Grafana: query `{app="qwen-adapter"} |= "[warmup]"` — returns warmup phase logs
- [ ] In Grafana Alerts: `QwenAdapterWarmupSlow` alert is visible in inactive state (healthy pod)
- [ ] `QwenAdapterWarmupFailed` alert is visible in inactive state (healthy pod)

## Technical Notes

**Prometheus client in Python (agent):**
```python
from prometheus_client import Gauge, Counter, start_http_server

WARMUP_DURATION = Gauge('qwen_adapter_warmup_duration_seconds', 'Warmup duration', ['feature'])
WARMUP_SUCCESS = Counter('qwen_adapter_warmup_success_total', 'Warmup successes', ['feature'])
WARMUP_FAILURE = Counter('qwen_adapter_warmup_failure_total', 'Warmup failures', ['feature'])
GPU_MEMORY = Gauge('qwen_adapter_gpu_memory_used_bytes', 'GPU VRAM used', ['feature'])

# In warmup.py main():
start = time.time()
# ... warmup steps ...
WARMUP_DURATION.labels(feature='qwen-adapter').set(time.time() - start)
WARMUP_SUCCESS.labels(feature='qwen-adapter').inc()
```

**If the agent uses a separate metrics port:** expose `:9090` for metrics and `:8000` for the API. Update Service and ServiceMonitor `port` accordingly. Confirm the convention with the team before starting.

## Definition of Done

- [ ] `/metrics` endpoint emitting all 6 metrics
- [ ] Service exposes named `metrics` port
- [ ] ServiceMonitor deployed with `release: prometheus` label — targets show UP in Prometheus
- [ ] PrometheusRule deployed with both alert rules — visible in Prometheus Alerts
- [ ] Pod has `app.kubernetes.io/name: qwen-adapter` label
- [ ] Loki stream `{app="qwen-adapter"}` queryable with warmup logs
- [ ] Verified in dev as part of S9 integration test (S9 depends on S11)
- [ ] Code reviewed and merged to feature branch
