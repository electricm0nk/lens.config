---
feature: terminus-infra-centralized-logging
doc_type: architecture
status: draft
goal: "Deliver a centralized log aggregation pipeline for all k3s workloads and VM-side services, with Loki as the backend, Alloy as the cluster-wide collector, and Grafana as the query surface."
key_decisions:
  - Use Grafana Loki (single-binary mode) as the log backend — already assumed in all existing docs, native Grafana integration, no licensing cost.
  - Use MinIO in-cluster as Loki prod object storage backend — S3-compatible single-instance StatefulSet in `monitoring` namespace, 100Gi local-path PVC; decouples Loki pod lifecycle from log storage.
  - Use local-path PVC for loki-dev only — dev instance is for upgrade staging/validation, not real log shipping.
  - Use Grafana Alloy as the DaemonSet collector — successor to Promtail, ships with OTel pipeline support for future OpenTelemetry convergence.
  - Single Alloy DaemonSet ships to prod Loki only — loki-dev is populated via synthetic data injection for upgrade staging; no alloy-dev DaemonSet.
  - Ship VM-side logs (Vault audit, Postgres JSON) via a lightweight Alloy service installation on each VM host (best-effort; no alert on shipper failure).
  - Add Loki as a Grafana datasource via the existing sidecar-managed datasource ConfigMap pattern.
  - Deploy Loki, Alloy, and Alertmanager via ArgoCD multi-source Helm apps following the existing prometheus/influxdb pattern.
  - Alertmanager is already deployed by the `prometheus` feature (complete) with `alertmanager.enabled: true` but no routes. This feature adds a Loki ruler alert route via `AlertmanagerConfig` CRD — no redeployment needed.
  - Coordinate Loki ruler alert routing with `prometheus-wiring` (finalizeplan-complete) to avoid config conflicts.
  - Store Loki auth and any tenant credentials in Vault, materialized via ESO — no secrets in git.
  - Should Loki use single-binary (SimpleScalable) or microservices mode? Single-binary recommended for current cluster size; re-evaluate at >10TB/day ingest.
resolved_questions:
  - Log retention: dev 7d (upgrade staging, no enforcement), prod 30d. Confirmed.
  - Loki ruler alert rules: in scope for this feature.
  - VM-side log shipping SLA: best-effort accepted — no alert on shipper failure.
  - Vault audit log durability: best-effort accepted — MinIO on local-path PVC survives pod restarts but not node loss; no zero-loss guarantee.
depends_on:
  - docs/terminus/infra/prometheus-wiring/architecture.md
  - docs/terminus/infra/secrets/architecture.md
  - docs/terminus/infra/k3s/architecture.md
blocks:
  - ArgoCD alert rules (ArgoCDAppDegraded, ArgoCDSyncFailureRateHigh) — currently have no data source
  - Vault audit trail centralization
  - Postgres query error analysis
updated_at: 2026-04-23T00:00:00Z
stepsCompleted: [1]
inputDocuments:
  - prometheus-wiring/docs/architecture.md
  - influxdb-monitoring/docs/architecture.md
  - platforms/k3s/helm/grafana/values.yaml
  - platforms/k3s/argocd/apps/prometheus.yaml
  - platforms/k3s/helm/kube-prometheus-stack/values.yaml
---

# Centralized Logging Architecture (Terminus Infra)

## Problem Statement and Scope

Terminus runs 25+ distinct workload deployments across 24 namespaces on a 9-node k3s cluster (3 control-plane, 6 worker nodes). Currently all logs are emitted to container stdout/stderr only and are accessible only via `kubectl logs` — ephemeral, per-pod, manual, and destroyed when a pod restarts or is evicted. There is no central destination.

**Operational impact:**
- Crash-loop bugs require `kubectl logs --previous` on each affected pod individually.
- Cross-service failure sequences (e.g., fourdogs-central → temporal-worker → fourdogs-emailfetcher) cannot be correlated without jumping between namespaces.
- Temporal workflow failures reference activity names but not the application error that caused them — finding the root cause requires manual log hunting.
- ArgoCD sync failures and application degradations generate Prometheus alert rules (`ArgoCDAppDegraded`, `ArgoCDSyncFailureRateHigh`) that fire on `argocd_app_info` metrics — but ArgoCD metrics are not being scraped at all (ServiceMonitor gap), and Prometheus itself is in CrashLoopBackOff, so no alerts fire.
- VM-side services (Vault, Postgres) emit logs to local disk only. Vault audit logs, which record every secret access, are not inspectable without SSHing into the Vault VM.

