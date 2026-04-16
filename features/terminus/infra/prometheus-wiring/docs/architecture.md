---
feature: prometheus-wiring
doc_type: architecture
status: draft
goal: Deliver a GitOps-managed Prometheus monitoring stack on k3s with secure service discovery, actionable alerts, and deterministic release/rollback flows.
key_decisions:
  - Standardize on kube-prometheus-stack Helm chart managed by ArgoCD for both dev and prod overlays.
  - Route all monitoring stack rollouts through Temporal ReleaseWorkflow activities that trigger and gate ArgoCD reconciliation.
  - Use ServiceMonitor and PodMonitor CRDs as the default scrape onboarding interface; avoid ad hoc static job definitions for in-cluster workloads.
  - Keep Prometheus and Alertmanager internal (ClusterIP); expose Grafana via Traefik ingress with cert-manager issued TLS for *.trantor.internal.
  - Store alert receiver secrets in Vault and materialize via External Secrets Operator; no secret values in git.
  - Start with local-path StorageClass retention (dev 15d, prod 30d) and document migration path to durable storage class when available.
open_questions:
  - Should long-term metrics retention be added via remote_write (VictoriaMetrics/Mimir/Thanos) in this feature or deferred to follow-up?
  - Do we require multi-cluster federation in scope, or single-cluster Terminus only for this phase?
  - Should Alertmanager route to email only initially, or include Slack/PagerDuty endpoints at launch?
depends_on:
  - docs/terminus/infra/k3s/architecture.md
  - docs/terminus/infra/secrets/architecture.md
  - docs/terminus/architecture-scrubbed.md
blocks:
  - Dashboard and SLO automation for platform services that currently lack standardized metrics labels
updated_at: 2026-04-16T14:25:56Z
---

# Prometheus Wiring Architecture (Terminus Infra)

## Problem Statement And Scope

Terminus currently has partial observability capabilities (service-local checks and ad hoc verification), but lacks a standardized, cluster-native metrics and alerting substrate. This creates three operational gaps:

1. Failures are detected late or manually.
2. Service health and release risk are not visible through consistent SLO dashboards.
3. New workloads onboard monitoring inconsistently, increasing maintenance cost.

This feature wires Prometheus-based monitoring into Terminus infra for k3s environments, including discovery, scrape, alerting, dashboards, and rollout mechanics.

In scope:
- Monitoring stack deployment and GitOps wiring.
- Cluster/service scrape onboarding contract.
- Alertmanager secret handling and routing baseline.
- SLO dashboard strategy and initial alert catalog.
- Validation, rollback, and phased implementation plan.

Out of scope:
- Full long-term metrics archive platform (remote_write backend).
- Enterprise multi-cluster federation.
- Replacement of existing logging/audit pipelines.

## Current State And Constraints

### Platform Constraints

- Runtime substrate is k3s on Proxmox.
- Cluster delivery is GitOps via ArgoCD.
- TLS is issued by cert-manager using Vault PKI.
- Internal DNS domain patterns include trantor.internal.
- Release sequencing preference is Temporal ReleaseWorkflow first, with ArgoCD sync triggered and gated by workflow activities.
- Normal operations must not rely on direct kubectl mutation as a deployment mechanism.

### Design Constraints

- No plaintext credentials in repository.
- Must support separate dev and prod overlays.
- Must remain compatible with existing namespace and label conventions used by ArgoCD app-of-apps.
- Must provide an onboarding path for any service exposing Prometheus format metrics.

## Architecture Overview

### High-Level Components

1. ArgoCD Applications
- monitoring-infra (base operators/charts)
- monitoring-dev overlay
- monitoring overlay

2. Helm chart deployment
- kube-prometheus-stack in each environment
- values files define retention, resources, ingress, and alert routing

3. Discovery contracts
- ServiceMonitor and PodMonitor CRDs are the primary in-cluster discovery mechanism
- PrometheusRule CRDs hold SLO and infra alert policies

4. Visualization and routing
- Grafana for dashboarding
- Alertmanager for alert fanout

5. Secret path
- Vault -> ExternalSecret -> Kubernetes Secret -> Alertmanager/Grafana references

6. Release orchestration path
- Temporal ReleaseWorkflow activity triggers ArgoCD sync for monitoring apps
- Workflow waits for sync+health before marking release step successful

### Data Flow

1. Workloads expose /metrics endpoints.
2. ServiceMonitor/PodMonitor objects select endpoints by labels.
3. Prometheus scrapes targets, stores series in TSDB PVC.
4. Prometheus evaluates alert rules.
5. Alerts are forwarded to Alertmanager.
6. Alertmanager routes to configured receivers.
7. Grafana reads Prometheus data for SLO dashboards.

