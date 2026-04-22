---
feature: terminus-infra-centralized-logging
doc_type: stories
created_at: 2026-04-23T00:00:00Z
---

# Stories — terminus-infra-centralized-logging

12 stories across 6 epics. Each story maps to a deliverable merge unit.

---

## Epic E1 — Log Backend Foundation

### E1-S1: Synology S3 Credentials — Vault + ExternalSecret Setup

**As an** operator,  
**I want** S3 credentials for Synology NAS stored in Vault and materialized as a Kubernetes Secret,  
**so that** Loki can authenticate to the S3 backend without credentials in git.

**Acceptance Criteria:**
- [ ] Synology NAS S3 service account created; bucket `terminus-loki-prod` created
- [ ] Credentials stored in Vault KV at `secret/terminus/loki/s3-creds` (`access_key_id`, `secret_access_key`)
- [ ] ExternalSecret manifest at `platforms/k3s/k8s/monitoring/loki-s3-creds-es.yaml` created
- [ ] ExternalSecret syncs successfully — `loki-s3-creds` Secret present in `monitoring` namespace
- [ ] S3 endpoint reachable from within cluster: `curl http://nas.trantor.internal/` returns non-error

**Files to create/modify:**
- `platforms/k3s/k8s/monitoring/loki-s3-creds-es.yaml` (ExternalSecret manifest)

**Notes:** Confirm Synology NAS hostname before deploying. If `nas.trantor.internal` is incorrect, update `architecture.md` S3 endpoint field.

---

### E1-S2: Loki Prod Deployment — Helm Values + ArgoCD App

**As an** operator,  
**I want** Loki deployed in the `monitoring` namespace with Synology NAS S3 storage,  
**so that** cluster logs can be ingested and retained for 30 days with post-mortem durability.

**Acceptance Criteria:**
- [ ] `platforms/k3s/helm/loki/values.yaml` created — storage: s3, Synology endpoint, retention 30d, `compactor.retention_enabled: true`
- [ ] ArgoCD multi-source app manifest at `platforms/k3s/argocd/apps/loki.yaml` created (wave 3)
- [ ] `loki` pod healthy in `monitoring` namespace
- [ ] Manual log push test succeeds: `curl -XPOST http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push` with test payload
- [ ] Logs stored to Synology bucket — object visible in `terminus-loki-prod` bucket
- [ ] 30-day retention config active — `compactor` running; `GET /loki/api/v1/rules` returns HTTP 200

**Files to create/modify:**
- `platforms/k3s/helm/loki/values.yaml` (new)
- `platforms/k3s/argocd/apps/loki.yaml` (new)

**Dependencies:** E1-S1 complete (loki-s3-creds Secret present)

---

### E1-S3: Loki Dev Deployment — Helm Values + ArgoCD App (Upgrade Staging)

**As an** operator,  
**I want** a loki-dev instance in `monitoring-dev` with local-path PVC,  
**so that** I can safely test Loki chart upgrades against a non-production instance before applying to prod.

**Acceptance Criteria:**
- [ ] `platforms/k3s/helm/loki/values-dev.yaml` created — storage: filesystem, 20Gi local-path PVC
- [ ] ArgoCD app manifest at `platforms/k3s/argocd/apps/loki-dev.yaml` created (wave 3)
- [ ] `loki-dev` pod healthy in `monitoring-dev` namespace
- [ ] 20Gi PVC bound via `local-path` StorageClass
- [ ] loki-dev accepts log push (for synthetic data injection in upgrade testing)

**Files to create/modify:**
- `platforms/k3s/helm/loki/values-dev.yaml` (new)
- `platforms/k3s/argocd/apps/loki-dev.yaml` (new)

**Notes:** loki-dev is NOT wired to Alloy. It exists for upgrade staging only. No real log collection in dev.

---

## Epic E2 — Cluster Log Collection

### E2-S1: Alloy DaemonSet Deployment — Helm Values + ArgoCD App

