---
feature: prometheus
doc_type: ux-design
status: draft
goal: "Provide Kubernetes-native metric scraping for k3s with Grafana dashboards for both ops and developer consumption"
key_decisions:
  - Prometheus is the scraping/storage layer; Grafana is the visualization layer — not replacing InfluxDB
  - GitOps-managed via ArgoCD/Helm — no manual cluster ops
  - Two dashboard tiers: platform-ops (infrastructure focus) and developer-facing (workload focus)
open_questions:
  - Alertmanager routing targets (email? Slack? webhook into Semaphore?) not yet decided
depends_on: []
blocks: []
updated_at: "2026-04-14T23:30:00Z"
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/infra/terminus-infra-ai-substrate-service-plan.md
  - docs/terminus/infra/influxdb-monitoring/architecture.md
---

# UX Design Specification — Prometheus Monitoring

**Author:** Todd Hintzmann
**Date:** 2026-04-14
**Feature:** `prometheus` | Domain: `terminus` | Service: `infra`

---

## 1. Overview

Prometheus complements the existing InfluxDB monitoring stack by providing **Kubernetes-native metric scraping** — targeting pods, nodes, k3s control-plane components, and namespace-level workload health. InfluxDB remains the store for infrastructure time-series data pushed from apps; Prometheus owns the "scrape and query" layer for Kubernetes primitives.

Grafana becomes the unified visualization surface for **both** data sources.

---

## 2. Users & Goals

| User | Context | Primary Goal |
|------|---------|-------------|
| Platform engineer (ops) | Manages the cluster, deploys infra | Deploy and operate the Prometheus stack via GitOps; see node/pod/k3s health at a glance |
| Developer | Ships services to `fourdogs` / `terminus-platform` namespaces | Query their workload's CPU, memory, error rates without needing cluster access |

---

## 3. UX Surfaces

Only Grafana dashboards are in scope for this feature. Prometheus UI and Alertmanager routing are noted as out-of-scope for the initial release.

### 3.1 Operator Dashboard — "k3s Infrastructure Health"

**Audience:** Platform engineers  
**Access:** Grafana (same instance used by InfluxDB datasource)

#### Panels

| Panel | Metric Source | Description |
|-------|-------------|-------------|
| Node CPU (%) — all nodes | `node_cpu_seconds_total` | Stacked bar per node; threshold line at 80% |
| Node Memory (%) — all nodes | `node_memory_MemAvailable_bytes` | Gauge per node; red at <10% free |
| Pod Restarts (24h) — all namespaces | `kube_pod_container_status_restarts_total` | Table sorted by restart count descending |
| k3s API Server Latency | `apiserver_request_duration_seconds` | P50 / P99 line chart |
| Persistent Volume Capacity | `kubelet_volume_stats_capacity_bytes` | Heatmap by PV name |
| Node Network I/O | `node_network_transmit_bytes_total` / `node_network_receive_bytes_total` | Line chart per node interface |

#### Layout

```
[ Node CPU ]  [ Node Memory ]  [ Pod Restarts ]
[        k3s API Latency             ]
[ PV Capacity ]     [ Node Network I/O ]
```

#### Filters / Variables

- `$node` — multi-select dropdown (all nodes)
- `$namespace` — multi-select dropdown (all namespaces)
- Time range: default last 3h

---

### 3.2 Developer Dashboard — "Workload Health"

**Audience:** Developers querying their own namespaces  
**Access:** Grafana (read-only role for dev namespaces)

#### Panels

| Panel | Metric Source | Description |
|-------|-------------|-------------|
| Pod CPU Request Utilization | `container_cpu_usage_seconds_total` vs `kube_pod_container_resource_requests` | % of requested CPU used, per pod |
| Pod Memory Request Utilization | `container_memory_working_set_bytes` vs `kube_pod_container_resource_requests` | % of requested memory used, per pod |
| HTTP Request Rate | `http_requests_total` (if instrumented) | Requests/sec per pod; "no data" state handled gracefully |
| Pod Status Summary | `kube_pod_status_phase` | Color-coded status badges: Running / Pending / Failed / Unknown |
| Container Restarts | `kube_pod_container_status_restarts_total` | Per-container restart count for selected namespace |

#### Layout

```
[ CPU Utilization ]  [ Memory Utilization ]
[ HTTP Request Rate (if available)         ]
[ Pod Status Summary ]  [ Container Restarts ]
```

#### Filters / Variables

- `$namespace` — single-select, scoped to namespaces the user has access to
- `$pod` — multi-select, filtered by `$namespace`
- Time range: default last 1h

---

## 4. GitOps Deployment Model (Ops UX)

The operator's interaction with Prometheus is entirely through git — no manual kubectl or Helm CLI operations.

### Deployment Topology (mirrors InfluxDB pattern)

```
terminus.infra repo
└── platforms/k3s/
    ├── argocd/apps/
    │   ├── prometheus.yaml         ← prod ArgoCD app
    │   └── prometheus-dev.yaml     ← dev ArgoCD app
    └── helm/prometheus/
        ├── values.yaml             ← prod values
        └── values-dev.yaml         ← dev values
```

### Operator Workflow

1. Edit `values.yaml` or `values-dev.yaml` in a PR branch
2. ArgoCD detects the merge to `main` and syncs the app
3. Prometheus pods roll with new config; Grafana datasource auto-resolves via service discovery
4. No out-of-band secret injection required for scraping (in-cluster service account)

### Secrets Boundary

- Admin/remote write credentials (if added later) → Vault + ESO
- Scrape configs for in-cluster targets → plain Helm values (no secrets)

---

## 5. Grafana Datasource Registration

The Prometheus datasource is registered in Grafana via Helm values provisioning (not manually):

```yaml
# values.yaml snippet (Grafana chart)
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.monitoring.svc.cluster.local
        isDefault: false   # InfluxDB remains default
        jsonData:
          timeInterval: "15s"
```

Operators confirm the datasource is live by checking Grafana → Data Sources → Prometheus → "Save & Test".

---

## 6. Out of Scope (This Feature)

| Item | Rationale |
|------|-----------|
| Alertmanager routing rules | Notification targets not yet decided; separate feature |
| Prometheus native UI / PromQL explorer | Grafana covers the query use case; raw UI adds complexity |
| Remote write / long-term retention | InfluxDB handles long-term; Prometheus retention set to 15d |
| Replacing InfluxDB | Each system covers different sources; coexistence is the model |

---

## 7. Open Questions

1. **Alertmanager targets** — where should firing alerts route? (Slack? Email? Semaphore webhook?)
2. ~~**Dev namespace scoping**~~ — ✅ **Decided:** Single Prometheus instance; dev and prod namespaces scoped via `$namespace` Grafana variable. Dev pod churn is acceptable at current scale.
3. ~~**Grafana org/folder**~~ — ✅ **Decided:** Unified folder; solo operator, both surfaces matter equally.
