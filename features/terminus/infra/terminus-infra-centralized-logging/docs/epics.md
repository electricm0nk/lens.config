---
feature: terminus-infra-centralized-logging
doc_type: epics
created_at: 2026-04-23T00:00:00Z
---

# Epics — terminus-infra-centralized-logging

## Overview

6 epics map to the 7-phase implementation sequence in `architecture.md`. Phases 0 and 1 are combined into Epic 1 since Phase 0 prerequisites gate Phase 1 and have no standalone value.

| Epic | Title | Phases | Stories | Status |
|------|-------|--------|---------|--------|
| E1 | Log Backend Foundation | 0 + 1 | 3 | not-started |
| E2 | Cluster Log Collection | 2 | 2 | not-started |
| E3 | ArgoCD Observability Gap Fix | 3 | 1 | not-started |
| E4 | Grafana Integration | 4 + 5 | 2 | not-started |
| E5 | Alert Rules and Routing | 6 | 2 | not-started |
| E6 | VM Log Shipping | 7 | 2 | not-started |

---

## Epic E1 — Log Backend Foundation

**Goal:** Deploy MinIO in-cluster as S3-compatible object storage, then deploy Loki (prod + dev). Prod Loki uses MinIO S3 backend (100Gi local-path PVC, `terminus-loki-prod` bucket). Dev Loki uses 20Gi local-path PVC (upgrade staging only). Includes all Phase 0 prerequisites (MinIO ArgoCD app, Vault credentials, ExternalSecret).

**Acceptance Criteria:**
- `minio` pod healthy in `monitoring` namespace; `terminus-loki-prod` bucket exists
- `loki` pod healthy in `monitoring` namespace; logs ingested from manual push test
- `loki-dev` pod healthy in `monitoring-dev` namespace; 20Gi PVC bound
- `loki-s3-creds` ExternalSecret materializes successfully in `monitoring` namespace
- ArgoCD apps `minio`, `loki`, and `loki-dev` sync cleanly with no drift

**Dependencies:** Vault and ESO operational (verified by `secrets` feature)

**Stories:**
- E1-S1: MinIO Deploy + S3 Credentials — ArgoCD app + Vault + ExternalSecret
- E1-S2: Loki Prod Deployment — Helm values + ArgoCD app
- E1-S3: Loki Dev Deployment — Helm values + ArgoCD app (upgrade staging)

---

## Epic E2 — Cluster Log Collection

**Goal:** Deploy Grafana Alloy as a DaemonSet on all k3s nodes. Alloy reads pod logs from `/var/log/pods/` and ships them to prod Loki. All workload namespaces covered; label strategy applied.

**Acceptance Criteria:**
- Alloy DaemonSet running on all nodes (control-plane + workers) — no evictions
- Log streams visible in Loki with correct `cluster`, `namespace`, `app`, `pod`, `container`, `env` labels
- No label cardinality explosion (max ~6 labels per stream)
- ArgoCD app `alloy` syncs cleanly

**Dependencies:** E1 complete (Loki healthy and accepting pushes)

**Stories:**
- E2-S1: Alloy DaemonSet Deployment — Helm values + ArgoCD app
- E2-S2: Alloy Log Stream Validation — verify label strategy and log coverage

---

## Epic E3 — ArgoCD Observability Gap Fix

**Goal:** Enable Prometheus metrics scraping for all ArgoCD components (controller, server, repoServer, applicationSet) via ServiceMonitors in ArgoCD Helm values. Enables the 3 existing ArgoCD alert rules and the ArgoCD delivery dashboard.

**Acceptance Criteria:**
- `argocd_app_info`, `argocd_app_sync_total` visible in Prometheus
- All 3 existing ArgoCD PrometheusRules (`ArgoCDAppDegraded`, `ArgoCDAppOutOfSync`, `ArgoCDSyncFailureRateHigh`) in ALERTS state (firing or pending with no-fire when healthy)
- ArgoCD Delivery Health dashboard functional in Grafana

