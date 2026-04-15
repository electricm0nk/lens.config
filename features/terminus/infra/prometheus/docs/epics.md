---
feature: prometheus
doc_type: epics
status: draft
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/infra/prometheus/prd.md
  - docs/terminus/infra/prometheus/ux-design.md
  - docs/terminus/infra/prometheus/architecture.md
updated_at: "2026-04-15T01:15:00Z"
---

# Prometheus Monitoring — Epic Breakdown

## Overview

This document decomposes the Prometheus Monitoring feature into epics and stories. The deployment is GitOps-only (ArgoCD/Helm), targets the `monitoring` namespace (prod, no dev environment), and integrates with an existing Grafana instance via pre-wired datasource. Stories are sized for single-agent completion.

---

## Requirements Inventory

### Functional Requirements

```
FR-1.1: Prometheus deployed to `monitoring` namespace via ArgoCD/Helm, status Healthy/Synced
FR-1.2: (Deferred — dev environment out of scope for this feature)
FR-1.3: Scrape node-exporter metrics from all k3s nodes (DaemonSet)
FR-1.4: Scrape kube-state-metrics (pod status, resource requests, restart counts)
FR-1.5: Scrape k3s control-plane metrics (API server latency via kubelet ServiceMonitor)
FR-2.1: Grafana datasource provisioned (URL already in grafana/values.yaml — activates on deploy)
FR-2.2: Prometheus datasource isDefault: false (InfluxDB remains default)
FR-2.3: Grafana datasource "Save & Test" passes
FR-3.1: Operator/built-in kube dashboards deployed (forceDeployDashboards)
FR-3.2: "k3s Infrastructure Health" dashboard rendered with live data
FR-3.3: "Workload Health" dashboard rendered with live data
FR-3.4: $namespace variable filter on Workload Health dashboard
FR-4.1: 15d TSDB retention configured
FR-4.2: 15s scrape interval configured
FR-4.3: Resource requests/limits set on Prometheus pod
FR-5.1: No credentials in git; Vault/ESO pattern for any future secrets
FR-5.2: Alertmanager deployed (no routing rules for MVP)
```

### Non-Functional Requirements

```
NFR-PERF-1: Scrape interval 15s (default) — tuning deferred
NFR-PERF-2: Retention 15d; long-term storage deferred to InfluxDB
NFR-REL-1: ArgoCD selfHeal: true — auto-recovers from drift
NFR-SEC-1: No secrets in git
NFR-SEC-2: In-cluster scrape via service account bearer token — no external credentials
NFR-MAINT-1: All config changes via PR; no kubectl apply or helm upgrade in prod
```

### Additional Requirements (from Architecture)

```
ARCH-1: Chart: kube-prometheus-stack (releaseName: prometheus), targetRevision: "67.*"
ARCH-2: k3s compatibility: disable kubeControllerManager, kubeScheduler, kubeProxy, kubeEtcd
ARCH-3: Kubelet TLS: insecureSkipVerify: true, cAdvisor: true
ARCH-4: forceDeployDashboards: true in values.yaml
ARCH-5: Dashboard ConfigMaps carry label grafana_dashboard: "1", annotation grafana_folder: "Terminus Infra"
ARCH-6: No namespace.yaml in prometheus-infra (monitoring namespace owned by influxdb-infra)
ARCH-7: ArgoCD app labels: terminus.io/component: prometheus
ARCH-8: Multi-source ArgoCD app pattern (chart source + values ref from repo)
ARCH-9: emptyDir storage for MVP
```

### UX Design Requirements

```
UX-1: "k3s Infrastructure Health" dashboard — 6 panels: Node CPU, Node Memory, Pod Restarts, k3s API Latency, PV Capacity, Node Network I/O
UX-2: Layout: 3-2-2 panel grid; filters: $node, $namespace, time range default 3h
UX-3: "Workload Health" dashboard — 5 panels: CPU Utilization, Memory Utilization, Pod Restarts, HTTP Error Rate, Container Status
UX-4: Layout: 2-2-1 panel grid; filter: $namespace; time range default 1h
UX-5: Grafana folder: "Terminus Infra"
UX-6: Read-only dashboard access for developers (no Grafana edit permission needed)
```

---

## FR Coverage Map

| Epic | Stories | FRs Covered |
|------|---------|-------------|
| Epic 1: Stack Deployment | 1-1, 1-2, 1-3 | FR-1.1, FR-1.3, FR-1.4, FR-1.5, FR-4.1, FR-4.2, FR-4.3, FR-5.2, NFR-REL-1, NFR-MAINT-1, ARCH-1–9 |
| Epic 2: Grafana Integration | 2-1, 2-2 | FR-2.1, FR-2.2, FR-2.3, FR-3.1–3.4, FR-5.1, UX-1–6 |

---