**In scope:**
- MinIO in-cluster deployment (single-instance StatefulSet, `monitoring` namespace, 100Gi local-path PVC), bucket creation, and ESO secret for Loki prod object storage credentials.
- Alertmanager deployment in `monitoring` namespace — **already deployed** by `prometheus` feature (complete); no action needed here. This feature adds Loki ruler alert routing via `AlertmanagerConfig` CRD.
- Loki deployment in `monitoring` namespace; `loki-dev` in `monitoring-dev` for upgrade staging only (populated via synthetic data injection, not real log shipping).
- Grafana Alloy DaemonSet — scrapes container logs from all namespaces on all nodes; ships to prod Loki only.
- Loki datasource added to Grafana (prod and dev).
- VM-side Alloy agent for Vault audit log and Postgres JSON log shipping (best-effort; no alert on shipper failure).
- Grafana dashboard set for platform log search (ArgoCD, Temporal, Traefik, fourdogs services).
- ArgoCD ServiceMonitor fix — closes the metrics scrape gap that currently makes all ArgoCD alert rules inert.
- Log-based Loki ruler alert rules for critical error patterns (crash loops, Vault audit anomalies).
- Dev and prod overlays following existing monitoring app conventions.

**Out of scope:**
- Replacing InfluxDB or Prometheus — this is additive.
- SIEM integration (deferred; Loki pipeline is SIEM-compatible per existing docs).
- Multi-cluster log federation.
- Log-based SLOs (deferred; metric SLOs are the Prometheus-wiring scope).
- Long-term cold log archive beyond MinIO in-cluster (e.g., external NAS, glacier-tier, multi-site replication) — MinIO is the prod backend; additional archive tiers are out of scope.

---

## Current State and Constraints

### What Exists

| Component | State | Notes |
|-----------|-------|-------|
| kube-prometheus-stack | Deployed; Prometheus CrashLoopBackOff resolved; Alertmanager deployed (no routes) | Alertmanager routing for Loki ruler alerts added in Phase 6 via AlertmanagerConfig CRD |
| Grafana (prod + dev) | Healthy | InfluxDB + Prometheus datasources; no Loki datasource |
| InfluxDB (prod + dev) | Healthy | Metrics backend for dashboards |
| ArgoCD | Healthy (all pods running) | No ServiceMonitor; logs to stdout only |
| Temporal (server + workers) | Healthy | Logs to stdout only |
| Traefik | Healthy | Access logs to stdout only |
| fourdogs-central, kaylee, emailfetcher, etailpet-trigger | Healthy | Application logs to stdout only |
| inference-gateway | Healthy | Logs to stdout only |
| terminus-portal | Healthy | Logs to stdout only |
| External Secrets Operator | Healthy | Logs to stdout only |
| cert-manager | Healthy | Logs to stdout only |
| Vault (external VM) | Operational | Audit log at `/var/log/vault/audit.log` on VM |
| Postgres (external VM) | Operational | JSON log at `/var/log/postgresql/` on VM |
| actions-runner | 9 deployments | One per repo; logs to stdout |

### Platform Constraints