**As an** operator,  
**I want** Grafana Alloy deployed as a DaemonSet on all k3s nodes,  
**so that** pod logs from all workloads are continuously collected and shipped to prod Loki.

**Acceptance Criteria:**
- [ ] `platforms/k3s/helm/alloy/values.yaml` created — DaemonSet mode, reads `/var/log/pods/`, ships to `loki.monitoring.svc.cluster.local:3100`
- [ ] Control-plane toleration set: `node-role.kubernetes.io/control-plane: NoSchedule`
- [ ] Label strategy applied: `cluster=terminus`, `namespace`, `app` (from `app.kubernetes.io/name`), `pod`, `container`, `node`, `env` (derived from namespace suffix)
- [ ] ArgoCD app manifest at `platforms/k3s/argocd/apps/alloy.yaml` created (wave 4)
- [ ] `alloy` DaemonSet healthy on all 9 nodes (3 CP + 6 workers)

**Files to create/modify:**
- `platforms/k3s/helm/alloy/values.yaml` (new)
- `platforms/k3s/argocd/apps/alloy.yaml` (new)

**Dependencies:** E1-S2 complete (prod Loki healthy)

---

### E2-S2: Alloy Log Stream Validation

**As an** operator,  
**I want** to verify log streams are flowing correctly from all namespaces to Loki,  
**so that** I'm confident no workloads are missing from the log pipeline.

**Acceptance Criteria:**
- [ ] LogQL query `{cluster="terminus", namespace="argocd"}` returns recent logs in Grafana/Loki API
- [ ] LogQL query `{cluster="terminus", namespace="monitoring"}` returns recent Prometheus/Grafana logs
- [ ] LogQL query `{cluster="terminus", env="dev"}` returns logs from `monitoring-dev` namespace
- [ ] Label cardinality: `{cluster, namespace, app, pod, container, node, env}` — no unexpected high-cardinality labels
- [ ] No Alloy error logs related to failed pushes to Loki in `kubectl logs -n monitoring ds/alloy`

**Files to create/modify:** (validation task — no new files expected; runbook notes go in story file)

**Dependencies:** E2-S1 complete

---

## Epic E3 — ArgoCD Observability Gap Fix

### E3-S1: ArgoCD ServiceMonitor Fix — Metrics and Namespace Label

**As an** operator,  
**I want** ArgoCD to expose Prometheus metrics via ServiceMonitors,  
**so that** the 3 existing ArgoCD alert rules and the ArgoCD delivery dashboard have data.

**Acceptance Criteria:**
- [ ] `platforms/k3s/helm/argocd/values.yaml` updated — `controller.metrics.enabled`, `server.metrics.enabled`, `repoServer.metrics.enabled`, `applicationSet.metrics.enabled` all true; each with `serviceMonitor.enabled: true` and `additionalLabels: {release: prometheus}`
- [ ] `argocd` namespace has label `observability.terminus.io/scrape: "true"` applied
- [ ] `argocd_app_info` metric visible in Prometheus UI
- [ ] `argocd_app_sync_total` metric visible in Prometheus UI
- [ ] No `kube-state-metrics` duplicate metric conflict warnings in Prometheus

**Files to create/modify:**
- `platforms/k3s/helm/argocd/values.yaml` (update)
- ArgoCD namespace manifest or `prometheus-infra.yaml` for namespace label (update)

**Notes:** Independent of Loki — can be done in Sprint 1 parallel with E1. No Loki dependency.

---

## Epic E4 — Grafana Integration

### E4-S1: Loki Grafana Datasource — Sidecar ConfigMaps

**As a** platform engineer,  
**I want** Loki available as a Grafana datasource in both prod and dev Grafana instances,  
**so that** I can run LogQL queries from the Grafana Explore view.