## Epic 1: Prometheus Stack Deployment

**User Outcome:** Platform engineers can deploy and verify Prometheus scraping all k3s targets via ArgoCD, with no manual cluster operations.

**Stories:**

---

### Story 1-1: ArgoCD apps and Helm values for kube-prometheus-stack

**Type:** Infrastructure / GitOps  
**Estimate:** M

**As a** platform engineer,  
**I want** the kube-prometheus-stack deployed to the `monitoring` namespace via ArgoCD (multi-source pattern),  
**so that** Prometheus is lifecycle-managed by ArgoCD with no manual Helm operations.

#### Acceptance Criteria

- [ ] `platforms/k3s/argocd/apps/prometheus-infra.yaml` created — wave 1, source: `platforms/k3s/k8s/prometheus-infra`, destination: `monitoring`, no `CreateNamespace` (namespace pre-exists via influxdb-infra)
- [ ] `platforms/k3s/argocd/apps/prometheus.yaml` created — wave 2, multi-source: `prometheus-community/kube-prometheus-stack` chart `targetRevision: "67.*"` + values ref from `terminus.infra` repo; `releaseName: prometheus`; `syncPolicy.automated.prune: true, selfHeal: true`; `CreateNamespace=false`
- [ ] App labels include `terminus.infra/component: prometheus`, `terminus.io/environment: prod`, `app.kubernetes.io/part-of: terminus-infra-k3s`
- [ ] `platforms/k3s/helm/prometheus/values.yaml` created with:
  - `grafana.enabled: false`, `grafana.forceDeployDashboards: true`, sidecar label `grafana_dashboard: "1"`, namespace `monitoring`
  - k3s disabled components: `kubeControllerManager.enabled: false`, `kubeScheduler.enabled: false`, `kubeProxy.enabled: false`, `kubeEtcd.enabled: false`
  - `kubelet.serviceMonitor.https: true`, `insecureSkipVerify: true`, `cAdvisor: true`
  - `prometheus.prometheusSpec.retention: 15d`, `scrapeInterval: 15s`, `storageSpec: {}` (emptyDir)
  - Resource requests: `cpu: 200m, memory: 512Mi`; limits: `cpu: 500m, memory: 1Gi`
  - `alertmanager.enabled: true`, resource requests: `cpu: 50m, memory: 64Mi`
- [ ] PR to `terminus.infra` passes `helm template` lint (no rendering errors for the chart + values combination)

#### Implementation Notes

- Mirror `influxdb.yaml` / `influxdb-infra.yaml` structure exactly
- Do NOT create `namespace.yaml` in `prometheus-infra` — namespace is owned by `influxdb-infra`
- Pre-deploy verification: confirm `influxdb-infra` ArgoCD app is Healthy and `monitoring` namespace exists

---

### Story 1-2: ArgoCD sync and target verification

**Type:** Infrastructure / Validation  
**Estimate:** S

**As a** platform engineer,  
**I want** to verify Prometheus is synced and scraping all expected targets,  
**so that** I have confirmed observability coverage of the k3s cluster before wiring dashboards.

#### Acceptance Criteria

- [ ] ArgoCD reports `prometheus-infra` and `prometheus` apps as `Healthy/Synced`
- [ ] All Prometheus pods in `monitoring` namespace are `Running`: prometheus-server, alertmanager, node-exporter (all nodes), kube-state-metrics, prometheus-operator
- [ ] `kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090 -n monitoring` → `/targets` page shows all scraped targets with status `up`; no persistent `down` targets
- [ ] node-exporter targets: one per k3s node (3 targets if 3 nodes)
- [ ] kube-state-metrics target: 1 target, `up`
- [ ] kubelet target: all nodes, `up`
- [ ] k3s API server target: `up`
- [ ] Disabled targets (`kubeControllerManager`, `kubeScheduler`, `kubeProxy`, `kubeEtcd`) not present in target list

#### Implementation Notes

- Run post-deploy validation playbook from architecture section 9.5
- If node-exporter DaemonSet pods are not scheduling, check node tolerations — toleration `operator: Exists` should handle control-plane nodes

---

### Story 1-3: Grafana datasource activation verification

**Type:** Infrastructure / Validation  
**Estimate:** XS

**As a** platform engineer,  
**I want** to verify the Prometheus datasource in Grafana resolves after deployment,  
**so that** dashboard queries will run without additional configuration.

#### Acceptance Criteria

- [ ] Grafana UI → Connections → Data Sources → Prometheus → "Save & Test" returns green (no changes to Grafana Helm values required — datasource URL pre-wired)
- [ ] Datasource `isDefault: false` confirmed (InfluxDB remains default)
- [ ] PromQL test query `up` returns results with `job="prometheus"` label present

#### Implementation Notes

