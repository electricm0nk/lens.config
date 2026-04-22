# Adversarial Review: terminus-infra-centralized-logging / techplan

**Reviewed:** 2026-04-22T18:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

## Summary

The architecture is sound and well-scoped. Two critical findings (double-ship DaemonSet design and silent Loki retention failure) were resolved during the review session by eliminating `alloy-dev` and adding `compactor.retention_enabled: true`. Three significant scope additions emerged: Synology NAS S3 as the prod storage backend (replacing the planned local-path PVC), Alertmanager deployment (never installed — required for all alert delivery), and a pre-condition Phase 0 in the implementation sequence. All high and medium findings were either resolved, accepted with documented rationale, or converted to tracked open items. No blockers remain. The architecture can advance to FinalizePlan with the remaining medium/low findings documented below.

## Findings

### Critical — All Resolved During Review

| # | Dimension | Finding | Resolution |
|---|-----------|---------|------------|
| C1 | Coverage Gaps | `alloy-dev` DaemonSet would double-ship all logs to both Loki instances. No defined purpose different from `alloy`. | **Resolved** — `alloy-dev` eliminated. Single `alloy` DaemonSet ships to prod Loki only. `loki-dev` populated via synthetic data injection for upgrade staging. Architecture updated. |
| C2 | Coverage Gaps | `compactor.retention_enabled: true` absent from Loki values. Without this flag, `limits_config.retention_period` is silently ignored and PVC fills until crash. | **Resolved** — flag added to `loki/values.yaml`. Architecture updated. |

### High — All Resolved or Accepted

| # | Dimension | Finding | Resolution |
|---|-----------|---------|------------|
| H1 | Logic Flaws | Alertmanager is target of Loki ruler alerts but was never deployed. No delivery path for any alert rule. | **Resolved** — Alertmanager deployment added to feature scope. Enabled via existing kube-prometheus-stack Helm values in Phase 0. Architecture updated. |
| H2 | Coverage Gaps | VM Alloy routing: `loki-push-dev.trantor.internal` vs `loki-push.trantor.internal` split requires two Alloy configs per VM with no documented purpose. | **Resolved** — VMs ship to prod Loki only via `loki-push.trantor.internal`. `*.trantor.internal` wildcard DNS resolves to `10.0.0.126`. Single ingress route needed. Architecture updated. |
| H3 | Coverage Gaps | `local-path` PVC for prod Loki: node failure = up to 30 days of log data loss. No RTO/RPO documented. Vault audit logs at risk. | **Resolved** — scope expanded. Prod Loki uses Synology NAS S3 backend. Remaining risk (Synology is single device, no HA) explicitly accepted. See Accepted Risks. Architecture updated. |
| H4 | Complexity / Risk | Prometheus CrashLoopBackOff blocks ArgoCD ServiceMonitor fix and Alertmanager. No tracked owner. | **Resolved** — Prometheus CrashLoopBackOff confirmed fixed. Alertmanager deployment added to this feature's scope as Phase 0 pre-condition. |
| H5 | Cross-Feature | `prometheus-wiring` (finalizeplan-complete) defines Alertmanager routing. Loki ruler alerts may be silently dropped if no route matches. | **Open — tracked.** Phase 6 step 2 explicitly requires coordination with `prometheus-wiring` Alertmanager routing config before enabling ruler alerts. |
| H6 | Assumptions | Alloy `loki.source.kubernetes_logs` compatibility with containerd 2.0.4 on Ubuntu 24.04 not verified. | **Open — tracked.** Added as Phase 2 verification step. Risk is low (Alloy 1.x is tested against containerd 2.x) but must be confirmed during deploy. |

### Medium / Low