### Control Flow

1. Git commit updates chart/values/rules.
2. Temporal ReleaseWorkflow invokes ArgoCD sync activity for monitoring apps.
3. ArgoCD reconciles manifests in wave order.
4. Temporal waits for app healthy status and smoke checks.
5. Release is completed or rolled back based on workflow outcome.

## Deployment And Integration Design

### Repository And Layout (Target)

Expected infra repository paths:
- platforms/k3s/argocd/apps/prometheus.yaml
- platforms/k3s/argocd/apps/prometheus-dev.yaml
- platforms/k3s/helm/kube-prometheus-stack/values.yaml
- platforms/k3s/helm/kube-prometheus-stack/values-dev.yaml
- platforms/k3s/helm/kube-prometheus-stack/rules/
- platforms/k3s/helm/kube-prometheus-stack/dashboards/

### Service Discovery Strategy

Primary:
- ServiceMonitor for stable service endpoints.
- PodMonitor for daemonsets/sidecars where service abstraction is unsuitable.

Label contract:
- Namespace opt-in label: observability.terminus.io/scrape=true
- Service label: app.kubernetes.io/name=<service>
- Metrics port naming convention: metrics

Prometheus selectors:
- serviceMonitorSelector and podMonitorSelector match release-scoped labels to avoid cross-environment contamination.

Static scrape configs:
- Allowed only for non-Kubernetes endpoints with explicit review (for example external VM exporters).

### Scrape And Rule Configuration Baseline

Global scrape interval:
- prod: 30s
- dev: 60s

Evaluation interval:
- prod: 30s
- dev: 60s

Rule categories:
- Platform health: API server errors, node pressure, pod crash loops.
- ArgoCD delivery health: app degraded/out-of-sync.
- Temporal release health: workflow activity failure and queue saturation.
- Service SLO burn alerts: latency and availability fast/slow burn pairs.

### Retention And Storage Assumptions

Assumed StorageClass:
- local-path (k3s default), RWO PVC.

Retention:
- dev: 15d
- prod: 30d

TSDB sizing assumptions:
- Initial cardinality target < 2M active series per env.
- Start with conservative requests/limits and tune after 2 weeks of real load.

Future-compatible extension:
- Reserve config surface for remote_write but keep disabled in initial rollout.

## Security And Secrets Handling

1. Secret sources
- Alert receiver credentials and webhook URLs originate in Vault.
- ESO syncs them into namespaced Kubernetes Secrets.

2. Exposure model
- Prometheus and Alertmanager remain ClusterIP only.
- Grafana exposed via Traefik ingress on trantor.internal with cert-manager TLS.

3. Access control
- Grafana admin bootstrap secret from Vault.
- RBAC-scoped service accounts for Prometheus operator and scrape jobs.
- NamespaceNetworkPolicy restricts ingress to monitoring components where feasible.

4. Data safety
- Disable anonymous Grafana access.
- Encrypt transport for exposed ingress endpoints.
- Prevent secret material from appearing in ConfigMaps, values files, or logs.

## Observability SLOs, Alerts, And Dashboard Strategy

### SLO Baseline

1. Monitoring pipeline availability
- SLO: 99.9% successful target scrape over rolling 30 days for tier-1 services.

2. Alert delivery pipeline
- SLO: 99.5% alert notification success to primary channel over rolling 30 days.

3. Dashboard freshness
- SLO: P95 query freshness <= 90 seconds for golden signal dashboards.

### Alert Strategy

- Multi-window multi-burn for availability and latency SLOs.
- Distinct severities: warning and critical.
- Dead-man switch alert to validate routing path continuously.
- Suppression and grouping by service, env, severity to reduce noise.

### Dashboard Strategy

Tier 1 dashboards:
- Cluster health
- ArgoCD sync and drift
- Temporal release workflow health
- Service golden signals (latency, traffic, errors, saturation)

Operational standards:
- Dashboards provisioned from git, not manually edited in UI.
- Each dashboard panel maps to an owning team and an alert where appropriate.

## Failure Modes And Rollback

### Common Failure Modes

1. CRD ordering errors
- Symptom: ServiceMonitor or PrometheusRule rejected before CRDs exist.
- Mitigation: ArgoCD sync-wave ordering and app dependency sequencing.

2. Missing alert secrets
- Symptom: Alertmanager config invalid or receivers disabled.
- Mitigation: ESO resources in earlier wave, release gate validates Secret readiness.

3. TSDB disk pressure
- Symptom: Prometheus crashloop or ingestion throttling.
- Mitigation: retention reduction override, PVC expansion path, high-cardinality triage.