- No changes to `grafana/values.yaml` or `grafana` ArgoCD app required
- Datasource URL `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090` resolves once the prometheus service is running

---

## Epic 2: Grafana Dashboard Integration

**User Outcome:** Platform engineers and developers can open Grafana and see live k3s infrastructure and workload metrics without any manual Grafana configuration.

**Stories:**

---

### Story 2-1: k3s Infrastructure Health dashboard

**Type:** Infrastructure / GitOps  
**Estimate:** M

**As a** platform engineer,  
**I want** the "k3s Infrastructure Health" Grafana dashboard deployed via ConfigMap,  
**so that** I can monitor node health, pod restarts, and k3s API latency from Grafana without manual dashboard import.

#### Acceptance Criteria

- [ ] `platforms/k3s/k8s/prometheus-infra/k3s-infrastructure-health-dashboard.yaml` created — ConfigMap in `monitoring` namespace, label `grafana_dashboard: "1"`, annotation `grafana_folder: "Terminus Infra"`, data key `k3s-infrastructure-health.json`
- [ ] Dashboard JSON implements all 6 UX panels using specified metric sources (see UX-1, UX-2):
  - Node CPU: `rate(node_cpu_seconds_total{mode!="idle"}[5m])` stacked by node, 80% threshold line
  - Node Memory: `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` gauge per node, red threshold at 90%
  - Pod Restarts (24h): `increase(kube_pod_container_status_restarts_total[24h])` table sorted desc
  - k3s API Latency: `histogram_quantile(0.50|0.99, rate(apiserver_request_duration_seconds_bucket[5m]))` P50/P99 line
  - PV Capacity: `kubelet_volume_stats_capacity_bytes` by PV name
  - Node Network I/O: `rate(node_network_transmit_bytes_total[5m])` and `rate(node_network_receive_bytes_total[5m])` per interface
- [ ] Dashboard variables: `$node` (multi-select, `label_values(node_cpu_seconds_total, instance)`), `$namespace` (multi-select, `label_values(kube_pod_info, namespace)`), time range default 3h
- [ ] ConfigMap applied to cluster → dashboard appears in Grafana under "Terminus Infra" folder without Grafana restart
- [ ] Live data renders in all panels (no "No data" state on any panel when queried against a running cluster)
- [ ] Community dashboard 1860 (Node Exporter Full) used as reference for node metrics PromQL; custom panel wiring per UX spec takes precedence

#### Implementation Notes

- OQ-3 resolved: PromQL sourced from UX spec + community dashboard 1860 reference
- If `forceDeployDashboards` emits a built-in node-exporter dashboard that conflicts, use a distinct `uid` for this custom dashboard
- Verify ConfigMap appears in Grafana dynamically (sidecar in-container watch, no pod restart needed)

---

### Story 2-2: Workload Health dashboard

**Type:** Infrastructure / GitOps  
**Estimate:** M

**As a** developer,  
**I want** the "Workload Health" Grafana dashboard deployed via ConfigMap with a `$namespace` filter,  
**so that** I can monitor CPU, memory, and error rates for my pods without needing cluster access.

#### Acceptance Criteria

- [ ] `platforms/k3s/k8s/prometheus-infra/workload-health-dashboard.yaml` created — ConfigMap in `monitoring` namespace, label `grafana_dashboard: "1"`, annotation `grafana_folder: "Terminus Infra"`, data key `workload-health.json`
- [ ] Dashboard JSON implements all 5 UX panels (see UX-3, UX-4):
  - CPU Utilization: `rate(container_cpu_usage_seconds_total{namespace="$namespace", container!=""}[5m])` by pod
  - Memory Utilization: `container_memory_working_set_bytes{namespace="$namespace", container!=""}` by pod
  - Pod Restarts: `increase(kube_pod_container_status_restarts_total{namespace="$namespace"}[1h])` by pod
  - HTTP Error Rate: `rate(http_requests_total{namespace="$namespace", status=~"5.."}[5m])` — graceful no-data if metric absent
  - Container Status: `kube_pod_container_status_running{namespace="$namespace"}` table
- [ ] Dashboard variable: `$namespace` (single-select, `label_values(kube_pod_info, namespace)`, default: `fourdogs`)
- [ ] Time range default: 1h
- [ ] ConfigMap applied → dashboard appears in Grafana under "Terminus Infra" folder
- [ ] `$namespace=fourdogs` filter shows only fourdogs pods; switching namespace updates all panels
- [ ] HTTP Error Rate panel handles no-data gracefully (shows empty state, not error)

#### Implementation Notes

- Container metrics come from cAdvisor via kubelet scrape (`cAdvisor: true` in values.yaml)
- HTTP Error Rate requires workloads to expose `http_requests_total` with `status` label — if not present, panel shows empty state (expected behavior)
- Default namespace `fourdogs` per user journey 3 in the PRD
