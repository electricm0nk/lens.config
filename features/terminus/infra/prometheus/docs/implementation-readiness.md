---
feature: prometheus
doc_type: implementation-readiness
status: pass-with-conditions
updated_at: "2026-04-15T01:15:00Z"
---

# Prometheus Monitoring — Implementation Readiness Report

## Summary

**Verdict: READY WITH PRE-DEPLOY CONDITIONS**

Planning artifacts are complete and approved. Two implementation assumptions (OQ-3 resolved, OQ-5 known risk) are carried forward as story-level notes. One pre-deploy environment condition must be verified before Story 1-1 sync.

---

## Artifacts Reviewed

| Artifact | Status |
|----------|--------|
| `docs/terminus/infra/prometheus/prd.md` | ✅ Complete |
| `docs/terminus/infra/prometheus/ux-design.md` | ✅ Complete |
| `docs/terminus/infra/prometheus/architecture.md` | ✅ Complete |
| `docs/terminus/infra/prometheus/businessplan-adversarial-review.md` | ✅ pass-with-warnings |
| `docs/terminus/infra/prometheus/techplan-adversarial-review.md` | ✅ pass-with-warnings |
| `docs/terminus/infra/prometheus/finalizeplan-review.md` | ✅ pass-with-warnings |
| `docs/terminus/infra/prometheus/epics.md` | ✅ Complete |

---

## Pre-Deploy Environment Checklist

These conditions must be met before Story 1-1 (ArgoCD sync) is attempted:

- [ ] **`influxdb-infra` ArgoCD app is `Healthy/Synced`** — the `monitoring` namespace is owned by `influxdb-infra`; prometheus-infra requires it to already exist
- [ ] **`monitoring` namespace exists on the cluster** — `kubectl get ns monitoring` returns `Active`
- [ ] **ArgoCD has access to `prometheus-community` Helm repo** — verify `argocd repo list` includes `https://prometheus-community.github.io/helm-charts`
- [ ] **`terminus.infra` git repo is accessible from ArgoCD** — multi-source app uses repo ref for values file; verify ArgoCD can reach `git@github.com:electricm0nk/terminus.infra.git`

---

## Open Questions Carried Forward

### OQ-3: Dashboard JSON Authorship (Medium)

**Status:** Resolved via implementation convention  
**Resolution:** PromQL authored per UX spec panel-by-panel; community dashboard 1860 (Node Exporter Full) used as PromQL reference for node metrics. Custom dashboard UIDs assigned to avoid clash with `forceDeployDashboards` built-in dashboards.  
**ACs defined in:** Story 2-1 and Story 2-2

### OQ-5: CRD Sync Race on First Deploy (Low)

**Status:** Known risk, mitigated by wave ordering  
**Risk:** kube-prometheus-stack CRDs (ServiceMonitor, PodMonitor, PrometheusRule) must exist before ArgoCD syncs CRD-dependent resources. On first deploy, CRD apply and CR apply could race.  
**Mitigation:** ArgoCD `wave-2` on `prometheus` app provides a natural sequencing gap after `prometheus-infra` wave-1. If CRD race is observed on first sync, re-sync the `prometheus` app manually; on subsequent syncs it self-heals.  
**ACs defined in:** Story 1-2 post-deploy validation

---

## Architecture Assumptions Verified

| Assumption | Source | Status |
|-----------|--------|--------|
| Grafana datasource pre-wired at `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090` | `grafana/values.yaml` inspection | ✅ Confirmed |
| `monitoring` namespace owned by `influxdb-infra` — no namespace.yaml in prometheus-infra | Architecture ADR-001 + infra review | ✅ Confirmed |
| `grafana.enabled: false` + `forceDeployDashboards: true` combo supported by chart | ADR-004 implementation note | ⚠️ Assumption — verify at Story 1-1 |
| emptyDir TSDB acceptable for MVP (disk pressure not a concern) | User confirmed in session | ✅ Confirmed |
| Grafana sidecar watchMethod: dynamic (no restart needed for ConfigMap dashboards) | User confirmed: separate Grafana container | ✅ Confirmed |
| Dev environment out of scope | User decision | ✅ Final |

---

## Risk Register

| Risk | Severity | Mitigation | Owner Story |
|------|----------|-----------|-------------|
| `grafana.enabled: false` + `forceDeployDashboards: true` may not emit built-in dashboards | Medium | Verify chart behavior at Story 1-1; fallback: deploy sidecar-only ConfigMap dashboards only | Story 1-1 |
| CRD sync race on first ArgoCD apply | Low | Wave ordering + manual re-sync if needed | Story 1-2 |
| HTTP Error Rate panel shows no data if workloads don't expose `http_requests_total` | Low | Empty state acceptable by design (ACs specify graceful no-data) | Story 2-2 |
| node-exporter DaemonSet pods may fail to schedule on control-plane nodes without tolerations | Low | `operator: Exists` toleration in values.yaml handles tainted nodes | Story 1-2 |

---

## Implementation Order

```
Sprint 1
├── Story 1-1: ArgoCD apps + Helm values (M) ← can start immediately after pre-deploy checklist
├── Story 1-2: ArgoCD sync + target verification (S) ← depends on 1-1
├── Story 1-3: Grafana datasource activation (XS) ← depends on 1-1 (parallel with 1-2 ok)
├── Story 2-1: k3s Infrastructure Health dashboard (M) ← depends on 1-1 + 1-2
└── Story 2-2: Workload Health dashboard (M) ← depends on 1-1 + 1-2 (parallel with 2-1 ok)
```

**Blocking path:** 1-1 → 1-2 → 2-1, 2-2

---

## Sign-off

| Phase | Status | Notes |
|-------|--------|-------|
| Business Plan | ✅ Approved | pass-with-warnings |
| Tech Plan | ✅ Approved | pass-with-warnings |
| Finalize Plan | ✅ Approved | pass-with-warnings — F1 (OQ-3) resolved in stories |
| Implementation Readiness | ✅ READY | Pre-deploy conditions must be met before Story 1-1 |
