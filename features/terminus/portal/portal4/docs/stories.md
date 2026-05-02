---
feature: portal4
doc_type: stories
status: draft
goal: "Complete story list for portal4 — 18 stories across 4 sprints"
key_decisions: []
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Stories: portal4

18 stories across 4 sprints. Story files in `stories/`.

---

## Sprint 1 — Visual Fixes and Service Config

| Story ID | File | Title | Estimate | Status |
|----------|------|-------|---------|--------|
| portal4-1-01 | stories/portal4-1-01-grafana-css-overflow.md | Fix Grafana card CSS overflow in ServiceGrid | S | backlog |
| portal4-1-02 | stories/portal4-1-02-argocd-semaphore-icon-slugs.md | Fix ArgoCD and Semaphore icon slugs | S | backlog |
| portal4-1-03 | stories/portal4-1-03-temporal-service-card.md | Add Temporal service card to services.js | S | backlog |
| portal4-1-04 | stories/portal4-1-04-fourdogs-dev-url-split.md | Add Fourdogs dev URL and split Fourdogs card metadata | M | backlog |

## Sprint 2 — Go Sidecar Foundation and Helm Wiring

| Story ID | File | Title | Estimate | Status |
|----------|------|-------|---------|--------|
| portal4-2-01 | stories/portal4-2-01-go-sidecar-scaffold.md | Scaffold Go sidecar module with health endpoint | M | backlog |
| portal4-2-02 | stories/portal4-2-02-sidecar-dockerfile-ci.md | Add sidecar Dockerfile and Semaphore CI build job | M | backlog |
| portal4-2-03 | stories/portal4-2-03-helm-chart-sidecar-nginx.md | Update Helm chart: sidecar container, nginx proxy_pass for /api/* | M | backlog |
| portal4-2-04 | stories/portal4-2-04-eso-externalsecret-pat.md | Wire PAT via Vault ESO ExternalSecret in Helm chart | L | backlog |
| portal4-2-05 | stories/portal4-2-05-serviceaccount-rbac.md | Create ServiceAccount and RBAC for k8s pod listing | M | backlog |

## Sprint 3 — Tab Navigation and Release Pipeline Tab

| Story ID | File | Title | Estimate | Status |
|----------|------|-------|---------|--------|
| portal4-3-01 | stories/portal4-3-01-tab-navigation.md | Implement tab navigation bar in App.jsx (useState) | M | backlog |
| portal4-3-02 | stories/portal4-3-02-github-proxy-route.md | Implement /api/github/repos/:owner/:repo/branches/:branch sidecar route | L | backlog |
| portal4-3-03 | stories/portal4-3-03-release-pipeline-tab.md | Build Release Pipeline tab component (ReleasePipelineTab.jsx) | M | backlog |
| portal4-3-04 | stories/portal4-3-04-release-pipeline-config.md | Replace githubActions.js with releasePipeline.js config | S | backlog |

## Sprint 4 — Deep Health Signals and Prometheus Metrics

| Story ID | File | Title | Estimate | Status |
|----------|------|-------|---------|--------|
| portal4-4-01 | stories/portal4-4-01-k8s-pods-routes.md | Implement /api/k8s/pods and /api/k8s/pods/all sidecar routes | M | backlog |
| portal4-4-02 | stories/portal4-4-02-pods-tab.md | Build Pods tab component (PodsTab.jsx) | M | backlog |
| portal4-4-03 | stories/portal4-4-03-prometheus-proxy.md | Implement /api/metrics/query sidecar route (Prometheus proxy + CA trust) | M | backlog |
| portal4-4-04 | stories/portal4-4-04-fourdogs-health-panel.md | Build FourDogsHealthPanel.jsx and wire per-microservice health signals | L | backlog |
| portal4-4-05 | stories/portal4-4-05-metrics-wiring.md | Wire metrics.js to live Prometheus endpoints in MetricsPanel | S | backlog |
