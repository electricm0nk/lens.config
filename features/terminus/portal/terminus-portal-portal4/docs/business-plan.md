---
feature: terminus-portal-portal4
doc_type: business-plan
status: draft
goal: "Transform the Terminus Portal from a basic service status board into an operational command centre with tab navigation, a Go backend sidecar, deep per-microservice health signals, live k8s pod health view, and a Release Pipeline tab giving full develop-to-production deployment visibility."
key_decisions:
  - Small Go sidecar in the portal pod to proxy GitHub API, k8s API, and Prometheus — PAT and service account never exposed to browser
  - Full tab navigation bar replacing the single-page layout
  - Release Pipeline tab as a first-class feature, not a panel
  - PAT sourced from Vault at terminus/github key Portal_PAT via ESO
  - Overview tab becomes a pure quick-HUD: pod health + service links only
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Business Plan: terminus-portal-portal4

## Executive Summary

The Terminus Portal today shows service cards with a single ONLINE/UNREACHABLE signal per service and an unwired metrics panel. This works as a basic "is the cluster up" check but fails as an operational tool: Fourdogs consists of five deployed microservices yet the portal shows one card; emailfetcher and the etailpet trigger workers have no visibility; the GitHub Actions panel cannot show private repos and shows only aggregate counts rather than release pipeline state. When something goes wrong — old code silently deployed to dev, a trigger worker failing quietly, an emailfetcher import stalling — there is no fast-path to that signal. Portal4 closes this gap by introducing a backend sidecar that unlocks private-data access, restructuring the UI into tabs optimised for two use cases (quick health HUD vs. deep release pipeline visibility), and wiring every Fourdogs microservice to a health signal appropriate to its type.

## Business Context

The operator (sole platform owner) manages a homelab platform serving a live production POS integration (Four Dogs Pet Supplies). Debugging incidents currently requires opening multiple browser tabs — ArgoCD, GitHub, kubectl — to triangulate what is deployed where and whether it is working. The root problem is not skill or tooling gap; it is that all the signal exists but it is not aggregated anywhere. Portal3 established the domain/service layout and GitHub Actions visibility foundations. Portal4 builds the operational layer on top of that foundation.

The cost of not building this: continued manual debugging loops when releases get stuck or workers fail silently. The etailpet-sales-trigger incident and the emailfetcher stall scenarios from recent weeks are concrete examples of signal that exists in logs and probes but has no visible surface in the portal.

## Stakeholders

| Stakeholder | Role | Interest |
|-------------|------|----------|
| Todd Hintzmann | Sole operator and developer | Reduce incident diagnosis time; see release pipeline state at a glance |
| Four Dogs Pet Supplies | Production consumer | Stable, observable platform reduces outage risk |

## Success Criteria

| Criterion | Measure |
|-----------|---------|
| Pod health visible on Overview tab | All k8s namespaces shown with pod ready/restart counts; no kubectl required for first-pass triage |
| Fourdogs microservice health depth | Each of the 5 Fourdogs workloads (central-ui, central API, emailfetcher, etailpet-trigger, etailpet-sales-trigger) shows a signal appropriate to its type — not just a TCP ping |
| Release Pipeline tab functional | Every tracked repo shows current tag/SHA on develop and main branches; drift is visible without opening GitHub |
| Private repo pipeline data visible | GitHub Actions workflow runs visible for all private repos via PAT proxy (no 403/404 errors) |
| Prometheus metrics wired | At least CPU load and pod count populated from Prometheus on the Overview HUD |
| ArgoCD and Semaphore icons fixed | Both icons render correctly |
| Temporal agent card added | Temporal appears as a service card |
| Fourdogs dev-site link present | Card links to `https://central-dev.fourdogspetsupplies.com/login` |
| Version watermark on portal pages | Portal's own deployed version (git tag) is visible in the UI |
| No secrets in source code or browser | PAT and k8s service account credentials remain in-cluster at all times |

## Scope

### In Scope