4. Label explosion
- Symptom: cardinality growth and memory pressure.
- Mitigation: metric relabel drops for unbounded labels; onboarding review checklist.

5. Bad rule deployment
- Symptom: alert storm or false positives.
- Mitigation: lint+unit test rules pre-merge; staged dev soak before prod promotion.

### Rollback Design

- Rollback trigger: failed Temporal release gate or post-release smoke failure.
- Mechanism: revert monitoring commit in infra repo and rerun Temporal ReleaseWorkflow.
- Scope control: environment-specific rollback (dev first, prod independent).
- Last resort: disable specific rule groups through values toggle while retaining scrape path.

## Testing And Validation Plan

### CI Validation

1. Helm rendering checks
- helm template for dev and prod overlays.

2. Policy/lint checks
- kubeconform/kubeval against rendered manifests.
- Prometheus rule lint and unit tests.

3. Static quality checks
- Verify required labels, namespace targeting, and ingress TLS annotations.

### Runtime Validation

1. Dev rollout
- Temporal-triggered ArgoCD sync succeeds.
- Prometheus targets up ratio >= 98% for baseline set.
- Alertmanager test alert reaches configured receiver.
- Grafana dashboard queries return data for golden signals.

2. Prod rollout
- Repeat validation with stricter success threshold and change window controls.

### Acceptance Criteria

1. Monitoring stack deploys in dev and prod through Temporal-gated ArgoCD reconciliation.
2. At least 3 core platform domains are onboarded via ServiceMonitor/PodMonitor.
3. Alertmanager receivers are backed by Vault/ESO-managed secrets only.
4. SLO dashboards for cluster, ArgoCD, and Temporal are provisioned from git and populated.
5. Rollback procedure is executed successfully at least once in dev.

## ADR-Style Decision Table

| ADR | Decision | Status | Rationale | Consequence |
| --- | --- | --- | --- | --- |
| ADR-001 | Use kube-prometheus-stack as baseline distribution | Accepted | Mature operator stack and CRD ecosystem | Locks us to chart upgrade cadence and CRD compatibility checks |
| ADR-002 | Release path is Temporal -> ArgoCD sync gate | Accepted | Aligns with platform release governance and deterministic sequencing | Requires workflow hooks for infra monitoring apps |
| ADR-003 | ServiceMonitor/PodMonitor required for in-cluster onboarding | Accepted | Consistent discovery and reduced bespoke scrape drift | Teams must follow label contract and metrics port standards |
| ADR-004 | Keep Prometheus/Alertmanager internal; expose Grafana only | Accepted | Reduces attack surface while preserving operator usability | Requires bastion/VPN/internal DNS access patterns |
| ADR-005 | Local-path PVC retention (15d dev, 30d prod) for initial phase | Accepted | Works with current k3s storage baseline and delivery speed | Long-term retention requires later remote_write/durable storage initiative |
| ADR-006 | Alert receiver secrets via Vault + ESO only | Accepted | Meets security posture and avoids git secret leakage | ESO health becomes a hard dependency for alerting readiness |

## Phased Implementation Plan

### Phase 0: Foundations

- Add ArgoCD app manifests and helm values skeleton for dev/prod.
- Install/verify CRDs and operator dependencies.
- Wire Temporal release activities to trigger/wait monitoring app sync.

Exit criteria:
- Clean render in CI, successful dev deployment, basic target discovery active.

### Phase 1: Core Wiring

- Add default ServiceMonitor/PodMonitor selectors and label contract docs.
- Add baseline PrometheusRule packs (cluster, ArgoCD, Temporal).
- Stand up Grafana ingress and cert-manager TLS.

Exit criteria:
- Core dashboards live, rule evaluation healthy, alert path proven with test notifications.

### Phase 2: SLO And Service Onboarding

- Onboard tier-1 services to golden signal dashboards.
- Add fast/slow burn SLO alerts and runbook links.
- Validate cardinality and resource headroom.

Exit criteria:
- Acceptance criteria met for onboarding and dashboard coverage.

### Phase 3: Hardening

- Execute rollback game day in dev.
- Tune retention/resources from observed load.
- Produce TechPlan-ready implementation backlog (epics/stories) from this architecture.

Exit criteria:
- Documented operational runbook and validated rollback confidence.

## TechPlan Hand-off Notes

Implementation stories should be grouped by:
1. ArgoCD and Helm artifact creation.
2. Temporal release integration.
3. Prometheus discovery/rules/dashboards.
4. Vault/ESO secret plumbing.
5. Validation automation and rollback drills.

This document is implementation-ready for TechPlan decomposition into epics and story files.
