---
feature: prometheus
doc_type: prd
status: draft
stepsCompleted: [1, 2, 2b, 2c, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12-complete]
inputDocuments:
  - docs/terminus/infra/prometheus/ux-design.md
  - docs/terminus/infra/terminus-infra-ai-substrate-service-plan.md
  - docs/terminus/infra/influxdb-monitoring/architecture.md
classification:
  projectType: Infrastructure / GitOps tooling
  domain: Platform engineering — observability
  complexity: medium
  context: brownfield
workflowType: prd
goal: "Close the Kubernetes observability gap in the Terminus stack with Prometheus scraping and Grafana visualization"
key_decisions: []
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-04-14T23:45:00Z"
---

# Product Requirements Document — Prometheus Monitoring

**Author:** Todd Hintzmann
**Date:** 2026-04-14
**Feature:** `prometheus` | Domain: `terminus` | Service: `infra`

---

## Executive Summary

Prometheus Monitoring closes the observability gap in the Terminus stack. InfluxDB captures Proxmox hypervisor metrics via native push; Prometheus captures Kubernetes-layer metrics via native scrape. Grafana serves both as a unified visualization surface. Together, the three form a complete, non-overlapping observability stack across the full infrastructure vertical — from bare metal to running containers.

Target users: platform engineers operating the k3s cluster, and developers whose workloads run on it. Operators need infrastructure-level visibility (node health, k3s control plane, PV capacity). Developers need workload-level visibility (pod resource utilization, container restarts, HTTP error rates) without requiring cluster access.

Deployment is fully GitOps-managed via ArgoCD/Helm, mirroring the existing InfluxDB pattern. Credentials stay in Vault. No manual cluster operations.

### What Makes This Special

Each tool does what it does natively. Prometheus is not a replacement or workaround — it owns the Kubernetes scrape surface the way InfluxDB owns Proxmox. The architectural seam between hypervisor and cluster is the current blind spot; Prometheus closes it precisely, without overloading any existing system. Grafana becomes a single pane of glass without any single datasource being forced to do more than it should.

### Project Classification

- **Type:** Infrastructure / GitOps tooling
- **Domain:** Platform engineering — observability
- **Complexity:** Medium (standard OSS stack; integration with existing Vault/ESO, ArgoCD, Grafana)
- **Context:** Brownfield — k3s, InfluxDB, Grafana, and ArgoCD are all operational

---

## Success Criteria

### User Success

- Platform engineer can open Grafana and see live k3s node and pod metrics without any manual query setup
- Developer can filter the workload dashboard to their namespace and see CPU/memory utilization for their pods

### Business Success

- The Kubernetes observability gap is closed — the full infrastructure vertical (Proxmox → k3s → workloads) is visible in a single Grafana instance
- Iteration is unblocked — a working baseline allows incremental improvements (alerting, additional exporters, retention tuning) without a redesign

### Technical Success

- Prometheus is deployed to the `monitoring` namespace via ArgoCD and is in `Healthy/Synced` state
- At least node, pod, and k3s control-plane targets are actively scraped with no persistent `down` targets
- Grafana datasource resolves and "Save & Test" passes
- No secrets in git; Vault/ESO pattern followed if credentials are needed

### Measurable Outcomes

- Scrape interval: 15s (default) — tuning deferred
- Retention: 15 days — long-term storage left to InfluxDB
- All thresholds are advisory, not gates — success is "it works and renders data"

## Product Scope

### MVP

- Prometheus deployed via ArgoCD/Helm (prod + dev environments)
- Node exporter, kube-state-metrics, k3s metrics scraping configured
- Grafana datasource provisioned via Helm values
- "k3s Infrastructure Health" and "Workload Health" dashboards rendering live data

### Growth Features (Post-MVP)

- Alertmanager with routing rules (targets TBD — deferred)
- Additional exporters (blackbox, cert-exporter, etc.)
- Remote write / long-term retention strategy
- Dashboard variable enhancements and annotations

### Vision (Future)

- Unified alerting across both datasources (InfluxDB + Prometheus) via Grafana unified alerting
- Automated dashboard provisioning from code (Grafonnet or similar)