- Go backend sidecar: GitHub proxy (`/api/github/*`), k8s proxy (`/api/k8s/*`), Prometheus proxy (`/api/metrics/*`)
- PAT injection via Vault ESO (Vault path: `terminus/github`, key: `Portal_PAT`)
- Tab navigation bar: **Overview**, **Release Pipeline**, **Pods** (or combined with Overview)
- Overview tab: quick HUD — service cards (fixed), pod health summary, Prometheus metrics (CPU, memory, pods running), links
- Release Pipeline tab: per-repo develop/main branch latest tag+SHA+commit message+age, workflow run status per branch, runner status
- Pods tab (or Overview section): all namespaces, pod name, ready state, restart count, age — grouped by namespace
- Fourdogs service cards expanded: central-ui (up/down), central API (up/down via `/v1/health`), emailfetcher (last import timestamp + error count via fourdogs-central health stub), etailpet-trigger (k8s pod phase + restart count — no HTTP probe exists), etailpet-sales-trigger (k8s pod phase + restart count — no HTTP probe exists)
- Temporal agent service card added
- ArgoCD and Semaphore icon fixes
- Fourdogs card dev-site link (`https://central-dev.fourdogspetsupplies.com/login`)
- Version watermark: portal's own git tag embedded at build time, displayed in header or footer
- Helm chart update: sidecar container, ServiceAccount with k8s read RBAC, ESO ExternalSecret for PAT
- Repo list for Release Pipeline tab (10 repos): `terminus-portal`, `terminus.infra`, `terminus.platform`, `lens.config`, `fourdogs-central`, `fourdogs-central-ui`, `fourdogs-kaylee-agent`, `terminus-inference-gateway`, `terminus-inference-qwen-warmup`, `fourdogs` (prototype — optional/deprioritised)

### Out of Scope

- Prometheus alerting rules or dashboards (Grafana owns that)
- Log streaming or log search in the portal
- Write operations (no kubectl apply, no ArgoCD sync triggers from portal)
- Authentication / access control on the portal itself
- Mobile/responsive layout changes
- etailpet trigger health beyond pod status (no HTTP probe endpoints exist in current deployments)

## Risks and Mitigations

| Risk | Probability | Mitigation |
|------|-------------|------------|
| k8s API RBAC misconfiguration leaks cross-namespace data or allows write access | Low | ServiceAccount is read-only, explicitly scoped to `get`/`list`/`watch` on pods only; reviewed in tech plan |
| GitHub API rate limits hit by Release Pipeline tab polling | Medium | Cache responses in sidecar with 5-min TTL; tab is not auto-refreshed at high frequency |
| PAT rotation breaks Release Pipeline tab | Low | ESO syncs from Vault; pod restart picks up new secret; operator runbook documented |
| Fourdogs health checks use external URLs that require Google auth | Confirmed | API health endpoints (`/v1/health`) are separate from the login page; internal cluster addresses used for health probes where available |
| Sidecar adds pod startup complexity and a new build artifact | Medium | Sidecar is a single static Go binary in a scratch/distroless image; minimal operational overhead |
| import_audit table query requires DB read access from sidecar | Low — avoided | emailfetcher health signal comes from the central API's health endpoint or a dedicated `/health/emailfetcher` stub, not direct DB access from the portal sidecar |

## Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| Vault `terminus/github` → `Portal_PAT` | Platform | Must be provisioned before deploy; read-only PAT with `repo` scope |
| ESO ClusterSecretStore | Platform | Existing; same pattern as fourdogs services |
| Prometheus at `https://prometheus.trantor.internal` | Platform | Must be reachable from within the portal pod |
| k8s API (in-cluster) | Platform | ServiceAccount projected token; standard in-cluster client config |
| fourdogs-central `/v1/health` endpoint | Application | Existing; used for API health check |
| fourdogs-central-ui deployment health | Application | HTTP check against deployed pod or service |
| fourdogs-central `/v1/health/emailfetcher` stub | Application | New endpoint needed in fourdogs-central; reads `import_audit` last success + error count |
| etailpet-trigger k8s pod status | Platform | No HTTP probe or Service exists; pod phase + restart count via `/api/k8s/pods` sidecar route |
| etailpet-sales-trigger k8s pod status | Platform | No HTTP probe or Service exists; pod phase + restart count via `/api/k8s/pods` sidecar route |

## Timeline Expectations

No external deadline. This is an operator-tooling initiative. Prioritised ahead of new feature work because operational visibility reduces debugging overhead across all other initiatives.