- k3s 1.32 on Proxmox, 3 CP + 6 worker nodes (Ubuntu 24.04, containerd 2.0.4).
- All cluster delivery is GitOps via ArgoCD (app-of-apps, `terminus-infra-k3s-root`).
- TLS is cert-manager + Vault PKI (`*.trantor.internal`).
- Secrets: Vault → ESO → Kubernetes Secret. No plaintext credentials in git.
- Storage: `local-path` StorageClass (node-local RWO). No shared/distributed storage class yet.
- Namespace label for Prometheus scrape opt-in: `observability.terminus.io/scrape: "true"`.
- Existing monitoring Helm apps follow multi-source pattern: chart from upstream registry + values from `terminus.infra` repo (same `$values` ref pattern).
- Dev overlays mirror prod with suffix `-dev` on app names and namespace names.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         k3s Cluster                                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  All Nodes (DaemonSet)                                        │   │
│  │  Grafana Alloy                                                │   │
│  │  - reads /var/log/pods/* (all namespaces, all containers)    │   │
│  │  - applies namespace/pod/container labels                     │   │
│  │  - ships to Loki push API                                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                 │                                    │
│                                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  monitoring namespace                                         │   │
│  │  Loki (single-binary)                    50Gi local-path PVC │   │
│  │  - ingests from Alloy push               retention: 30d      │   │
│  │  - serves LogQL query API                                     │   │
│  │  - Loki ruler: log-based alert rules     → Alertmanager      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                 │                                    │
│                                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Grafana (existing, monitoring namespace)                     │   │
│  │  + Loki datasource (new ConfigMap via sidecar)               │   │
│  │  + Log dashboards (ArgoCD, Temporal, fourdogs, Traefik)      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

External VMs (Proxmox)
┌─────────────────────────────────────────────────────────────┐
│  Vault VM                        Postgres VM                │
│  Alloy (systemd service)         Alloy (systemd service)    │
│  reads: /var/log/vault/audit.log reads: /var/log/postgresql/│
│  ships → Loki push API (internal) ships → Loki push API     │
└─────────────────────────────────────────────────────────────┘

ArgoCD ServiceMonitor Fix (in-scope, same deploy wave)
┌─────────────────────────────────────────────────────────────┐
│  argocd namespace                                           │
│  argocd Helm values: metrics enabled for all components     │
│  argocd namespace label: observability.terminus.io/scrape   │
│  → Prometheus now scrapes argocd_app_info, sync metrics     │
│  → Existing alert rules and dashboard become functional     │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Decisions

### 1. Log Backend: Grafana Loki

**Mode:** Single-binary (monolithic). All components (ingester, querier, ruler, compactor) run in one pod.

**Rationale:**
- Already assumed by every relevant doc in the codebase (`secrets/architecture.md`, `postgres/architecture.md`, `secrets/prd.md` FR35, `emailfetcher/stories.md`). This is not a new choice — it's closing a known gap.
- Native Grafana integration: Loki datasource is first-class in the Grafana version already deployed.
- Single-binary is appropriate for current cluster size (9 nodes, ~25 workloads). SimpleScalable is the next step if ingest exceeds 10GB/day.
- No external object storage required for MVP — `local-path` PVC matches the Prometheus pattern.
- No per-node licensing cost (vs Splunk, Datadog).

**Deployment:** Helm chart `grafana/loki`, version `6.*`, ArgoCD multi-source app following `prometheus.yaml` pattern.

**Storage:**
- Prod: MinIO in-cluster S3-compatible object store. Single-instance StatefulSet in `monitoring` namespace, 100Gi `local-path` PVC. Credentials in Vault KV (`secret/terminus/loki/s3-creds`), materialized via ESO into `loki-s3-creds` secret in `monitoring` namespace. Bucket: `terminus-loki-prod`. Endpoint: `http://minio.monitoring.svc.cluster.local:9000`. Retention: 30 days.
- Dev (`loki-dev`): 20Gi `local-path` PVC — upgrade staging only; no retention enforcement; not a real log ingestion target.
- Rationale for MinIO over raw local-path for prod: decouples Loki compute from log storage; Loki pod can be rescheduled without data loss (storage stays on the MinIO StatefulSet PVC, which persists across Loki pod restarts). Node-local only — does not survive permanent node loss, which is accepted for homelab scale.

**Service:** `loki.monitoring.svc.cluster.local:3100` (ClusterIP, internal only). No external ingress needed.

### 2. Cluster Log Collector: Grafana Alloy DaemonSet

**Rationale:**
- Alloy is the official Grafana agent successor (Promtail is maintenance-only as of 2024).
- Ships with an OTel pipeline built in — positions Terminus to adopt OpenTelemetry structured logs without replacing the collector.
- Native Loki push API support via `loki.write` component.
- Reads container logs directly from `/var/log/pods/` on each node — no API server load, no sidecar containers needed.

**DaemonSet scope:** All nodes (3 CP + 6 workers). Tolerations for control-plane taint. Ships to prod Loki (`loki.monitoring.svc.cluster.local:3100`) only. `loki-dev` is not a real log destination — it is populated via synthetic data injection for upgrade staging.

**Log sources collected:**
- `/var/log/pods/*` — all containers in all namespaces.
- `/var/log/journal` — systemd journal (k3s agent, containerd, node-level events).

**Label strategy** (applied at collection time, immutable after ingestion):

| Label | Value | Source |
|-------|-------|--------|
| `cluster` | `terminus` | Static in Alloy config |
| `namespace` | `fourdogs-central-dev` | Pod metadata |
| `app` | `fourdogs-central` | `app.kubernetes.io/name` label |
| `pod` | `fourdogs-central-abc-xyz` | Pod name |
| `container` | `api` | Container name |
| `node` | `k3s-worker-01` | Node name |
| `env` | `dev` \| `prod` | Derived from namespace suffix |

**Cardinality notes:** No `log_level` label at collection time — log level should be extracted via LogQL pipeline expressions at query time, not indexed as a label. Keeps label cardinality low and ingest cost proportional.

**Secret handling:** No auth required for Loki push from in-cluster Alloy — Loki is ClusterIP-only, no auth configured for internal push (multi-tenancy disabled for single-cluster MVP). If multi-tenancy is enabled later, `X-Scope-OrgID` header is injected by Alloy config, credential from Vault via ESO.

### 3. VM-Side Log Collector: Alloy (systemd service)

Two VM targets initially:

**Vault VM** (`vault.trantor.internal`):
- Source: `/var/log/vault/audit.log` (JSON, one event per line)
- Alloy `local.file_match` → `loki.source.file` → `loki.write`
- Labels: `cluster=terminus`, `service=vault`, `log_type=audit`
- Priority: High — audit logs must survive pod-cycle gaps. Alloy file position is persisted on disk.

**Postgres VM** (`postgres.primary.trantor.internal`):
- Source: `/var/log/postgresql/*.log` (JSON log format, confirmed in postgres/architecture.md)
- Alloy `local.file_match` → `loki.source.file` → `loki.write`
- Labels: `cluster=terminus`, `service=postgres`, `log_type=db`

**Alloy config for VM:** Stored in `platforms/vm/alloy/` in this repo, deployed via Ansible playbook (same delivery mechanism as Vault and Postgres initial provisioning).

**Loki push endpoint from VM:** `http://loki.monitoring.svc.cluster.local:3100` is not routable from VMs. Options:
- Option A (recommended): Expose Loki push API on a ClusterIP with MetalLB LoadBalancer annotation — internal IP only, not externally routed.
- Option B: Route via Traefik internal ingress (`loki-push.trantor.internal`) — consistent with how other internal services are accessed from VMs.

**Decision:** Option B (Traefik ingress) — consistent with the existing operational pattern. VMs already resolve `*.trantor.internal` via internal DNS. This keeps Loki itself ClusterIP and avoids adding a MetalLB LB entry for a single push endpoint.

### 4. Grafana Loki Datasource

**Method:** ConfigMap with label `grafana_datasource: "1"` — same pattern as the existing Grafana sidecar picks up. No changes to `grafana/values.yaml` required for the datasource itself.

**Dev and prod:** Two ConfigMaps — one in `monitoring`, one in `monitoring-dev` — pointing at respective Loki endpoints.

**No auth:** Internal cluster access, Loki auth disabled for single-tenant MVP.

### 5. ArgoCD ServiceMonitor Fix

**What's missing:** `argocd` Helm values have no `server.metrics`, `controller.metrics`, `repoServer.metrics`, or `serviceMonitor` stanzas. The comment in `monitoring-servicemonitors.yaml` that says "ServiceMonitors for ArgoCD are managed via their respective Helm values" was never implemented.

**Fix — three changes:**

1. Add to `platforms/k3s/helm/argocd/values.yaml`:
```yaml
controller:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: prometheus
server:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: prometheus
repoServer:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: prometheus
applicationSet:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: prometheus
```

2. Add namespace label to `argocd` namespace (in `prometheus-infra.yaml` or the argocd namespace manifest):
```yaml
labels:
  observability.terminus.io/scrape: "true"
```

3. No changes needed to `monitoring-servicemonitors.yaml` — ArgoCD's own Helm values now own the ServiceMonitor objects as the comment intended.

**Effect:** `argocd_app_info`, `argocd_app_sync_total`, and all other ArgoCD metrics become available in Prometheus. The three existing alert rules (`ArgoCDAppDegraded`, `ArgoCDAppOutOfSync`, `ArgoCDSyncFailureRateHigh`) and the ArgoCD Delivery Health Grafana dashboard become functional immediately after Prometheus is also repaired.

**Note:** Prometheus CrashLoopBackOff must be diagnosed and fixed before this provides value. That is a pre-condition for this feature going live on the metrics side.

---

## GitOps Delivery Pattern

### New ArgoCD Applications

Following the established multi-source + sync-wave pattern:

| App Name | Chart | Namespace | Wave | Purpose |
|----------|-------|-----------|------|---------|
| `kube-prometheus-stack` (update) | `prometheus-community/kube-prometheus-stack` | `monitoring` | 2 | Enable Alertmanager — update existing app values only |
| `minio` | `bitnami/minio 14.*` | `monitoring` | 2 | In-cluster S3 object store for Loki prod — 100Gi local-path PVC |
| `loki` | `grafana/loki 6.*` | `monitoring` | 3 | Prod Loki backend (MinIO S3 storage) |
| `loki-dev` | `grafana/loki 6.*` | `monitoring-dev` | 3 | Dev Loki backend (local-path, upgrade staging only) |
| `alloy` | `grafana/alloy 0.*` | `monitoring` | 4 | Cluster log collector DaemonSet — ships to prod Loki only |

Wave rationale: Loki must be healthy before Alloy tries to push logs. Both are after the namespace and base infra waves (0-2). No `alloy-dev` — single Alloy ships to prod Loki only.

### New Helm Values Files

```
platforms/k3s/helm/
├── minio/
│   └── values.yaml          # MinIO standalone, 100Gi local-path PVC, terminus-loki-prod bucket
├── loki/
│   ├── values.yaml          # prod — MinIO S3 backend, 30d retention
│   └── values-dev.yaml      # dev — 20Gi local-path PVC, upgrade staging (no real Alloy shipping)
└── alloy/
    └── values.yaml          # ships to prod Loki only; no alloy-dev variant
```

### New Grafana Datasource ConfigMaps

```
platforms/k3s/k8s/
├── monitoring/
│   └── loki-datasource.yaml           # prod Loki datasource ConfigMap
└── monitoring-dev/
    └── loki-datasource-dev.yaml       # dev Loki datasource ConfigMap
```

Deployed by existing `monitoring-servicemonitors` ArgoCD app (or a new `loki-infra` app at wave 3.5 if separation is preferred). The sidecar in Grafana already watches for `grafana_datasource: "1"` ConfigMaps.

### New Grafana Dashboard ConfigMaps

```
platforms/k3s/k8s/prometheus-infra/
├── dashboard-logs-argocd.yaml       # ArgoCD log search + error rate
├── dashboard-logs-temporal.yaml     # Temporal workflow error logs
├── dashboard-logs-fourdogs.yaml     # fourdogs-* application logs
├── dashboard-logs-platform.yaml     # Traefik, ESO, cert-manager
└── dashboard-logs-vault.yaml        # Vault audit log search
```

Each dashboard uses Loki as datasource and LogQL for queries. They slot into the existing sidecar dashboard pickup (`grafana_dashboard: "1"` label).

### VM Alloy Delivery

```
platforms/vm/
└── alloy/
    ├── config.alloy.vault.j2      # Jinja2 template — Vault VM Alloy config
    ├── config.alloy.postgres.j2   # Jinja2 template — Postgres VM Alloy config
    └── deploy-alloy-vm.yaml       # Ansible playbook — install + configure Alloy
```

Delivery: Ansible playbook run from SemaphoreUI (consistent with how Vault and Postgres VMs are managed). The playbook installs the Alloy binary, writes the rendered config, and enables the systemd service.

---

## Security Considerations

### In-Cluster (Alloy DaemonSet)

- Alloy runs as a privileged DaemonSet (required to read `/var/log/pods/` on the host).
- ServiceAccount with minimal RBAC: `get`/`list`/`watch` on `pods`, `namespaces` only (for pod metadata enrichment).
- No Loki auth for internal push — Loki is ClusterIP, not reachable from outside the cluster.
- No log content encryption in transit within the cluster (acceptable for internal private network).

### VM-Side Alloy

- Alloy on VM communicates with Loki via internal Traefik ingress (`loki-push.trantor.internal`).
- TLS: cert-manager-issued certificate (consistent with other `*.trantor.internal` endpoints).
- No client auth for Loki push API in single-tenant mode (VMs are on private internal network only).
- Vault audit log file must be readable by the `alloy` user on the VM. The Vault audit log device is currently world-readable if Vault is configured that way — verify and tighten file permissions so only `vault` and `alloy` users can read.

### Log Content

- Logs may contain sensitive request paths, user identifiers, and internal service names.
- No PII filtering at collection time in MVP. Vault audit logs are already redacted by Vault's HMAC-based secret value masking.
- Loki has no row-level access control in single-tenant mode — anyone with Grafana access sees all logs.
- Post-MVP: enable Loki multi-tenancy + Grafana datasource permissions per team if finer access control is needed.

---

## Loki Configuration Key Decisions

### Storage Mode: `s3` (Synology NAS)

```yaml
# loki/values.yaml — key excerpt
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: s3
    s3:
      endpoint: http://minio.monitoring.svc.cluster.local:9000
      bucketnames: terminus-loki-prod
      region: us-east-1                # required field; MinIO ignores the value
      access_key_id: ${LOKI_S3_ACCESS_KEY}    # ESO-materialized from Vault
      secret_access_key: ${LOKI_S3_SECRET_KEY}
      insecure: true                   # MinIO in-cluster HTTP — no TLS needed internally
      s3forcepathstyle: true           # required for MinIO path-style access
  schemaConfig:
    configs:
      - from: "2026-04-22"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h
  compactor:
    retention_enabled: true   # REQUIRED: without this, retention_period is silently ignored
  limits_config:
    retention_period: 720h   # 30d prod
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32
    max_streams_per_user: 0  # unlimited in single-tenant mode
```

### Ruler (Log-Based Alerts)

Loki ruler enables alerting on log content without requiring Prometheus metric exporters. Rules are defined inline in `loki/values.yaml` via `loki.rulesConfig` — Helm populates the local rules directory at deploy time. No PrometheusRule CRDs are used.

```yaml
# loki/values.yaml — ruler and rules sections
ruler:
  enabled: true
  storage:
    type: local
    local:
      directory: /rules
  rule_path: /rules/fake  # single-tenant path
  alertmanager_url: http://alertmanager-operated.monitoring.svc.cluster.local:9093  # verify service name after Alertmanager deploy

rulesConfig:
  terminus-platform.yaml: |
    groups:
      - name: terminus-platform
        rules:
          - alert: CrashLoopDetected
            expr: count_over_time({namespace=~".+"} |= "reason=CrashLoopBackOff" [5m]) > 0
            labels:
              severity: critical
          - alert: OOMKillDetected
            expr: count_over_time({namespace=~".+"} |= "OOMKilled" [5m]) > 0
            labels:
              severity: critical
          - alert: VaultAuditDeny
            expr: count_over_time({service="vault",log_type="audit"} | json | type="response" | error!="" [5m]) > 5
            labels:
              severity: warning
          - alert: TemporalWorkflowPanic
            expr: count_over_time({namespace="temporal"} |= "panic" [5m]) > 0
            labels:
              severity: critical
          - alert: ArgoCDSyncError
            expr: count_over_time({namespace="argocd"} |= "level=error" [5m]) > 10
            labels:
              severity: warning
```

**Alert rule notes:**
- `CrashLoopDetected` uses `|= "reason=CrashLoopBackOff"` (structured field match) to avoid false positives from scripts or monitors that print the phrase as plain text.
- Alertmanager receiver routing for Loki ruler alerts must be coordinated with `prometheus-wiring` (at `finalizeplan-complete`) — a matching route/receiver must exist in Alertmanager config or alerts will be silently dropped.

---

## Implementation Sequence

This feature has a dependency on Prometheus being operational. Loki can be deployed independently, but the ArgoCD ServiceMonitor fix only becomes functional once Prometheus is healthy.

### Phase 0 — Prerequisites (before any Loki deploy)
1. Deploy MinIO in `monitoring` namespace: ArgoCD app + values.yaml (100Gi local-path PVC, `defaultBuckets: terminus-loki-prod`, wave 2). Confirm MinIO pod healthy and `terminus-loki-prod` bucket exists.
2. Store MinIO root credentials in Vault KV (`secret/terminus/loki/s3-creds`), create ExternalSecret to materialize as `loki-s3-creds` in `monitoring` namespace. MinIO `existingSecret` references this secret — ESO must materialize before MinIO pod starts.
3. Verify Alertmanager is running: `kubectl get pods -n monitoring | grep alertmanager` — deployed by `prometheus` feature (complete). No redeployment needed. Note the Alertmanager service name for Loki ruler `alertmanager_url`.

### Phase 1 — Loki Backend (after Phase 0 prereqs complete)
1. Add `platforms/k3s/helm/loki/values.yaml` (prod — MinIO S3 backend, 30d retention) and `values-dev.yaml` (dev — local-path, upgrade staging)
2. Add `platforms/k3s/argocd/apps/loki.yaml` and `loki-dev.yaml`
3. Add Loki datasource ConfigMaps to `monitoring` and `monitoring-dev`
4. Commit to `terminus-infra-centralized-logging-plan` → ArgoCD deploys Loki
5. Verify Loki pod healthy, S3 connection to MinIO established, LogQL API responsive

### Phase 2 — Alloy DaemonSet (after Phase 1)
1. Add `platforms/k3s/helm/alloy/values.yaml` (prod only — no alloy-dev variant)
2. Add `platforms/k3s/argocd/apps/alloy.yaml`
3. Commit and sync — Alloy DaemonSet deploys to all 9 nodes
4. Verify logs flowing in Grafana Loki datasource (LogQL `{cluster="terminus"}`)
5. Validate all namespaces present in label browser

### Phase 3 — Grafana Dashboards
1. Create 5 dashboard ConfigMaps (ArgoCD, Temporal, fourdogs, platform, Vault)
2. Verify sidecar picks them up in Grafana
3. Confirm LogQL queries return data for each dashboard panel

### Phase 4 — ArgoCD ServiceMonitor Fix
1. Update `platforms/k3s/helm/argocd/values.yaml` with metrics + ServiceMonitor stanzas
2. Add `observability.terminus.io/scrape: "true"` label to `argocd` namespace
3. Sync ArgoCD app — new ServiceMonitors created
4. Verify in Prometheus targets: `argocd-*` endpoints appear as UP
5. Verify ArgoCD Delivery Health dashboard shows data (requires Prometheus also fixed)

### Phase 5 — VM Alloy Agents
1. Add `platforms/vm/alloy/` directory with Ansible playbook and Alloy config templates
2. Add Traefik internal ingress for `loki-push.trantor.internal`
3. Run Ansible playbook via SemaphoreUI against Vault VM
4. Verify Vault audit log stream in Grafana
5. Run Ansible playbook against Postgres VM
6. Verify Postgres log stream in Grafana

### Phase 6 — Loki Ruler Alert Rules
1. Add `loki.rulesConfig` block to `platforms/k3s/helm/loki/values.yaml` with inline rule groups.
2. Add `AlertmanagerConfig` CRD to `platforms/k3s/k8s/monitoring/loki-alertmanager-config.yaml` for Loki ruler receiver routing. Coordinate with `prometheus-wiring` feature: the Alertmanager config owned by `prometheus-wiring` must not conflict with this route. If `prometheus-wiring` is not yet in dev, add a minimal catch-all route here and plan to migrate to the full `prometheus-wiring` routing config when it ships.
3. Sync `loki` ArgoCD app — Helm populates the local rules directory; ruler loads rules automatically.
4. Verify rules visible in Loki ruler API (`/loki/api/v1/rules`).
5. Verify alerts route to Alertmanager (check Alertmanager active alerts UI or API).
6. Test CrashLoopDetected rule by intentionally crashing a test pod.

---

## Dev/Prod Overlay Summary

| Concern | Dev (upgrade staging) | Prod |
|---------|-----------------------|------|
| Loki storage | 20Gi local-path PVC | MinIO in-cluster S3 (`terminus-loki-prod` bucket, 100Gi local-path PVC) |
| Retention | No enforcement (upgrade staging only) | 30d (`compactor.retention_enabled: true`) |
| Log ingestion | Synthetic data injection only — no real Alloy shipping | Alloy DaemonSet ships all cluster logs |
| Ingress hostname | Not exposed (no real push traffic) | `loki-push.trantor.internal` |
| Grafana datasource | In `monitoring-dev`, points to dev Loki | In `monitoring`, points to prod Loki |
| Ruler alerts | Disabled | Enabled — routes to Alertmanager |
| ArgoCD ServiceMonitor | Applies to same `argocd` namespace (shared) | Same — single ArgoCD instance |

---

## Affected Repositories

| Repo | Changes |
|------|---------|
| `terminus.infra` (this repo) | All Helm values, ArgoCD apps, ConfigMaps, Ansible playbook |
| `terminus.infra` | ArgoCD Helm values update (ServiceMonitor fix) |
| No application repos need changes | Log collection is infrastructure-side only; application pods need no modification |

---

## ADR-001: Loki over Splunk / Datadog / ELK

**Status:** Decided

**Context:** Need a log aggregation backend for a self-hosted k3s cluster on private Proxmox hardware.

**Decision:** Grafana Loki.

**Rationale:**
- All existing planning docs already reference Loki specifically (not "a log solution") — this is closing a known architectural gap.
- Grafana is already deployed and healthy. Loki is a first-class Grafana datasource — zero additional UI tooling needed.
- Loki's label-based index model (labels only, not full-text index) means storage costs scale with label cardinality, not log volume. At current cluster size this is 10-50x cheaper on storage than ELK.
- Self-hosted, no licensing fees, no data egress to cloud vendor.
- Splunk: enterprise licensing, designed for data centers with dedicated Splunk admins. Overengineered for a 9-node homelab-scale k3s cluster.
- ELK: Elasticsearch full-text index is expensive on disk and CPU for this workload size. Kibana is a separate app to manage. Loki + Grafana is already 80% of the ELK surface with 20% of the operational overhead.
- Datadog: SaaS, per-host pricing, data leaves the private network. Not appropriate for this environment.

**Consequences:** LogQL is less familiar than SQL-like query languages. Mitigation: dashboard library covers the most common queries; power users can learn LogQL in a single session.

---

## ADR-004: MinIO In-Cluster as Loki Prod Object Storage

**Status:** Decided (revised 2026-04-23 — Synology NAS S3 not available on DS1618+ hardware)

**Context:** Need S3-compatible object storage for Loki prod log backend. Synology NAS S3 API was the original decision but is not supported on the DS1618+ hardware model.

**Decision:** MinIO in-cluster (single-instance StatefulSet, `monitoring` namespace, `bitnami/minio` Helm chart).

**Rationale:**
- Synology DS1618+ does not support the S3-compatible API (requires the S3 Server package available only on specific DiskStation models). Not viable.
- MinIO is the de-facto open-source S3-compatible object store for self-hosted k8s environments. Well-maintained `bitnami/minio` chart with ArgoCD multi-source support.
- Decouples Loki pod lifecycle from log storage: Loki pod can be rescheduled/restarted without data loss; storage persists on the MinIO StatefulSet PVC independently.
- Consistent with existing GitOps delivery pattern: ArgoCD multi-source app, `local-path` PVC, credentials in Vault + ESO.
- No external hardware dependency — entirely in-cluster.

**Accepted trade-off:** `local-path` PVC is node-local. MinIO data does not survive permanent loss of the node hosting the PVC. This is the same failure mode as Prometheus TSDB and is accepted for homelab scale. Mitigation: 30d retention means data age is bounded; no long-term cold archive requirement.

**Consequences:** Prod Loki depends on MinIO pod availability. If MinIO is down, Loki ingest fails (Alloy buffers briefly). MinIO adds one more pod to the `monitoring` namespace (minimal resource footprint: 100m CPU / 256Mi memory request).

---

## ADR-002: Alloy over Promtail as Cluster Collector

**Status:** Decided

**Context:** Need a DaemonSet log collector for the k3s cluster.

**Decision:** Grafana Alloy.

**Rationale:**
- Promtail is now maintenance-only in the Grafana ecosystem; Alloy is the supported path.
- Alloy ships with a built-in OpenTelemetry pipeline — this is the de facto direction for structured logs and traces. Adopting Alloy now avoids a future collector replacement.
- Helm chart is available and well-maintained (`grafana/alloy`).
- Config language (River/Alloy HCL) is more expressive than Promtail YAML for complex pipelines.

**Consequences:** Alloy config syntax is different from Promtail. Existing Promtail documentation doesn't apply. Mitigation: Grafana provides an Alloy config migration guide; most Loki-related Alloy configs are simple and well-documented.

---

## ADR-003: Single-Tenant Loki (No Multi-Tenancy in MVP)

**Status:** Decided

**Context:** Loki supports multi-tenancy via `X-Scope-OrgID` header.

**Decision:** Single-tenant mode for MVP.

**Rationale:**
- Single cluster, single operator (electricm0nk). All teams who need log access already have Grafana access.
- Multi-tenancy adds operational complexity (per-tenant retention policies, auth token management) without current benefit.
- Migration path is additive: enable multi-tenancy later by setting `auth_enabled: true` in Loki config and injecting tenant headers in Alloy. No data migration required.

**Consequences:** All Grafana users with Loki datasource access see all logs. Acceptable at current team size.