| # | Severity | Dimension | Finding | Status |
|---|----------|-----------|---------|--------|
| M1 | Medium | Logic Flaws | Phase 6 originally referenced "PrometheusRule equivalent for Loki ruler" — Loki ruler does not use PrometheusRule CRDs. | **Resolved** — architecture updated to use `loki.rulesConfig` inline in values.yaml. |
| M2 | Medium | Coverage Gaps | Vault audit log file permissions not included as an implementation step (only mentioned in security section). | **Tracked** — added explicitly to Phase 5 step 3a: verify and tighten `/var/log/vault/audit.log` permissions so only `vault` and `alloy` users can read. |
| M3 | Medium | Coverage Gaps | Postgres VM hostname was TBD. | **Resolved** — `postgres.primary.trantor.internal`. Architecture updated. |
| M4 | Medium | Coverage Gaps | No self-monitoring for Alloy — log pipeline failure unobservable. | **Accepted** — out of scope for this feature. A `kube_daemonset_status_number_ready` alert for the Alloy DaemonSet is tracked as a follow-up item for a monitoring hygiene feature. |
| M5 | Medium | Complexity / Risk | `alloy-dev` underspecification creates implementation ambiguity. | **Resolved** — eliminated (see C1). |
| M6 | Medium | Assumptions | Grafana dashboard sidecar (`grafana_dashboard: "1"`) not verified against deployed Grafana version. | **Open — tracked.** Phase 3 step 1a: verify sidecar is configured for dashboard pickup in the existing Grafana values before creating dashboard ConfigMaps. |
| M7 | Medium | Assumptions | No disk availability check on the node where Loki schedules. 100Gi PVC on insufficient disk won't bind. | **N/A** — moot. Prod Loki uses Synology S3; no PVC on cluster nodes. Dev Loki uses 20Gi local-path (upgrade staging only). |
| L1 | Low | Coverage Gaps | No baseline log ingest rate estimate to validate storage sizing. | **Accepted** — Synology S3 removes the hard sizing constraint. Retention enforcement is time-based (30d). Monitor ingest rate after Phase 2 deploy via Loki metrics. |
| L2 | Low | Complexity / Risk | Phase 4 (ArgoCD ServiceMonitor) pre-condition (Prometheus healthy) not reflected in phase steps. | **Resolved** — Phase 4 now includes explicit pre-condition check. |
| L3 | Low | Assumptions | `CrashLoopDetected` LogQL `\|= "CrashLoopBackOff"` prone to false positives. | **Resolved** — changed to `\|= "reason=CrashLoopBackOff"` in architecture and rulesConfig. |

## Accepted Risks

| Risk | Rationale |
|------|-----------|
| Synology NAS is a single device with no off-site replication. Extended NAS downtime = Loki cannot ingest. Short downtime (~minutes) is buffered by Alloy. | Explicitly accepted for homelab-scale environment. Synology is more fault-tolerant than cluster-local storage for the primary use case (post-mortem after trantor failure). |
| Vault audit logs in Loki are best-effort: no zero-loss guarantee, no HA. | Explicitly accepted. These are homelab audit logs, not compliance-regulated records. |
| VM-side log shipping (Vault, Postgres) is best-effort: no alert if Alloy agent on VM stops sending. | Explicitly accepted. Self-monitoring follow-up tracked separately. |

## Party-Mode Challenge

> Round conducted during review session.

**Stavros (Platform SRE):** The `alloy-dev` DaemonSet thing is the kind of thing that gets merged in a PR on a Friday and no one notices until Monday when the log ingest rate is double. A single Alloy with two push targets is cleaner — or just one target. Glad we caught it.

**Ilaria (Security Engineer):** The Vault audit log goes to Loki, stored on a single Synology, with no row-level access control. If the answer is "this is homelab-scale and best-effort is fine," write that down and we can move on. It's written down now. Good.

**Dmitri (Monitoring Lead):** Who alerts on the Alloy crash loop? Who alerts on "Loki hasn't received any logs in 10 minutes"? Your observability stack's own health is completely unmonitored. This is accepted out of scope for now — but it should be the first thing on the monitoring hygiene backlog after this ships.

**Blind-spot challenge questions asked and answered:**
1. `alloy-dev` purpose → eliminated ✓
2. Alertmanager health → Prometheus fixed; Alertmanager added to scope ✓
3. Vault audit log compliance posture → best-effort explicitly accepted ✓
4. `compactor.retention_enabled` → added to values ✓
5. VM Alloy routing → prod Loki only; wildcard DNS confirmed ✓

## Verdict

**pass-with-warnings**

The two Critical findings are resolved. Architecture is updated and internally consistent. Remaining open items (H5 Alertmanager routing coordination, H6 containerd compatibility verification, M6 Grafana sidecar check) are all verification steps during implementation, not architecture design gaps. Phase 0 (Synology S3 + Alertmanager) is a new sequencing requirement that must complete before Phase 1. FinalizePlan may proceed.