**Acceptance Criteria:**
- [ ] `platforms/k3s/k8s/monitoring/loki-datasource.yaml` created — ConfigMap with `grafana_datasource: "1"` label, pointing to `http://loki.monitoring.svc.cluster.local:3100`
- [ ] `platforms/k3s/k8s/monitoring-dev/loki-datasource-dev.yaml` created — pointing to `http://loki-dev.monitoring-dev.svc.cluster.local:3100`
- [ ] Loki datasource visible in Grafana prod `Configuration > Data Sources`
- [ ] Loki datasource visible in Grafana dev `Configuration > Data Sources`
- [ ] Grafana sidecar did not require manual restart (sidecar auto-picks up ConfigMap)

**Files to create/modify:**
- `platforms/k3s/k8s/monitoring/loki-datasource.yaml` (new)
- `platforms/k3s/k8s/monitoring-dev/loki-datasource-dev.yaml` (new)

**Dependencies:** E1-S2 complete (prod Loki healthy), E1-S3 complete (dev Loki healthy)

---

### E4-S2: Log Dashboards — Platform Log Search Dashboards

**As a** platform engineer,  
**I want** 5 Grafana dashboards for searching and exploring log data,  
**so that** I can quickly triage incidents using logs from ArgoCD, Temporal, fourdogs, platform services, and Vault.

**Acceptance Criteria:**
- [ ] `platforms/k3s/k8s/prometheus-infra/dashboard-logs-argocd.yaml` created and visible in Grafana
- [ ] `platforms/k3s/k8s/prometheus-infra/dashboard-logs-temporal.yaml` created and visible in Grafana
- [ ] `platforms/k3s/k8s/prometheus-infra/dashboard-logs-fourdogs.yaml` created and visible in Grafana
- [ ] `platforms/k3s/k8s/prometheus-infra/dashboard-logs-platform.yaml` created and visible in Grafana (Traefik, ESO, cert-manager)
- [ ] `platforms/k3s/k8s/prometheus-infra/dashboard-logs-vault.yaml` created and visible in Grafana
- [ ] Each dashboard uses Loki datasource and LogQL queries
- [ ] Dashboard ConfigMaps have `grafana_dashboard: "1"` label; picked up by sidecar without manual restart

**Files to create/modify:**
- 5 x `platforms/k3s/k8s/prometheus-infra/dashboard-logs-*.yaml` (new)

**Dependencies:** E4-S1 complete (Loki datasource available in Grafana), E2-S2 complete (log streams validated)

---

## Epic E5 — Alert Rules and Routing

### E5-S1: Loki Ruler Rules — Log-Based Alert Rules in values.yaml

**As an** operator,  
**I want** 5 log-based alert rules evaluated by the Loki ruler,  
**so that** I receive alerts for CrashLoops, OOMKills, Vault auth denials, Temporal panics, and ArgoCD errors without needing Prometheus metric exporters.

**Acceptance Criteria:**
- [ ] `loki.ruler.enabled: true` in `platforms/k3s/helm/loki/values.yaml`
- [ ] `loki.rulesConfig` block populated with all 5 rules: CrashLoopDetected, OOMKillDetected, VaultAuditDeny, TemporalWorkflowPanic, ArgoCDSyncError
- [ ] `CrashLoopDetected` uses `|= "reason=CrashLoopBackOff"` filter (not plain string match)
- [ ] Ruler rules visible via `GET /loki/api/v1/rules` on Loki API
- [ ] `CrashLoopDetected` fires within 5 minutes when a test pod is put into CrashLoopBackOff state

**Files to create/modify:**
- `platforms/k3s/helm/loki/values.yaml` (update — add ruler and rulesConfig sections)

**Dependencies:** E1-S2 complete (Loki running), E5-S2 (AlertmanagerConfig) can proceed in parallel

---

### E5-S2: Alertmanager Routing for Loki Alerts — AlertmanagerConfig CRD

**As an** operator,  
**I want** Loki ruler alerts to route to Alertmanager,  
**so that** log-based alerts follow the same notification path as Prometheus alerts.