---

## User Journeys

### Journey 1 — Platform Engineer: Monday Morning Ops Check

It's 8am. A deployment ran overnight. Todd opens Grafana, switches to the "k3s Infrastructure Health" dashboard, and immediately sees all three nodes green, pod restart count at zero for the last 12h, and PV capacity well within bounds. He checks the k3s API latency panel — P99 looks normal. He closes the tab and gets coffee. *The story ends before anything bad could hide.*

### Journey 2 — Platform Engineer: Incident Response

A service feels sluggish. Todd opens the "Workload Health" dashboard, sets `$namespace=fourdogs`, and sees one pod's memory utilization spiking to 95% of its request limit. He restarts it, watches the metric drop in real time, and files a ticket to raise the limit. *Without Prometheus, this would have been a kubectl logs spelunking session.*

### Journey 3 — Developer: Self-Service Debug

A dev sees errors in their app. They open Grafana, filter the Workload Health dashboard to their namespace, check HTTP error rate and restart count. They identify a container OOMKilling and fix their memory request. *They never needed cluster access or a Slack message to ops.*

### Journey Requirements Summary

| Journey | Reveals |
|---|---|
| Ops morning check | Node/pod/k3s health panels, sane defaults, fast dashboard load |
| Ops incident response | Real-time metrics, namespace filter, restart trends |
| Dev self-service | Namespace-scoped dashboard, read-only Grafana role, graceful no-data handling |

---

## Infrastructure / GitOps Tooling — Specific Requirements

### Technical Architecture Considerations

- **Helm chart:** `kube-prometheus-stack` (bundles Prometheus, node-exporter, kube-state-metrics, Alertmanager) or standalone `prometheus-community/prometheus` — decision deferred to TechPlan
- **ArgoCD app pattern:** Mirror `influxdb.yaml` / `influxdb-dev.yaml` — separate prod and dev ArgoCD apps pointing to separate Helm values files
- **Namespace:** `monitoring` (prod), `monitoring-dev` (dev) — consistent with existing InfluxDB deployment
- **Scrape targets:** node-exporter (per-node DaemonSet), kube-state-metrics (cluster-level), k3s API server metrics endpoint
- **Grafana datasource:** Provisioned via Grafana Helm values (`datasources.yaml`) — not manually added; `isDefault: false` (InfluxDB remains default)

### Deployment Constraints

- All config changes via PR to `terminus.infra` repo → ArgoCD syncs automatically
- No `kubectl apply` or `helm upgrade` commands run manually in prod
- In-cluster scrape requires no secrets; Vault + ESO for any future remote-write credentials

### Implementation Considerations

- **Dev environment first:** Deploy to `monitoring-dev` and validate before prod rollout
- **Dashboard provisioning:** Dashboards as ConfigMaps via Helm, not manually saved in Grafana UI
- **Resource limits:** Conservative requests/limits on Prometheus pod given finite k3s node capacity
- **Retention:** `--storage.tsdb.retention.time=15d` in Helm values

---

## Functional Requirements

### FR-1: Prometheus Deployment

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | Prometheus deployed to `monitoring` namespace via ArgoCD/Helm, status `Healthy/Synced` | Must |
| FR-1.2 | Separate dev deployment to `monitoring-dev` namespace with its own Helm values | Must |
| FR-1.3 | Scrape node-exporter metrics from all k3s nodes (DaemonSet) | Must |
| FR-1.4 | Scrape kube-state-metrics (pod status, resource requests, restart counts) | Must |
| FR-1.5 | Scrape k3s control-plane metrics (API server latency, etcd if applicable) | Must |
| FR-1.6 | All scrape targets show `UP` status; no persistent `down` targets at steady state | Must |
| FR-1.7 | Metric retention configured to 15 days | Should |
| FR-1.8 | Scrape interval 15s (default); configurable via Helm values | Should |

### FR-2: Grafana Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | Prometheus registered as a Grafana datasource via Helm values provisioning | Must |
| FR-2.2 | Datasource `Save & Test` returns success after deployment | Must |
| FR-2.3 | InfluxDB remains the default Grafana datasource | Must |
| FR-2.4 | Prometheus datasource available for dashboard queries alongside InfluxDB | Must |