**Dependencies:** Prometheus operational (prometheus feature, complete)

**Stories:**
- E3-S1: ArgoCD ServiceMonitor Fix — values.yaml update + namespace label

---

## Epic E4 — Grafana Integration

**Goal:** Add Loki as a Grafana datasource (via sidecar ConfigMap) for both prod and dev, and create 5 log dashboards covering the key platform components.

**Acceptance Criteria:**
- Loki datasource visible in Grafana (both prod and dev Grafana instances)
- 5 log dashboards deployed and showing data: ArgoCD, Temporal, fourdogs, platform (Traefik/ESO/cert-manager), Vault
- No sidecar reload required — ConfigMap pickup is automatic

**Dependencies:** E1 and E2 complete (Loki healthy and logs flowing)

**Stories:**
- E4-S1: Loki Grafana Datasource — sidecar ConfigMaps for prod and dev
- E4-S2: Log Dashboards — 5 dashboard ConfigMaps

---

## Epic E5 — Alert Rules and Routing

**Goal:** Enable Loki ruler with 5 log-based alert rules (CrashLoopDetected, OOMKillDetected, VaultAuditDeny, TemporalWorkflowPanic, ArgoCDSyncError). Add `AlertmanagerConfig` CRD for Loki ruler alert routing. Coordinate with `prometheus-wiring`.

**Acceptance Criteria:**
- Loki ruler rules visible in Loki ruler API (`/loki/api/v1/rules`)
- `CrashLoopDetected` rule fires when a test pod is put into CrashLoopBackOff
- Alerts route to Alertmanager (visible in Alertmanager `/api/v2/alerts`)
- `AlertmanagerConfig` CRD does not conflict with `prometheus-wiring` Alertmanager baseline (coordination verified)

**Dependencies:** E1 complete (Loki ruler requires Loki), Alertmanager running (verified by `prometheus` feature)

**Open item:** Coordinate with `prometheus-wiring` feature before implementing AlertmanagerConfig CRD — ensure `alertmanagerConfigSelector` includes our route namespace (H5 from techplan review)

**Stories:**
- E5-S1: Loki Ruler Rules — `rulesConfig` in values.yaml
- E5-S2: Alertmanager Routing for Loki Alerts — `AlertmanagerConfig` CRD

---

## Epic E6 — VM Log Shipping

**Goal:** Install Grafana Alloy on Vault VM and Postgres VM as systemd services. Ship Vault audit logs and Postgres query logs to prod Loki via Traefik internal ingress (`loki-push.trantor.internal`). Delivery via Ansible playbook from SemaphoreUI.

**Acceptance Criteria:**
- Alloy running as systemd service on `vault.trantor.internal` and `postgres.primary.trantor.internal`
- Vault audit log streams visible in Loki with labels `service=vault, log_type=audit`
- Postgres log streams visible in Loki with labels `service=postgres, log_type=db`
- Alloy file position persisted across service restarts (no duplicate log shipping on restart)
- Vault audit log file permissions set so only `vault` and `alloy` users can read

**Dependencies:** E1 complete (Loki healthy with Traefik ingress on `loki-push.trantor.internal`)

**Stories:**
- E6-S1: Alloy VM Ansible Playbook — Vault VM and Postgres VM configuration templates
- E6-S2: VM Log Stream Validation — verify streams in Loki, verify file permissions

---

## Sprint Allocation Suggestion

| Sprint | Epics | Rationale |
|--------|-------|-----------|
| Sprint 1 | E1 + E3 | Backend foundation + quick ArgoCD fix (independent; E3 has no Loki dependency) |
| Sprint 2 | E2 + E4 | Collector + Grafana integration (depends on E1) |
| Sprint 3 | E5 + E6 | Alerts + VM shipping (E5 depends on E1; E6 depends on E1) |