**Acceptance Criteria:**
- [ ] `platforms/k3s/k8s/monitoring/loki-alertmanager-config.yaml` created — `AlertmanagerConfig` CRD with a receiver matching Loki ruler namespace
- [ ] Alertmanager `alertmanagerConfigSelector` picks up the CRD (verify or update `kube-prometheus-stack` values to set selector)
- [ ] Test alert from Loki ruler appears in Alertmanager `/api/v2/alerts` endpoint
- [ ] No conflict with any existing `AlertmanagerConfig` resources
- [ ] **Coordination gate**: `prometheus-wiring` feature stories reviewed for `alertmanagerConfigSelector` conflict before this story is closed; coordination note added if conflict found

**Files to create/modify:**
- `platforms/k3s/k8s/monitoring/loki-alertmanager-config.yaml` (new)
- Possibly `platforms/k3s/helm/kube-prometheus-stack/values.yaml` (update if `alertmanagerConfigSelector` not set)

**Dependencies:** Alertmanager running (prometheus feature, complete), E5-S1 (ruler rules active)

**Open item (H5):** Coordinate with `prometheus-wiring` before merging this story. If `prometheus-wiring` has entered dev by then, review their Alertmanager config for conflicts.

---

## Epic E6 — VM Log Shipping

### E6-S1: Alloy VM Ansible Playbook — Vault and Postgres VM Log Shipping

**As an** operator,  
**I want** Alloy installed on Vault VM and Postgres VM as systemd services,  
**so that** Vault audit logs and Postgres query logs are shipped to prod Loki via `loki-push.trantor.internal`.

**Acceptance Criteria:**
- [ ] `platforms/vm/alloy/config.alloy.vault.j2` created — Vault audit log source, ships to `http://loki-push.trantor.internal/loki/api/v1/push`, labels `service=vault, log_type=audit`
- [ ] `platforms/vm/alloy/config.alloy.postgres.j2` created — Postgres log source, ships to `http://loki-push.trantor.internal/loki/api/v1/push`, labels `service=postgres, log_type=db`
- [ ] `platforms/vm/alloy/deploy-alloy-vm.yaml` Ansible playbook created — installs Alloy binary, renders config, enables `alloy.service` systemd unit
- [ ] Alloy version pinned in playbook — no auto-update on playbook re-run
- [ ] Alloy position file path configurable (default: `/var/lib/alloy/positions.yaml`)
- [ ] Playbook is idempotent — re-running does not duplicate log entries

**Files to create/modify:**
- `platforms/vm/alloy/config.alloy.vault.j2` (new)
- `platforms/vm/alloy/config.alloy.postgres.j2` (new)
- `platforms/vm/alloy/deploy-alloy-vm.yaml` (new Ansible playbook)

**Dependencies:** E1-S2 complete (Loki healthy); Traefik ingress `loki-push.trantor.internal` created (included in E1-S2 scope or separate story task)

**Notes:** `loki-push.trantor.internal` ingress must be created pointing at the Loki push port. Add as a task within E1-S2 or as a separate task in this story if not already covered.

---

### E6-S2: VM Log Stream Validation and File Permission Hardening

**As an** operator,  
**I want** to validate VM log streams in Loki and harden Vault audit log file permissions,  
**so that** VM logs are confirmed flowing and the audit log access surface is minimized.

**Acceptance Criteria:**
- [ ] LogQL query `{service="vault", log_type="audit"}` returns Vault audit events in Grafana (post-deployment)
- [ ] LogQL query `{service="postgres", log_type="db"}` returns Postgres log entries in Grafana
- [ ] Alloy file position persists across service restart: `sudo systemctl restart alloy && sleep 30` — no duplicate log entries in Loki
- [ ] Vault audit log file permissions: `chmod 640 /var/log/vault/audit.log` and `chown vault:alloy /var/log/vault/audit.log` applied via Ansible playbook
- [ ] `alloy` system user is a member of the `vault` group OR vault audit log is group-readable by `alloy` group

**Files to create/modify:**
- Update `platforms/vm/alloy/deploy-alloy-vm.yaml` to include file permission hardening task

**Dependencies:** E6-S1 complete (Alloy deployed on both VMs)