### FR-3: Operator Dashboard — "k3s Infrastructure Health"

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | Dashboard provisioned as ConfigMap via Helm, persists across Grafana pod restarts | Must |
| FR-3.2 | Node CPU utilization panel (all nodes, stacked, 80% threshold line) | Must |
| FR-3.3 | Node memory utilization panel (all nodes, gauge, red at <10% free) | Must |
| FR-3.4 | Pod restart count table (all namespaces, sorted descending, last 24h) | Must |
| FR-3.5 | k3s API server latency panel (P50/P99 line chart) | Must |
| FR-3.6 | Persistent volume capacity heatmap | Should |
| FR-3.7 | Node network I/O line chart | Should |
| FR-3.8 | Dashboard variables: `$node` (multi-select), `$namespace` (multi-select), time range default 3h | Must |

### FR-4: Developer Dashboard — "Workload Health"

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | Dashboard provisioned as ConfigMap via Helm | Must |
| FR-4.2 | Pod CPU request utilization panel (% of requested CPU, per pod) | Must |
| FR-4.3 | Pod memory request utilization panel (% of requested memory, per pod) | Must |
| FR-4.4 | Container restart count panel (per container, selected namespace) | Must |
| FR-4.5 | Pod status summary panel (Running/Pending/Failed/Unknown badges) | Must |
| FR-4.6 | HTTP request rate panel — graceful "no data" state for uninstrumented apps | Should |
| FR-4.7 | Dashboard variables: `$namespace` (single-select), `$pod` (multi-select filtered by namespace), time range default 1h | Must |

### FR-5: GitOps & Operations

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-5.1 | All configuration managed in `terminus.infra` repo under `platforms/k3s/` | Must |
| FR-5.2 | ArgoCD app files follow existing pattern (`argocd/apps/prometheus.yaml`, `prometheus-dev.yaml`) | Must |
| FR-5.3 | Helm values files: `helm/prometheus/values.yaml`, `helm/prometheus/values-dev.yaml` | Must |
| FR-5.4 | No secrets committed to git | Must |

---

## Non-Functional Requirements

### Performance

| ID | Requirement |
|----|-------------|
| NFR-P.1 | Grafana dashboard load time < 5s under normal k3s load |
| NFR-P.2 | Prometheus pod memory usage < 512Mi at 15d retention with current cluster size |
| NFR-P.3 | Scrape duration < 10s per target; alert if consistently exceeded |

### Reliability

| ID | Requirement |
|----|-------------|
| NFR-R.1 | Prometheus pod restarts automatically via k3s pod lifecycle management |
| NFR-R.2 | TSDB survives pod restart with persistent volume; no metric data loss on restart |
| NFR-R.3 | Dashboard ConfigMaps survive Grafana pod restarts |

### Security

| ID | Requirement |
|----|-------------|
| NFR-S.1 | No credentials in git; Vault/ESO for any future remote-write secrets |
| NFR-S.2 | Prometheus endpoint not exposed outside cluster (in-cluster only) |
| NFR-S.3 | Grafana developer access read-only; no edit or admin permissions for devs |

### Maintainability

| ID | Requirement |
|----|-------------|
| NFR-M.1 | Helm values files self-documenting with inline comments on non-obvious settings |
| NFR-M.2 | Dashboard JSON stored in version control; reproducible from scratch via ArgoCD sync |
| NFR-M.3 | Upgrade path via Helm chart version bump in values file; no manual migration steps |

---

## Constraints & Assumptions

- **Assumption:** k3s cluster has sufficient headroom for Prometheus + node-exporter DaemonSet (estimated ~256Mi/pod)
- **Assumption:** Grafana Helm chart already deployed and upgradeable to add datasource provisioning
- **Constraint:** Single Prometheus instance for both dev and prod namespaces (label-based scoping via `$namespace`)
- **Constraint:** Alertmanager deferred — bundled if using `kube-prometheus-stack` but rules left empty for now
- **Out of scope:** Remote write, long-term retention, Prometheus native UI hardening, replacing InfluxDB
